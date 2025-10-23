---
title: "cocos spine 换装"
titleColor: "#aaa,#5d3bbf"
titleIcon: "asset:cocos"
tags: ["Cocos Creator"]
categories: ["Code"]
description: "cocos spine 换装"
publishDate: "2023-03-12"
updatedDate: "2023-03-12"
password: "1231234"
---

#### ChangeSkin

```js
import { _decorator, Component, Node, sp, Texture2D, resources } from 'cc';
const { ccclass, property } = _decorator;

@ccclass('ChangeSkin')
export class ChangeSkin extends Component {
    @property(sp.Skeleton)
    skin1Skel: sp.Skeleton = null!;

    private skinIdx1: number = 0;

    start() {
        // 3.8.4 不需要重写 updateRegion
    }

    protected async onChangeSkin1() {
        if (this.skinIdx1 === 0) {
            this.skinIdx1 = 1;
            await this.modifyAttachment(this.skin1Skel, 'a/clothes_a');
        } else {
            this.skinIdx1 = 0;
            await this.modifyAttachment(this.skin1Skel, '');
        }
    }

    private async modifyAttachment(player: sp.Skeleton, imgName: string) {
        if (imgName) {
            const tex2d = await this.loadTex2d(imgName);
            (player as any).setSlotTexture('clothes2', tex2d, true);
        } else {
            (player as any).setSlotTexture('clothes2', null);
        }
    }

    private loadTex2d(name: string): Promise<Texture2D | null> {
        return new Promise((resolve) => {
            resources.load(`children/${name}/texture`, Texture2D, (err, tex2d) => {
                if (err) {
                    console.error(`loadTex2d ${name} error:`, err);
                    resolve(null);
                } else {
                    resolve(tex2d);
                }
            });
        });
    }
}
```
