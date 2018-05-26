# README
## 特性
  + 原生JavaScript (ECMAScript 2015)
  + 支持储存，读档，自动载入，记录最高得分
  + 移动端友好
  + 仿原应用的CSS

---

## 实现

### 定义
- 方格(Grid)：指在网页背景上显示的，由HTML tag所定义的预留空间
- 方块(Tile)：指叠放在方格上的真正可操纵对象

---

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
定义GameManager类
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
	- `Actuator`为`HTMLActuator`类的实例，用来处理与HTML DOM交互有关的事件
	- `StorageManager`为`LocalStorageManager`类的实例，用来处理本地数据的读取和保存
	- `startTiles`设置初始方块数量，默认为2
	- `inputManager.on()`绑定事件与对应的方法，在`KeyboardInputManager`中按栈模型处理
	- 通过`setup()`方法初始化游戏

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

    5. `addStartTiles()` : 创造最初始方块
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

    7. `actuate()` : 主驱动程序
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
        - `this.storageManager.getBestScore()`为历史最高得分，将其与`this.score`当前得分比较来更新历史记录
        - 通过判断`this.over`决定游戏是否继续，若继续则通过`this.storageManager.setGameState()`储存当前局面状态
        - `this.serialize()`返回序列化的当前每个方格的状态
        - 最后调用`this.actuator.actuate()`将当前状态通过HTML DOM绘制

    8. `serialize()` : 将当前每个方格的状态作为对象返回
        ```
        serialize() {
            return {
                grid: this.grid.serialize(),
                score: this.score,
                over: this.over,
                won: this.won,
                keepPlaying: this.keepPlaying
            };
        }
        ```

    9.  `prepareTiles()` : 储存所有方块的位置并重置合并信息
        ```
        prepareTiles() {
            this.grid.eachCell(function (x, y, tile) {
                if (tile) {
                    tile.mergedFrom = null;
                    tile.savePosition();
                }
            });
        }
        ```
        - 调用`this.grid.eachCell()`对每个方格执行函数，该函数检查当前方格上是否存在方块，若存在则储存其位置并将其合并源位置清空        - 

    10. `moveTile(tile, cell)` : 将一个方块移动到另一个方格上
        ```
        moveTile(tile, cell) {
            this.grid.cells[tile.x][tile.y] = null;
            this.grid.cells[cell.x][cell.y] = tile;
            tile.updatePosition(cell);
        }
        ```
        - 接受参数`tile`为欲移动的方块，`cell`为移动目的方格
        - 把坐标为`(tile.x, tile.y)`的方格设为无方块
        - 把坐标为`(cell.x, cell.y)`的方格设为存在方块`tile`
        - 调用`tile.updatePosition()`方法更新方块内部储存的坐标信息

    11. `move(direction)` : 方块移动合并事件驱动函数
        ```
        move(direction) {
            var self = this;
            if (this.isGameTerminated())
            return;
            var cell, tile;
            var vector = this.getVector(direction);
            var traversals = this.buildTraversals(vector);
            var moved = false;
            this.prepareTiles();
            traversals.x.forEach(function (x) {
                traversals.y.forEach(function (y) {
                    cell = { x: x, y: y };
                    tile = self.grid.cellContent(cell);
                    if (tile) {
                        var positions = self.findFarthestPosition(cell, vector);
                        var next = self.grid.cellContent(positions.next);
                        if (next && next.value === tile.value && !next.mergedFrom) {
                            var merged = new Tile(positions.next, tile.value * 2);
                            merged.mergedFrom = [tile, next];
                            self.grid.insertTile(merged);
                            self.grid.removeTile(tile);
                            tile.updatePosition(positions.next);
                            self.score += merged.value;
                            if (merged.value === 2048)
                            self.won = true;
                        }
                        else {
                            self.moveTile(tile, positions.farthest);
                        }
                        if (!self.positionsEqual(cell, tile)) {
                            moved = true;
                        }
                    }
                });
            });
            if (moved) {
                this.addRandomTile();
                if (!this.movesAvailable()) {
                    this.over = true;
                }
                this.actuate();
            }
        }
        ```
        - 首先判断`this.isGameTerminated()`，若为`True`则直接退出
        - 通过`this.getVector()`获取移动的方向和距离，通过`this.buildTraversals()`获取遍历方格坐标的数组
        - 对`traversals`中的每个坐标，先通过`this.grid.cellContent()`来判断该方格上是否存在方块；若存在，通过`this.findFarthestPosition()`来寻找在该方向上该方块(`tile`，下同)能移动的最大距离和第一个遇见或出界的方块(`next`，下同)。
        - 当且仅当`next`存在(未出界)，`tile`与`next`数值相等，`next`在此次坐标更新中未移动过这三个条件同时成立时，在`next`对应的坐标上新建一个数值为`2 * tile.value`的方块并储存此次移动信息；将这次合并所获得的分数加在总分数上，若此时总分等于2048则令`this.won`值为`True`
        - 若不满足上述三个条件之一，则只需调用`this.moveTile()`方法将当前方块移至其在当前方向上最远所能到达的位置
        - 调用`this.positionsEqual()`方法来检查上述步骤是否出错，若无误则在一个可用位置随机生成一个新的方块；此时调用`this.movesAvailable()`检查是否可以继续移动，若不能继续则游戏结束

    12. `getVector(direction)` : 将用户输入转换为方向向量
        ```
        getVector(direction) {
            var map = {
                0: { x: 0, y: -1 }, // Up
                1: { x: 1, y: 0 }, //Right
                2: { x: 0, y: 1 }, //Down
                3: { x: -1, y: 0 } // Left
            };
            return map[direction];
        }
        ```
        - 返回输入`direction`所对应的方向向量

    13. `buildTraversals(vector)` ： 返回一个方格坐标遍历数组
        ```
        buildTraversals(vector) {
            var traversals = { x: [], y: [] };
            for (var pos = 0; pos < this.size; pos++) {
                traversals.x.push(pos);
                traversals.y.push(pos);
            }
            if (vector.x === 1)
                traversals.x = traversals.x.reverse();
            if (vector.y === 1)
                traversals.y = traversals.y.reverse();
            return traversals;
        }
        ```
        - 对每个方格将其位置信息保存在`traversals`内
        - 最后两个`if`语句确保此遍历数组永远是从移动方向的最远端开始遍历

    14. `findFarthestPosition(cell, vector)` ： 返回指定坐标在指定方向的最远可用位置
        ```
        findFarthestPosition(cell, vector) {
            var previous;
            do {
                previous = cell;
                cell = { x: previous.x + vector.x, y: previous.y + vector.y };
            } while (this.grid.withinBounds(cell) && this.grid.cellAvailable(cell));
            return {
                farthest: previous,
                next: cell
            };
        }
        ```
        - 从当前位置开始，每次沿着指定方向迭代前进后检查当前方格是否可用(`this.grid.cellAvailable()`)以及是否出界(`this.grid.withinBounds()`)
        - 返回界内最远可用方格的位置以及第一个不可用或出界位置

    15. `movesAvailable()` ： 检查是否存在可移动方块
        ```
        movesAvailable() {
            return this.grid.cellsAvailable() || this.tileMatchesAvailable();
        }
        ```
        - 当存在可用方格(可生成新的方块)或存在可合并方块时返回`True`，否则返回`False`

    16. `tileMatchesAvailable()` ： 检查是否存在可合并方块
        ```
        tileMatchesAvailable() {
            var self = this;
            var tile;
            for (var x = 0; x < this.size; x++) {
                for (var y = 0; y < this.size; y++) {
                    tile = this.grid.cellContent({ x: x, y: y });
                    if (tile) {
                        for (var direction = 0; direction < 4; direction++) {
                            var vector = self.getVector(direction);
                            var cell = { x: x + vector.x, y: y + vector.y };
                            var other = self.grid.cellContent(cell);
                            if (other && other.value === tile.value) {
                                return true;
                            }
                        }
                    }
                }
            }
            return false;
        }
        ```
        - 对于每个方格，如果其上存在方块，则以其坐标为原点对每个方向都检查在此方向上第一个遇见的方块其数值是否与当前方块一致

    17. `positionsEqual(first, second)` ： 检查`first`与`second`坐标是否一致
        ```
        positionsEqual(first, second) {
            return first.x === second.x && first.y === second.y;
        }
        ```

---

### `/js/bind_polyfill.js` : 改写绑定事件
```
Function.prototype.bind = Function.prototype.bind || function (target) {
  var self = this;
  return function (args) {
    if (!(args instanceof Array)) {
      args = [args];
    }
    self.apply(target, args);
  };
};
```
对于存在绑定事件的函数对象不改变其原有事件；对于不存在绑定事件的函数对象，将其默认绑定至一个预定义函数上，该函数将调用时的`this`替换为`target`，同时把`args`数组作为参数传递给`this`对象。通过此函数使得所有的函数对象都通过`apply()`接受一个数组作为输入参数。

---

### `/js/classlist_polyfill.js` : 初始化与操作 HTML DOM 相关的设置

### `/js/animframe_polyfill.js` : 处理在不同浏览器上与更新动画相关的设置
```
var lastTime = 0;
  var vendors = ['webkit', 'moz'];
  for (var x = 0; x < vendors.length && !window.requestAnimationFrame; ++x) {
    window.requestAnimationFrame = window[vendors[x] + 'RequestAnimationFrame'];
    window.cancelAnimationFrame = window[vendors[x] + 'CancelAnimationFrame'] ||
      window[vendors[x] + 'CancelRequestAnimationFrame'];
  }
```
对不同浏览器分别重写调用动画和回调动画函数并绑定给函数对象。

若上述预设值均不被当前浏览器接受，则手动重写该函数。`id`是回调列表中唯一的标识，可以传这个值给`window.cancelAnimationFrame()`以取消回调函数。
```
if (!window.requestAnimationFrame) {
    window.requestAnimationFrame = function (callback) {
      var currTime = new Date().getTime();
      var timeToCall = Math.max(0, 16 - (currTime - lastTime));
      var id = window.setTimeout(function () {
        callback(currTime + timeToCall);
      },
      timeToCall);
      lastTime = currTime + timeToCall;
      return id;
    };
  }

if (!window.cancelAnimationFrame) {
    window.cancelAnimationFrame = function (id) {
      clearTimeout(id);
    };
  }
```

---

### `/js/keyboard_input_manager.js` : 处理输入事件
定义`KeyboardInputManager`类
1. 构造函数
    ```
    constructor() {
        this.events = {};
        if (window.navigator.msPointerEnabled) {
            this.eventTouchstart = "MSPointerDown";
            this.eventTouchmove = "MSPointerMove";
            this.eventTouchend = "MSPointerUp";
        }
        else {
            this.eventTouchstart = "touchstart";
            this.eventTouchmove = "touchmove";
            this.eventTouchend = "touchend";
        }
        this.listen();
    }
    ```
    - `this.events`为事件数组，按栈模型处理
    - 定义移动端访问手指滑动事件参数；通过`window.navigator.msPointerEnabled`判断如果浏览器为`IE`系列，则定义略有不同
    - 调用`this.listen()`开始监听输入

2. 对象方法
    1. `` ： 
        ```
        
        ```
        - 