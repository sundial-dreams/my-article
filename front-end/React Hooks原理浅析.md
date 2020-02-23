## 揭开React Hooks神秘面纱

![wallhaven-565628](assets/React%20Hooks%E5%8E%9F%E7%90%86%E6%B5%85%E6%9E%90-assets/wallhaven-565628.png)

### React Hooks简介

React Hooks是React 16.8的新增特性，它可以让你在不编写类的情况下使用state和其他React功能。**hook翻译过来就是“钩子”，即在函数式组件中“<u>钩入</u>”React state以及生命周期等特性的函数**。一个简单的hook类似于

```javascript
function Count(props) {
  const [count, setCount] = useState(0); // 钩入state
  useEffect(() => { // 钩入组件生命周期
    console.log("component did mount");
  }, []);
  return (
  	<div onClick={() => setCount(v => v - 1)}>
      count: {count}
    </div>
  )
}
```



### Why React Hooks？

那么，为什么会有React Hooks呢，简而言之四个字：**逻辑复用**。React没用提供将**可复用代码**附加到组件中去的途径，对于这种问题一般都使用**render props**和**高阶组件**来解决。然而这类方案需要重新组织结构，而且很麻烦，写出来的代码像下面的那样，难以理解

```javascript
// redux connect高阶组件
export default connect(
  ({ reducer: { isLogin } }) => ({ isLogin })
)(Avatar);
```

而且当我们在使用Class在写组件时会把UI与逻辑混在Class里，导致逻辑复用困难。相信很多读者都碰到过逻辑一致而UI不一致的问题，至少在字节跳动实习期间，大部分的重复代码都是这种问题造成的。比如下面的UI

![image-20200223183612370](assets/React%20Hooks%E5%8E%9F%E7%90%86%E6%B5%85%E6%9E%90-assets/image-20200223183612370.png)

两个部分都有PK逻辑，使用Class来编写每一个组件，每个组件都得重新写一遍PK逻辑。由于后端提供的接口比较恶心，为了尽可能复用代码，对PK接口也进行了尽可能的封装：

**API.match: **设置定时器，每3s给后端push一个发起匹配的消息，并在匹配成功后```resolve```结果，匹配失败后```reject```。

**API.cancelMatch:** 清除定时器，并给后端push一个取消匹配的消息，取消成功```resolve```，取消失败（即在取消前一刻匹配成功了）```reject```。

```javascript
const PK_STATUS = {
  NO_MATCH: 0, // 不匹配
  MATCHING: 1, // 匹配中
  PK: 2 // 正在PK
};
// banner * 2
class PKBanner extends React.Component {
  constructor (props) {
    super(props);
    this.state = {
      status: PK_STATUS.NO_MATCH
    };
    this.handleMatch = this.handleMatch.bind(this);
    this.handleCancelMatch = this.handleCancelMatch.bind(this);
    this.autoCancelMatch = this.autoCancelMatch.bind(this);
  }

  handleMatch () { // 发起匹配
    this.setState({ status: PK_STATUS.MATCHING });
    API.match().then(data => this.setState({ status: PK_STATUS.PK })).catch(() => { // 出错情况
      toast("当前不能发起PK");
      this.setState({ status: PK_STATUS.NO_MATCH });
      API.cancelMatch();
    });
  }

  handleCancelMatch () { // 取消匹配
    API.cancelMatch().then(() => this.setState({ status: PK_STATUS.NO_MATCH }), data => { // 在取消时又匹配成功了
      this.setState({ status: PK_STATUS.PK });
    });
  }

  autoCancelMatch () { // 自动取消匹配
    if (this.state.status === PK_STATUS.MATCHING) {
      API.cancelMatch().then(() => {
        toast('匹配超时');
        this.setState({ status: PK_STATUS.NO_MATCH })
      }, data => { // 在取消匹配时匹配成功了
        this.setState({ status: PK_STATUS.PK });
      })
    }
  }

  render () {
    // do something
  }
}
```

这还是对接口进行了比较好的封装，如果把每3s给后端push匹配消息以及push取消匹配消息等逻辑直接往Class里面写可能会更多的重复代码。所以可以考虑把这种重复逻辑抽象成一个**useMatch hook**

```javascript
function useMatch () {
  const [status, setStatus] = useState(PK_STATUS.NO_MATCH);
  function match () {
    API.match().then(data => setStatus( PK_STATUS.PK)).catch(() => {
      toast("当前不能发起PK");
      setStatus(PK_STATUS.NO_MATCH );
      API.cancelMatch();
    });
  }
  function cancelMatch () {
    API.cancelMatch().then(() => setStatus( PK_STATUS.NO_MATCH ), data => {
      setStatus(PK_STATUS.PK);
    });
  }
  function autoCancelMatch () {
    if (status === PK_STATUS.MATCHING) {
      API.cancelMatch().then(() => {
        toast('匹配超时');
        setStatus(PK_STATUS.NO_MATCH)
      }, data => {
        setStatus(PK_STATUS.PK);
      })
    }
  }
  return [status, match, cancelMatch, autoCancelMatch];
}
```

然后在每一个需要使用PK逻辑的组件都能够将逻辑“钩入”组件中（也许这就是hooks的精髓）

```javascript
function PKBanner (props) {
  const [status, match, cancelMatch, autoCancelMatch] = useMatch();
  // do something
}
```



### 揭开React Hooks神秘面纱-原理浅析

其实React Hooks的原理和redux的实现有些相似（[redux原理浅析](https://juejin.im/post/5d0ccfd751882532d134beda)），都是手动```render```，比如```useState``` hook实现

```javascript
import ReactDOM from "react-dom";
import React from "react";

const MyReact = function () {
  function isFunc (fn) {
    return typeof fn === "function";
  }
  let state = null; // 全局变量保存state
  function useState (initState) {
    state = state || initState;
    function setState(newState) {
      // 保存新的state，注意setState里面可接个函数setState(v => v + 1);
      state = isFunc(newState) ? newState(state) : newState; 
      render(); // 手动调用render => ReactDOM.render => diff => 重构UI
    }
    return [state, setState];
  }

  return { useState }
}();

function App(props) {
  const [count, setCount] = MyReact.useState(0);
  return (
    <div onClick={() => setCount(count => count + 1)}>
      count: { count }
    </div>
  )
}

function render () {
  ReactDOM.render(<App/>, document.getElementById("root"));
}

render();

```

上面实现了一个简单的```useState``` hook，其基本原理并不复杂（理解```App```会一直调用）。当```App```执行时，调用```MyReact.useState(0)```将state初始为0（```state = state || initState```），并将```state```和```setState```返回，也就是```count```和```setCount```，当点击事件发生时，```state = newState```并且手动调用```render```，然后执行```App```，调用```MyReact.useState(0)```，上一个步骤```state = newState```将state赋值，因此```state = state || initState```得到的就是上一个的```state```，在UI上的变化就是```count + 1```了。

同理我们可以实现```useEffect``` hook，```useEffect```有几个特点：接受```callback```和```depArr```两个参数；当```depArr```不存在时```callback```会一直执行；```depArr```存在且和上一个```depArr```不相等时执行```callback```；

```javascript
let deps = null;
function useEffect (callback, depArr) {
  const hasDepsChange = deps ? deps.some((v, i) => depArr[i] !== v) : true; //检查依赖数组是否变化
  if (!depArr || hasDepsChange) { // depArr = undefined 或者 依赖数组变化时才调用callback
    deps = depArr; // 记录下上一个依赖数组
    isFunc(callback) && callback();
  }
}
```

每一次调用```useEffect```时都对依赖数组做一个判断，来判断```callback```是否需要执行，所以像下面的代码，依赖数组一直没变化，所以```callback```也就只会执行一次

```javascript
MyReact.useEffect(() => {
  console.log("count: ", count);
}, []);
```

上面的实现存在问题，即```state```只有一个，所以写两个hooks前一个会被覆盖，比如

```javascript
const [count, setCount] = MyReact.useState(0);
const [text, setText] = MyReact.useState("");
```

我们可以使用一个hooks数组来记录第几个hook，修改后的```useState```的实现

```javascript
let hooksOfState = []; // 记录state的数组
let hooksOfStateIndex = 0; // 当前索引
function useState (initState) {
  const currentIndex = hooksOfStateIndex; // 记录这是第几个useState
  hooksOfState[currentIndex] = hooksOfState[currentIndex] || initState;
  function setState(newState) { // 利用闭包的特性，这里的currentIndex就是对应编号的useState
    hooksOfState[currentIndex] = isFunc(newState) ? newState(hooksOfState[currentIndex]) : newState;
    render();
  }
  return [hooksOfState[hooksOfStateIndex++], setState]; // 记得索引++
}
```

而```useEffect```也是类似

```javascript
let hooksOfEffect = []; // 记录依赖数组
let hooksOfEffectIndex = 0; // 记录索引
function useEffect (callback, depArr) {
  const currentIndex = hooksOfEffectIndex;
  const hasDepsChange = !hooksOfEffect[currentIndex] || hooksOfEffect[currentIndex].some((v, i) => depArr[i] !== v);
  if (!depArr || hasDepsChange) {
    hooksOfEffect[currentIndex] = depArr; // 这两行顺序很重要，如果callback是一个setState的操作，导致render，可以避免出现栈溢出
    isFunc(callback) && callback();
  }
  hooksOfEffectIndex ++; // 索引++
}
```

需要注意在每一次```render```时都需要将索引设为0，保证```render```后还能按顺序找到对应的hooks

```javascript
function resetIndex () {
    hooksOfStateIndex = 0;
    hooksOfEffectIndex = 0;
}

function render () {
  MyReact.resetIndex(); // render时将索引设置为初始值
  ReactDOM.render(<App/>, document.getElementById("root"));
}
```

这里可以思考，为什么hooks不让写在条件分支或循环内，答案也很明显，如果写在条件分支或者循环内将无法保证下一次```render```时hooks的顺序。

```javascript
function App(props) {
  const [a, setA] = useState('a'); // 索引0
  if (condition) {
    const [b, setB] = useState('b'); // 假设条件执行，索引为1，如果在下一次render时条件不成立，则跳过这个hook
  }

  const [c, setC] = useState('c'); // 索引 2，假设render时上一个条件不成立，这里的索引将会变成1，导致发生意想不到的错误

  return <div> </div>;
}
```

其他hooks比如```useMemo```，```useCallback```的实现也是类似

```javascript
let hooksOfMemo = []; // {deps: , result: }
let hooksOfMemoIndex = 0;
function useMemo (callback, depArr) {
  const currentIndex = hooksOfMemoIndex;
  const hasDepsChange = !hooksOfMemo[currentIndex] || hooksOfMemo[currentIndex].deps.some((v, i) => depArr[i] !== v); // 
  if (!depArr || hasDepsChange) {
    hooksOfMemo[currentIndex] = {
      deps: depArr,
      result: isFunc(callback) && callback()
    };
  }
  hooksOfMemoIndex ++;
  return hooksOfMemo[currentIndex].result;
}

let hooksOfCallback = []; // {deps: , callback}
let hooksOfCallbackIndex = 0;
function useCallback (callback, depArr) {
  const currentIndex = hooksOfCallbackIndex;
  const hasDepsChange = !hooksOfCallback[currentIndex] || hooksOfCallback[currentIndex].deps.some((v, i) => depArr[i] !== v);
  if (!depArr || hasDepsChange) {
    hooksOfCallback[currentIndex] = {
      deps: depArr,
      callback: callback
    };
  }
  hooksOfCallbackIndex ++;
  return hooksOfCallback[currentIndex].callback;
}
```

当然上面的实现并不是React中Hooks的真正实现，它的实现比这要复杂很多，比如React中是通过类似单链表的形式来代替数组的，通过next按顺序串联所有的hook。并且hooks的数据作为组件的一个信息，存储在这些节点上，伴随组件整个生命周期。

### 完整代码

```javascript
import ReactDOM from "react-dom";
import React from "react";

const MyReact = function () {
  function isFunc (fn) {
    return typeof fn === "function";
  }

  let hooksOfState = [];
  let hooksOfStateIndex = 0;

  function useState (initState) {
    const currentIndex = hooksOfStateIndex;
    hooksOfState[currentIndex] = hooksOfState[currentIndex] || initState;

    function setState (newState) {
      hooksOfState[currentIndex] = isFunc(newState) ? newState(hooksOfState[currentIndex]) : newState;
      render();
    }

    return [hooksOfState[hooksOfStateIndex ++], setState];
  }

  let hooksOfEffect = [];
  let hooksOfEffectIndex = 0;

  function useEffect (callback, depArr) {
    const currentIndex = hooksOfEffectIndex;
    const hasDepsChange = !hooksOfEffect[currentIndex] || hooksOfEffect[currentIndex].some((v, i) => depArr[i] !== v);
    if (!depArr || hasDepsChange) {
      hooksOfEffect[currentIndex] = depArr;
      isFunc(callback) && callback();
    }
    hooksOfEffectIndex ++;
  }

  let hooksOfMemo = []; // { deps, result }
  let hooksOfMemoIndex = 0;

  function useMemo (callback, depArr) {
    const currentIndex = hooksOfMemoIndex;
    const hasDepsChange = !hooksOfMemo[currentIndex] || hooksOfMemo[currentIndex].deps.some((v, i) => depArr[i] !== v);
    if (!depArr || hasDepsChange) {
      hooksOfMemo[currentIndex] = {
        deps: depArr,
        result: isFunc(callback) && callback()
      };
    }
    hooksOfMemoIndex ++;
    return hooksOfMemo[currentIndex].result;
  }

  let hooksOfCallback = []; // {deps, callback}
  let hooksOfCallbackIndex = 0;

  function useCallback (callback, depArr) {
    const currentIndex = hooksOfCallbackIndex;
    const hasDepsChange = !hooksOfCallback[currentIndex] || hooksOfCallback[currentIndex].deps.some((v, i) => depArr[i] !== v);
    if (!depArr || hasDepsChange) {
      hooksOfCallback[currentIndex] = {
        deps: depArr,
        callback: callback
      };
    }
    hooksOfCallbackIndex ++;
    return hooksOfCallback[currentIndex].callback;
  }

  function resetIndex () {
    hooksOfStateIndex = 0;
    hooksOfEffectIndex = 0;
    hooksOfMemoIndex = 0;
    hooksOfCallbackIndex = 0;
  }

  return { resetIndex, useEffect, useState, useMemo, useCallback }
}();

function App (props) {
  const [count, setCount] = MyReact.useState(0);
  const [text, setText] = MyReact.useState("count");
  MyReact.useEffect(() => {
    console.log("count: ", count);
  }, [count]);
  MyReact.useEffect(() => {
    console.log("changed");
    // setText("change start");
  }, []);
  const sum = MyReact.useMemo(function () {
    let result = 0;
    for (let i = 0; i < count; i ++) result += i;
    return result;
  }, [count]);
  const fn = MyReact.useCallback(() => 20 + count, [count]);
  return (
    <div onClick={ () => setCount(count => count + 1) }>
      <div>{ text }: { count } </div>
      <div>sum: { sum }, const: { fn() }</div>
    </div>
  )
}

function render () {
  MyReact.resetIndex();
  ReactDOM.render(<App/>, document.getElementById("root"));
}

render();

```

### 参考

[React Hooks文档](https://zh-hans.reactjs.org/docs/hooks-overview.html)

[Deep dive: How do React hooks really work?](https://www.netlify.com/blog/2019/03/11/deep-dive-how-do-react-hooks-really-work/)