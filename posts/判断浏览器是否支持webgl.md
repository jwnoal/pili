---
title: "判断浏览器是否支持webgl"
titleColor: "#aaa,#0ae9ad"
titleIcon: "asset:markdown"
tags: ["webgl"]
categories: ["Code"]
description: "判断浏览器是否支持webgl"
publishDate: "2023-03-12"
updatedDate: "2023-03-12"
---

#### 检测 webgl 是否可用

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

#### 监听 webgl lost

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

#### 模拟 webgl lost

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
  var lost = gl.getExtension("WEBGL_lose_context"); // 使用 var 变量提升
  lost.loseContext();
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

#### 模拟 webgl restored

```js
// 必须先使用前面模拟webgl lost方法
lost.restoreContext();
```
