#100行代码教你实现类Webpack的JS打包器

### 前言

​	早期JavaScript只需要实现简单的页面交互，几行代码即可搞定。随着浏览器性能的提升以及前端技术的不断发展，JavaScript代码日益膨胀，此时就需要一个完善的模块化机制来解决这个问题。因此诞生了**CommonJS**(NodeJS), **AMD**(sea.js), **ES6 Module**(ES6, Webpack), **CMD**(require.js)等模块化规范。

**什么是模块化？**

> 模块化是一种处理复杂系统分解为更好的可管理模块的方式，用来分割，组织和打包软件。每一个模块完成一个特定的子功能，所有的模块按照某种方式组装起来，成为一个整体，完成整个系统的所有要求功能。

**模块化的好处是什么？**

1. 模块间解耦，提高模块的复用性。
2. 避免命名冲突。
3. 分离以及按需加载。
4. 提高系统的维护性。

随着Webpack的盛行，理解Webpack是如何将模块打包的也是前端人的基本素养，因此本文将从Webpack角度去实现一个类似于Webpack的JS模块打包器。

### JS模块化的演进

说到模块化，就不得不提一下JavaScript模块化的发展进程了。早期JavaScript模块化模式比较简单粗暴，将**一个模块定义为一个全局函数**

```javascript
function module1() {
  // code
}
function module2() {
	// code  
}
```

这种方案非常简单，但问题也很明显：**污染全局命名空间，引起命名冲突或数据不安全，而且模块间的依赖关系并不明显**。

在此基础上，又有了**namespace模式**，利用一个对象来对模块进行包装

```javascript
var module1 = {
  data: {  }, // 数据区域
  func1: function() {}
  func2: function() {}
}
```

这种方案的问题依然是数据不安全，外面能直接修改```module1```的```data```

因此又有了**IIFE模式，利用自执行函数（闭包）**

```javascript
!function(window) {
  var data = {};
  function func1() {
    data.hello = "hello";
  }
  function func2() {
    data.world = "world";
  }
  window.module1 = { func1, func2 };
} (window)
```

数据定义为私有，外部只能通过模块暴露的方法来对```data```进行操作，但这依然没有解决模块依赖的问题

基于IIFE，又提出了一种新的模块化方案，即在**IIFE的基础上引入了依赖**（现代模块化的基石，Webpack、NodeJS等模块化都是基于此实现的）

```javascript
!function (window, module2) {
  var data = {};
  function func1() {
    data.world = "world";
    module2.hello();
  }
  window.module1 = { func1 };
} (window, { hello: function() {}, });
```

这样使IIFE模块化的依赖关系变得更明显，又保证了IIFE模块化独有的特性。这种模块化方案也是本文JS模块打包器的模块化思路。

### 打包器设计思路

#### 一、原理

使用**引用依赖的IIFE模块化方案**，首先将每一个模块都封装成一个闭包函数，并传入```require```,```module```,```exports```参数

```javascript
function (require, module, exports) {
  // 模块化代码
  var module1 = require("module2");
  console.log(module1.add(1, 2));
  exports.sub = function (a, b) { return a - b }
}
```

并且使用一个```modules```对象来管理每个模块

```javascript
var modules = {
  "module1": [
    ["module2"], // dependencies，模块依赖数组
    function (require, module, exports) {
      // 模块化代码
      var module1 = require("module2");
      console.log(module1.add(1, 2));
      exports.sub = function (a, b) { return a - b }      
    }
  ],
  "module2": [
    [],
    function (require, module, exports) {
      exports.add = function (a, b) {
        return a + b
      }
    }
  ], 
}
```

这里的重点在于实现require函数来加载每一个模块

```javascript
function require(moduleId) {
  var deps = modules[moduleId][0]; // modules就是上面的modules，而moduleId就是上面的"module2" 
  var fn = modules[moduleId][1]; // 封装的闭包函数
  var module = { exports: {} };
  fn(require, module, module.exports); // 核心在这，将函数执行，并将require传进去
  return module.exports; 
}
```

稍微组织一下代码

```javascript
!function (modules) {
  function require (moduleId) {
    var deps = modules[moduleId][0];
    var fn = modules[moduleId][1];
    var module = { exports: {} };
    fn(require, module, module.exports);
    return module.exports;
  }
  // 从根模块依次加载
  require("module1"); // 假设module1是根模块
}({
  "module1": [
    ["module2"], // dependencies，依赖模块数组
    function (require, module, exports) {
      var module2 = require("module2");
      console.log("module2: ", module2.add(1, 2));
      exports.sub = function (a, b) {
        return a - b
      }
    }
  ],
  "module2": [
    [],
    function (require, module, exports) {
      exports.add = function (a, b) {
        return a + b;
      }
    }
  ],
});
```

运行如下

![image-20200104123108065](/Users/dengpengfei/Library/Application Support/typora-user-images/image-20200104123108065.png)

这就是本文的目标，将多个js文件打包成类似上面这样的模块化代码。

#### 二、准备工作

目录结构如下

​	mypack

​			I__  src // 打包目录

​					|__  tools

​							 |__  a.js

​							 |__  b.js

​					|__  func

​							 |__ func.js 

​					|__ add.js 

​					|__ index.js // 模块打包入口

​           |__ index.js	// 打包器代码

​	   	|__ config.js // 打包配置

在config.js中可以先写上打包配置

```javascript
const path = require("path");
module.exports = {
  entry: "src/index.js",
  output: {
    filename: "bundle.js",
    path: path.resolve(__dirname, "./dist")
  }
};
```

#### 三、从文件中解析出模块

每个文件使用ES6语法，并使用ES6模块化规范(import/exports)，引入和导出模块使用

```javascript
import { add } from "../add";
export default function func() { 
	console.log("func");
}
```

那么首先需要将ES6代码转码成ES5，并将"../add" import的相对路径解析出来。读者可能已经想到了，使用[babel](https://babeljs.io/)就能做到这件事，因此安装babel依赖

```bash
npm install --save @babel/core @babel/traverse @babel/preset-env
```

babel [core](https://babeljs.io/docs/en/babel-core)用于解析出抽象语法树ast并重新生成新的code，[traverse](https://babeljs.io/docs/en/babel-traverse)用于遍历抽象语法树

使用```transformFileAsync```来生成抽象语法树，其语法格式为

```javascript
transformFileAsync(filename: string, options?: Object)
```

传入js文件和[options](https://babeljs.io/docs/en/options)异步生成ast

```javascript
const { transformFileAsync } = require("@babel/core");

transformFileAsync("./src/index.js", { sourceType: "module", ast: true }).then((result) => {
  console.log(result.ast);
});
```

```ast: true```开启ast支持，默认为false，返回的结果中ast为null，```sourceType: "module"``` 表示使用ES6 module

sourceType可以是 "script" | "module" | "unambiguous"，默认其实是"module"

+ **"script"**: 使用正常的script标签里的js语法解析文件，没有import/export，并且不是严格模式
+ **"module"**: 使用ES6 module解析文件，自动为严格模式，支持import/export语法
+ **"unambiguous"**: babel根据文件中是否出现import/exports来确定是否处于"module"模式还是"script"模式

我们可以看到解析出来的ast长什么样

<img src="/Users/dengpengfei/Library/Application Support/typora-user-images/image-20200104134037434.png" alt="image-20200104134037434" style="zoom:40%;" />

console.log打印的不完全，建议去[astexplorer](https://astexplorer.net/)网站上去看

![image-20200104134324843](/Users/dengpengfei/Library/Application Support/typora-user-images/image-20200104134324843.png)

抽象语法树ast有了，那么只需要遍历这颗语法树，然后找到import语句的位置，并将import的文件找出来，细心的读者可能发现了，上图下面的箭头就指着不就是嘛？没错，这里可以使用babel [traverse](https://babeljs.io/docs/en/babel-traverse) 去遍历ast，traverse其实是根据对应的type来遍历ast的，type就是上图第一个箭头指着的，import表达式的type就是```ImportDeclaration```，在上面代码的基础上，将

```javascript
const { transformFileAsync } = require("@babel/core");
const traverser = require("@babel/traverse");

transformFileAsync("./src/index.js", { sourceType: "module", ast: true }).then((result) => {
  traverser.default(result.ast, {
    ImportDeclaration({ node }) { // import
      console.log(node.source.value); // 打印出import的文件
    }
  });
});
```

结果如下

<img src="/Users/dengpengfei/Library/Application Support/typora-user-images/image-20200104135549932.png" alt="image-20200104135549932" style="zoom:50%;" />

我们可以正常的从文件中解析出使用了某个模块(文件)，现在只需要将ES6的代码转回ES5就可以了

```javascript
const { transformFileAsync, transformFromAstAsync } = require("@babel/core");

transformFileAsync("./src/index.js", { sourceType: "module", ast: true }).then(({ast}) => {
  // 使用@babel/preset-env插件来转化代码
  transformFromAstAsync(ast, null, { presets: ["@babel/preset-env"], }).then(({code}) => {
    console.log(code);
  });
});
```

执行代码

<img src="/Users/dengpengfei/Library/Application Support/typora-user-images/image-20200104141117648.png" alt="image-20200104141117648" style="zoom:50%;" />

编译后的代码正好有个```require```函数，这也是上面实现的```require```函数。

整理了一下思路，现在稍微组织一下代码，将上面的内容封装成一个解析js文件的依赖和编译回ES5代码的函数，

```javascript
const { transformFileAsync, transformFromAstAsync } = require("@babel/core");
const traverser = require("@babel/traverse");

async function getDepsAndCode(filename) {
  const { ast } = await transformFileAsync(filename, { sourceType: "module", ast: true });
  const { code } = await transformFromAstAsync(ast, null, { presets: ["@babel/preset-env"] });
  const deps = [];
  traverser.default(ast, {
    ImportDeclaration({node}) {
      deps.push(node.source.value);
    }
  });
  return { code, deps }
}

async function main () {
  const { code, deps } = await getDepsAndCode("./src/index.js");
  console.log(code);
  console.log();
  console.log(deps);
}
main().catch(console.error);
```

运行结果如下

<img src="/Users/dengpengfei/Library/Application Support/typora-user-images/image-20200104142408666.png" alt="image-20200104142408666" style="zoom:50%;" />



#### 四、依赖图构建

读取一个文件，可以解析出它的依赖，接下来就是从入口依次构建依赖图，其实就是类似于上文中的modules的结构

```javascript
const graph = {
  "./src/index.js": { // 以文件绝对路径当作moduleId可以避免出现同名模块
    code,
    mapping: { // 相对路径到绝对路径的一个映射，因为使用的绝对路径当moduleId，但require时还是用相对路径
      "./add": "src/add"
    }
  },
  "src/add": {
    code,
    mapping: {}
  }
};
```

从入口文件递归（深搜）构建graph，在搜索时有个要点，就是要把当前的目录给记录下来，保证求绝对路径时能正确

```javascript
const path = require("path");
const config = require("./config");

function resolveJsFile(filename) {
  if (fs.existsSync(filename)) return filename;
  if (fs.existsSync(filename + ".js")) return filename + ".js";
  return filename;
}

async function makeDepsGraph(entry) {
  const graph = {};
  async function makeDepsGraph (filename) {
    if (graph[filename]) return; // 防止重复加载模块
    const mapping = {}; // 定义相对路径到绝对路径的一个映射
    const dirname = path.dirname(filename); // 注意保存上一个目录名，这样能找到模块的绝对路径
    const { code, deps } = await getDepsAndCode(resolveJsFile(filename));
    graph[filename] = { code }; // 解决循环依赖
    for (let dep of deps) {
      mapping[dep] = path.join(dirname, dep); // dep是相对路径，path.join(dirname, dep)是绝对路径
      await makeDepsGraph(mapping[dep]); // 深搜，不使用广搜
    }
    graph[filename].mapping = mapping;
  }
  await makeDepsGraph(entry);
  return graph;
}

async function main () {
  const graph = await makeDepsGraph(config.entry);
  console.log(graph);
}
main().catch(console.error);
```

执行结果

<img src="/Users/dengpengfei/Library/Application Support/typora-user-images/image-20200104151209168.png" alt="image-20200104151209168" style="zoom:50%;" />

#### 五、生成bundle.js

上一步构建出依赖图graph，接下来就是生成bundle代码，按照上面介绍的原理，我们可以根据graph生成modules

```javascript
let modules = "";
for (let filename of Object.keys(graph)) {
  modules += `'${ filename }': {
   mapping: ${ JSON.stringify(graph[filename].mapping) },
   fn: function (require, module, exports) {
     ${ graph[filename].code }
   }  
  },`
}
```

有了modules，在包裹上一个自执行函数即可生成bundle

还记得require函数嘛

```javascript
function require(moduleId) {
  var deps = modules[moduleId][0]; // modules就是上面的modules，而moduleId就是上面的"module2" 
  var fn = modules[moduleId][1]; // 封装的闭包函数
  var module = { exports: {} };
  fn(require, module, module.exports); // 核心在这，将函数执行，并将require传进去
  return module.exports; 
}
```

这里的require函数没有对模块进行缓存并且没有对循环依赖进行处理

> 循环依赖：即A依赖B，而B又依赖了A，举个例子
>
> a.js
>
> ```javascript
> import { b } from "./b";
> b();
> export function a() { console.log("a") }
> ```
>
> b.js
>
> ```javascript
> import { a } from "./a";
> a();
> export function b() { console.log("b") }
> ```

如果直接使用上面的require函数的话，会一直在这两个模块中来回require，直到栈溢出

因此，对require函数进行改进

```javascript
var cache = {}; // 缓存模块
var count = {}; // 模块计数，大于2表示存在循环引用
function require(moduleId) {
  if (cache[moduleId]) return cache[moduleId];
  count[moduleId] || (count[moduleId] = 0);
  count[moduleId] ++;
  
  var mapping = modules[moduleId].mapping;
  var fn = modules[moduleId].fn;
  function _require(id) { // id是相对路径
    var mId = mapping[id]; // 使用mapping映射为绝对路径
    if (count[mId] >= 2) return {}; // 循环引用返回空对象
    return require(mId);
  }
  var module = { exports: {} };
  fn(_require, module, module.exports);
  return module.exports;
}
```

因此bundle也变成了

```javascript
const bundle = `
   !function (modules) { 
      var cache = {}; 
      var count = {};
      function require(moduleId) {
        if (cache[moduleId]) return cache[moduleId];
        count[moduleId] || (count[moduleId] = 0);
        count[moduleId] ++;
    
        var mapping = modules[moduleId].mapping;
        var fn = modules[moduleId].fn;
        function _require(id) { 
          var mId = mapping[id]; 
          if (count[mId] >= 2) return {};
          return require(mId);
        }
    
        var module = { exports: {} };
        fn(_require, module, module.exports);
        return module.exports;
      }
      require('${entry}');
   } ({${modules}})`; 
```

整理一下代码，将bundle写入到相应的文件中，代码如下

```javascript
async function writeJsBundle (entry) {
  const graph = await makeDepsGraph(entry);
  let modules = "";
  for (let filename of Object.keys(graph)) {
    modules += `'${ filename }': {
     mapping: ${ JSON.stringify(graph[filename].mapping) },
     fn: function (require, module, exports) {
       ${ graph[filename].code }
     }  
    },`
  }

  const bundle = `
   !function (modules) {
      var cache = {}; 
      var count = {};
      function require(moduleId) {
        if (cache[moduleId]) return cache[moduleId];
        count[moduleId] || (count[moduleId] = 0);
        count[moduleId] ++;
    
        var mapping = modules[moduleId].mapping;
        var fn = modules[moduleId].fn;
        function _require(id) { 
          var mId = mapping[id]; 
          if (count[mId] >= 2) return {};
          return require(mId);
        }
    
        var module = { exports: {} };
        fn(_require, module, module.exports);
        return module.exports;
      }
      require('${ entry }');
    } ({${ modules }})`;
  await mkdir(config.output.path); 
  await writeFile(`${ config.output.path }/${ config.output.filename }`, bundle); 
}
```

### 完整代码

完整的代码如下，本人从不骗人，说好100行就100行😊😊😊

```javascript
const fs = require("fs");
const path = require("path");
const { promisify } = require("util");
const { transformFileAsync, transformFromAstAsync } = require("@babel/core");
const traverser = require("@babel/traverse");
const config = require("./config");
const mkOneDir = promisify(fs.mkdir);
const writeFile = promisify(fs.writeFile);

async function mkdir (dir) {
  const dirs = dir.split("/").filter(Boolean);
  let cur = "";
  for (let d of dirs) {
    cur += d;
    if (!fs.existsSync(cur)) await mkOneDir(cur);
    cur += "/"
  }
}

async function getDepsAndCode (filename) {
  const { ast } = await transformFileAsync(filename, { sourceType: "module", ast: true });
  const { code } = await transformFromAstAsync(ast, null, { presets: ["@babel/preset-env"] });
  const deps = [];
  traverser.default(ast, {
    ImportDeclaration ({ node }) { deps.push(node.source.value); }
  });
  return { code, deps }
}

function resolveJsFile (filename) {
  if (fs.existsSync(filename)) return filename;
  if (fs.existsSync(filename + ".js")) return filename + ".js";
  return filename;
}

async function makeDepsGraph (entry) {
  const graph = {};

  async function makeDepsGraph (filename) {
    if (graph[filename]) return; 
    const mapping = {}; 
    const dirname = path.dirname(filename);
    const { code, deps } = await getDepsAndCode(resolveJsFile(filename));
    graph[filename] = { code }; 
    for (let dep of deps) {
      mapping[dep] = path.join(dirname, dep); 
      await makeDepsGraph(mapping[dep]); 
    }
    graph[filename].mapping = mapping;
  }

  await makeDepsGraph(entry);
  return graph;
}

async function writeJsBundle (entry) {
  const graph = await makeDepsGraph(entry);
  let modules = "";
  for (let filename of Object.keys(graph)) {
    modules += `'${ filename }': {
     mapping: ${ JSON.stringify(graph[filename].mapping) },
     fn: function (require, module, exports) {
       ${ graph[filename].code }
     }  
    },`
  }

  const bundle = `
   !function (modules) {
      var cache = {}; 
      var count = {};
      
      function require(moduleId) {
        if (cache[moduleId]) return cache[moduleId];
        count[moduleId] || (count[moduleId] = 0);
        count[moduleId] ++;
        var mapping = modules[moduleId].mapping;
        var fn = modules[moduleId].fn;
        
        function _require(id) { 
          var mId = mapping[id]; 
          if (count[mId] >= 2) return {};
          return require(mId);
        }
        var module = { exports: {} };
        fn(_require, module, module.exports);
        return module.exports;
      }
      
      require('${ entry }');
    } ({${ modules }})`;
  await mkdir(config.output.path);
  await writeFile(`${ config.output.path }/${ config.output.filename }`, bundle);
}

async function main () {
  await writeJsBundle(config.entry);
}

main().catch(console.error);
```

### github地址

https://github.com/sundial-dreams/mypack

### 参考

**babel**: https://babeljs.io/

**前端模块化**: https://juejin.im/post/5c17ad756fb9a049ff4e0a62

**手写一个js打包器:** https://juejin.im/post/5e04c935e51d4557ea02c097



