---
title: "unity 定时管理类"
titleColor: "#aaa,#20B2AA"
titleIcon: "asset:unity"
tags: ["Untiy"]
categories: ["Code"]
description: "unity 定时管理类"
publishDate: "2023-03-10"
updatedDate: "2023-03-10"
---

```csharp
using System;
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class TimeManager : MonoBehaviour
{
    // 使用字典保存 Coroutine 和其执行器，以便快速查找和管理
    private static Dictionary<Coroutine, CoroutineExecutor> coroutineExecutors =
        new Dictionary<Coroutine, CoroutineExecutor>();

    // 辅助类，用于执行 Coroutine 并管理其状态
    private class CoroutineExecutor : MonoBehaviour
    {
        public bool IsPaused { get; private set; } = false;

        public void Pause()
        {
            IsPaused = true;
        }

        public void Resume()
        {
            IsPaused = false;
        }
    }

    /// <summary>
    /// 在指定延迟后执行一个 Action。
    /// </summary>
    /// <param name="delay">延迟秒数。</param>
    /// <param name="onComplete">延迟后执行的 Action。</param>
    /// <returns>返回 Coroutine 引用，可用于手动停止、暂停和恢复。</returns>
    public static Coroutine Delay(float delay, Action onComplete)
    {
        if (delay < 0)
        {
            Debug.LogError("延迟时间不能为负数！");
            return null;
        }

        GameObject executorObject = new GameObject("TimeManagerExecutor");
        DontDestroyOnLoad(executorObject);
        CoroutineExecutor executor = executorObject.AddComponent<CoroutineExecutor>();

        IEnumerator coroutineRoutine = DelayCoroutine(delay, onComplete, executorObject, executor);
        Coroutine coroutine = executor.StartCoroutine(coroutineRoutine);

        coroutineExecutors.Add(coroutine, executor);

        return coroutine;
    }

    private static IEnumerator DelayCoroutine(
        float delay,
        Action onComplete,
        GameObject executorObject,
        CoroutineExecutor executor
    )
    {
        float startTime = Time.time;
        while (Time.time < startTime + delay)
        {
            while (executor.IsPaused)
            {
                yield return null;
            }
            yield return null;
        }

        onComplete?.Invoke();
        Cleanup(executorObject);
    }

    /// <summary>
    /// 每隔一段时间重复执行一个 Action，直到达到指定时长或次数。
    /// </summary>
    /// <param name="interval">重复的间隔秒数。</param>
    /// <param name="duration">持续的总时长（可选）。</param>
    /// <param name="count">重复的次数（可选）。</param>
    /// <param name="onRepeat">每次重复执行的 Action。</param>
    /// <returns>返回 Coroutine 引用，可用于手动停止、暂停和恢复。</returns>
    public static Coroutine Repeat(
        float interval,
        Action onRepeat,
        float duration = -1,
        int count = -1
    )
    {
        if (interval <= 0)
        {
            Debug.LogError("重复间隔必须大于0！");
            return null;
        }

        GameObject executorObject = new GameObject("TimeManagerExecutor");
        DontDestroyOnLoad(executorObject);
        CoroutineExecutor executor = executorObject.AddComponent<CoroutineExecutor>();

        IEnumerator coroutineRoutine = RepeatCoroutine(
            interval,
            duration,
            count,
            onRepeat,
            executorObject,
            executor
        );
        Coroutine coroutine = executor.StartCoroutine(coroutineRoutine);

        coroutineExecutors.Add(coroutine, executor);

        return coroutine;
    }

    private static IEnumerator RepeatCoroutine(
        float interval,
        float duration,
        int count,
        Action onRepeat,
        GameObject executorObject,
        CoroutineExecutor executor
    )
    {
        float timer = 0f;
        int currentCount = 0;

        while (true)
        {
            while (executor.IsPaused)
            {
                yield return null;
            }

            if ((duration > 0 && timer >= duration) || (count > 0 && currentCount >= count))
            {
                break;
            }

            onRepeat?.Invoke();
            currentCount++;
            timer += interval;
            yield return new WaitForSeconds(interval);
        }

        Cleanup(executorObject);
    }

    /// <summary>
    /// 等待某个条件为真时，再执行一个 Action。
    /// </summary>
    /// <param name="predicate">等待的条件，返回 true 时继续执行。</param>
    /// <param name="onComplete">条件满足时执行的 Action。</param>
    /// <returns>返回 Coroutine 引用，可用于手动停止、暂停和恢复。</returns>
    public static Coroutine WaitUntil(Func<bool> predicate, Action onComplete)
    {
        return WaitUntil(predicate, -1f, onComplete, null);
    }

    /// <summary>
    /// 等待某个条件在指定时间内为真，否则超时。
    /// </summary>
    /// <param name="predicate">等待的条件，返回 true 时继续执行。</param>
    /// <param name="timeout">超时时间（秒）。</param>
    /// <param name="onComplete">条件满足时执行的 Action。</param>
    /// <param name="onTimeout">超时时执行的 Action。</param>
    /// <returns>返回 Coroutine 引用，可用于手动停止、暂停和恢复。</returns>
    public static Coroutine WaitUntil(
        Func<bool> predicate,
        float timeout,
        Action onComplete,
        Action onTimeout
    )
    {
        if (predicate == null)
        {
            Debug.LogError("等待条件不能为空！");
            return null;
        }

        GameObject executorObject = new GameObject("TimeManagerExecutor");
        DontDestroyOnLoad(executorObject);
        CoroutineExecutor executor = executorObject.AddComponent<CoroutineExecutor>();

        IEnumerator coroutineRoutine = WaitUntilCoroutine(
            predicate,
            timeout,
            onComplete,
            onTimeout,
            executorObject,
            executor
        );
        Coroutine coroutine = executor.StartCoroutine(coroutineRoutine);

        coroutineExecutors.Add(coroutine, executor);

        return coroutine;
    }

    private static IEnumerator WaitUntilCoroutine(
        Func<bool> predicate,
        float timeout,
        Action onComplete,
        Action onTimeout,
        GameObject executorObject,
        CoroutineExecutor executor
    )
    {
        float startTime = Time.time;
        bool conditionMet = false;

        // 循环直到条件满足或超时
        while (!conditionMet && (timeout < 0 || Time.time - startTime < timeout))
        {
            while (executor.IsPaused)
            {
                yield return null;
            }

            if (predicate())
            {
                conditionMet = true;
            }
            yield return null;
        }

        // 根据结果执行不同的回调
        if (conditionMet)
        {
            onComplete?.Invoke();
        }
        else
        {
            onTimeout?.Invoke();
        }

        Cleanup(executorObject);
    }

    /// <summary>
    /// 手动停止一个由 TimeManager 启动的 Coroutine。
    /// </summary>
    public static void Stop(Coroutine coroutine)
    {
        if (coroutine == null)
            return;
        if (coroutineExecutors.TryGetValue(coroutine, out CoroutineExecutor executor))
        {
            executor.StopCoroutine(coroutine);
            Cleanup(executor.gameObject);
        }
    }

    /// <summary>
    /// 暂停一个由 TimeManager 启动的 Coroutine。
    /// </summary>
    public static void Pause(Coroutine coroutine)
    {
        if (coroutine == null)
            return;
        if (coroutineExecutors.TryGetValue(coroutine, out CoroutineExecutor executor))
        {
            executor.Pause();
        }
    }

    /// <summary>
    /// 恢复一个由 TimeManager 启动的 Coroutine。
    /// </summary>
    public static void Resume(Coroutine coroutine)
    {
        if (coroutine == null)
            return;
        if (coroutineExecutors.TryGetValue(coroutine, out CoroutineExecutor executor))
        {
            executor.Resume();
        }
    }

    /// <summary>
    /// 停止所有由 TimeManager 启动的 Coroutine。
    /// </summary>
    public static void StopAll()
    {
        foreach (var executor in coroutineExecutors.Values)
        {
            if (executor != null && executor.gameObject != null)
            {
                executor.StopAllCoroutines();
                Destroy(executor.gameObject);
            }
        }
        coroutineExecutors.Clear();
    }

    // 清理资源：从字典中移除并销毁执行器
    private static void Cleanup(GameObject executorObject)
    {
        if (executorObject != null)
        {
            CoroutineExecutor executor = executorObject.GetComponent<CoroutineExecutor>();
            if (executor != null)
            {
                List<Coroutine> keysToRemove = new List<Coroutine>();
                foreach (var entry in coroutineExecutors)
                {
                    if (entry.Value == executor)
                    {
                        keysToRemove.Add(entry.Key);
                    }
                }
                foreach (var key in keysToRemove)
                {
                    coroutineExecutors.Remove(key);
                }
            }
            Destroy(executorObject);
        }
    }
}

// 每隔 1 秒重复 5 次
// myRepeatedTask = TimeManager.Repeat(1f, () => {
//     Debug.Log("我在重复执行...");
// }, count: 5);

// 等待条件满足
// bool isReady = false;
// TimeManager.WaitUntil(
//     () => isReady,
//     5f, // -1 表示不设置超时
//     () =>
//     {
//         Debug.Log("<b><color=green>示例1：条件已满足，任务完成！</color></b>");
//     },
//     () =>
//     {
//         Debug.Log("<b><color=red>示例1：任务超时！</color></b>");
//     }
// );

```
