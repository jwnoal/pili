---
title: "cocos引擎WebGL上下文丢失与恢复处理机制"
titleColor: "#aaa,#5d3bbf"
titleIcon: "asset:cocos"
tags: ["Cocos Creator"]
categories: ["Code"]
description: "cocos引擎WebGL上下文丢失与恢复处理机制"
publishDate: "2023-03-12"
updatedDate: "2023-03-12"
---

#### 创建脚本并挂在到根节点

```js
import { _decorator, Component, director, game } from "cc";
const { ccclass, property } = _decorator;

@ccclass("HandleWebGLLoss")
export class HandleWebGLLoss extends Component {
  onLoad() {
    const canvas = game.canvas;

    canvas.addEventListener(
      "webglcontextlost",
      (event) => {
        event.preventDefault();
        console.log("Context Lost");
        director.pause();
      },
      false
    );

    canvas.addEventListener(
      "webglcontextrestored",
      () => {
        console.log("Context Restored");
        // director.resume();
        location.reload();
      },
      false
    );
  }
}
```

模拟 WBGL 上下文丢失于恢复：[地址](https://pili.run/posts/%E5%88%A4%E6%96%AD%E6%B5%8F%E8%A7%88%E5%99%A8%E6%98%AF%E5%90%A6%E6%94%AF%E6%8C%81webgl/)
