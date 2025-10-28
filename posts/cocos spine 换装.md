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

#### 外部资源换装

```js
import { _decorator, Component, Node, sp, Texture2D, resources } from 'cc';
const { ccclass, property } = _decorator;

@ccclass('ChangeSkin')
export class ChangeSkin extends Component {
    @property(sp.Skeleton)
    skin1Skel: sp.Skeleton = null!;

    private skinIdx1: number = 0;

    start() {
        this.modifyAttachment('头发_前发cloth_hairF_00/B_cloth_01_hairF_00', "cloth_hairF_00");
    }

    protected async onChangeSkin2() {
        if (this.skinIdx1 === 0) {
            this.skinIdx1 = 1;
            await this.modifyAttachment('cloth_shoesL_00/B_cloth_01_shoesL_00', "cloth_shoesL_00");
        } else {
            this.skinIdx1 = 0;
            await this.modifyAttachment("", "cloth_shoesL_00");
        }
    }

    private async modifyAttachment(imgName: string, slot: string) {
        if (imgName) {
            const tex2d = await this.loadTex2d(imgName);
            if (tex2d) {
                this.skin1Skel.setSlotTexture(slot, tex2d, true);
            }
        } else {
            let tex2d = new Texture2D();
            this.skin1Skel.setSlotTexture(slot, tex2d, true);
        }
    }

    private loadTex2d(name: string): Promise<Texture2D | null> {
        return new Promise((resolve) => {
            resources.load(`yifu/${name}/texture`, Texture2D, (err, tex2d) => {
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

#### 皮肤叠加换装

```ts
import { _decorator, Component, sp } from "cc";
const { ccclass, property } = _decorator;

@ccclass("SpineCustomizer")
export class SpineCustomizer extends Component {
  @property(sp.Skeleton)
  public skeleton: sp.Skeleton = null!; // 引用你的Spine节点

  // 用于存储当前穿戴的皮肤部件名称
  private currentOutfit: Map<string, string> = new Map();

  start() {
    // 初始化时可以设置一套默认服装
    this.changeBodyPart("accessories", "accessories/acc1");
    this.changeBodyPart("white", "white");
    this.changeBodyPart("face", "face/face1");
    this.changeBodyPart("hair", "hair/hair6");
    this.changeBodyPart("shirt", "shirt/shirt2");
    this.changeBodyPart("shoe", "shoe/shoe9");
  }

  /**
   * 更换身体部位的皮肤
   * @param slotName 插槽名称，如 'head', 'body'
   * @param skinName 皮肤名称，如在Spine中设置的 'head/headB'
   */
  changeBodyPart(slotName: string, skinName: string) {
    if (!this.skeleton) {
      console.error("Skeleton component is missing!");
      return;
    }

    // 方法1：直接使用setSkin（推荐，性能更好）
    this._setSkinDirectly(slotName, skinName);

    // 方法2：使用组合皮肤（如果需要同时显示多个皮肤部件）
    this._buildCombinedSkin();
  }

  /**
   * 方法1：直接设置皮肤（推荐）
   */
  private _setSkinDirectly(slotName: string, skinName: string) {
    // 获取 Spine 运行时对象
    const spine = this.skeleton["_skeleton"];
    if (!spine) {
      console.error("Spine skeleton not found!");
      return;
    }

    // 查找皮肤
    const newSkin = spine.data.findSkin(skinName);
    if (!newSkin) {
      console.error(`Skin not found: ${skinName}`);
      return;
    }

    // 更新当前装备记录
    this.currentOutfit.set(slotName, skinName);

    // 设置皮肤
    spine.setSkin(newSkin);
    spine.setSlotsToSetupPose();

    // 更新动画状态
    this._updateAnimationState();
  }

  /**
   * 方法2：构建组合皮肤（如果需要组合多个皮肤部件）
   */
  private _buildCombinedSkin() {
    const spine = this.skeleton["_skeleton"];
    if (!spine) {
      console.error("Spine skeleton not found!");
      return;
    }

    // 创建新的组合皮肤
    const combinedSkin = new sp.spine.Skin("combined-skin");

    // 遍历所有当前穿戴的部件，将它们添加到组合皮肤中
    this.currentOutfit.forEach((skinName) => {
      const skinPart = spine.data.findSkin(skinName);
      if (skinPart) {
        combinedSkin.addSkin(skinPart);
      }
    });

    // 设置组合皮肤
    spine.setSkin(combinedSkin);
    spine.setSlotsToSetupPose();

    // 更新动画状态
    this._updateAnimationState();
  }

  /**
   * 更新动画状态
   */
  private _updateAnimationState() {
    if (!this.skeleton) return;

    // 获取当前动画状态并重新应用
    const currentTrack = this.skeleton.getCurrent(0);
    if (currentTrack) {
      // 重新应用当前动画以保持流畅
      this.skeleton.setAnimation(
        0,
        currentTrack.animation.name,
        currentTrack.loop
      );
    }
  }

  /**
   * 重置到默认皮肤
   */
  resetToDefaultSkin() {
    const spine = this.skeleton["_skeleton"];
    if (spine) {
      spine.setToSetupPose();
      spine.setSlotsToSetupPose();
      this.currentOutfit.clear();
      this._updateAnimationState();
    }
  }

  /**
   * 获取当前穿搭配置 (可用于保存)
   */
  // getCurrentOutfit(): Record<string, string> {
  //     return Object.fromEntries(this.currentOutfit);
  // }

  /**
   * 加载一套已保存的穿搭配置
   * @param outfitConfig 穿搭配置对象
   */
  // loadOutfit(outfitConfig: Record<string, string>) {
  //     this.currentOutfit.clear();
  //     for (const [slotName, skinName] of Object.entries(outfitConfig)) {
  //         this.changeBodyPart(slotName, skinName);
  //     }
  // }
}
```
