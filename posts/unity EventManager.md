---
title: "unity EventManager"
titleColor: "#aaa,#20B2AA"
titleIcon: "asset:unity"
tags: ["Untiy"]
categories: ["Code"]
description: "unity EventManager"
publishDate: "2023-03-10"
updatedDate: "2023-03-10"
---

#### EventSystem 使用

```csharp
// 注册事件监听

this.RegisterEvent(EventType.OnPlayerDamaged, OnDamaged); // 使用EventHelper（推荐，简化MonoBehaviour操作）

EventManager.Instance.RegisterListener(EventType.OnPlayerDamaged, this, OnDamaged); // 直接使用EventManager

// 触发事件
this.TriggerEvent(EventType.OnPlayerDamaged, 50); // 带数据
this.TriggerEvent(EventType.OnGameStart); // 无数据

EventManager.Instance.TriggerEvent(EventType.OnPlayerDamaged, 50);

// 异步触发事件
await this.TriggerEventAsync(EventType.OnPlayerDamaged, 50);
await EventManager.Instance.TriggerEventAsync(EventType.OnPlayerDamaged, 50);

// 注销事件监听
this.UnregisterEvent(EventType.OnPlayerDamaged);
this.UnregisterAllEvents();

// 添加事件拦截器
this.AddInterceptor((eventType, data) =>
{
    if (eventType == EventType.OnPlayerDamaged && data is int damage)
    {
        Debug.Log($"拦截到伤害事件: {damage}");
        return damage < 10; // 小于10的伤害不传播
    }
    return true; // 其他事件继续传播
});
// 移除拦截器：
this.RemoveInterceptor(MyInterceptorFunction);

// 查看事件统计（编辑器下）
EventManager.Instance.PrintEventStatistics();
EventManager.Instance.ResetEventStatistics();
```

#### EventSystem

```csharp
using System;
using System.Collections.Concurrent;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using UnityEngine;

namespace EventSystem
{
    /// <summary>
    /// 事件类型枚举
    /// </summary>
    public enum EventType
    {
        // 通用事件
        OnGameStart,
        OnGamePause,
        OnGameResume,
        OnGameEnd,

        // UI事件
        OnButtonClick,
        OnSliderValueChanged,
        OnToggleChanged,

        // 游戏事件
        OnPlayerDamaged,
        OnPlayerDeath,
        OnEnemyKilled,
        OnLevelUp,

        // 自定义事件
        CustomEvent1,
        CustomEvent2,
        CustomEvent3,
    }

    /// <summary>
    /// 事件中心 - 单例管理类
    /// </summary>
    public class EventManager : MonoBehaviour
    {
        private static EventManager _instance;

        // 事件监听字典（使用ConcurrentDictionary以支持线程安全）
        private ConcurrentDictionary<EventType, List<EventListener>> _eventListeners =
            new ConcurrentDictionary<EventType, List<EventListener>>();

        // **修复：添加缺失的事件拦截器**
        private event Func<EventType, object, bool> OnEventIntercept;

        // 事件统计
        private ConcurrentDictionary<EventType, int> _eventStatistics =
            new ConcurrentDictionary<EventType, int>();

        // 调试模式
        [SerializeField]
        private bool debugMode = false;

        [SerializeField]
        private bool enableStatistics = true;

        // 锁对象，用于线程安全
        private readonly object _lock = new object();

        // 对象池：复用监听器列表，避免GC
        private Queue<List<EventListener>> _listenerPool = new Queue<List<EventListener>>();

        public static EventManager Instance
        {
            get
            {
                if (_instance == null)
                {
                    GameObject go = new GameObject("EventManager");
                    _instance = go.AddComponent<EventManager>();
                    DontDestroyOnLoad(go);
                }
                return _instance;
            }
        }

        void Awake()
        {
            if (_instance != null && _instance != this)
            {
                Destroy(gameObject);
                return;
            }

            _instance = this;
            DontDestroyOnLoad(gameObject);

            // 初始化对象池（预分配一些列表）
            for (int i = 0; i < 10; i++)
            {
                _listenerPool.Enqueue(new List<EventListener>(10));
            }
        }

        /// <summary>
        /// 获取或创建事件监听列表（懒加载）
        /// </summary>
        private List<EventListener> GetListeners(EventType eventType)
        {
            return _eventListeners.GetOrAdd(eventType, _ => new List<EventListener>());
        }

        /// <summary>
        /// 从池借用列表
        /// </summary>
        private List<EventListener> BorrowList()
        {
            return _listenerPool.Count > 0 ? _listenerPool.Dequeue() : new List<EventListener>(10);
        }

        /// <summary>
        /// 归还列表到池
        /// </summary>
        private void ReturnList(List<EventListener> list)
        {
            list.Clear();
            _listenerPool.Enqueue(list);
        }

        /// <summary>
        /// 注册事件监听
        /// </summary>
        public void RegisterListener(
            EventType eventType,
            object listener,
            Action<object> callback,
            bool autoUnregister = true,
            int priority = 0,
            bool oneShot = false
        )
        {
            if (listener == null || callback == null)
            {
#if UNITY_EDITOR
                Debug.LogWarning("注册事件监听失败：监听器或回调为空");
#endif
                return;
            }

            lock (_lock)
            {
                CleanupInvalidListeners(eventType);

                var eventListener = new EventListener(
                    listener,
                    callback,
                    autoUnregister,
                    priority,
                    oneShot
                );
                var listeners = GetListeners(eventType);

                // 按优先级插入（优先级高的在前）
                int index = 0;
                while (index < listeners.Count && listeners[index].Priority > priority)
                    index++;

                listeners.Insert(index, eventListener);

#if UNITY_EDITOR
                if (debugMode)
                {
                    Debug.Log(
                        $"注册事件监听: {eventType} -> {listener.GetType().Name} (优先级: {priority}, 一次性: {oneShot})"
                    );
                }
#endif
            }
        }

        /// <summary>
        /// 注销事件监听
        /// </summary>
        public void UnregisterListener(EventType eventType, object listener)
        {
            lock (_lock)
            {
                if (!_eventListeners.ContainsKey(eventType))
                    return;

                var listeners = _eventListeners[eventType];
                listeners.RemoveAll(l => l.Target == listener);

#if UNITY_EDITOR
                if (debugMode)
                {
                    Debug.Log($"注销事件监听: {eventType} -> {listener.GetType().Name}");
                }
#endif
            }
        }

        /// <summary>
        /// 注销对象的所有事件监听
        /// </summary>
        public void UnregisterAllListeners(object listener)
        {
            lock (_lock)
            {
                foreach (var eventType in _eventListeners.Keys.ToList())
                {
                    _eventListeners[eventType].RemoveAll(l => l.Target == listener);
                }

#if UNITY_EDITOR
                if (debugMode)
                {
                    Debug.Log($"注销所有事件监听: {listener.GetType().Name}");
                }
#endif
            }
        }

        /// <summary>
        /// 添加事件拦截器（返回bool以决定是否继续传播）
        /// </summary>
        public void AddInterceptor(Func<EventType, object, bool> interceptor)
        {
            OnEventIntercept += interceptor;
        }

        /// <summary>
        /// 移除事件拦截器
        /// </summary>
        public void RemoveInterceptor(Func<EventType, object, bool> interceptor)
        {
            OnEventIntercept -= interceptor;
        }

        /// <summary>
        /// 触发事件
        /// </summary>
        public void TriggerEvent(EventType eventType, object data = null)
        {
            // 执行事件拦截 **（现在可以正常工作了）**
            if (OnEventIntercept != null)
            {
                var interceptors = OnEventIntercept.GetInvocationList();
                foreach (Func<EventType, object, bool> interceptor in interceptors)
                {
                    if (!interceptor(eventType, data))
                    {
#if UNITY_EDITOR
                        if (debugMode)
                        {
                            Debug.Log($"事件 {eventType} 被拦截阻止传播");
                        }
#endif
                        return; // 阻止传播
                    }
                }
            }

            lock (_lock)
            {
                CleanupInvalidListeners(eventType);

                if (!_eventListeners.ContainsKey(eventType))
                    return;

#if UNITY_EDITOR
                if (debugMode)
                {
                    Debug.Log($"触发事件: {eventType} {(data != null ? $"数据: {data}" : "")}");
                }
#endif

#if UNITY_EDITOR
                // 更新事件统计（仅编辑器）
                if (enableStatistics)
                {
                    _eventStatistics.AddOrUpdate(eventType, 1, (_, v) => v + 1);
                }
#endif

                // 使用池化列表避免GC
                var listeners = _eventListeners[eventType];
                var cache = BorrowList();
                cache.AddRange(listeners);

                // 处理一次性监听器
                List<EventListener> toRemove = new List<EventListener>();

                foreach (var listener in cache)
                {
                    if (listener.IsValid)
                    {
                        try
                        {
                            listener.Callback(data);
                            if (listener.OneShot)
                            {
                                toRemove.Add(listener);
                            }
                        }
                        catch (Exception e)
                        {
#if UNITY_EDITOR
                            Debug.LogError($"事件回调执行错误: {eventType} -> {e.Message}");
#endif
                        }
                    }
                }

                // 移除一次性监听器
                foreach (var remove in toRemove)
                {
                    listeners.Remove(remove);
                }

                ReturnList(cache);
            }
        }

        /// <summary>
        /// 触发泛型事件
        /// </summary>
        public void TriggerEvent<T>(EventType eventType, T data)
        {
            TriggerEvent(eventType, data);
        }

        /// <summary>
        /// 异步触发事件（使用Task以真正异步）
        /// </summary>
        public async Task TriggerEventAsync(EventType eventType, object data = null)
        {
            // 执行事件拦截（同步，因为拦截应阻塞）
            if (OnEventIntercept != null)
            {
                var interceptors = OnEventIntercept.GetInvocationList();
                foreach (Func<EventType, object, bool> interceptor in interceptors)
                {
                    if (!interceptor(eventType, data))
                    {
#if UNITY_EDITOR
                        if (debugMode)
                        {
                            Debug.Log($"事件 {eventType} 被拦截阻止传播");
                        }
#endif
                        return;
                    }
                }
            }

            List<EventListener> cache;
            lock (_lock)
            {
                CleanupInvalidListeners(eventType);

                if (!_eventListeners.ContainsKey(eventType))
                    return;

#if UNITY_EDITOR
                if (debugMode)
                {
                    Debug.Log($"异步触发事件: {eventType}");
                }
#endif

#if UNITY_EDITOR
                // 更新事件统计（仅编辑器）
                if (enableStatistics)
                {
                    _eventStatistics.AddOrUpdate(eventType, 1, (_, v) => v + 1);
                }
#endif

                // 使用池化列表
                var listeners = _eventListeners[eventType];
                cache = BorrowList();
                cache.AddRange(listeners);
            }

            // 异步执行回调
            List<Task> tasks = new List<Task>();
            List<EventListener> toRemove = new List<EventListener>();

            foreach (var listener in cache)
            {
                if (listener.IsValid)
                {
                    tasks.Add(
                        Task.Run(() =>
                        {
                            try
                            {
                                listener.Callback(data);
                                if (listener.OneShot)
                                {
                                    toRemove.Add(listener);
                                }
                            }
                            catch (Exception e)
                            {
#if UNITY_EDITOR
                                Debug.LogError($"异步事件回调执行错误: {eventType} -> {e.Message}");
#endif
                            }
                        })
                    );
                }
            }

            await Task.WhenAll(tasks);

            // 归还池化列表
            ReturnList(cache);

            // 移除一次性监听器（回主线程）
            UnityMainThreadDispatcher.Instance.Enqueue(() =>
            {
                lock (_lock)
                {
                    if (_eventListeners.TryGetValue(eventType, out var listeners))
                    {
                        foreach (var remove in toRemove)
                        {
                            listeners.Remove(remove);
                        }
                    }
                }
            });
        }

        /// <summary>
        /// 清理无效监听器
        /// </summary>
        private void CleanupInvalidListeners(EventType eventType)
        {
            if (!_eventListeners.ContainsKey(eventType))
                return;

            var listeners = _eventListeners[eventType];
            int removed = listeners.RemoveAll(l => !l.IsValid);

#if UNITY_EDITOR
            if (debugMode && removed > 0)
            {
                Debug.Log($"清理事件: {eventType} -> 移除 {removed} 个无效监听器");
            }
#endif
        }

#if UNITY_EDITOR
        /// <summary>
        /// 打印事件统计信息（仅编辑器）
        /// </summary>
        public void PrintEventStatistics()
        {
            if (!enableStatistics)
                return;

            Debug.Log("===== 事件统计 =====");
            foreach (var kvp in _eventStatistics.OrderByDescending(x => x.Value))
            {
                Debug.Log($"{kvp.Key}: {kvp.Value}次");
            }
        }

        /// <summary>
        /// 重置事件统计（仅编辑器）
        /// </summary>
        public void ResetEventStatistics()
        {
            _eventStatistics.Clear();
        }
#endif
    }

    /// <summary>
    /// 事件监听器封装类（使用弱引用以防内存泄漏）
    /// </summary>
    public class EventListener
    {
        private readonly WeakReference _targetReference;
        public Action<object> Callback { get; }
        public bool AutoUnregister { get; }
        public int Priority { get; }
        public bool OneShot { get; }

        public EventListener(
            object target,
            Action<object> callback,
            bool autoUnregister,
            int priority = 0,
            bool oneShot = false
        )
        {
            _targetReference = new WeakReference(target);
            Callback = callback;
            AutoUnregister = autoUnregister;
            Priority = priority;
            OneShot = oneShot;
        }

        /// <summary>
        /// 获取目标对象
        /// </summary>
        public object Target => _targetReference.Target;

        /// <summary>
        /// 检查监听器是否有效
        /// </summary>
        public bool IsValid
        {
            get
            {
                var target = _targetReference.Target;
                if (target == null)
                    return false;

                if (AutoUnregister && target is UnityEngine.Object unityObj)
                {
                    return unityObj != null;
                }
                return true;
            }
        }
    }

    /// <summary>
    /// 事件监听助手类 - 简化组件中的事件监听
    /// </summary>
    public static class EventHelper
    {
        /// <summary>
        /// 注册事件监听
        /// </summary>
        public static void RegisterEvent(
            this MonoBehaviour component,
            EventType eventType,
            Action<object> callback,
            int priority = 0,
            bool oneShot = false
        )
        {
            EventManager.Instance.RegisterListener(
                eventType,
                component,
                callback,
                true,
                priority,
                oneShot
            );
        }

        /// <summary>
        /// 注册泛型事件监听
        /// </summary>
        public static void RegisterEvent<T>(
            this MonoBehaviour component,
            EventType eventType,
            Action<T> callback,
            int priority = 0,
            bool oneShot = false
        )
        {
            Action<object> wrapper = obj =>
            {
                if (obj is T typedObj)
                    callback(typedObj);
            };
            EventManager.Instance.RegisterListener(
                eventType,
                component,
                wrapper,
                true,
                priority,
                oneShot
            );
        }

        /// <summary>
        /// 注销事件监听
        /// </summary>
        public static void UnregisterEvent(this MonoBehaviour component, EventType eventType)
        {
            EventManager.Instance.UnregisterListener(eventType, component);
        }

        /// <summary>
        /// 注销所有事件监听
        /// </summary>
        public static void UnregisterAllEvents(this MonoBehaviour component)
        {
            EventManager.Instance.UnregisterAllListeners(component);
        }

        /// <summary>
        /// 触发事件
        /// </summary>
        public static void TriggerEvent(
            this MonoBehaviour component,
            EventType eventType,
            object data = null
        )
        {
            EventManager.Instance.TriggerEvent(eventType, data);
        }

        /// <summary>
        /// 触发泛型事件
        /// </summary>
        public static void TriggerEvent<T>(
            this MonoBehaviour component,
            EventType eventType,
            T data
        )
        {
            EventManager.Instance.TriggerEvent(eventType, data);
        }

        /// <summary>
        /// 异步触发事件
        /// </summary>
        public static async Task TriggerEventAsync(
            this MonoBehaviour component,
            EventType eventType,
            object data = null
        )
        {
            await EventManager.Instance.TriggerEventAsync(eventType, data);
        }

        /// <summary>
        /// 添加事件拦截器
        /// </summary>
        public static void AddInterceptor(
            this MonoBehaviour component,
            Func<EventType, object, bool> interceptor
        )
        {
            EventManager.Instance.AddInterceptor(interceptor);
        }

        /// <summary>
        /// 移除事件拦截器
        /// </summary>
        public static void RemoveInterceptor(
            this MonoBehaviour component,
            Func<EventType, object, bool> interceptor
        )
        {
            EventManager.Instance.RemoveInterceptor(interceptor);
        }
    }

    /// <summary>
    /// Unity主线程调度器（用于异步回调后回主线程）
    /// </summary>
    public class UnityMainThreadDispatcher : MonoBehaviour
    {
        private static UnityMainThreadDispatcher _instance;
        private readonly Queue<Action> _executionQueue = new Queue<Action>();

        public static UnityMainThreadDispatcher Instance
        {
            get
            {
                if (_instance == null)
                {
                    GameObject go = new GameObject("UnityMainThreadDispatcher");
                    _instance = go.AddComponent<UnityMainThreadDispatcher>();
                    DontDestroyOnLoad(go);
                }
                return _instance;
            }
        }

        void Update()
        {
            lock (_executionQueue)
            {
                while (_executionQueue.Count > 0)
                {
                    _executionQueue.Dequeue().Invoke();
                }
            }
        }

        public void Enqueue(Action action)
        {
            lock (_executionQueue)
            {
                _executionQueue.Enqueue(action);
            }
        }
    }
}

```
