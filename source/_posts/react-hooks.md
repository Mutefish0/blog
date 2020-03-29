---
title: React Hooks之useState的原理剖析
date: 2020-03-29 09:54:51
tags: react hooks useState
---
在最近的项目在项目中使用了React Hooks进行开发。Hooks把状态、副作用与组件渲染逻辑进行了解耦，可以比较方便实现代码复用，同时替代了声明周期函数，使得代码大大简化。

## 使用Hooks实现一个简单的计数器组件
```jsx
import React, { useState } from 'react';

export default function App() {
  const [counter, setCounter] = useState(1);

  return (
    <div>
      <p>counter: {counter}</p>
      <hr/>
      <button onClick={() => setCounter(counter + 1)}>add</button>
    </div>
  );
}
```

CodeSandbox
<iframe
     src="https://codesandbox.io/embed/aged-browser-s934g?fontsize=14&hidenavigation=1&theme=dark"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="aged-browser-s934g"
     allow="geolocation; microphone; camera; midi; vr; accelerometer; gyroscope; payment; ambient-light-sensor; encrypted-media; usb"
     sandbox="allow-modals allow-forms allow-popups allow-scripts allow-same-origin"
   ></iframe>

### 疑问
useState这个useState它是如何保存状态的？假如useState是一个纯函数，那么每次渲染的时候，都会执行`useState(1)`，那么，每次返回的`[counter, setCounter]`应该不变才对，所以useState一定有副作用。首先，`counter`这个状态一定保存在`App`这个函数之外，其次，`setCounter`能够更新这个状态，同时能够强制`App`组件的重渲染。我们来尝试实现它。

### 实现一个模拟的强制刷新
```jsx
import React, { useState as reactUseState } from "react";

let forceUpdate;

function Wrapper(Component) {
  return () => {
    const [, setValue] = reactUseState();
    forceUpdate = () => setValue(Math.random());
    return <Component />;
  }
}

function App() {
  const now = Date.now();
  return (
    <div>
      <span>now: {now}</span>
      <hr/>
      <button onClick={forceUpdate}>force update</button>
    </div>
  );
}


export default Wrapper(App);

```

CodeSandbox
<iframe
     src="https://codesandbox.io/embed/mystifying-tereshkova-3qiw7?fontsize=14&hidenavigation=1&theme=dark"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="mystifying-tereshkova-3qiw7"
     allow="geolocation; microphone; camera; midi; vr; accelerometer; gyroscope; payment; ambient-light-sensor; encrypted-media; usb"
     sandbox="allow-modals allow-forms allow-popups allow-scripts allow-same-origin"
   ></iframe>

### 实现一个useState
```jsx
import React, { useState as reactUseState } from "react";

let forceUpdate;
function Wrapper(Component) {
  return () => {
    const [, setValue] = reactUseState();
    forceUpdate = () => setValue(Math.random());
    return <Component />;
  };
}

// 保存状态
let _state = null;
function useState(defaultValue) {
  if (_state == null) {
    _state = defaultValue;
  }
  return [
    _state,
    nextState => {
      _state = nextState;
      forceUpdate();
    }
  ];
}

function App() {
  const [counter, setCounter] = useState(1);

  return (
    <div>
      counter: {counter}
      <hr />
      <button onClick={() => setCounter(counter + 1)}>add</button>
    </div>
  );
}

export default Wrapper(App);

```

CodeSandbox
<iframe
     src="https://codesandbox.io/embed/optimistic-lewin-cpcvv?fontsize=14&hidenavigation=1&theme=dark"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="optimistic-lewin-cpcvv"
     allow="geolocation; microphone; camera; midi; vr; accelerometer; gyroscope; payment; ambient-light-sensor; encrypted-media; usb"
     sandbox="allow-modals allow-forms allow-popups allow-scripts allow-same-origin"
   ></iframe>

### 整理代码
我们在同目录下新一个文件`my-hooks.js`，把Wrapper、useState的代码拆进去
```js
import React, { useState as reactUseState } from "react";

let forceUpdate;
export function Wrapper(Component) {
  return () => {
    const [, setValue] = reactUseState();
    forceUpdate = () => setValue(Math.random());
    return <Component />;
  };
}

// 保存状态
let _state = null;
export function useState(defaultValue) {
  if (_state == null) {
    _state = defaultValue;
  }
  return [
    _state,
    nextState => {
      _state = nextState;
      forceUpdate();
    }
  ];
}
```
## 多个计数器
我们来尝试额外增加一个计数器

### 先用原生的`useState`实现
```jsx
import React, { useState } from "react";

export default function App() {
  const [counter1, setCounter1] = useState(1);
  const [counter2, setCounter2] = useState(1);

  return (
    <div>
      <p>counter1: {counter1}</p>
      <button onClick={() => setCounter1(counter1 + 1)}>add</button>
      <hr />
      <p>counter2: {counter2}</p>
      <button onClick={() => setCounter2(counter2 + 1)}>add</button>
    </div>
  );
}
```
可以看到，两个状态正常work，互相不影响。
CodeSandbox
<iframe
     src="https://codesandbox.io/embed/summer-bash-3fu29?fontsize=14&hidenavigation=1&theme=dark"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="summer-bash-3fu29"
     allow="geolocation; microphone; camera; midi; vr; accelerometer; gyroscope; payment; ambient-light-sensor; encrypted-media; usb"
     sandbox="allow-modals allow-forms allow-popups allow-scripts allow-same-origin"
   ></iframe>

### 尝试用我们造的`useState`轮子实现
```jsx
import React from "react";
import { Wrapper, useState } from "./my-hooks";

function App() {
  const [counter1, setCounter1] = useState(1);
  const [counter2, setCounter2] = useState(1);

  return (
    <div>
      <p>counter1: {counter1}</p>
      <button onClick={() => setCounter1(counter1 + 1)}>add</button>
      <hr />
      <p>counter2: {counter2}</p>
      <button onClick={() => setCounter2(counter2 + 1)}>add</button>
    </div>
  );
}

export default Wrapper(App);
```
可以明显发现，在这种场景下，之前的实现不Work了。两个状态混到一起去了，互相之前有影响。那么怎么把这两个状态区分开来呢？
参考[这篇文章](https://dev.to/wuz/linked-lists-in-the-wild-react-hooks-3ep8)发现，React为每个组件维护了一个链表结构，通过链表结构来一一对应不同的状态。我们尝试来实现。
修改`my-hooks.js`
```js
import React, { useState as reactUseState } from "react";

// 一个空的头结点
let _stateNodeHead = { state: "", next: null };
let _currentStateNode = _stateNodeHead;

let forceUpdate;
export function Wrapper(Component) {
  return () => {
    const [, setValue] = reactUseState();
    forceUpdate = () => setValue(Math.random());
    // 重新渲染之前，重置指针到链表头部
    _currentStateNode = _stateNodeHead;
    return <Component />;
  };
}

export function useState(defaultValue) {
  // 尝试移动节点，如果不存在则创建一个
  if (!_currentStateNode.next) {
    _currentStateNode.next = {
      state: defaultValue,
      next: null
    };
  }

  const node = _currentStateNode.next;
  // 移动节点
  _currentStateNode = node;

  return [
    node.state,
    value => {
      // 改变当前节点的状态
      node.state = value;
      // 刷新
      forceUpdate();
    }
  ];
}
```
CodeSandbox
<iframe
     src="https://codesandbox.io/embed/lucid-mayer-n3mv5?fontsize=14&hidenavigation=1&theme=dark"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="lucid-mayer-n3mv5"
     allow="geolocation; microphone; camera; midi; vr; accelerometer; gyroscope; payment; ambient-light-sensor; encrypted-media; usb"
     sandbox="allow-modals allow-forms allow-popups allow-scripts allow-same-origin"
   ></iframe>
从结果可以看到，完美实现了功能

## 渲染多个平级组件
我们尝试新增一个平级的组件
```jsx
import React from "react";
import { Wrapper, useState } from "./my-hooks";


function App() {
  const [counter1, setCounter1] = useState(1);
  const [counter2, setCounter2] = useState(2);

  return (
    <div>
      <p>counter1: {counter1}</p>
      <button onClick={() => setCounter1(counter1 + 1)}>add</button>
      <hr />
      <p>counter2: {counter2}</p>
      <button onClick={() => setCounter2(counter2 + 1)}>add</button>
    </div>
  );
}

function MainApp() {
  return (
  <div>
    <h2>App 1</h2>
    <App />
    <h2>App 2</h2>
    <App />
  </div>
  );
}

export default Wrapper(MainApp);

```
CodeSandbox
<iframe
     src="https://codesandbox.io/embed/gallant-tree-lfhu2?fontsize=14&hidenavigation=1&theme=dark"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="gallant-tree-lfhu2"
     allow="geolocation; microphone; camera; midi; vr; accelerometer; gyroscope; payment; ambient-light-sensor; encrypted-media; usb"
     sandbox="allow-modals allow-forms allow-popups allow-scripts allow-same-origin"
   ></iframe>

可以发现平级的是没问题的。

## 渲染嵌套的组件
```js
import React from "react";
import { Wrapper, useState } from "./my-hooks";

function App() {
  const [counter1, setCounter1] = useState(1);
  const [counter2, setCounter2] = useState(2);

  return (
    <div>
      <p>counter1: {counter1}</p>
      <button onClick={() => setCounter1(counter1 + 1)}>add</button>
      <hr />
      <p>counter2: {counter2}</p>
      <button onClick={() => setCounter2(counter2 + 1)}>add</button>
    </div>
  );
}

function MainApp() {
  const [counter1, setCounter1] = useState(1);
  const [counter2, setCounter2] = useState(2);

  return (
    <div>
      <h2>App 1</h2>
      <App />
      <h2>App 2</h2>
      <div>
        <p>counter1: {counter1}</p>
        <button onClick={() => setCounter1(counter1 + 1)}>add</button>
        <hr />
        <p>counter2: {counter2}</p>
        <button onClick={() => setCounter2(counter2 + 1)}>add</button>
      </div>
    </div>
  );
}

export default Wrapper(MainApp);
```
CodeSandbox
<iframe
     src="https://codesandbox.io/embed/competent-water-v8r5r?fontsize=14&hidenavigation=1&theme=dark"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="competent-water-v8r5r"
     allow="geolocation; microphone; camera; midi; vr; accelerometer; gyroscope; payment; ambient-light-sensor; encrypted-media; usb"
     sandbox="allow-modals allow-forms allow-popups allow-scripts allow-same-origin"
   ></iframe>

   可以发现，嵌套的渲染也是work的

## 条件渲染
```js
import React from "react";
import { Wrapper, useState } from "./my-hooks";

function App() {
  const [counter1, setCounter1] = useState(1);
  const [counter2, setCounter2] = useState(2);

  return (
    <div>
      <p>counter1: {counter1}</p>
      <button onClick={() => setCounter1(counter1 + 1)}>add</button>
      <hr />
      <p>counter2: {counter2}</p>
      <button onClick={() => setCounter2(counter2 + 1)}>add</button>
    </div>
  );
}

function MainApp() {
  const [showApp1, setShowApp1] = useState(false);

  return (
    <div>
      <h2>App 1</h2>
      {showApp1 && <App />}
      <h2>App 2</h2>
      <App />
      <hr />
      <button onClick={() => setShowApp1(!showApp1)}>toggle</button>
    </div>
  );
}

export default Wrapper(MainApp);
```

CodeSandbox
<iframe
     src="https://codesandbox.io/embed/eloquent-cdn-ptzyf?fontsize=14&hidenavigation=1&theme=dark"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="eloquent-cdn-ptzyf"
     allow="geolocation; microphone; camera; midi; vr; accelerometer; gyroscope; payment; ambient-light-sensor; encrypted-media; usb"
     sandbox="allow-modals allow-forms allow-popups allow-scripts allow-same-origin"
   ></iframe>

可以发现，这里出现了串数据的现象。从这里我们可以分析原因，因为不管是任意的平级或者嵌套的组件渲染，都不会改变调用useState的顺序，所以他们总是能够正确得对应上相关的状态。但是条件渲染，则改变了它的顺序，所以出现了串数据的情况。
注意到，我们实现的setState是存储在全局的一个链表，也就是说所有组件的状态都共用这一个链表。如果每个组件，都单独占用一个独立的链表，问题就可以解决了。但是，有什么办法才能知道当前渲染的组件对应哪一个链表呢？参考[这边文章](https://indepth.dev/inside-fiber-in-depth-overview-of-the-new-reconciliation-algorithm-in-react/)发现，React Fiber算法正式可以解决这个问题，因为每次执行渲染函数的时候，都能够知道当前函数所对应的组件信息，那么如果我们把setState所需要的链表信息在这里面，每次渲染之前取一下，就可以完美实现了。

由于React Fiber算法又是一个比较大的主题，这次就不展开了。下次有时间再专门写一篇React Fiber的博客。