---
title: "判断浏览器是否支持webgl"
titleColor: "#aaa,#0ae9ad"
titleIcon: "asset:markdown"
tags: ["webgl"]
categories: ["Code"]
description: "判断浏览器是否支持webgl"
publishDate: "2025-03-10"
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
    console.log(e);
  },
  false
);
```

#### 模拟 webgl lost

```js
const contexts = ["webgl2", "webgl", "experimental-webgl"];
let gl = null;

for (const type of contexts) {
  gl = canvas.getContext(type);
  if (gl) break;
}

if (!gl) {
  console.log("该浏览器不支持WebGL");
} else {
  gl.getExtension("WEBGL_lose_context").loseContext();
}
```
