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

#### 动画使用

```csharp
// 高度动画
rectTransform = GetComponent<RectTransform>();
rectTransform.DOSizeDelta(new Vector2(50f, 100f), 0.5f).SetLink(gameObject); // 动画


// 位置动画
obj.transform.DOMove(new Vector3(5, 0, 0), 1f);
obj.transform.DOLocalMove(new Vector3(2, 0, 0), 1f);

// 旋转动画
obj.transform.DORotate(new Vector3(0, 90, 0), 1f);
obj.transform.DOLocalRotate(new Vector3(0, 180, 0), 1f);

// 缩放动画
obj.transform.DOScale(new Vector3(2, 2, 2), 1f);
obj.transform.DOScale(0.5f, 1f); // 统一缩放

// 透明
damageText.DOFade(0f, 0.2f);

// 更改颜色
damageText.DOColor(Color.red, "_Color", 1f);

// 延迟和回调
obj.transform.DOMoveX(5, 1f)
    .SetDelay(2f) // 延迟2秒
    .OnStart(() => Debug.Log("动画开始"))
    .OnComplete(() => Debug.Log("动画完成"))
    .OnUpdate(() => Debug.Log("动画更新"));

// 序列动画
Sequence sequence = DOTween.Sequence();

// 添加动画到序列
sequence.Append(transform.DOMoveX(5, 1f));        // 顺序执行
sequence.Join(transform.DOScale(2, 1f));          // 同时执行
sequence.AppendInterval(1f);                       // 间隔
sequence.Prepend(transform.DOMoveY(2, 0.5f));     // 在开头添加

sequence.Play(); // 播放序列

// 序列控制
Sequence sequence = DOTween.Sequence();
sequence
    .Append(transform.DOMoveX(5, 1f))
    .SetAutoKill(false)    // 不自动销毁
    .SetRecyclable(true);  // 可回收

sequence.Play();
sequence.Pause();
sequence.Rewind();
sequence.TogglePause();
```

#### 延时器使用

```csharp
Tween myDelay = DOVirtual.DelayedCall(
    2.0f,
    () =>
    {
        Debug.Log("2秒后执行的操作");
    }
);

myDelay.Pause(); // 暂停
myDelay.Play(); // 继续
myDelay.Kill(); // 取消

// 每2秒执行一次
DOVirtual.DelayedCall(2f, () => Debug.Log("每2秒执行一次")).SetLoops(-1, LoopType.Restart); // 无限循环

// 延迟一帧执行
DOVirtual.DelayedCall(
    0,
    () =>
    {
        Debug.Log("下一帧执行");
    }
);

// 条件满足后执行
bool condition = false;
DOVirtual.DelayedCall(
    0.1f,
    () =>
    {
        if (condition)
        {
            Debug.Log("条件满足");
        }
        else
        {
            // 条件不满足，重新检查
            DOVirtual.DelayedCall(0.1f, () => CheckCondition());
        }
    }
);
```

#### 对象池使用

```csharp
// 初始化时配置（推荐在游戏启动时调用）
DOTween.Init(
    recycleAllByDefault: true, // 所有Tween默认会被回收
    useSafeMode: true,         // 安全模式检测错误
    logBehaviour: LogBehaviour.ErrorsOnly
);

Tween myTween = transform.DOMoveX(5, 1f)
    .SetAutoKill(false) // 设置为false，这样Tween完成后不会被自动销毁
    .SetRecyclable(true); // 设置为可回收

// 使用完成后手动回收
myTween.OnComplete(() => {
    if(myTween != null && myTween.active) {
        myTween.Recycle();
    }
});

DOTween.Clear(); // 清空所有 Tween，包括对象池中的
DOTween.ClearCachedTweens(); // 只清空对象池中的 Tween

// 获取当前对象池中缓存的 Tween 数量
int cachedTweens = DOTween.TotalCachedTweens();
```
