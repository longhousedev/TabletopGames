# Core class contracts

Full method-level contracts for the three mandatory classes. The `gametemplate` package (`GT*` classes) contains skeletons of all of these with javadoc; this file adds the details and the reasons.

## GameState — `extends AbstractGameState`

Data only. Prefer `core.components` types over raw collections where possible — components have IDs, ownership, and visibility built in. No initialisation in the constructor; all setup happens in `ForwardModel._setup()`.

```java
public class MyGameState extends AbstractGameState {
    // Example fields
    List<PartialObservableDeck<MyCard>> playerHands;
    Deck<MyCard> drawDeck;
    int[] victoryPoints;   // primitives/arrays are fine too

    public MyGameState(AbstractParameters gameParameters, int nPlayers) {
        super(gameParameters, nPlayers);
    }
    @Override protected GameType _getGameType() { return GameType.MyGame; }
}
```

### `List<Component> _getAllComponents()`
Return every component in the game, **including nested ones** (a `Deck` counts as one component, but if actions reference the cards inside it by ID, the cards must be reachable too — adding a Deck adds its contents automatically since `Deck` is a component containing components; but components stashed in plain maps/lists of your own must be added explicitly). Called once after setup to build the ID→component lookup used by `getComponentById`. A component missing here means `getComponentById` returns null mid-game.

### `MyGameState _copy(int playerId)`
The single most important method in the game.

- Return type must be the concrete class, not `AbstractGameState`.
- Construct a fresh state and deep-copy **every** field. Components have `.copy()` methods.
- `playerId == -1`: full-information faithful copy. Copy everything exactly.
- `playerId != -1`: this is agent `playerId` imagining the state — anything they cannot see must be *redeterminised*: collect unseen cards (opponents' hidden hands + face-down decks), shuffle them together, redeal into the same structure. Use the provided `redeterminisationRnd`, **not** `getRnd()` — using the main RNG here desynchronises the game's random stream across simulations. `utilities.DeterminisationUtilities` has helpers (e.g. `reshuffle(...)` across several decks keeping visible cards fixed).
- Framework-managed fields (turn/round counters, phase, player results) are copied by the framework; you copy only your own fields.

### `double getGameScore(int playerId)`
The true score per the printed rules (victory points, money, …). Return 0 if the game has no running score (e.g. pure knockout games). Used for final rankings and as the default tiebreak.

### `double _getHeuristicScore(int playerId)`
An estimate in [0, 1] of how well `playerId` is doing (0 ≈ losing, 1 ≈ winning), used by MCTS/OSLA when a rollout hits a non-terminal state. The template pattern:

```java
if (isNotTerminal()) {
    return /* e.g. myScore / plausibleMaxScore */;
}
return getPlayerResults()[playerId].value;  // WIN=1, DRAW=0, LOSE=-1 at terminal
```

A crude scaled score is fine to start; a better heuristic just makes agents stronger.

### `boolean _equals(Object o)` and `int hashCode()`
Compare/hash **all** of your fields (the framework combines this with its own fields via the public `equals`). `ForwardModelTester` exercises exactly this contract: copy must be equal and hash-equal to the original.

### Useful inherited API (do not reimplement)
- `getCurrentPlayer()`, `getNPlayers()`, `getRoundCounter()`, `getTurnCounter()`, `getFirstPlayer()`
- `getRnd()` — the seeded game RNG; use for all in-game randomness
- `getGamePhase()` / `setGamePhase(IGamePhase)` — see components-and-data.md
- `isNotTerminal()`, `getPlayerResults()`, `logEvent(...)`
- Optional overrides for special games: `getNTeams()`/`getTeam(player)` for team games; `getTiebreak(playerId, tier)`/`getTiebreakLevels()` for multi-level tiebreaks; `getOrdinalPosition(playerId)` for insta-win games with bespoke ranking.

## ForwardModel — `extends StandardForwardModel`

Stateless: no instance fields. All state lives in the GameState; the same ForwardModel instance serves every copy of the state simultaneously during MCTS search.

### `void _setup(AbstractGameState firstState)`
Cast to your state class and initialise **every** field: create components, shuffle (`deck.shuffle(state.getRnd())`), deal, set the starting phase. Runs once per game and again on reset — so everything must be created here, not in the state constructor.

### `List<AbstractAction> _computeAvailableActions(AbstractGameState gameState)`
All legal actions for `gameState.getCurrentPlayer()` right now (switch on `getGamePhase()` if the game has phases). Must never be empty while the game is ongoing — if the rules can leave a player with no move, return a `DoNothing` (in `core.actions`) or a game-specific Pass action. Generating an action does not execute it; construct actions with the parameters/IDs they need.

### `void _afterAction(AbstractGameState currentState, AbstractAction actionTaken)`
Called after the framework has executed each action. This is where the game *flows*:

1. Apply rule consequences (draw back up to hand size, resolve triggered effects, change phase, …).
2. Check for game end. If ended: `gs.setGameStatus(CoreConstants.GameResult.GAME_END)` and `gs.setPlayerResult(WIN_GAME/LOSE_GAME/DRAW_GAME, p)` for every player, then return.
3. Otherwise advance play: `endPlayerTurn(gs)` (next player in order, skipping eliminated ones), `endPlayerTurn(gs, nextPlayer)` (explicit next player), `endRound(gs)` / `endRound(gs, firstPlayerOfNextRound)` (increments round counter; auto-ends the game if `maxRounds` in parameters is reached).

The framework never advances the turn itself. If `_afterAction` doesn't call one of these (and the action isn't part of an in-progress extended sequence), the same player acts again — which is sometimes exactly what you want (e.g. multi-action turns), but must be deliberate. Note: while an `IExtendedSequence` is in progress, do **not** end turns here; let the sequence complete first (check `currentState.isActionInProgress()`).

### `void _beforeAction(...)` (optional)
Hook before an action executes. Rarely needed; delete it if unused.

## Parameters — `extends AbstractParameters` (or `TunableParameters`)

One field per rule constant, with sensible defaults:

```java
public class MyParameters extends AbstractParameters {
    public int handSize = 5;
    public int pointsToWin = 15;

    @Override protected MyParameters _copy() { MyParameters p = new MyParameters(); p.handSize = handSize; ... return p; }
    @Override protected boolean _equals(Object o) { return o instanceof MyParameters p && p.handSize == handSize && ...; }
    @Override public int hashCode() { return Objects.hash(super.hashCode(), handSize, pointsToWin); }
}
```

Access anywhere via `(MyParameters) gameState.getGameParameters()`. Inherited useful fields: the random seed, `maxRounds`, `timeoutRounds`.

Extending `evaluation.optimisation.TunableParameters` instead lets the framework's NTBEA-based tuner search parameter space automatically — implement `addTunableParameter(...)` registrations in the constructor and `_reset()` to pull current values (see `games.tictactoe.TicTacToeGameParameters` for a minimal example).
