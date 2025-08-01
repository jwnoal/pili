---
title: "Unity Spine"
titleColor: "#aaa,#0acee9"
titleIcon: "asset:unity"
tags: ["Untiy"]
categories: ["Code"]
description: "Unity Spine插件的使用"
publishDate: "2023-03-10"
updatedDate: "2023-03-10"
---

插件名：spine-unity

下载地址：[https://zh.esotericsoftware.com/spine-unity-download#%E4%B8%8B%E8%BD%BD](https://zh.esotericsoftware.com/spine-unity-download#%E4%B8%8B%E8%BD%BD)

下载对应版本的 unitypackage, 将包拖动到 Assets 中。

下载 UPM 包，解压后放入 Packages 文件夹中。  
然后在 Unity 中打开 Package Manager (Window > Package Manager), 选择 + 图标, 单击 Add package from disk..., 选择 package.json 文件.

美术给的文件有：

- 骨骼动画文件：xxx.json
- 贴图文件：xxx.atlas
- 图片文件：xxx.png

将文件放入 Assets/Resources/Spine 目录下。

**unity 无法识别 xxx.atlas 文件，打开此文件目录，重命名为 xxx.atlas.txt，回到 unity，会自动生成 Material 和 Atlas.asset 文件**。

点击 SkeletonData 文件，Inspector 面板中有 Atlas Asset 选项，选择 Atlas.asset 文件。

如果是在 UI 里面展示 Spine， 在节点处右键，Spine/SkeletonGraphic，将 SkeletonData 拖动到 SkeletonGraphic(Unity UI Canvas) SkeletonData Asset 中。

如果展示出的动画有不正常的描边， SkeletonGraphicDefault(Material) Shader 中勾选 Straight Alpha Texture 选项。

相关代码：

```unity
using System;
using DG.Tweening;
using Spine;
using Spine.Unity;
using UnityEngine;

public class GameManager : MonoBehaviour
{
    public GameObject man;
    public GameObject game;

    SkeletonGraphic skeletonGraphic;
    Spine.AnimationState animationState;

    void Awake()
    {
        skeletonGraphic = man.GetComponent<SkeletonGraphic>();
        animationState = skeletonGraphic.AnimationState;
        animationState.Complete += OnSpineAnimationComplete;
    }

    // Start is called before the first frame update

    void Start()
    {
    }

    // Update is called once per frame
    void Update() { }

    void playMan()
    {
        man.SetActive(true);
        // animationState.SetAnimation(0, "run", true);
    }

    public void OnSpineAnimationComplete(TrackEntry trackEntry)
    {
        // Add your implementation code here to react to complete events
        Debug.Log("Animation completed: " + trackEntry.Animation.Name);
    }

}

```
