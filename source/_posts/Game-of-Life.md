---
title: Game of Life
date: 2019-10-24 23:38:08
tags:
---

<style>
  .align-center {
    text-align: center;
  }
</style>

## 康威生命游戏
康威生命游戏（英语：Conway's Game of Life），又称康威生命棋，是英国数学家约翰·何顿·康威在1970年发明的细胞自动机。
它最初于1970年10月在《科学美国人》杂志上马丁·葛登能的“数学游戏”专栏出现。以下是几个例子：


![](https://upload.wikimedia.org/wikipedia/commons/e/e5/Gospers_glider_gun.gif)
<p class="align-center">
<i>可持续繁殖模式：“高斯帕机枪”不断制造“滑翔机”</i>
</p>

![](https://upload.wikimedia.org/wikipedia/commons/e/e6/Conways_game_of_life_breeder_animation.gif)
<p class="align-center">
<i>康威生命游戏中的较复杂演化模式：“播种机”模式不断生成“高斯帕机枪”来制造“滑翔机”</i>
</p>

## 细胞自动机
细胞自动机，是指方格形状细胞的集合。每个细胞有自己的状态，并且会根据相邻细胞的状态使用一个简单的规则，来改变自己的状态，一代一代演化下去。

### 最简一维细胞自动机
如下图所示，包含4个细胞，组成一行。每个细胞有两种状态，0和1，分别表现为黑色和白色，每个细胞最多有2个邻居（左、右）
<style>
  .grid {
    display: flex;
    justify-content: center;
  }
  .cell-wrapper {
    text-align: center;
  }
  .cell {
    margin: 8px;
    width: 60px;
    height: 60px;
    background-color: #000;
    border: 1px solid #eee;
  }
  .cell.alive {
    background-color: #fff;
  }
  .tip {
    display: inline-block;
  }
</style>
<div class="grid">
  <div class="cell-wrapper">
    <div class="cell"></div>
    <div class="tip">0</div>
  </div>
  <div class="cell-wrapper">
    <div class="cell alive"></div>
    <div class="tip">1</div>
  </div>
  <div class="cell-wrapper">
    <div class="cell"></div>
    <div class="tip">0</div>
  </div>
  <div class="cell-wrapper">
    <div class="cell alive"></div>
    <div class="tip">1</div>
  </div>
</div>

### 演化
每个细胞会更具相邻的细胞状态应用同一个规则，不断的进行演化。
例如应用一个规则：左右细胞状态相同则变成状态0，不同则变成状态1。
<div>
  第一代
  <div class="grid">
    <div class="cell-wrapper">
      <div class="cell alive"></div>
      <div class="tip">1</div>
    </div>
    <div class="cell-wrapper">
      <div class="cell alive"></div>
      <div class="tip">1</div>
    </div>
    <div class="cell-wrapper">
      <div class="cell alive"></div>
      <div class="tip">1</div>
    </div>
    <div class="cell-wrapper">
      <div class="cell alive"></div>
      <div class="tip">1</div>
    </div>
  </div>
</div>

<div>
  第二代
  <div class="grid">
    <div class="cell-wrapper">
      <div class="cell alive"></div>
      <div class="tip">1</div>
    </div>
    <div class="cell-wrapper">
      <div class="cell"></div>
      <div class="tip">0</div>
    </div>
    <div class="cell-wrapper">
      <div class="cell"></div>
      <div class="tip">0</div>
    </div>
    <div class="cell-wrapper">
      <div class="cell alive"></div>
      <div class="tip">1</div>
    </div>
  </div>
</div>

<div>
  第三代
  <div class="grid">
    <div class="cell-wrapper">
      <div class="cell"></div>
      <div class="tip">0</div>
    </div>
    <div class="cell-wrapper">
      <div class="cell alive"></div>
      <div class="tip">1</div>
    </div>
    <div class="cell-wrapper">
      <div class="cell alive"></div>
      <div class="tip">1</div>
    </div>
    <div class="cell-wrapper">
      <div class="cell"></div>
      <div class="tip">0</div>
    </div>
  </div>
</div>

### 代码实现
下面是实现上述规1维细胞自动机的代码
```html
<html>
  <head>
    <style>
      body {
        background-color: #000;
      }
      .tip {
        color: #fff;
      }
      #container {
        position: fixed;
        display: flex;
        top: 50%;
        left: 50%;
        transform: translate(-50%, -50%);
        border: 1px solid #eee;
      }
      .cell {
        display: inline-block;
        width: 80px;
        height: 80px;
      }
      .cell.pos {
        background-color: #fff;
      }
      .cell.neg {
        background-color: #000;
      }
    </style>
  </head>
  <body>
    <div class="tip">初始状态：1 1 1 1 1 1</div>
    <div class="tip">规则：左右异或 </div>
    <div id="container"></div>
  </body>
  <script>
    const INTERVAL = 300;

    function Cell(state) {
      let container = document.getElementById('container');
      let dom = document.createElement('div');
      dom.classList.add('cell');
      container.appendChild(dom);
      this.dom = dom;
      this.state = state;
      this.leftState = 0;
      this.rightState = 0;
      this.set(state);
    }

    Cell.prototype.set = function(state) {
      this.state = state;
      if (state) {
        this.dom.classList.remove('neg');
        this.dom.classList.add('pos');
      } else {
        this.dom.classList.add('neg');
        this.dom.classList.remove('pos');
      }
    }

    Cell.prototype.next = function() {
      const nextState = this.leftState ^ this.rightState;

      this.set(nextState);
    }

    function Gird(initialState) {
      this.cells = [];
      for (let i = 0; i < initialState.length; i++) {
        this.cells.push(new Cell(initialState[i]));
      }
      for (let i = 0; i < initialState.length; i++) {
        const cell = this.cells[i];
        cell.leftState = this.cells[i - 1] ? this.cells[i - 1].state : 0;
        cell.rightState = this.cells[i + 1] ? this.cells[i + 1].state : 0;
      }
    }

    Gird.prototype.run = function() {
      setInterval(() => {
        for (let i = 0; i < this.cells.length; i++) {
          this.cells[i].next();
        }
        for (let i = 0; i < initialState.length; i++) {
          const cell = this.cells[i];
          cell.leftState = this.cells[i - 1] ? this.cells[i - 1].state : 0;
          cell.rightState = this.cells[i + 1] ? this.cells[i + 1].state : 0;
        }
      }, INTERVAL);
    }

    const initialState = [1, 1, 1, 1, 1, 1];
    const ca = new Gird(initialState);
    ca.run();
  </script>
</html>
```

## 二维细胞自动机与生命游戏
二维细胞自动机是行、列进行排列的表格状。容易想到，每个细胞最多有8个邻居（上、下、左、右、对角线4个）。
上面讲了生命游戏其实就是一个二维细胞自动机。

1. 它应用以下规则：
- 孤独致死：活着的邻居总数小于2，死亡
- 存活：活着的邻居总数等于2或者3，继续存活
- 人口过多致死：活着的邻居总数大于3，死亡
- 出生，活着的邻居总数正好等于3，则出生

2. 初始化
不同的初始化情况，则可以演化出不同的图案。
康威研究了各种各样初始化条件，总结除了很多“Pattern”。初始化的时候，把这些Pattern丢进去，会演化出很有的行为。

我们可以用一个二维状态数组表示Pattern。比如信号灯：
![](https://upload.wikimedia.org/wikipedia/commons/9/95/Game_of_life_blinker.gif)
```js
[
  [1],
  [1],
  [1]
]
```

### 代码实现
可以支持填写Pattern，然后点击运行，即可观看生命的演化
```html
<html>

<head>
  <style>
    body {
      background-color: #000;
    }

    .tip {
      color: #fff;
    }

    .tip #pattern, .tip #count {
      font-size: 20px;
    }

    #container {
      position: fixed;
      top: 50%;
      left: 50%;
      transform: translate(-50%, -50%);
      border: 1px solid #eee;
    }

    .row {
      display: flex;
    }

    .cell {
      display: inline-block;
      width: 10px;
      height: 10px;
    }

    .cell.pos {
      background-color: #fff;
    }

    .cell.neg {
      background-color: #000;
    }
  </style>
</head>

<body>
  <div class="tip">
    Pattern:
    <textarea type="text" id="pattern" cols="8"></textarea>
    <br>
    Count: &nbsp; 
    <input type="text" id="count">
    <br>
    <button id="run">运行</button>
  </div>
  <div id="container"></div>
</body>
<script>
  function applyPattern(initialState, pattern, count = 1) {
    const lenRow = initialState.length;
    const lenColumn = initialState[0].length;

    const lenPatternRow = pattern.length;
    const lenPatternColumn = pattern[0].length;

    for (let i = 0; i < count; i++) {
      const randomX = Math.floor(Math.random() * (lenColumn - lenPatternColumn));
      const randomY = Math.floor(Math.random() * (lenRow - lenPatternRow));

      for (let y = 0; y < lenPatternRow; y++) {
        for (let x = 0; x < lenPatternColumn; x++) {
          initialState[randomY + y][randomX + x] = pattern[y][x];
        }
      }
    }

  }

  const INTERVAL = 100;

  function Cell(state) {
    let dom = document.createElement('div');
    dom.classList.add('cell');
    this.dom = dom;
    this.state = state;
    this.leftState = 0;
    this.rightState = 0;
    this.upState = 0;
    this.bottomState = 0;
    this.leftTopState = 0;
    this.rightTopState = 0;
    this.leftBottomState = 0;
    this.rightBottomState = 0;
    this.set(state);
  }

  Cell.prototype.set = function (state) {
    this.state = state;
    if (state) {
      this.dom.classList.remove('neg');
      this.dom.classList.add('pos');
    } else {
      this.dom.classList.add('neg');
      this.dom.classList.remove('pos');
    }
  }

  Cell.prototype.next = function () {
    let nextState = this.state;

    const neighborCount = 
      this.leftState + this.rightState + this.upState + this.bottomState 
      + this.leftTopState + this.rightTopState + this.leftBottomState + this.rightBottomState;

    // 1.孤独致死
    if (neighborCount < 2) {
      nextState = 0;
    }

    // 2.活到下一代
    if (neighborCount === 2 || neighborCount === 3) {
      nextState = this.state;
    }

    // 3.人口过多致死
    if (neighborCount > 3) {
      nextState = 0;
    }

    // 4. 出生
    if (!this.state && neighborCount === 3) {
      nextState = 1;
    }

    this.set(nextState);
  }

  function Gird(initialState) {
    this.interval = null;
    this.rows = [];
    this.lenRow = initialState.length;
    this.lenColumn = initialState[0].length;

    const container = document.getElementById('container');
    for (let y = 0; y < this.lenRow; y++) {
      const domRow = document.createElement('div');
      domRow.classList.add('row');
      const row = [];
      for (let x = 0; x < this.lenColumn; x++) {
        const cell = new Cell(initialState[y][x]);
        row.push(cell);
        domRow.appendChild(cell.dom);
      }
      this.rows.push(row);
      container.appendChild(domRow);
    }

    for (let y = 0; y < this.lenRow; y++) {
      for (let x = 0; x < this.lenColumn; x++) {
        const cell = this.rows[y][x];
        cell.leftState = this.rows[y] && this.rows[y][x - 1] ? this.rows[y][x - 1].state : 0;
        cell.rightState = this.rows[y] && this.rows[y][x + 1] ? this.rows[y][x + 1].state : 0;
        cell.upState = this.rows[y - 1] && this.rows[y - 1][x] ? this.rows[y - 1][x].state : 0;
        cell.bottomState = this.rows[y + 1] && this.rows[y + 1][x] ? this.rows[y + 1][x].state : 0;
        cell.leftTopState = this.rows[y - 1] && this.rows[y - 1][x - 1] ? this.rows[y - 1][x - 1].state : 0;
        cell.rightTopState = this.rows[y - 1] && this.rows[y - 1][x + 1] ? this.rows[y - 1][x + 1].state : 0;
        cell.leftBottomState = this.rows[y + 1] && this.rows[y + 1][x - 1] ? this.rows[y + 1][x - 1].state : 0;
        cell.rightBottomState = this.rows[y + 1] && this.rows[y + 1][x + 1] ? this.rows[y + 1][x + 1].state : 0;
      }
    }
  }

  Gird.prototype.run = function () {
    this.interval = setInterval(() => {
      for (let y = 0; y < this.lenRow; y++) {
        for (let x = 0; x < this.lenColumn; x++) {
          const cell = this.rows[y][x];
          cell.next();
        }
      }

      for (let y = 0; y < this.lenRow; y++) {
        for (let x = 0; x < this.lenColumn; x++) {
          const cell = this.rows[y][x];
          cell.leftState = this.rows[y] && this.rows[y][x - 1] ? this.rows[y][x - 1].state : 0;
          cell.rightState = this.rows[y] && this.rows[y][x + 1] ? this.rows[y][x + 1].state : 0;
          cell.upState = this.rows[y - 1] && this.rows[y - 1][x] ? this.rows[y - 1][x].state : 0;
          cell.bottomState = this.rows[y + 1] && this.rows[y + 1][x] ? this.rows[y + 1][x].state : 0;
          cell.leftTopState = this.rows[y - 1] && this.rows[y - 1][x - 1] ? this.rows[y - 1][x - 1].state : 0;
          cell.rightTopState = this.rows[y - 1] && this.rows[y - 1][x + 1] ? this.rows[y - 1][x + 1].state : 0;
          cell.leftBottomState = this.rows[y + 1] && this.rows[y + 1][x - 1] ? this.rows[y + 1][x - 1].state : 0;
          cell.rightBottomState = this.rows[y + 1] && this.rows[y + 1][x + 1] ? this.rows[y + 1][x + 1].state : 0;
        }
      }
    }, INTERVAL);
  }

  Gird.prototype.destroy = function() {
    clearInterval(this.interval);
    for (let row of this.rows) {
      for (let cell of row) {
        cell.dom.remove();
      }
    }
  }

  let gird = null;

  document.getElementById('run').addEventListener('click', () => {
    const pattern = JSON.parse(document.getElementById('pattern').value);
    const count = parseInt(document.getElementById('count').value);
    if (gird) {
      gird.destroy();
    }

    const ROW = 50;
    const COLUMN = 50;

    const initialState = [];


    for (let y = 0; y < ROW; y++) {
      const row = [];
      for (let x = 0; x < COLUMN; x++) {
        row.push(0);
      }
      initialState.push(row);
    }

    applyPattern(initialState, pattern, count);

    gird = new Gird(initialState);
    gird.run();
  });
</script>

</html>
```
