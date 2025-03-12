---
title: "cocos处理webgl lost"
titleColor: "#aaa,#5d3bbf"
titleIcon: "asset:cocos"
tags: ["Cocos Creator"]
categories: ["Code"]
description: "cocos处理webgl lost"
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
