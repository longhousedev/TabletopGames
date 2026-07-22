# Testing, running, and verifying a game

## Build

Java 21, Maven. From the repo root:

```bash
mvn -q compile          # compile main sources
mvn -q test-compile     # also compile tests
mvn -q test -Dtest="games.mygame.*"   # run one game's tests (JUnit 4)
```

## Unit tests

Tests live in `src/test/java/games/<gamename>/` and drive the ForwardModel directly — no Game object or players needed:

```java
public class BasicRules {
    MyForwardModel fm = new MyForwardModel();
    MyParameters params = new MyParameters();
    MyGameState state;

    @Before
    public void setup() {
        state = new MyGameState(params, 3);   // 3 players
        fm.setup(state);                      // public wrapper around _setup
    }

    @Test
    public void dealSizesCorrect() {
        assertEquals(params.handSize, state.playerHands.get(0).getSize());
    }

    @Test
    public void playingCardMovesItToDiscard() {
        List<AbstractAction> actions = fm.computeAvailableActions(state);
        fm.next(state, actions.get(0));       // executes action + _afterAction + turn advance
        assertEquals(1, state.getCurrentPlayer());
    }
}
```

Key entry points: `fm.setup(state)`, `fm.computeAvailableActions(state)`, `fm.next(state, action)`. To force specific situations, set state fields directly before acting (tests are in the same package tree, and most game state fields are package-visible or public).

What to cover, roughly in order of value:

1. Setup invariants (deck sizes, hand sizes, starting player/phase).
2. Available actions in hand-crafted situations — both that legal moves appear and that illegal ones don't.
3. Action effects and turn/round flow (`getCurrentPlayer`, counters advance correctly).
4. Game-end detection and correct win/lose/draw results for all players.
5. Copy behaviour: `state.copy()` equals the original; `state.copy(playerId)` hides what it should (e.g. opponent hand contents differ across repeated copies, own hand identical); mutating a copy leaves the original untouched.

See `src/test/java/games/gofish/BasicTurns.java` or `games/cantstop` tests for real examples.

## ForwardModelTester — automated copy-bug detection

`evaluation.ForwardModelTester` plays full games with random (or other) agents and, at every decision, copies the state, checks `equals`/`hashCode` consistency, applies actions to copies, and verifies historic states are never mutated. It is the single best detector of shallow-copy bugs — run it before considering the game done:

```bash
mvn -q compile exec:java -Dexec.mainClass=evaluation.ForwardModelTester \
    "-Dexec.args=game=MyGame nPlayers=3 nGames=10"
```

Useful args: `agent=random|osla|mcts` (mcts explores deeper lines), `seed=...` to reproduce a failure, `verbose`. A failure names the decision number and the hash component that diverged — almost always a field missing from `_copy` or from `hashCode`.

## Playing the game

- **Headless / quick sanity run:** `core.Game.main` — `mvn -q compile exec:java -Dexec.mainClass=core.Game "-Dexec.args=game=MyGame gui=false"`. Edit the player list inside `main` if you want specific agents.
- **GUI (human play):** run `gui.Frontend` and pick the game, or `core.Game.main` with `gui=true` (needs a GUIManager registered in GameType).
- **Tournaments / experiments:** `evaluation.RunGames` runs batches of games between configured agents and reports win rates — useful for checking that MCTS beats random (a sanity check that the game rewards skill; if random ties MCTS, suspect the heuristic, the action space, or the rules).

## Heuristics for stronger play

The default `_getHeuristicScore` is what MCTS maximises. If agents play badly, improve it: reward proxies for winning (points, material, tempo) scaled to [0,1]. Alternatively implement `core.interfaces.IStateHeuristic` as a separate class (see `games.tictactoe.TicTacToeHeuristic`) and pass it to agents at construction; making it extend `TunableParameters` lets the framework tune its weights.

## Parameter tuning (optional)

If Parameters extends `evaluation.optimisation.TunableParameters`, the NTBEA-based `evaluation.optimisation.ParameterSearch` can optimise game parameters (or agent parameters) automatically. Register each tunable in the constructor with `addTunableParameter("name", defaultValue, possibleValues)` and wire `_reset()` to copy searched values back into fields.
