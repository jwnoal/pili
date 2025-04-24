---
title: 'unity URP 无法设置Transparency Sort Mode'
titleColor: '#aaa,#0acee9'
titleIcon: 'asset:unity'
tags: [ 'Untiy' ]
categories: [ 'Code' ]
description: '实现2D 按y轴排序进行层级叠放功能。'
publishDate: '2023-03-10'
updatedDate: '2023-03-10'
password: 'pilipal'
---

#### 普通管线设置
![](https://cdn.jiangwei.zone/blog/20250424113921915.png)

URP 使用脚本
```csharp
if UNITY_EDITOR
using UnityEditor;
#endif

using UnityEngine;
using UnityEngine.Rendering;

#if UNITY_EDITOR
[InitializeOnLoad]
#endif

class TransparencySortGraphicsHelper
{
    static TransparencySortGraphicsHelper()
    {
        OnLoad();
    }

    [RuntimeInitializeOnLoadMethod]
    static void OnLoad()
    {
        GraphicsSettings.transparencySortMode = TransparencySortMode.CustomAxis;
        GraphicsSettings.transparencySortAxis = new Vector3(0.0f, 1.0f, 0.0f);
    }
}
```