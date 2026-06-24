# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Start here

A more detailed, up-to-date summary of the codebase structure is maintained at
**https://deepwiki.com/GAIGResearch/TabletopGames**. Consult it first (e.g. via WebFetch) before
doing any detailed analysis of the source — it is faster than reading the code directly and gives a
fuller architectural picture than the overview below.

## What this is

TAG (Tabletop Games Framework) is a Java benchmark for board/card game AI research. It provides a
common API so that ~40 games and a range of AI agents can be run against each other, tuned, and
analysed through a shared infrastructure. This fork (`Advisers` branch) adds **advisers** (players
that suggest moves to other players) and an **LLM** module that generates/uses heuristics via
langchain4j.

## Build, test, run

Maven project, **Java 21**.

```bash
mvn package                 # builds target/TAG.jar (jar-with-dependencies, main class core.TAG)
```

Tests are **skipped by default** (`maven.test.skip=true` in pom.xml). To run them you must override:

```bash
mvn test -Dmaven.test.skip=false                       # all tests (JUnit 4 + Mockito)
mvn test -Dmaven.test.skip=false -Dtest=CustomDiceTest  # single test class
mvn test -Dmaven.test.skip=false -Dtest=CustomDiceTest#methodName  # single method
```

Running the framework — `core.TAG` dispatches on its first argument to a sub-module:

```bash
java -jar target/TAG.jar <EntryPoint> [args...]
# EntryPoints: RunGames, ParameterSearch, FrontEnd, FrontEndSimple,
#              ExpertIteration, OneStepDeviations, SkillLadder
java -jar target/TAG.jar RunGames --help    # each entry point has its own --help
```

From an IDE, run `core.TAG`, or run `gui.Frontend` directly for the GUI. Most entry points are
configured via JSON config files (see `--help` / `RunArg`) or command-line `key=value` args parsed
by `evaluation/RunArg.java`. A `Dockerfile` builds and packages the same jar with the `data/`
directory.

## Core architecture

A game is the combination of four cooperating pieces, all in `core`:

- **`AbstractGameState`** — all mutable game data. Implement `_copy(playerId)` (redeterminises hidden
  info for that player's perspective), `_getAllComponents()`, `getGameScore()`, `_getHeuristicScore()`,
  `_equals()`. State is the single source of truth; the forward model never holds game data.
- **`AbstractForwardModel`** — the rules engine. Implement `_setup()`, `_next(state, action)`,
  `_computeAvailableActions(state)`, `endPlayerTurn(state)`. Most games extend
  **`StandardForwardModel`** (use `StandardForwardModelWithTurnOrder` only for legacy turn-order games).
- **`AbstractParameters`** — tunable game parameters (implements `ITunableParameters` so games can be
  optimised/searched).
- **`AbstractGUIManager`** subclass — Swing rendering (optional but expected).

`core.Game` is the orchestrator: it owns the state, forward model, and the list of `AbstractPlayer`s,
drives the turn loop, broadcasts `Event`s to `IGameListener`s, and tracks per-tick statistics.

**Actions** (`core.actions`) implement `AbstractAction`; multi-step decisions use
`IExtendedSequence` to keep an action "in progress" across several player decisions.

**Registering a game**: every game lives in `games.<name>` (e.g. `games.catan`) and is wired into the
central enum **`games.GameType`**, which maps the game to its state/forward-model/parameters/GUI
classes plus metadata (min/max players, categories). This enum is the canonical list of games — add
new games here.

## Players (agents)

`core.AbstractPlayer` defines an agent via `_getAction(state, possibleActions)` and `copy()`. Notable
implementations under `players/`:

- `simple` — `RandomPlayer`, `OSLAPlayer` (one-step look-ahead), `FirstActionPlayer`.
- `mcts` (full MCTS, heavily configurable), `basicMCTS`, `rhea` (rolling-horizon EA), `rmhc`.
- `heuristics` — `IStateHeuristic` / `IActionHeuristic` implementations (linear, logistic, decision-tree,
  learned-value, etc.) used to bias the search agents.
- `decorators` (implement `IPlayerDecorator`) — wrap a player to **filter the action list** before it
  decides (e.g. forbidding action classes, epsilon-randomness).
- `observers` — **the Advisers feature** (`Advisers` branch). `GameAdviser` is an `IGameListener`
  composed of an `AbstractPlayer` (decides what is "best") plus an `IAdviceFilter` (decides *which*
  players are advised and *when* to intervene), writing advice to a file.

`players.PlayerFactory.createPlayer(jsonFileOrName)` constructs agents from JSON descriptors — this is
how tournaments and the GUI instantiate configurable players (see `json/players/`).

## Evaluation & analytics

`evaluation/` is the experiment-running side:

- `RunGames` — run tournaments of agents over one or more games (round-robin, one-vs-all, etc.).
- `tournaments/` — `RoundRobinTournament`, `SkillLadder`/`SkillGrid` (vary budget), result analyses.
- `optimisation/` — `ParameterSearch` (NTBEA-based tuning of tunable parameters), `OneStepDeviations`.
- `metrics/`, `listeners/`, `loggers/`, `summarisers/` — `IGameListener` + `Event` pipeline that
  records game data to files/stats for analysis. Listener configs live in `metrics/*.json`.
- `features/`, `heuristics/` — state/action feature extraction feeding learned heuristics and
  `ExpertIteration`.

## LLM module

`llm/` integrates langchain4j (OpenAI, Anthropic, Gemini, Mistral backends; see pom.xml). `JavaCoder`
asks an LLM to write Java heuristic code; `GamePromptGenerator` builds game-state prompts;
`StringHeuristic` compiles LLM-produced heuristic strings (uses `javaparser`). Requires provider API
keys in the environment.

## Config & data locations

- `json/` — player definitions, search spaces, listener and learner configs consumed at runtime.
- `metrics/` — metric/listener JSON configs; `metrics/out/` is the default results directory.
- `data/` — game assets (images, card/board definitions) loaded by some games; shipped into the
  Docker image.

## Python

`core.PyTAG` + `players/python` expose a Python-facing API (PyTAG) for RL/gym-style use.
