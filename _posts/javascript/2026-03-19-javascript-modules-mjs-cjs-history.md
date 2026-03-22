---
title: "JavaScript 模块演进史：从脚本标签到 .mjs 与 .cjs"
date: 2026-03-19 14:00:00 +0800
categories: [JavaScript, Web]
tags: [ES Modules, CommonJS, mjs, cjs, Node.js, 模块化]
---

当我们今天在 Node.js 中看到 `.mjs` 和 `.cjs` 两种文件扩展名时，背后其实是一段跨越二十年的模块化探索史。JavaScript 从最初「没有模块」的脚本语言，一路演进到拥有官方标准模块系统，其间经历了多个相互竞争的方案。理解这段历史，不仅能解释为什么会有 `.mjs` 和 `.cjs` 的区分，更能帮助我们理解现代前端工具链（如 Webpack、Vite、Rollup）在解决什么问题。

## 🕰️ 史前时代：没有模块（1995–2009）

JavaScript 诞生于 1995 年，最初被设计为在浏览器中嵌入 HTML 页面、处理简单交互的脚本语言。在很长一段时间里，**「模块」根本不是一个语言层面的概念**。

### 脚本标签的「模块」：全局污染

早期的代码组织方式极其朴素：多个 `<script>` 标签按顺序加载，所有变量默认挂在全局对象（`window`）上。

```html
<script src="utils.js"></script>
<script src="app.js"></script>
```

这带来了几个问题：
*   **命名冲突**：两个文件都定义了 `getData()`，后者覆盖前者。
*   **依赖顺序敏感**：必须把被依赖的脚本放在前面，顺序错乱就会报错。
*   **没有封装**：任何代码都能读写全局变量，无法做真正的模块隔离。

这一时期，开发者只能用 **IIFE（立即执行函数表达式）** 和**命名空间对象**来人为制造「伪模块」，避免污染全局。但这不是语言标准，只是约定俗成的写法。

## 🏗️ 奠基时代：CommonJS 与 Node.js（2009 年）

2009 年，Ryan Dahl 发布 Node.js，把 JavaScript 带到了服务端。服务端需要读取文件、依赖本地模块，而浏览器的那套「脚本标签 + 全局变量」在 Node 中完全不适用。于是，Node.js 采用了 **CommonJS** 作为其默认模块系统。

### CommonJS 的核心：require 与 module.exports

CommonJS 由 Mozilla 工程师 Kevin Dangoor 等人于 2009 年初提出，旨在为 JavaScript 在浏览器外运行提供一个标准的模块 API。其核心语法非常简单：

```javascript
// utils.cjs（历史上是 .js，Node 默认按 CJS 解析）
function add(a, b) {
  return a + b;
}
module.exports = { add };

// main.cjs
const { add } = require('./utils.cjs');  // .cjs 需显式写出扩展名，require 不会自动解析
console.log(add(1, 2)); // 3
```

*   **`require()`**：同步加载模块，在模块首次被引用时执行，结果会被缓存。
*   **`module.exports`**：定义模块对外暴露的内容。
*   **加载方式**：同步、阻塞式。在服务端读本地文件可以接受，但在浏览器中会卡住整个页面。

Node.js 的迅速崛起，让 CommonJS 成了事实上的**服务端 JavaScript 模块标准**。时至今日，npm 生态中仍有大量包使用或兼容 CommonJS 格式，`.cjs` 扩展名就是为了明确标识「这是一个 CommonJS 模块」。

## ⚔️ 分叉时代：浏览器与 AMD/UMD（2010–2015）

Node.js 有文件系统，可以同步 `require`；浏览器没有文件系统，且需要异步加载脚本。于是，**浏览器端**催生了另一套模块方案。

### AMD（Asynchronous Module Definition）

AMD 由 Kris Zyp 提出规范，由 James Burke 的 RequireJS 实现并推广，专为**异步加载**设计：

```javascript
// 定义模块
define(['dependency1', 'dependency2'], function (dep1, dep2) {
  return { myFunction: function () { /* ... */ } };
});

// 使用模块
require(['myModule'], function (myModule) {
  myModule.myFunction();
});
```

AMD 适合浏览器，但语法繁琐，且与 Node.js 的 CommonJS 不兼容。于是出现了 **UMD（Universal Module Definition）**：通过运行时检测环境，同一份代码既可以 `define`（AMD），也可以 `module.exports`（CommonJS），甚至暴露为全局变量。UMD 的本质是一层兼容胶水，让一个库同时支持多种环境。

这一时期，JavaScript 世界形成了**双轨制**：Node 用 CommonJS，浏览器用 AMD 或 UMD，两者无法直接互通。

## 🌟 标准化时代：ES Modules 的诞生（2015–2020）

转机来自 ECMAScript 2015（ES6）。TC39 在语言规范层面引入了 **ES Modules**，这是 JavaScript 历史上**第一种官方标准的模块系统**。

### ES Modules 的核心：import 与 export

```javascript
// math.mjs
export function add(a, b) {
  return a + b;
}

// main.mjs
import { add } from './math.mjs';
console.log(add(1, 2)); // 3
```

与 CommonJS 相比，ES Modules 有几个重要区别：

| 特性         | CommonJS              | ES Modules                    |
| :----------- | :-------------------- | :---------------------------- |
| 加载时机     | 运行时、同步          | 静态分析、可预解析            |
| Tree Shaking  | 难以实现              | 原生支持，利于打包优化        |
| 循环依赖     | 支持但行为复杂        | 规范明确定义，行为可预测      |
| 顶层 await   | 不支持                | 支持（ES2022）                |
| 浏览器原生   | 不支持                | 支持（`<script type="module">`）|

ES Modules 是**静态的**：`import` 和 `export` 在语法解析阶段就能确定依赖关系，因此打包工具可以安全地做死代码消除（Tree Shaking）。而 CommonJS 的 `require()` 可以是动态的（如 `require(variable)`），工具难以静态分析。

### 从规范到实现：漫长的支持之路

*   **浏览器**：2017 年起，主流浏览器陆续支持 `<script type="module">`（Chrome 61、Safari 10.1、Firefox 60 等）。
*   **Node.js**：2017 年 9 月，Node 8.5.0 在 `--experimental-modules` 下支持 `.mjs`；Node 12（2019 年 4 月）起支持 `"type": "module"`；Node 13（2019 年 10 月）起不再强制要求 `--experimental-modules` 标志。但 Node 的默认仍是 CommonJS，以保持向后兼容。
*   **扩展名问题**：在 Node 中，同一目录下既有 CommonJS 又有 ES Modules 时，仅靠 `package.json` 的 `"type": "module"` 无法区分单个 `.js` 文件。于是，**`.mjs` 和 `.cjs`** 应运而生。

## 📦 .mjs 与 .cjs：扩展名的语义化（2017 年至今）

### 为什么需要明确的扩展名？

在 Node.js 中：
*   未设置 `"type": "module"` 时，`.js` 按 **CommonJS** 解析。
*   设置 `"type": "module"` 后，`.js` 按 **ES Modules** 解析。

这会导致一个问题：如果你想在同一个项目中**混用**两种格式（例如迁移期的遗留代码），仅凭 `.js` 无法判断该文件是哪种模块类型。Node 需要一个**不受 `package.json` 影响的、基于文件本身的约定**。

### .mjs 与 .cjs 的语义

| 扩展名 | 含义                     | 解析方式                          |
| :----- | :----------------------- | :-------------------------------- |
| `.mjs` | **M**odule JavaScript    | 始终按 ES Modules 解析            |
| `.cjs` | **C**ommonJS JavaScript  | 始终按 CommonJS 解析              |

使用扩展名后：
*   `.mjs` 文件一定使用 `import`/`export`，无需 `"type": "module"`。
*   `.cjs` 文件一定使用 `require`/`module.exports`，即使项目设置了 `"type": "module"` 也不会被误解析。
*   Node 的 `require()` 仅自动解析 `.js`、`.json`、`.node`，**不包含 `.cjs`**，因此 `require('./foo.cjs')` 必须写全扩展名；而 ESM 的 `import` 在引用本地文件时也需写明扩展名（如 `'./foo.mjs'`）。

这样，**文件格式由扩展名决定，与项目配置解耦**，混用两种格式时不会产生歧义。

### 实践建议

*   **新项目**：优先使用 ES Modules（`.mjs` 或 `"type": "module"` + `.js`），享受 Tree Shaking、静态分析和未来语言特性（如顶层 await）。
*   **旧项目迁移**：可保留 `.cjs` 作为过渡，逐步将关键路径改为 `.mjs`。
*   **库作者**：同时发布 ESM 和 CJS 构建（dual package），通过 `package.json` 的 `exports` 字段让打包工具自动选择合适格式。

## 总结

JavaScript 模块的演进史，就是一部**从「无模块」到「多标准并存」再到「官方标准逐渐统一」**的历史：

1.  **史前**：脚本标签 + 全局变量，无模块概念。
2.  **Node 时代**：CommonJS 成为服务端事实标准，`.cjs` 是其明确标识。
3.  **浏览器分叉**：AMD、UMD 满足异步加载，但与 Node 生态割裂。
4.  **标准化**：ES6 引入 ES Modules，成为语言官方标准，`.mjs` 是其明确标识。
5.  **共存期**：`.mjs` 与 `.cjs` 通过扩展名明确语义，使 Node 能够安全地混用两种格式，支撑漫长的迁移与过渡。

理解这段历史，不仅能解释「为什么有 .mjs 和 .cjs」，更能帮助我们理性看待当前工具链中的各种模块格式配置——它们都是历史演进留下的痕迹，而未来终将逐步收敛到 ES Modules 为主。
