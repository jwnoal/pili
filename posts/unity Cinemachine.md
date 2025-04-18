---
title: 'unity Cinemachine'
titleColor: '#aaa,#20B2AA'
titleIcon: 'asset:unity'
tags: [ 'Untiy' ]
categories: [ 'Code' ]
description: 'Unity 虚拟相机的使用'
publishDate: '2023-03-10'
updatedDate: '2023-03-10'
---

一般场景中只会有一个Unity相机，但可以有多个Virtual Cameras    

### 安装
在 Package Manager 中搜索 Cinemachine，然后安装 Cinemachine 到项目中。

### 使用

#### 在Main Camra 上挂载组件 CinemachineBrain
![](https://cdn.jiangwei.zone/blog/20250418103034401.png)

#### CinemachineBrain 中的 Live Camera中需要绑定虚拟相机节点  

创建节点Virtual Camera，添加组件 CinemachineVirtualCamera, Cinemachine Confiner 2D
![](https://cdn.jiangwei.zone/blog/20250418103733789.png)

#### CinemachineVirtualCamera 中的 Follow 绑定跟随的目标物体，Look At 绑定相机的方向。   

创建节点Virtual Camera Follow

#### Cinemachine Confiner 2D 中的 Bounding Shape 2D 可以添加相机可移动区域  

创建节点Virtual Camera Collider 添加 Polygon Collider 2D组件
![](https://cdn.jiangwei.zone/blog/20250418103539517.png)

Edit Collider 可以设置区域   

#### 节点展示
![](https://cdn.jiangwei.zone/blog/20250418104426689.png)

移动Virtual Camera Follow节点位置即可移动相机

#### 禁止相机触边反弹
Cinemachine Confiner 2D -> Damping 选项可以设置反弹力度，设置为0即可。
