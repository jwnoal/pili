---
title: "Unity ECS配置"
titleColor: "#aaa,#0acee9"
titleIcon: "asset:unity"
tags: ["Untiy"]
categories: ["Code"]
description: "Unity ECS下载及配置"
publishDate: "2023-03-10"
updatedDate: "2023-03-10"
password: "1231234"
---

#### 安装 Entity

package manager 安装套件 Entities 和 Entities Graphics

- Add package by name (com.unity.entities.graphics)

#### 设置

Edit -> Project Settings -> Editor -> Enter Play Mode Options // 编译速度比较快 Reload Domain、Reload Scene 不勾选
unity -> Preferences/settings -> Entities -> Scene View Mode 改为 Runtime Data

#### 设置 URP

Project Settings -> Graphics -> 点击 URP-HighFidelity, 跳转目录找到 URP-HighFidelity-Renderer 后点击 -> Rendering -> Rendering Path 选择 Forward+

#### 查看 entity

Window -> Entities -> Entiteies Hierarchy // 显示所有 entity

#### 创建 EntityScene

右键 New Sub Scene -> Empty Scene

#### 确定启用 Burst

Jobs -> Burst -> Enable Burst Compilation

#### gpu 动画插件

GPU ECS Animation Baker

#### 导航插件

Agent Navigation
