---
title: 'Unity ECS配置'
titleColor: '#aaa,#0acee9'
titleIcon: 'asset:unity'
tags: [ 'Untiy' ]
categories: [ 'Code' ]
description: 'Unity ECS下载及配置'
publishDate: '2023-03-10'
updatedDate: '2023-03-10'
password: 'pili'
---

#### 安装Entity
package manager 安装套件 Entities 和 Entities Graphics
- Add package by name (com.unity.entities.graphics)

#### 设置
Edit -> Project Settings -> Editor -> Enter Play Mode Options // 编译速度比较快 Reload Domain、Reload Scene不勾选
unity -> Preferences/settings -> Entities -> Scene View Mode 改为 Runtime Data

#### 设置URP
Project Settings -> Graphics -> 点击URP-HighFidelity, 跳转目录找到 URP-HighFidelity-Renderer 后点击 -> Rendering -> Rendering Path 选择Forward+

#### 查看entity
Window -> Entities -> Entiteies Hierarchy // 显示所有 entity

#### 创建 EntityScene
右键 New Sub Scene -> Empty Scene

#### gpu动画插件
GPU ECS Animation Baker

#### 导航插件
Agent Navigation