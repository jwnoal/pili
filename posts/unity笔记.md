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

#### 设置定位

注意要使用 localPosition

```csharp
feijian.GetComponent<RectTransform>().localPosition = new Vector2(x, 649f);
```

#### Mask

父节点设置 Image 组件和 Mask 组件，子节点就可以被蒙版遮挡。

#### 脚本执行顺序

在 Unity 编辑器中，打开 Edit→ Project Settings→ Script Execution Order

#### 层级

ui 如果没有 Order in Layer，可以试试添加 Canvas 组件，并勾选 Override Sorting  
粒子需要在他 Rendering 中改变 Sorting Layer  
spine 使用 skeletonAnimation，可以设置 Sorting Layer
