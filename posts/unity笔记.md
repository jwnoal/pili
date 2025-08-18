---
title: "unity 笔记"
titleColor: "#aaa,#20B2AA"
titleIcon: "asset:unity"
tags: ["Untiy"]
categories: ["Code"]
description: "unity笔记，随便记记"
publishDate: "2023-03-10"
updatedDate: "2023-03-10"
---

#### 获取节点

```csharp
Transform woman = transform.Find("role_woman");
if (woman != null)
{
    woman.gameObject.SetActive(true);
}
```

#### 设置节点高度

```csharp
rectTransform = GetComponent<RectTransform>();
rectTransform.sizeDelta = new Vector2(50f, 800f);
// rectTransform.DOSizeDelta(new Vector2(50f, 100f), 0.5f); // 动画
```

#### Mask

父节点设置 Image 组件和 Mask 组件，子节点就可以被蒙版遮挡。
