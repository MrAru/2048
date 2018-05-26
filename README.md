# README
## 特性
  + 原生JavaScript (ECMAScript 2015)
  + 支持储存，读档，自动载入，记录最高得分
  + 移动端友好
  + 仿原应用的CSS

---

## 实现
### `index.html`
构造 4 x 4 `<div>`方格作为背景并统一`class='grid-cell'`,预留一个`class='tile-container'`的`<div>`标签用来在背景方格上描绘方块。

---

### `/js/application.js` : 初始化渲染
```
window.requestAnimationFrame(function () {
    new GameManager(4, KeyboardInputManager, HTMLActuator, LocalStorageManager);
});
```
在`index.html`中最后加载此文件，目的是使浏览器加载全部JS文件之后再开始渲染避免一些奇怪的bug。

---

### `/js/game_manager.js` : 游戏主事件驱动
1. 构造函数
	```
    constructor(size, InputManager, Actuator, StorageManager) {
        this.size = size;
        this.inputManager = new InputManager;
        this.storageManager = new StorageManager;
        this.actuator = new Actuator;
        this.startTiles = 2;
        this.inputManager.on("move", this.move.bind(this));
        this.inputManager.on("restart", this.restart.bind(this));
        this.inputManager.on("keepPlaying", this.keepPlaying.bind(this));
        this.setup();
    }
    ```
	- `size`控制背景方格边长（即 `size x size` ）
	- `InputManager`为`KeyboardInputManager`类的实例，按栈模型处理输入事件
	- `Actuator`为`HTMLActuator`类的实例，用来处理与HTMLDom交互有关的事件
	- `StorageManager`为`LocalStorageManager`类的实例，用来处理本地数据的读取和保存
	- `startTiles`设置初始方块数量，默认为2
	- `inputManager.on()`绑定事件与对应的方法，在`KeyboardInputManager`中按栈模型处理
	- `setup()`方法初始化游戏

2. 对象方法
	1. `setup()` ： 初始化游戏
        ```
        setup() {
            var previousState = this.storageManager.getGameState();
            if (previousState) {
                this.grid = new Grid(previousState.grid.size, previousState.grid.cells);
                this.score = previousState.score;
                this.over = previousState.over;
                this.won = previousState.won;
                this.keepPlaying = previousState.keepPlaying;
            }
            else {
                this.grid = new Grid(this.size);
                this.score = 0;
                this.over = false;
                this.won = false;
                this.keepPlaying = false;
                this.addStartTiles();
            }
            this.actuate();
        }
        ```
        - 如果存在上次游戏未结束退出则通过`this.storageManager.getGameState()`自动读取上次的存档
        - 若是新游戏则创建`size x size`的方格并初始化各类参数，更新第一次`actuate()`

    2. `restart()` ： 重启游戏并清除当前状态的储存
        ```
        restart() {
            this.storageManager.clearGameState();
            this.actuator.continueGame();
            this.setup();
        }
        ```
        - 调用`this.storageManager.clearGameState()`清除当前游戏状态的储存
        - 调用`this.actuator.continueGame()`清除当前可能在HTML中显示的输/赢弹窗
        - 调用`this.setup()`新建游戏

    3. `keepPlaying()` ： 允许玩家在得分超过2048后继续游戏
        ```
        keepPlaying() {
            this.keepPlaying = true;
            this.actuator.continueGame();
        }
        ```
        - 将`this.keepPlaying`的值修改为`Ture`
        - 调用`this.actuator.continueGame()`清除当前在HTML中显示的游戏胜利弹窗

    4. `isGameTerminated()` : 当用户失败或赢得游戏或不继续玩则返回`True`
        ```
        isGameTerminated() {
            return this.over || (this.won && !this.keepPlaying);
        }
        ```
        - `this.over`指示是否失败
        - `this.won`指示是否胜利
        - `this.keepPlaying`指示用户是否继续玩

    5. `addStartTiles()` : 初始化方块
        ```
        addStartTiles() {
            for (var i = 0; i < this.startTiles; i++) {
                this.addRandomTile();
            }
        }
        ```
        - `this.startTiles`在构造函数中定义，指示最初方块的个数
        - `this.addRandomTile()`为初始的每个方块分配随机的数值

    6. `addRandomTile()` : 为初始方块分配随机数值
        ```
        addRandomTile() {
            if (this.grid.cellsAvailable()) {
                var value = Math.random() < 0.9 ? 2 : 4;
                var tile = new Tile(this.grid.randomAvailableCell(), value);
                this.grid.insertTile(tile);
            }
        }
        ```
        - `this.gird.cellsAvailable()`指示当前方块是否可访问
        - `Math.random() < 0.9 ? 2 : 4`按照 9 : 1 的比例随机分配初始值为 2 和 4
        - `new Tile(this.grid.randomAvailableCell(), value)`在随机一个并未被使用的区域新建一个值为`value`的方块
        - `this.gird.insertTile(tile)`把该方块放入其所在背景方格

    7. `actuate()` : 
        ```
        actuate() {
            if (this.storageManager.getBestScore() < this.score) {
                this.storageManager.setBestScore(this.score);
            }
            if (this.over) {
                this.storageManager.clearGameState();
            }
            else {
                this.storageManager.setGameState(this.serialize());
            }
            this.actuator.actuate(this.grid, {
                score: this.score,
                over: this.over,
                won: this.won,
                bestScore: this.storageManager.getBestScore(),
                terminated: this.isGameTerminated()
                }
            );
        }
        ```


### 


### 
  