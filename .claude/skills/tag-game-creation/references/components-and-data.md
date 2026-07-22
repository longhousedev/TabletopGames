# Components, data files, constants, and phases

## Component catalog (`core.components`)

All extend `Component`, which provides `getComponentID()` (unique, framework-assigned), a name, an owner, and `copy()`. Build the game state out of these rather than raw collections when the objects are game pieces that actions need to reference.

| Class | Use for | Notes |
|---|---|---|
| `Deck<T extends Component>` | Ordered pile of cards/tiles | `new Deck<>("DrawPile", VisibilityMode.HIDDEN_TO_ALL)` or with an ownerId. `add`, `draw`, `peek`, `shuffle(rnd)`. |
| `PartialObservableDeck<T>` | Hands / piles with per-player visibility | `new PartialObservableDeck<>("Hand", ownerId, nPlayers, VisibilityMode.VISIBLE_TO_OWNER)`; per-card visibility flags for things like revealed cards. |
| `Card` | Base card; extend with game fields | `FrenchCard` is a ready-made standard 52-card implementation (see GoFish, Poker, Blackjack). |
| `Counter` | Bounded numeric track (health, money, supply) | `new Counter(startValue, min, max, "name")`; `increment`/`decrement`, `isMaximum`/`isMinimum`. |
| `Dice` | Die with N sides | `roll(rnd)`. |
| `Token` | A pawn/marker with a type string | |
| `GridBoard` | Rectangular grid of `BoardNode`s | `new GridBoard(w, h, new BoardNode(emptyName))`; `getElement(x,y)`, `setElement(x,y,node)` (see TicTacToe, Connect4). |
| `PartialObservableGridBoard` | Grid with per-player visibility | (see Stratego). |
| `GraphBoard` / `GraphBoardWithEdges` | Arbitrary connected locations | Node adjacency via `BoardNode` neighbours or explicit `Edge`s (see Pandemic's city map). |
| `Area` | Named bag of components | Loose grouping container. |

Visibility modes (`CoreConstants.VisibilityMode`): `VISIBLE_TO_ALL`, `HIDDEN_TO_ALL`, `VISIBLE_TO_OWNER`, `TOP_VISIBLE_TO_ALL`, `BOTTOM_VISIBLE_TO_ALL`, `MIXED_VISIBILITY`. These drive both GUI display and what `_copy(playerId)` should hide — but note the redeterminisation in `_copy` is still your code; visibility flags don't do it for you.

Components created during `_setup` get registered for `getComponentById` lookup via your `_getAllComponents()`. A `Deck` exposes its contents, so cards inside registered decks are findable by ID.

## JSON data files (optional)

For games with lots of content (card texts, board maps), put JSON under `data/<gamename>/` and pass that path as the final argument of the GameType enum entry. Load it via a `<Name>GameData extends AbstractGameData` class or your own reader invoked from `_setup`. Look at `data/pandemic/` + `games.pandemic` for the canonical example, or `games.descent2e` for heavier use. For games whose content is small (a fixed deck of 16 cards), plain Java constants are simpler and preferred.

## Constants class

A `<Name>Constants` class holds fixed game data that isn't a tunable parameter: card type enums, player token names, string hash keys. Example: `TicTacToeConstants` holds the player symbol tokens and the empty-cell name. Rule *numbers* belong in Parameters (tunable); *structural facts* belong in constants.

## Game phases (optional)

If different phases of a turn/round have different action sets, define:

```java
public enum MyGamePhase implements IGamePhase { Draft, Build, Score }
```

(usually an inner enum of the GameState). Set the initial phase in `_setup` with `state.setGamePhase(MyGamePhase.Draft)`, switch phases in `_afterAction`, and branch on `getGamePhase()` in `_computeAvailableActions`. `CoreConstants.DefaultGamePhase.Main` is the default if you never set one. The phase is part of framework state and is copied/compared automatically.
