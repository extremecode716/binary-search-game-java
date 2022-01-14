# binary-search-game-java

```java
import java.io.IOException;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ThreadLocalRandom;
import java.util.concurrent.TimeUnit;

/**
 * Binary Search Game <p>
 *
 * @author extremecode716
 */
public class Main {

    public static void main(String[] args) {
        GameManager.getInstance().addGame("Binary Search Game", new BinarySearchGame(1, 100));

        final Core core = Core.getInstance();
        core.init();
        while (!core.isGamesEmpty()) {
            core.run();
        }
    }
}

class Core {
    private final GameManager gameManager;
    private final ExitGameManager exitGameManager;

    private Core() {
        gameManager = GameManager.getInstance();
        exitGameManager = ExitGameManager.getInstance();
    }

    private static class Holder {
        private static final Core INSTANCE = new Core();
    }

    public static Core getInstance() {
        return Holder.INSTANCE;
    }

    public void init() {
        gameManager.init();
    }

    public void run() {
        update();
        rendering();
        clearGames();
    }

    private void update() {
        gameManager.progress();
    }

    private void rendering() {
        gameManager.rendering();
    }

    private void clearGames() {
        exitGameManager.run();
    }

    public boolean isGamesEmpty() {
        return gameManager.isGamesEmpty();
    }
}

class GameManager {
    private final Map<String, Game> games;

    private GameManager() {
        games = new ConcurrentHashMap<>();
    }

    private static class Holder {
        private static final GameManager INSTANCE = new GameManager();
    }

    public static GameManager getInstance() {
        return Holder.INSTANCE;
    }

    public void init() {
        games.forEach((gameName, game) -> game.awake());
        games.forEach((gameName, game) -> game.start());
    }

    public void progress() {
        games.forEach((gameName, game) -> {
            if (game.isPlayMode()) {
                game.update();
            }
        });
        games.forEach((gameName, game) -> {
            if (game.isPlayMode()) {
                game.lateUpdate();
            }
        });
        games.forEach((gameName, game) -> game.finalUpdate());

        // 물리 처리 (생략)
    }

    public void rendering() {
        games.forEach((gameName, game) -> game.rendering());
    }

    public void addGame(String gameName, Game game) {
        if (this.games.containsKey(gameName)) {
            System.out.printf("=== %s은 이미 실행중입니다. ===%n", gameName);
            return;
        }
        game.setName(gameName);
        this.games.put(gameName, game);
    }

    public Game removeGame(String gameName) {
        return this.games.remove(gameName);
    }

    public boolean isGamesEmpty() {
        return this.games.isEmpty();
    }
}

abstract class Game {
    public enum GameState {
        NONE,
        PLAY,
        PAUSE,
        STOP,
        END
    }

    protected String name;
    protected GameState gameState;
    protected boolean isExit;

    protected Game() {
        this.name = "";
        this.gameState = GameState.NONE;
        this.isExit = false;
    }

    public void awake() {
    }

    public void start() {
    }

    public void update() {
    }

    public void lateUpdate() {
    }

    public void finalUpdate() {
    }

    public void rendering() {
    }

    public boolean save() {
        return true;
    }

    public boolean load() {
        return true;
    }

    public String getName() {
        return this.name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public GameState getGameState() {
        return gameState;
    }

    public void changeGameState(GameState gameState) {
        this.gameState = gameState;
    }

    public boolean isPlayMode() {
        return GameState.PLAY == gameState;
    }

    public boolean isExit() {
        return isExit;
    }

    public void exit() {
        isExit = true;
    }
}

class BinarySearchGame extends Game {
    private static final int DEFAULT_MIN_N = 1;
    private static final int DEFAULT_MAX_N = 100;
    private static final long DEFAULT_TIMEOUT_MS = 5000;
    private static final long DEFAULT_UPDATE_INTERVAL_MS = 1000;

    private final int minN;
    private final int maxN;

    public BinarySearchGame() {
        super();
        minN = DEFAULT_MIN_N;
        maxN = DEFAULT_MAX_N;
    }

    public BinarySearchGame(int minN, int maxN) {
        super();
        this.minN = minN;
        this.maxN = maxN;
    }

    @Override
    public void awake() {
        this.gameState = GameState.PLAY;
        isExit = false;
    }

    @Override
    public void start() {
        final CountDownLatch latch = new CountDownLatch(1);

        new Thread(new GameBreaker(() -> {
            System.out.printf("== Enter 키를 누르면 %s이 종료됩니다 ==%n", getName());
            latch.countDown();
            System.in.read();
            ExitGameManager.getInstance().addExitGame(getName());
        })).start();

        try {
            if (!latch.await(DEFAULT_TIMEOUT_MS, TimeUnit.MILLISECONDS)) {
                System.out.printf("error CountDownLatch await timeout : %s ms%n", DEFAULT_TIMEOUT_MS);
                ExitGameManager.getInstance().addExitGame(getName());
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
            Thread.currentThread().interrupt();
            ExitGameManager.getInstance().addExitGame(getName());
        }
    }

    @Override
    public void update() {
        try {
            final int n = RandomUtil.RANDOM.nextInt(this.maxN - this.minN + 1) + this.minN;
            final int arraySize = RandomUtil.RANDOM.nextInt(n - this.minN + 1) + 1;
            int findNumber;
            int resultIndex;
            // 1. 중복되지 않은 무작위 배열 생성
            int[] uniqueRandomArray = CustomArrays.createUniqueRandomArray(this.minN, n + 1, arraySize);
            // 2. 배열 정렬
            CustomArrays.sort(uniqueRandomArray);
            findNumber = uniqueRandomArray[RandomUtil.RANDOM.nextInt(uniqueRandomArray.length)];
            // 3. 이진 탐색
            resultIndex = CustomArrays.binarySearch(uniqueRandomArray, findNumber);

            System.out.printf("[%s]%nARRAY => %s%nRESULT => 찾는 숫자 : %d 찾은 위치 : %d%n%n", getName(), Arrays.toString(uniqueRandomArray), findNumber, resultIndex);
            TimeUnit.MILLISECONDS.sleep(DEFAULT_UPDATE_INTERVAL_MS);
        } catch (Exception e) {
            e.printStackTrace();
            Thread.currentThread().interrupt();
            ExitGameManager.getInstance().addExitGame(getName());
        }
    }
}

class ExitGameManager {
    private final GameManager gameManager;
    private final Set<String> exitGames;

    private ExitGameManager() {
        gameManager = GameManager.getInstance();
        exitGames = ConcurrentHashMap.newKeySet();
    }

    private static class Holder {
        private static final ExitGameManager INSTANCE = new ExitGameManager();
    }

    public static ExitGameManager getInstance() {
        return Holder.INSTANCE;
    }

    public boolean addExitGame(String gameName) {
        return exitGames.add(gameName);
    }

    public boolean removeExitGame(String gameName) {
        return exitGames.remove(gameName);
    }

    public void run() {
        exitGames.forEach(gameName ->
                Optional.ofNullable(gameManager.removeGame(gameName)).ifPresent(game -> {
                    game.exit();
                    System.out.printf("%s을 종료합니다.%n", game.getName());
                }));
        exitGames.clear();
    }
}

class RandomUtil {
    public static final Random RANDOM;

    static {
        RANDOM = ThreadLocalRandom.current();
    }

    private RandomUtil() {
    }
}

@FunctionalInterface
interface GameBreakerFunc {
    void run() throws IOException, InterruptedException;

    default GameBreakerFunc andThen(GameBreakerFunc after) {
        Objects.requireNonNull(after);

        return () -> {
            run();
            after.run();
        };
    }
}

class GameBreaker implements Runnable {
    private final GameBreakerFunc gameBreakerFunc;

    public GameBreaker(GameBreakerFunc func) {
        gameBreakerFunc = func;
    }

    @Override
    public void run() {
        try {
            gameBreakerFunc.run();
        } catch (IOException | RuntimeException | InterruptedException e) {
            e.printStackTrace();
            Thread.currentThread().interrupt();
        }
    }
}

class CustomArrays {
    private CustomArrays() {
    }

    public static int[] createUniqueRandomArray(int numberOrigin, int numberBound, long size) {
        if (size > numberBound - numberOrigin)
            throw new IllegalArgumentException(String.format("(numberBound[%s] - numberOrigin[%s]):[%s] must be greater than size[%s]",
                    numberBound, numberOrigin, numberBound - numberOrigin, size));
        return RandomUtil.RANDOM.ints(numberOrigin, numberBound).distinct().limit(size).toArray();
    }

    public static void sort(int[] array) {
        Arrays.sort(array);
    }

    public static int binarySearch(int[] array, int key) {
        int low = 0;
        int high = array.length - 1;

        while (low <= high) {
            int mid = (low + high) >>> 1;
            int midVal = array[mid];

            if (midVal < key)
                low = mid + 1;
            else if (midVal > key)
                high = mid - 1;
            else
                return mid;
        }
        return -1;
    }
}
```
