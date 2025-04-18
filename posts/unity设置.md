---
title: 'Unity 设置'
titleColor: '#aaa,#20B2AA'
titleIcon: 'asset:unity'
tags: [ 'Untiy' ]
categories: [ 'Code' ]
description: 'Unity 的一些设置'
publishDate: '2023-03-10'
updatedDate: '2023-03-10'
---

#### 设置shadow distance
Edit -> Project Settings -> Graphics 点击 URP-HighFidelity，会出现文件，点击文件，Ispector 面板中，设置 Shadow / Max Distance 

#### 解决图片锯齿
勾选图片文件的 Generate Mip Maps 选项
点击Main Camera，在Inspector面板中，设置 Rendering -> Anti-aliasing 选项为：SMAA 或 FXAA

#### 查看视椎剔除
Windows -> Rendering -> OcclusionCulling 打开Occlusion面板
新建cube 设置为static, 在Occlusion面板中，点击Bake按钮，生成Occlusion数据
需要选中Occlusion面板中的Visualization, 然后在Scene视图中拖动物体或相机，就能看到效果