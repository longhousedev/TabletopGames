# GUI (optional)

A game is fully playable by agents with no GUI at all — pass `null` as the GUIManager class in the GameType entry and run headless. Add a GUI when humans need to play (or to debug visually).

## Minimal GUIManager

Extend `gui.AbstractGUIManager` (Swing-based):

```java
public class MyGUIManager extends AbstractGUIManager {
    MyBoardView view;   // your JComponent subclass

    public MyGUIManager(GamePanel parent, Game game, ActionController ac, Set<Integer> humanId) {
        super(parent, game, ac, humanId);
        if (game == null) return;
        MyGameState state = (MyGameState) game.getGameState();
        view = new MyBoardView(state);
        // standard furniture: info panel (round, scores, phase) + action buttons for the human player
        JPanel infoPanel = createGameStateInfoPanel("MyGame", state, defaultDisplayWidth, defaultInfoPanelHeight);
        JComponent actionPanel = createActionPanel(new IScreenHighlight[0], defaultDisplayWidth, defaultActionPanelHeight);
        parent.setLayout(new BorderLayout());
        parent.add(view, BorderLayout.CENTER);
        parent.add(infoPanel, BorderLayout.NORTH);
        parent.add(actionPanel, BorderLayout.SOUTH);
        parent.revalidate();
        parent.setVisible(true);
    }

    @Override public int getMaxActionSpace() { return /* upper bound on simultaneous actions */ 50; }

    @Override
    protected void _update(AbstractPlayer player, AbstractGameState gameState) {
        view.updateState((MyGameState) gameState);   // then framework repaints
    }
}
```

Check the exact constructor signature against a recently added game (e.g. `games.gofish.gui.GoFishGUIManager` or `games.tictactoe.gui.TicTacToeGUIManager`) rather than trusting docs — it has changed over time.

What the base class gives you for free:

- Player scores / game status / current player display via `createGameStateInfoPanel`
- Action history panel
- Clickable action buttons for human players via `createActionPanel` (labels come from `action.getString(gs)`)

## Views

Put custom drawing in `JComponent` subclasses in a `gui` subpackage of your game. Reusable views live in `gui.views` — `DeckView` (card piles/hands, handles hidden cards), `CardView`, `ComponentView`. Grid games typically paint cells directly (see `games.tictactoe.gui.TTTBoardView`); card games compose `DeckView`s (see GoFish or SushiGo GUIs).

For simple debugging without a GUI, implement `core.interfaces.IPrintable` on the game state and print it from tests instead.
