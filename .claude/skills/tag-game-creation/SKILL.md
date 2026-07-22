---
name: tag-game-creation
description: Implement a new tabletop game in the TAG (Tabletop Games) framework in this Java repo, so it can be played by AI agents (MCTS, RL, rule-based) and humans. Use this skill whenever the user wants to add, create, implement, or port a board game, card game, or dice game to the framework — even if they don't say "TAG" — and whenever they mention GameState, ForwardModel, GameType, the gametemplate package, making a game "playable by agents", or ask how games are structured in this codebase. Also use it when fixing or extending an existing game's rules, actions, or state copying.
---

# Creating a Game in the TAG Framework

TAG is a Java 21 / Maven framework where every game is a state machine driven by AI or human players. The engine (in `core`) runs the game loop; a game supplies the pieces below. Get the architecture right and any agent in the framework (Random, OSLA, MCTS, RL) can immediately play your game.

## Architecture: the five pieces

| Piece | Extends | Role |
|---|---|---|
| GameState | `core.AbstractGameState` | **Data only.** All information describing the current state. No game logic, not even in the constructor. |
| ForwardModel | `core.StandardForwardModel` | **Logic only, stateless.** Setup, available actions, rule consequences, game end. Never store fields on it. |
| Actions | `core.actions.AbstractAction` | The moves players make. Each applies its own effect to a GameState. |
| Parameters | `core.AbstractParameters` | Static settings (hand size, board size, rule variants). Never hard-code these numbers elsewhere. |
| GameType entry | enum in `games.GameType` | Registers the game with the framework so it can be instantiated by name. |

Optional: a Swing GUI (`gui.AbstractGUIManager`), a heuristic for agents, JSON data files, tunable parameters.

**Why the strict data/logic split matters:** agents like MCTS work by repeatedly *copying* the state and *simulating* forward with the ForwardModel. If logic hides in the state, or state hides in the forward model, or a copy is shallow, simulations corrupt each other and agents behave nonsensically — usually without crashing. Most bugs in new TAG games are copy/equals bugs, not rule bugs.

## The four contracts (violating these breaks agents silently)

1. **Deep copy.** `GameState._copy(playerId)` must deep-copy every mutable field. With `playerId != -1` it must also hide/re-randomise information that player cannot see (shuffle unseen cards back, redeal) — use `redeterminisationRnd`, never `getRnd()`, for that reshuffling. `Action.copy()` must do the same for its fields (return `this` only if all fields are final/immutable).
2. **equals + hashCode everywhere.** GameState `_equals`/`hashCode` must include every state field; Actions likewise. MCTS uses action hashCodes as tree keys and the framework uses state hashes to detect copy bugs.
3. **No component references in Actions.** Store `getComponentID()` ints; re-fetch with `gameState.getComponentById(id)` inside `execute()`. A stored reference points into *one particular copy* of the state and is stale in every other copy.
4. **All randomness from the state.** Use `gameState.getRnd()` for in-game randomness (shuffles, dice) so games are reproducible from a seed.

## Workflow

### 1. Study a comparable existing game

Pick the closest match from `src/main/java/games/` and skim its GameState + ForwardModel first — it answers most "how do I…" questions concretely:

- Simple grid/abstract: `tictactoe` (minimal), `dotsboxes`, `connect4`
- Cards with hidden hands: `gofish`, `loveletter`, `sushigo`, `hearts`
- Dice / push-your-luck: `cantstop`, `pickomino`, `diamant`
- Deck-building / multi-step actions: `dominion`, `monopolydeal`
- Boards from JSON data: `pandemic`, `battlelore`

### 2. Copy the template

Copy `src/main/java/gametemplate/` to `src/main/java/games/<gamename>/` and rename the `GT*` classes to `<Name>*` (e.g. `GTGameState` → `SplendorGameState`), fixing the `package` declarations. The template files contain detailed javadoc on every mandatory method — keep them open while implementing. Delete `GTExtendedSequenceAction` if the game has no multi-step actions, and `_beforeAction` if unused (it rarely is used).

### 3. Implement, in this order

1. **Parameters** — every rule number as a field with a default. Implement `_copy()`, `_equals()`, `hashCode()`.
2. **GameState** — fields (prefer `core.components` types: `Deck`, `PartialObservableDeck`, `Counter`, `GridBoard`, …), then `_getGameType()`, `_getAllComponents()`, `_copy(playerId)`, `getGameScore()`, `_getHeuristicScore()` (return in [0,1]; at terminal return `getPlayerResults()[playerId].value`), `_equals()`, `hashCode()`.
3. **Actions** — one class per distinct move type, parameterised (e.g. `PlayCard(cardId)`), not one class per concrete move.
4. **ForwardModel** — `_setup()` (create every component, deal, set starting phase), `_computeAvailableActions()` (never return an empty list for a live game — that crashes the run), `_afterAction()` (apply rule consequences, check game end, then `endPlayerTurn(state)` / `endRound(state)`; the framework never advances turns for you).

Read [references/core-classes.md](references/core-classes.md) before this step for the full method contracts, and [references/actions.md](references/actions.md) when writing actions — especially for multi-step decisions (`IExtendedSequence`). For the component catalog, visibility modes, JSON data files and game phases, see [references/components-and-data.md](references/components-and-data.md).

### 4. Register in `games.GameType`

Add imports and an enum entry:

```java
MyGame(2, 4,
        Arrays.asList(Cards, Strategy),                 // GameType.Category values
        Arrays.asList(HandManagement, SetCollection),   // GameType.Mechanic values
        MyGameState.class, MyForwardModel.class, MyParameters.class,
        MyGUIManager.class),   // or null if no GUI yet; add a trailing "data/mygame/" arg if the game loads JSON data
```

Categories/Mechanics are inner enums of `GameType` — pick the closest existing values (empty lists are acceptable). Game end for wins is signalled from the ForwardModel with `gs.setGameStatus(CoreConstants.GameResult.GAME_END)` plus `gs.setPlayerResult(WIN_GAME/LOSE_GAME/DRAW_GAME, player)` for each player.

### 5. Compile and verify

```bash
mvn -q compile                        # needs Java 21
```

Then verify behaviour, in increasing order of strength (details and commands in [references/testing-and-running.md](references/testing-and-running.md)):

1. **Unit tests** in `src/test/java/games/<gamename>/` (JUnit 4): construct `Parameters` + `GameState` + `ForwardModel` directly, call `fm.setup(state)`, drive it with `fm.next(state, action)`, and assert on state. Test setup counts, action lists in known positions, end-of-game detection, and copy/hiding behaviour.
2. **Full random playthroughs with copy checking** — `evaluation.ForwardModelTester` plays complete games while verifying every state copy stays `equals` to its original and past states are never mutated. This catches shallow copies automatically; run it before declaring the game done:
   ```bash
   mvn -q compile exec:java -Dexec.mainClass=evaluation.ForwardModelTester "-Dexec.args=game=MyGame nPlayers=3 nGames=10"
   ```
3. **Play against MCTS** (`agent=mcts` on ForwardModelTester, or run `core.Game`'s main with `game=MyGame`) — a sane game should show MCTS beating random players.

### 6. Optional extras

- **GUI** so humans can play: see [references/gui.md](references/gui.md). Without one, pass `null` in GameType and run headless.
- **Better heuristic** for stronger agent play: refine `_getHeuristicScore` or implement `IStateHeuristic` separately.
- **Parameter tuning**: extend `evaluation.optimisation.TunableParameters` instead of `AbstractParameters` to enable the framework's automatic optimisation tools.

## Pitfalls checklist (review before finishing)

- [ ] `_copy(playerId)` copies **every** mutable field; hidden info re-randomised with `redeterminisationRnd`
- [ ] `_getAllComponents()` returns every component (including nested ones) — missing ones break ID lookup
- [ ] `equals`/`hashCode` on state, parameters, and every action cover all fields
- [ ] Actions hold component IDs, never references; `copy()` returns the concrete type
- [ ] `_afterAction` always ends in `endPlayerTurn`/`endRound` (or game end) — otherwise the same player moves forever
- [ ] `_computeAvailableActions` never returns an empty list while the game is ongoing (add a Pass/DoNothing action if the rules allow no move)
- [ ] All rule numbers live in Parameters, read via `gs.getGameParameters()`
- [ ] `ForwardModelTester` passes over multiple full games
