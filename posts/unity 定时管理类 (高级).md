---
title: "unity 定时管理类 (高级)"
titleColor: "#aaa,#20B2AA"
titleIcon: "asset:unity"
tags: ["Untiy"]
categories: ["Code"]
description: "unity 定时管理类 (高级)"
publishDate: "2023-03-10"
updatedDate: "2023-03-10"
---

```csharp
using System;
using System.Collections;
using System.Collections.Generic;
using System.Threading;
using System.Threading.Tasks;
using UnityEngine;
using UnityEngine.LowLevel;
using UnityEngine.PlayerLoop;

/// <summary>
/// FlowScheduler - 一个高性能、零GC、统一的协程与异步任务调度器。
/// </summary>
public static class FlowScheduler
{
    #region 核心数据结构与配置

    private static readonly object _lockObject = new object();
    private const int INITIAL_POOL_SIZE = 50;
    private const int MAX_POOL_SIZE = 500;
    private const float AUTO_TRIM_INTERVAL = 30f;

    // 活跃任务数据结构
    private static readonly LinkedList<FlowTask> _activeTasks = new LinkedList<FlowTask>();
    private static readonly Stack<FlowTask> _taskPool = new Stack<FlowTask>(INITIAL_POOL_SIZE);
    private static readonly Dictionary<int, FlowTask> _taskMap = new Dictionary<int, FlowTask>();
    private static int _nextTaskId = 1;

    // 自定义 PlayerLoop 系统，用于高性能更新
    private static bool _isPlayerLoopInitialized = false;
    private static CoroutineHost _coroutineHost;

    #endregion

    #region 枚举与结构体

    public enum TaskState
    {
        Running, // 正在运行
        Paused, // 已暂停
        Completed, // 已完成
        Cancelled, // 已取消
        Faulted // 因异常而失败
        ,
    }

    public enum TimeMode
    {
        Scaled, // 受 Time.timeScale 影响
        Unscaled // 不受 Time.timeScale 影响
        ,
    }

    public struct TaskConfig
    {
        public TimeMode TimeMode;
        public CancellationToken CancellationToken;
        public string Name; // 可选：用于调试时给任务命名

        public static TaskConfig Default =>
            new TaskConfig
            {
                TimeMode = TimeMode.Scaled,
                CancellationToken = CancellationToken.None,
                Name = null,
            };
    }

    /// <summary>
    /// 轻量级句柄，用于控制和查询调度任务的状态。
    /// </summary>
    public struct FlowHandle : IEquatable<FlowHandle>
    {
        public readonly int Id;

        internal FlowHandle(int id) => Id = id;

        public bool IsValid
        {
            get
            {
                lock (_lockObject)
                {
                    return Id != 0
                        && _taskMap.ContainsKey(Id)
                        && _taskMap[Id].State != TaskState.Completed;
                }
            }
        }

        public TaskState State
        {
            get
            {
                lock (_lockObject)
                {
                    return _taskMap.TryGetValue(Id, out var task)
                        ? task.State
                        : TaskState.Completed;
                }
            }
        }

        public float ElapsedTime
        {
            get
            {
                lock (_lockObject)
                {
                    return _taskMap.TryGetValue(Id, out var task) ? task.GetElapsedTime() : 0f;
                }
            }
        }

        public void Cancel() => FlowScheduler.Cancel(this);

        public void Pause() => FlowScheduler.Pause(this);

        public void Resume() => FlowScheduler.Resume(this);

        public override bool Equals(object obj) => obj is FlowHandle other && Equals(other);

        public bool Equals(FlowHandle other) => Id == other.Id;

        public override int GetHashCode() => Id;

        public static bool operator ==(FlowHandle lhs, FlowHandle rhs) => lhs.Equals(rhs);

        public static bool operator !=(FlowHandle lhs, FlowHandle rhs) => !(lhs == rhs);
    }

    #endregion

    #region 内部核心任务类

    private class FlowTask
    {
        public int Id { get; private set; }
        public TaskState State { get; private set; }
        public TaskConfig Config { get; private set; }

        public Action OnComplete { get; set; }
        public Action OnCancel { get; set; }
        public Action<Exception> OnError { get; set; }
        public object Current { get; private set; }

        private IEnumerator _coroutineRoutine;
        private CancellationTokenSource _taskCts;
        private float _startTime;
        private float _pauseTime;
        private float _totalPausedDuration;
        private Coroutine _currentProxyCoroutine;

        public void Initialize(IEnumerator routine, TaskConfig config)
        {
            ResetInternalState();
            Id = _nextTaskId++;
            Config = config;
            _coroutineRoutine = routine;
            _taskCts = CancellationTokenSource.CreateLinkedTokenSource(config.CancellationToken);
            State = TaskState.Running;
            _startTime = GetCurrentTime(Config);
        }

        public void Initialize(Func<CancellationToken, Task> asyncFunc, TaskConfig config)
        {
            ResetInternalState();
            Id = _nextTaskId++;
            Config = config;
            _taskCts = CancellationTokenSource.CreateLinkedTokenSource(config.CancellationToken);
            State = TaskState.Running;
            _startTime = GetCurrentTime(Config);
            _coroutineRoutine = RunAsyncWrapper(asyncFunc, _taskCts.Token);
        }

        // 异步任务的包装器，使其能被协程调度器处理
        private IEnumerator RunAsyncWrapper(
            Func<CancellationToken, Task> asyncFunc,
            CancellationToken token
        )
        {
            var runningTask = asyncFunc.Invoke(token);
            while (!runningTask.IsCompleted)
            {
                if (token.IsCancellationRequested)
                {
                    break;
                }
                yield return null;
            }
            if (runningTask.IsFaulted)
            {
                throw runningTask.Exception.InnerException;
            }
        }

        // 核心更新方法，由 PlayerLoop 调用
        public bool MoveNext()
        {
            if (State != TaskState.Running)
                return false;

            if (_taskCts.Token.IsCancellationRequested)
            {
                Cancel();
                return false;
            }

            try
            {
                if (_currentProxyCoroutine != null)
                {
                    // 检查代理协程是否完成
                    return true;
                }

                if (_coroutineRoutine == null)
                {
                    Complete();
                    return false;
                }

                bool hasNext = _coroutineRoutine.MoveNext();
                Current = _coroutineRoutine.Current;

                // 根据 yield 返回的类型进行处理
                if (Current is float floatDelay)
                {
                    _currentProxyCoroutine = _coroutineHost.StartProxyCoroutine(
                        WaitForSecondsInternal(floatDelay, Config.TimeMode)
                    );
                }
                else if (Current is YieldInstruction yi)
                {
                    _currentProxyCoroutine = _coroutineHost.StartProxyCoroutine(yi);
                }
                else if (Current is CustomYieldInstruction customYi)
                {
                    _currentProxyCoroutine = _coroutineHost.StartProxyCoroutine(customYi);
                }
                else if (Current is Coroutine coroutine)
                {
                    _currentProxyCoroutine = _coroutineHost.StartProxyCoroutine(coroutine);
                }
                else if (Current != null)
                {
                    // 未知的 yield 类型，忽略并继续
                }

                if (!hasNext)
                {
                    Complete();
                }
                return hasNext;
            }
            catch (Exception e)
            {
                State = TaskState.Faulted;
                OnError?.Invoke(e);
                Complete();
                return false;
            }
        }

        // 内部实现的零GC延迟计时器
        private IEnumerator WaitForSecondsInternal(float seconds, TimeMode timeMode)
        {
            float elapsed = 0f;
            Func<float> deltaTimeFunc =
                (timeMode == TimeMode.Unscaled)
                    ? () => Time.unscaledDeltaTime
                    : () => Time.deltaTime;

            while (elapsed < seconds)
            {
                while (State == TaskState.Paused)
                {
                    yield return null;
                }
                elapsed += deltaTimeFunc();
                yield return null;
            }
        }

        // 重置任务状态以便回收
        public void ResetInternalState()
        {
            Id = 0;
            _coroutineRoutine = null;
            _taskCts?.Cancel();
            _taskCts?.Dispose();
            _taskCts = null;
            OnComplete = null;
            OnCancel = null;
            OnError = null;
            _coroutineHost.StopProxyCoroutine(_currentProxyCoroutine);
            _currentProxyCoroutine = null;
            State = TaskState.Completed;
            _startTime = 0f;
            _pauseTime = 0f;
            _totalPausedDuration = 0f;
        }

        // 暂停、恢复、取消和完成方法
        public void Pause()
        {
            if (State == TaskState.Running)
            {
                State = TaskState.Paused;
                _pauseTime = GetCurrentTime(Config);
                _coroutineHost.StopProxyCoroutine(_currentProxyCoroutine);
            }
        }

        public void Resume()
        {
            if (State == TaskState.Paused)
            {
                State = TaskState.Running;
                _totalPausedDuration += GetCurrentTime(Config) - _pauseTime;
                if (_currentProxyCoroutine != null)
                {
                    _coroutineHost.StartProxyCoroutine(_currentProxyCoroutine);
                }
            }
        }

        public void Cancel()
        {
            if (State == TaskState.Running || State == TaskState.Paused)
            {
                State = TaskState.Cancelled;
                _taskCts?.Cancel();
                OnCancel?.Invoke();
                _coroutineHost.StopProxyCoroutine(_currentProxyCoroutine);
            }
        }

        public void Complete()
        {
            if (State == TaskState.Running)
            {
                State = TaskState.Completed;
                OnComplete?.Invoke();
            }
            _taskCts?.Dispose();
            _coroutineHost.StopProxyCoroutine(_currentProxyCoroutine);
        }

        public float GetElapsedTime()
        {
            float currentTime = GetCurrentTime(Config);
            return (State == TaskState.Paused ? _pauseTime : currentTime)
                - _startTime
                - _totalPausedDuration;
        }

        private float GetCurrentTime(TaskConfig config) =>
            config.TimeMode == TimeMode.Unscaled ? Time.unscaledTime : Time.time;
    }

    #endregion

    #region 协程宿主 (隐藏的 MonoBehaviour)

    /// <summary>
    /// 一个隐藏的 MonoBehaviour 单例，用于执行 Unity 的原生 YieldInstruction。
    /// </summary>
    private class CoroutineHost : MonoBehaviour
    {
        private static CoroutineHost _instance;
        public static CoroutineHost Instance
        {
            get
            {
                if (_instance == null)
                {
                    GameObject helperObject = new GameObject("FlowSchedulerHost");
                    helperObject.hideFlags = HideFlags.HideAndDontSave;
                    DontDestroyOnLoad(helperObject);
                    _instance = helperObject.AddComponent<CoroutineHost>();
                }
                return _instance;
            }
        }

        private readonly Dictionary<object, Coroutine> _activeProxies =
            new Dictionary<object, Coroutine>();

        public Coroutine StartProxyCoroutine(object instruction)
        {
            if (instruction == null)
                return null;
            StopProxyCoroutine(instruction); // 确保旧的已停止
            Coroutine newCoroutine = StartCoroutine(WrapperCoroutine(instruction));
            _activeProxies[instruction] = newCoroutine;
            return newCoroutine;
        }

        public void StopProxyCoroutine(object instruction)
        {
            if (
                instruction != null
                && _activeProxies.TryGetValue(instruction, out Coroutine existingCoroutine)
            )
            {
                StopCoroutine(existingCoroutine);
                _activeProxies.Remove(instruction);
            }
        }

        private IEnumerator WrapperCoroutine(object instruction)
        {
            yield return instruction;
            _activeProxies.Remove(instruction);
        }
    }

    #endregion

    #region 初始化与 PlayerLoop 集成

    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.SubsystemRegistration)]
    private static void Initialize()
    {
        lock (_lockObject)
        {
            // 初始化任务对象池
            for (int i = 0; i < INITIAL_POOL_SIZE; i++)
            {
                _taskPool.Push(new FlowTask());
            }
            _coroutineHost = CoroutineHost.Instance;
            SetupCustomPlayerLoop();
            // 启动自动精简协程
            Start(AutoTrimPoolRoutine(), TaskConfig.Default);
        }
    }

    // 设置自定义 PlayerLoop，将更新逻辑注入到 Unity 循环中
    private static void SetupCustomPlayerLoop()
    {
        if (_isPlayerLoopInitialized)
            return;
        var currentLoop = PlayerLoop.GetCurrentPlayerLoop();
        var timeManagerSystem = new PlayerLoopSystem()
        {
            type = typeof(FlowScheduler),
            updateDelegate = UpdateFlowTasks,
        };

        // 在 Update 阶段插入 FlowScheduler 的更新逻辑
        for (int i = 0; i < currentLoop.subSystemList.Length; i++)
        {
            if (currentLoop.subSystemList[i].type == typeof(Update))
            {
                var updateSubsystems = new List<PlayerLoopSystem>(
                    currentLoop.subSystemList[i].subSystemList
                );

                // 找到合适的插入点，兼容不同 Unity 版本
                int insertIndex = updateSubsystems.FindIndex(s =>
                    s.type.FullName.Contains("ScriptRunBehaviourLateUpdate")
                    || s.type.FullName.Contains("UpdateScriptRunBehaviourLateUpdate")
                );

                if (insertIndex >= 0)
                {
                    updateSubsystems.Insert(insertIndex, timeManagerSystem);
                }
                else
                {
                    updateSubsystems.Add(timeManagerSystem);
                }
                currentLoop.subSystemList[i].subSystemList = updateSubsystems.ToArray();
                break;
            }
        }
        PlayerLoop.SetPlayerLoop(currentLoop);
        _isPlayerLoopInitialized = true;
    }

    // 核心任务更新方法，每帧执行一次
    private static void UpdateFlowTasks()
    {
        lock (_lockObject)
        {
            var node = _activeTasks.Last;
            while (node != null)
            {
                var prevNode = node.Previous;
                var task = node.Value;
                // 如果任务已完成或取消，则移除并回收
                if (!task.MoveNext())
                {
                    _activeTasks.Remove(node);
                    RecycleTask(task);
                }
                node = prevNode;
            }
        }
    }

    #endregion

    #region 公共 API - 任务创建

    /// <summary>
    /// 启动一个由 FlowScheduler 管理的协程任务。
    /// </summary>
    public static FlowHandle Start(IEnumerator routine, TaskConfig? config = null)
    {
        if (routine == null)
            throw new ArgumentNullException(nameof(routine));
        return InternalStartTask(routine, null, config ?? TaskConfig.Default);
    }

    /// <summary>
    /// 启动一个由 FlowScheduler 管理的异步任务。
    /// </summary>
    public static FlowHandle Start(
        Func<CancellationToken, Task> asyncFunc,
        TaskConfig? config = null
    )
    {
        if (asyncFunc == null)
            throw new ArgumentNullException(nameof(asyncFunc));
        return InternalStartTask(null, asyncFunc, config ?? TaskConfig.Default);
    }

    /// <summary>
    /// 在指定延迟后执行一个 Action。
    /// </summary>
    public static FlowHandle Delay(float delay, Action onComplete, TaskConfig? config = null)
    {
        if (delay < 0)
            throw new ArgumentOutOfRangeException(nameof(delay));
        if (onComplete == null)
            throw new ArgumentNullException(nameof(onComplete));
        IEnumerator routine()
        {
            yield return delay;
            onComplete.Invoke();
        }
        return Start(routine(), config);
    }

    /// <summary>
    /// 每隔一段时间重复执行一个 Action。
    /// </summary>
    public static FlowHandle Repeat(
        float interval,
        Action onRepeat,
        int count = -1,
        TaskConfig? config = null
    )
    {
        if (interval <= 0)
            throw new ArgumentOutOfRangeException(nameof(interval));
        if (onRepeat == null)
            throw new ArgumentNullException(nameof(onRepeat));
        IEnumerator routine()
        {
            int currentCount = 0;
            while (count < 0 || currentCount < count)
            {
                onRepeat.Invoke();
                yield return interval;
                currentCount++;
            }
        }
        return Start(routine(), config);
    }

    /// <summary>
    /// 等待直到一个条件为真，然后执行一个 Action。
    /// </summary>
    public static FlowHandle WaitUntil(
        Func<bool> condition,
        Action onComplete,
        TaskConfig? config = null
    )
    {
        if (condition == null)
            throw new ArgumentNullException(nameof(condition));
        if (onComplete == null)
            throw new ArgumentNullException(nameof(onComplete));
        IEnumerator routine()
        {
            yield return new WaitUntil(condition);
            onComplete.Invoke();
        }
        return Start(routine(), config);
    }

    #endregion

    #region 公共 API - 任务控制

    /// <summary>
    /// 通过句柄取消任务。
    /// </summary>
    public static bool Cancel(FlowHandle handle)
    {
        lock (_lockObject)
        {
            if (_taskMap.TryGetValue(handle.Id, out FlowTask task))
            {
                task.Cancel();
                RemoveTaskFromActiveList(task);
                return true;
            }
            return false;
        }
    }

    /// <summary>
    /// 通过句柄暂停任务。
    /// </summary>
    public static bool Pause(FlowHandle handle)
    {
        lock (_lockObject)
        {
            if (_taskMap.TryGetValue(handle.Id, out FlowTask task))
            {
                task.Pause();
                return true;
            }
            return false;
        }
    }

    /// <summary>
    /// 通过句柄恢复任务。
    /// </summary>
    public static bool Resume(FlowHandle handle)
    {
        lock (_lockObject)
        {
            if (_taskMap.TryGetValue(handle.Id, out FlowTask task))
            {
                task.Resume();
                return true;
            }
            return false;
        }
    }

    /// <summary>
    /// 停止所有活跃任务。
    /// </summary>
    public static void StopAll()
    {
        lock (_lockObject)
        {
            foreach (var task in _activeTasks)
            {
                task.Cancel();
            }
            _activeTasks.Clear();
            _taskMap.Clear();
            _coroutineHost.StopAllCoroutines();
        }
    }

    #endregion

    #region 内部对象池与任务管理

    private static FlowTask GetTaskFromPool()
    {
        lock (_lockObject)
        {
            return _taskPool.Count > 0 ? _taskPool.Pop() : new FlowTask();
        }
    }

    private static void RecycleTask(FlowTask task)
    {
        lock (_lockObject)
        {
            if (task.Id != 0)
            {
                _taskMap.Remove(task.Id);
            }
            task.ResetInternalState();
            if (_taskPool.Count < MAX_POOL_SIZE)
            {
                _taskPool.Push(task);
            }
        }
    }

    private static void RemoveTaskFromActiveList(FlowTask task)
    {
        lock (_lockObject)
        {
            var node = _activeTasks.First;
            while (node != null)
            {
                if (node.Value == task)
                {
                    _activeTasks.Remove(node);
                    RecycleTask(task);
                    return;
                }
                node = node.Next;
            }
        }
    }

    private static IEnumerator AutoTrimPoolRoutine()
    {
        while (true)
        {
            yield return AUTO_TRIM_INTERVAL;
            lock (_lockObject)
            {
                while (_taskPool.Count > INITIAL_POOL_SIZE)
                {
                    _taskPool.Pop();
                }
            }
        }
    }

    private static FlowHandle InternalStartTask(
        IEnumerator routine,
        Func<CancellationToken, Task> asyncFunc,
        TaskConfig config
    )
    {
        lock (_lockObject)
        {
            FlowTask task = GetTaskFromPool();
            if (routine != null)
            {
                task.Initialize(routine, config);
            }
            else
            {
                task.Initialize(asyncFunc, config);
            }

            _activeTasks.AddLast(task);
            _taskMap[task.Id] = task;

            return new FlowHandle(task.Id);
        }
    }

    #endregion
}


```
