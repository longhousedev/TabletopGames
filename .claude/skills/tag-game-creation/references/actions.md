# Actions

## AbstractAction contract

Every action extends `core.actions.AbstractAction`:

```java
public class PlayCard extends AbstractAction {
    public final int cardId;      // component ID, never a Card reference
    public final int playerId;

    public PlayCard(int playerId, int cardId) { this.playerId = playerId; this.cardId = cardId; }

    @Override
    public boolean execute(AbstractGameState gs) {
        MyGameState state = (MyGameState) gs;
        MyCard card = (MyCard) state.getComponentById(cardId);   // re-fetch fresh reference
        state.playerHands.get(playerId).remove(card);
        state.discardPile.add(card);
        // ...apply the card's effect...
        return true;   // false = could not execute (should be rare; only offer legal actions)
    }

    @Override public PlayCard copy() { return this; }  // all fields final → immutable → 'this' is correct
    @Override public boolean equals(Object o) { return o instanceof PlayCard p && p.cardId == cardId && p.playerId == playerId; }
    @Override public int hashCode() { return Objects.hash(cardId, playerId); }
    @Override public String toString() { return "Player " + playerId + " plays card " + cardId; }
}
```

Rules of thumb:

- **Parameterise, don't proliferate.** One `PlayCard(cardId)` class, not a class per card. Card-specific effects can dispatch on the card's type, or complex cards get their own action subclass.
- **Immutability is the easy path.** Make all fields final; `copy()` returns `this`. If a field is mutable you must deep-copy it, and the concrete return type is required either way.
- **equals/hashCode over all fields.** MCTS keys tree nodes by action equality; two logically identical actions must be equal, two different ones must not. Do not include the game state.
- **`getString(AbstractGameState)`** (optional) gives a human-readable description using live component names; `getString(gs, playerId)` exists for hiding info from a viewing player in the GUI.
- **Reuse core actions** in `core.actions` where they fit: `DoNothing`, `SetGridValueAction`, `DrawCard`, etc.

## Multi-step decisions: IExtendedSequence

Two situations call for `core.interfaces.IExtendedSequence`:

1. **Move groups** — one conceptual move with several parameters (choose a card, then a target). Offering every combination as flat actions explodes the action space (6 cards × 10 targets = 60 actions; as a sequence it's 6 then 10). Smaller action spaces help tree-search agents enormously.
2. **Triggered sub-decisions** — an action forces other players to respond (everyone discards down to 3; the defender chooses whether to block).

The sequence object acts as a temporary sub-forward-model: while one is active, the framework routes `computeAvailableActions` and current-player queries to it instead of your ForwardModel.

```java
public class DiscardDownTo implements IExtendedSequence {
    final int playerID;          // who must decide (sequences track their own actors)
    int discardsRemaining;       // mutable local state is fine — but then copy() must deep-copy

    @Override public List<AbstractAction> _computeAvailableActions(AbstractGameState state) {
        // one DiscardCard(cardId) per card in the deciding player's hand
    }
    @Override public int getCurrentPlayer(AbstractGameState state) { return playerID; }
    @Override public void _afterAction(AbstractGameState state, AbstractAction action) {
        if (action instanceof DiscardCard) discardsRemaining--;
    }
    @Override public boolean executionComplete(AbstractGameState state) { return discardsRemaining == 0; }
    @Override public DiscardDownTo copy() { /* new instance copying fields */ }
    // plus equals/hashCode
}
```

To start a sequence, an ordinary action's `execute()` calls:

```java
gs.setActionInProgress(new DiscardDownTo(targetPlayer, n));
```

The framework keeps a stack of in-progress sequences on the state (sequences can nest) and deep-copies it with the state. When `executionComplete` returns true the sequence pops and control returns to the ForwardModel.

Preferred modern style: keep the `IExtendedSequence` as its own small class triggered by a plain action, rather than one class that is both the action and the sequence (the template's `GTExtendedSequenceAction` shows the combined style, which also works — it then calls `gs.setActionInProgress(this)` in its own `execute`).

**Interaction with turn flow:** in `ForwardModel._afterAction`, check `state.isActionInProgress()` and skip end-of-turn processing while a sequence is running — the actions inside the sequence also arrive at `_afterAction`, and ending the turn mid-sequence breaks the game. Example:

```java
@Override
protected void _afterAction(AbstractGameState gs, AbstractAction action) {
    if (gs.isActionInProgress()) return;   // sequence still making decisions
    // normal consequences + endPlayerTurn(...)
}
```

Study `games.dominion` (Militia et al.) or `games.monopolydeal` for real examples.
