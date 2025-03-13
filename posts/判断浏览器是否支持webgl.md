---
title: "判断浏览器是否支持WebGL"
titleColor: "#aaa,#0ae9ad"
titleIcon: "asset:markdown"
tags: ["浏览器"]
categories: ["Code"]
description: "判断浏览器是否支持WebGL以及模拟WebGL 上下文丢失和恢复"
publishDate: "2023-03-12"
updatedDate: "2023-03-12"
---

#### 检测 WebGL 是否可用

```js
const canvas = document.getElementById("GameCanvas");
const contexts = ["webgl2", "webgl", "experimental-webgl"];
let gl = null;

for (const type of contexts) {
  gl = canvas.getContext(type);
  if (gl) break;
}

if (!gl) {
  console.log("该浏览器不支持WebGL");
}
```

#### 监听 WebGL webglcontextlost

```js
const canvas = document.getElementById("GameCanvas");
canvas.addEventListener(
  "webglcontextlost",
  function (e) {
    console.log("webglcontextlost", e);
  },
  false
);
```

#### 模拟 WebGL lost

```js
const canvas = document.getElementById("GameCanvas");
const contexts = ["webgl2", "webgl", "experimental-webgl"];
let gl = null;

for (const type of contexts) {
  gl = canvas.getContext(type);
  if (gl) break;
}

if (!gl) {
  console.log("该浏览器不支持WebGL");
} else {
  var lose_context = gl.getExtension("WEBGL_lose_context"); // 使用 var 变量提升
  lose_context.loseContext();
}
```

#### 监听 webglcontextrestored

```js
const canvas = document.getElementById("GameCanvas");
canvas.addEventListener(
  "webglcontextrestored",
  (e) => {
    console.log("webglcontextrestored", e);
  },
  false
);
```

#### 模拟 WebGL restored

```js
// 必须先使用前面模拟webgl lost方法
lose_context.restoreContext();
```
