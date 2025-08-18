---
title: "unity DO Tween"
titleColor: "#aaa,#20B2AA"
titleIcon: "asset:unity"
tags: ["Untiy"]
categories: ["Code"]
description: "unity DO Tween"
publishDate: "2023-03-10"
updatedDate: "2023-03-10"
---

#### 引用

```csharp
using DG.Tweening;
```

#### 设置节点高度动画

```csharp
rectTransform = GetComponent<RectTransform>();
rectTransform.DOSizeDelta(new Vector2(50f, 100f), 0.5f); // 动画
```
