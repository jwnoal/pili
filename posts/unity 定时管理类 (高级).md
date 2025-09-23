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
using UnityEngine.Profiling;
#if UNITY_EDITOR
using UnityEditor;
#endif

/// <summary>
/// FlowScheduler - 高性能、零GC、统一的协程与异步任务调度器
/// 版本 3.2 - Unity 2022.3兼容版
/// </summary>
public static class FlowScheduler
{
    #region 核心数据结构与配置

    private static readonly object _lockObject = new object();
    private static int _initialPoolSize = 50;
    private static int _maxPoolSize = 500;
    private const float AUTO_TRIM_INTERVAL = 30f;
    private const float POOL_ADJUST_INTERVAL = 10f;

    private static readonly LinkedList<FlowTask> _activeTasks = new LinkedList<FlowTask>();
    private static readonly Stack<FlowTask> _taskPool = new Stack<FlowTask>();
    private static readonly Dictionary<int, FlowTask> _taskMap = new Dictionary<int, FlowTask>();
    private static int _nextTaskId = 1;

    private static bool _isPlayerLoopInitialized = false;
    private static CoroutineHost _coroutineHost;

    // 性能计数器
    public static int TotalTasksCreated { get; private set; }
    public static int TotalTasksCompleted { get; private set; }
    public static int PeakActiveTasks { get; private set; }

    #endregion

    #region 枚举与结构体

    public enum TaskState
    {
        Running,
        Paused,
        Completed,
        Cancelled,
        Faulted,
    }

    public enum TimeMode
    {
        Scaled,
        Unscaled,
    }

    public enum InjectionPoint
    {
        AfterUpdate,
        AfterFixedUpdate,
        AfterPreLateUpdate,
        Custom,
    }

    public struct TaskConfig
    {
        public TimeMode TimeMode;
        public CancellationToken CancellationToken;
        public string Name;
        public int Priority;
        public string ProfilerName;

        public static TaskConfig Default =>
            new TaskConfig
            {
                TimeMode = TimeMode.Scaled,
                CancellationToken = CancellationToken.None,
                Name = null,
                Priority = 0,
                ProfilerName = null,
            };
    }

    public struct FlowHandle : IEquatable<FlowHandle>
    {
        public readonly int Id;
        public readonly int Generation;

        internal FlowHandle(int id, int generation)
        {
            Id = id;
            Generation = generation;
        }

        public bool IsValid
        {
            get
            {
                lock (_lockObject)
                {
                    return _taskMap.TryGetValue(Id, out var task)
                        && task.State != TaskState.Completed
                        && task.Generation == Generation;
                }
            }
        }

        public TaskState State
        {
            get
            {
                lock (_lockObject)
                {
                    if (_taskMap.TryGetValue(Id, out var task) && task.Generation == Generation)
                    {
                        return task.State;
                    }
                    return TaskState.Completed;
                }
            }
        }

        public float ElapsedTime
        {
            get
            {
                lock (_lockObject)
                {
                    if (_taskMap.TryGetValue(Id, out var task) && task.Generation == Generation)
                    {
                        return task.GetElapsedTime();
                    }
                    return 0f;
                }
            }
        }

        public void Cancel() => FlowScheduler.Cancel(this);

        public void Pause() => FlowScheduler.Pause(this);

        public void Resume() => FlowScheduler.Resume(this);

        public bool IsAlive => State == TaskState.Running || State == TaskState.Paused;

        public override bool Equals(object obj) => obj is FlowHandle other && Equals(other);

        public bool Equals(FlowHandle other) => Id == other.Id && Generation == other.Generation;

        public override int GetHashCode() => HashCode.Combine(Id, Generation);

        public static bool operator ==(FlowHandle lhs, FlowHandle rhs) => lhs.Equals(rhs);

        public static bool operator !=(FlowHandle lhs, FlowHandle rhs) => !lhs.Equals(rhs);
    }

    #endregion

    #region 内部核心任务类

    private class FlowTask : IComparable<FlowTask>
    {
        public int Id { get; private set; }
        public int Generation { get; private set; }
        public TaskState State { get; private set; }
        public TaskConfig Config { get; private set; }
        public Exception Exception { get; private set; }
        public float LastUpdateTime { get; private set; }
        public double StartTime { get; private set; }

        private IEnumerator _coroutineRoutine;
        private CancellationTokenSource _taskCts;
        private double _pauseTime;
        private double _totalPausedDuration;
        private Coroutine _currentProxyCoroutine;
        private bool _isProxyRunning;

        #region 状态控制方法
        public void MarkRunning() => State = TaskState.Running;

        public void MarkPaused() => State = TaskState.Paused;

        public void MarkCancelled() => State = TaskState.Cancelled;

        public void MarkCompleted() => State = TaskState.Completed;

        public void MarkFaulted() => State = TaskState.Faulted;

        public void SetException(Exception e)
        {
            Exception = e;
            MarkFaulted();
        }
        #endregion

        public void Initialize(int id, IEnumerator routine, TaskConfig config)
        {
            ResetInternalState();
            Id = id;
            Config = config;
            _coroutineRoutine = routine;
            _taskCts = CancellationTokenSource.CreateLinkedTokenSource(config.CancellationToken);
            MarkRunning();
            StartTime = GetCurrentTime(Config);
            LastUpdateTime = (float)StartTime;
        }

        public void Initialize(int id, Func<CancellationToken, Task> asyncFunc, TaskConfig config)
        {
            ResetInternalState();
            Id = id;
            Config = config;
            _taskCts = CancellationTokenSource.CreateLinkedTokenSource(config.CancellationToken);
            MarkRunning();
            StartTime = GetCurrentTime(Config);
            LastUpdateTime = (float)StartTime;
            _coroutineRoutine = RunAsyncWrapper(asyncFunc, _taskCts.Token);
        }

        private IEnumerator RunAsyncWrapper(
            Func<CancellationToken, Task> asyncFunc,
            CancellationToken token
        )
        {
            Task runningTask = null;
            Exception caughtException = null;

            try
            {
                runningTask = asyncFunc.Invoke(token);
            }
            catch (Exception e)
            {
                caughtException = e;
            }

            if (caughtException != null)
            {
                SetException(caughtException);
                yield break;
            }

            if (runningTask == null)
            {
                yield break;
            }

            while (!runningTask.IsCompleted)
            {
                if (token.IsCancellationRequested)
                {
                    break;
                }
                yield return null;
            }

            if (runningTask.IsFaulted && runningTask.Exception != null)
            {
                SetException(runningTask.Exception);
            }
        }

        public bool MoveNext()
        {
            // 更新最后执行时间
            LastUpdateTime = (float)GetCurrentTime(Config);

            // 先检查暂停状态 - 保持任务活跃但不执行
            if (State == TaskState.Paused)
                return true;

            // 再检查其他终止状态
            if (State != TaskState.Running)
                return false;

            if (_taskCts != null && _taskCts.Token.IsCancellationRequested)
            {
                MarkCancelled();
                return false;
            }

            try
            {
                if (_isProxyRunning)
                {
                    return true;
                }

                if (_coroutineRoutine == null)
                {
                    MarkCompleted();
                    return false;
                }

                bool hasNext = _coroutineRoutine.MoveNext();

                if (_isProxyRunning)
                {
                    return true;
                }

                var currentYield = _coroutineRoutine.Current;

                // 使用模式匹配优化类型判断
                switch (currentYield)
                {
                    case float floatDelay:
                        StartProxyCoroutine(WaitForSecondsInternal(floatDelay, Config.TimeMode));
                        break;
                    case Coroutine coroutine:
                        StartProxyCoroutine(coroutine);
                        break;
                    case YieldInstruction yieldInstruction:
                        StartProxyCoroutine(yieldInstruction);
                        break;
                    case CustomYieldInstruction customYield:
                        StartProxyCoroutine(customYield);
                        break;
                }

                if (!hasNext)
                {
                    MarkCompleted();
                }
                return hasNext;
            }
            catch (Exception e)
            {
                SetException(e);
                MarkCompleted();
                return false;
            }
        }

        private void StartProxyCoroutine(object instruction)
        {
            if (instruction == null)
                return;

            _isProxyRunning = true;
            _currentProxyCoroutine = _coroutineHost.StartProxyCoroutine(
                instruction,
                () =>
                {
                    _isProxyRunning = false;
                }
            );
        }

        private IEnumerator WaitForSecondsInternal(float seconds, TimeMode timeMode)
        {
            double elapsed = 0f;
            Func<double> deltaTimeFunc =
                (timeMode == TimeMode.Unscaled)
                    ? () => Time.unscaledDeltaTime
                    : () => Time.deltaTime;

            while (elapsed < seconds)
            {
                // 处理暂停状态
                while (State == TaskState.Paused)
                {
                    yield return null;
                }

                // 检查是否被取消
                if (State != TaskState.Running)
                {
                    yield break;
                }

                elapsed += deltaTimeFunc();
                yield return null;
            }
        }

        public void ResetInternalState()
        {
            _coroutineRoutine = null;
            if (_currentProxyCoroutine != null)
            {
                _coroutineHost?.StopProxyCoroutine(_currentProxyCoroutine);
                _currentProxyCoroutine = null;
            }
            _isProxyRunning = false;
            State = TaskState.Completed;
            StartTime = 0f;
            _pauseTime = 0f;
            _totalPausedDuration = 0f;
            Generation++;
            Exception = null;
            LastUpdateTime = 0f;

            if (_taskCts != null)
            {
                _taskCts.Dispose();
                _taskCts = null;
            }
        }

        public void Pause()
        {
            if (State == TaskState.Running)
            {
                MarkPaused();
                _pauseTime = GetCurrentTime(Config);
            }
        }

        public void Resume()
        {
            if (State == TaskState.Paused)
            {
                MarkRunning();
                _totalPausedDuration += GetCurrentTime(Config) - _pauseTime;
            }
        }

        public void Cancel()
        {
            if (State == TaskState.Running || State == TaskState.Paused)
            {
                MarkCancelled();
                _taskCts?.Cancel();
                if (_currentProxyCoroutine != null)
                {
                    _coroutineHost?.StopProxyCoroutine(_currentProxyCoroutine);
                }
            }
        }

        public float GetElapsedTime()
        {
            double currentTime = GetCurrentTime(Config);
            return (float)(
                (State == TaskState.Paused ? _pauseTime : currentTime)
                - StartTime
                - _totalPausedDuration
            );
        }

        private double GetCurrentTime(TaskConfig config) =>
            config.TimeMode == TimeMode.Unscaled ? Time.unscaledTimeAsDouble : Time.timeAsDouble;

        // 实现优先级比较
        public int CompareTo(FlowTask other)
        {
            return other.Config.Priority.CompareTo(Config.Priority);
        }
    }

    #endregion

    #region 协程宿主 (隐藏的 MonoBehaviour)

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

        private readonly object _proxyLock = new object();
        private readonly Dictionary<object, Coroutine> _activeProxies =
            new Dictionary<object, Coroutine>();

        public Coroutine StartProxyCoroutine(object instruction, Action onComplete)
        {
            if (instruction == null)
                return null;

            lock (_proxyLock)
            {
                StopProxyCoroutine(instruction);
                Coroutine newCoroutine = StartCoroutine(WrapperCoroutine(instruction, onComplete));
                _activeProxies[instruction] = newCoroutine;
                return newCoroutine;
            }
        }

        public void StopProxyCoroutine(object instruction)
        {
            if (instruction == null)
                return;

            lock (_proxyLock)
            {
                if (_activeProxies.TryGetValue(instruction, out Coroutine existingCoroutine))
                {
                    StopCoroutine(existingCoroutine);
                    _activeProxies.Remove(instruction);
                }
            }
        }

        public void StopAllCoroutinesInternal()
        {
            lock (_proxyLock)
            {
                StopAllCoroutines();
                _activeProxies.Clear();
            }
        }

        private IEnumerator WrapperCoroutine(object instruction, Action onComplete)
        {
            yield return instruction;

            lock (_proxyLock)
            {
                _activeProxies.Remove(instruction);
            }

            onComplete?.Invoke();
        }

        void OnDestroy()
        {
            lock (_proxyLock)
            {
                _activeProxies.Clear();
            }
        }
    }

    #endregion

    #region 初始化、清理与 PlayerLoop 集成

    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.SubsystemRegistration)]
    private static void Initialize()
    {
        Cleanup();

        lock (_lockObject)
        {
            // 初始化对象池
            for (int i = 0; i < _initialPoolSize; i++)
            {
                _taskPool.Push(new FlowTask());
            }

            _coroutineHost = CoroutineHost.Instance;
            SetupCustomPlayerLoop();

            // 启动管理协程
            Start(
                ManagementRoutine(),
                new TaskConfig
                {
                    Name = "FlowScheduler_Management",
                    Priority = int.MinValue, // 最低优先级
                }
            );
        }

#if UNITY_EDITOR
        EditorApplication.quitting += OnEditorQuitting;
#endif
    }

    private static void Cleanup()
    {
        lock (_lockObject)
        {
            foreach (var task in _activeTasks)
            {
                task.Cancel();
            }
            _activeTasks.Clear();
            _taskMap.Clear();
            _taskPool.Clear();
            _nextTaskId = 1;
            PeakActiveTasks = 0;
            TotalTasksCreated = 0;
            TotalTasksCompleted = 0;

            if (_coroutineHost != null)
            {
                _coroutineHost.StopAllCoroutinesInternal();
            }
        }
    }

#if UNITY_EDITOR
    private static void OnEditorQuitting()
    {
        EditorApplication.quitting -= OnEditorQuitting;
        Cleanup();
    }
#endif

    private static InjectionPoint _injectionPoint = InjectionPoint.AfterPreLateUpdate;
    private static string _customInjectionPoint;

    public static void ConfigurePlayerLoopInjection(InjectionPoint point, string customPoint = null)
    {
        if (_isPlayerLoopInitialized)
        {
            Debug.LogWarning(
                "[FlowScheduler] PlayerLoop already initialized. Changes will take effect on next initialization."
            );
        }

        _injectionPoint = point;
        _customInjectionPoint = customPoint;
    }

    private static void SetupCustomPlayerLoop()
    {
        if (_isPlayerLoopInitialized)
            return;

        var currentLoop = PlayerLoop.GetCurrentPlayerLoop();
        if (currentLoop.subSystemList == null)
            return;

        // 检查是否已经注入
        foreach (var system in currentLoop.subSystemList)
        {
            if (system.type == typeof(FlowScheduler))
            {
                _isPlayerLoopInitialized = true;
                return;
            }

            if (system.subSystemList != null)
            {
                foreach (var subSystem in system.subSystemList)
                {
                    if (subSystem.type == typeof(FlowScheduler))
                    {
                        _isPlayerLoopInitialized = true;
                        return;
                    }
                }
            }
        }

        var timeManagerSystem = new PlayerLoopSystem()
        {
            type = typeof(FlowScheduler),
            updateDelegate = UpdateFlowTasks,
        };

        // 增强PlayerLoop注入的健壮性
        for (int i = 0; i < currentLoop.subSystemList.Length; i++)
        {
            if (currentLoop.subSystemList[i].type == typeof(Update))
            {
                var updateSubsystems = new List<PlayerLoopSystem>(
                    currentLoop.subSystemList[i].subSystemList
                );

                // 增强的注入点选择逻辑
                int insertIndex = -1;
                string pointName = null;

                switch (_injectionPoint)
                {
                    case InjectionPoint.AfterUpdate:
                        insertIndex = updateSubsystems.FindIndex(s => s.type == typeof(Update));
                        pointName = "Update";
                        break;

                    case InjectionPoint.AfterFixedUpdate:
                        insertIndex = updateSubsystems.FindIndex(s =>
                            s.type == typeof(FixedUpdate)
                        );
                        pointName = "FixedUpdate";
                        break;

                    case InjectionPoint.AfterPreLateUpdate:
                        insertIndex = updateSubsystems.FindIndex(s =>
                            s.type == typeof(PreLateUpdate)
                        );
                        pointName = "PreLateUpdate";
                        break;

                    case InjectionPoint.Custom:
                        if (!string.IsNullOrEmpty(_customInjectionPoint))
                        {
                            insertIndex = updateSubsystems.FindIndex(s =>
                                s.type.Name.Contains(_customInjectionPoint)
                            );
                            pointName = _customInjectionPoint;
                        }
                        break;
                }

                if (insertIndex == -1)
                {
                    // 回退到智能选择
                    int[] priorityPoints = new[]
                    {
                        updateSubsystems.FindIndex(s =>
                            s.type.Name.Contains("ScriptRunBehaviourLateUpdate")
                        ),
                        updateSubsystems.FindIndex(s => s.type == typeof(PreLateUpdate)),
                        updateSubsystems.FindIndex(s => s.type == typeof(Update)),
                        updateSubsystems.Count - 1,
                    };

                    foreach (int index in priorityPoints)
                    {
                        if (index >= 0)
                        {
                            insertIndex = index;
                            pointName = updateSubsystems[index].type.Name;
                            break;
                        }
                    }
                }

                if (insertIndex >= 0)
                {
                    Debug.Log($"[FlowScheduler] Injecting at: {pointName}");
                    updateSubsystems.Insert(insertIndex + 1, timeManagerSystem);
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

    private static void UpdateFlowTasks()
    {
        Profiler.BeginSample("FlowScheduler.Update");

        try
        {
            List<FlowTask> tasksToProcess;
            lock (_lockObject)
            {
                tasksToProcess = new List<FlowTask>(_activeTasks);

                // 更新性能计数器
                if (tasksToProcess.Count > PeakActiveTasks)
                {
                    PeakActiveTasks = tasksToProcess.Count;
                }
            }

            // 按优先级排序（高优先级先执行）
            tasksToProcess.Sort();

            List<FlowTask> tasksToRemove = new List<FlowTask>();

            foreach (var task in tasksToProcess)
            {
                bool shouldRemove = false;
                try
                {
                    // Unity 2022.3兼容的Profiler标记
                    Profiler.BeginSample(task.Config.ProfilerName ?? "FlowTask");
                    shouldRemove = !task.MoveNext();
                }
                catch (Exception e)
                {
                    task.SetException(e);
                    shouldRemove = true;
                }
                finally
                {
                    Profiler.EndSample();
                }

                if (shouldRemove)
                {
                    tasksToRemove.Add(task);
                }
            }

            if (tasksToRemove.Count > 0)
            {
                lock (_lockObject)
                {
                    foreach (var task in tasksToRemove)
                    {
                        if (_activeTasks.Contains(task))
                        {
                            _activeTasks.Remove(task);
                            _taskMap.Remove(task.Id);
                            RecycleTask(task);
                            TotalTasksCompleted++;

#if UNITY_EDITOR || DEVELOPMENT_BUILD
                            TrackTaskHistory(task);
#endif
                        }
                    }
                }
            }
        }
        finally
        {
            Profiler.EndSample();
        }
    }

    #endregion

    #region 公共 API

    public static FlowHandle Start(IEnumerator routine, TaskConfig? config = null)
    {
        if (routine == null)
            throw new ArgumentNullException(nameof(routine));
        return InternalStartTask(routine, null, config ?? TaskConfig.Default);
    }

    public static FlowHandle Start(
        Func<CancellationToken, Task> asyncFunc,
        TaskConfig? config = null
    )
    {
        if (asyncFunc == null)
            throw new ArgumentNullException(nameof(asyncFunc));
        return InternalStartTask(null, asyncFunc, config ?? TaskConfig.Default);
    }

    public static FlowHandle Delay(float delay, Action onComplete, TaskConfig? config = null)
    {
        if (delay < 0)
            throw new ArgumentOutOfRangeException(nameof(delay));
        if (onComplete == null)
            throw new ArgumentNullException(nameof(onComplete));

        IEnumerator Routine()
        {
            yield return delay;
            onComplete.Invoke();
        }

        return Start(Routine(), config);
    }

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

        IEnumerator Routine()
        {
            int currentCount = 0;
            while (count < 0 || currentCount < count)
            {
                onRepeat.Invoke();
                yield return interval;
                currentCount++;
            }
        }

        return Start(Routine(), config);
    }

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

        IEnumerator Routine()
        {
            yield return new WaitUntil(condition);
            onComplete.Invoke();
        }

        return Start(Routine(), config);
    }

    public static FlowHandle WaitForTask(
        FlowHandle handleToWait,
        Action onComplete,
        TaskConfig? config = null
    )
    {
        if (onComplete == null)
            throw new ArgumentNullException(nameof(onComplete));

        IEnumerator Routine()
        {
            while (handleToWait.IsAlive)
            {
                yield return null;
            }
            onComplete.Invoke();
        }

        return Start(Routine(), config);
    }

    public static FlowHandle WhenAll(
        IEnumerable<FlowHandle> handles,
        Action onComplete,
        TaskConfig? config = null
    )
    {
        if (onComplete == null)
            throw new ArgumentNullException(nameof(onComplete));

        IEnumerator Routine()
        {
            foreach (var handle in handles)
            {
                while (handle.IsAlive)
                {
                    yield return null;
                }
            }
            onComplete.Invoke();
        }

        return Start(Routine(), config);
    }

    public static bool Cancel(FlowHandle handle)
    {
        lock (_lockObject)
        {
            if (
                _taskMap.TryGetValue(handle.Id, out FlowTask task)
                && task.Generation == handle.Generation
            )
            {
                task.Cancel();
                _activeTasks.Remove(task);
                _taskMap.Remove(task.Id);
                RecycleTask(task);
                return true;
            }
        }
        return false;
    }

    public static bool Pause(FlowHandle handle)
    {
        lock (_lockObject)
        {
            if (
                _taskMap.TryGetValue(handle.Id, out FlowTask task)
                && task.Generation == handle.Generation
            )
            {
                task.Pause();
                return true;
            }
        }
        return false;
    }

    public static bool Resume(FlowHandle handle)
    {
        lock (_lockObject)
        {
            if (
                _taskMap.TryGetValue(handle.Id, out FlowTask task)
                && task.Generation == handle.Generation
            )
            {
                task.Resume();
                return true;
            }
        }
        return false;
    }

    public static void CancelAll()
    {
        lock (_lockObject)
        {
            var tasks = new List<FlowTask>(_activeTasks);
            foreach (var task in tasks)
            {
                task.Cancel();
                _activeTasks.Remove(task);
                _taskMap.Remove(task.Id);
                RecycleTask(task);
            }
        }
    }

    public static void PauseAll()
    {
        lock (_lockObject)
        {
            foreach (var task in _activeTasks)
            {
                if (task.State == TaskState.Running)
                {
                    task.Pause();
                }
            }
        }
    }

    public static void ResumeAll()
    {
        lock (_lockObject)
        {
            foreach (var task in _activeTasks)
            {
                if (task.State == TaskState.Paused)
                {
                    task.Resume();
                }
            }
        }
    }

    #endregion

    #region 内部工具与对象池

    private static FlowHandle InternalStartTask(
        IEnumerator routine,
        Func<CancellationToken, Task> asyncFunc,
        TaskConfig config
    )
    {
        FlowTask task;
        lock (_lockObject)
        {
            task = _taskPool.Count > 0 ? _taskPool.Pop() : new FlowTask();
            int taskId = _nextTaskId++;
            if (_nextTaskId == int.MaxValue)
                _nextTaskId = 1;

            if (routine != null)
                task.Initialize(taskId, routine, config);
            else if (asyncFunc != null)
                task.Initialize(taskId, asyncFunc, config);

            _activeTasks.AddLast(task);
            _taskMap[taskId] = task;
            TotalTasksCreated++;

#if UNITY_EDITOR || DEVELOPMENT_BUILD
            TrackTaskHistory(task);
#endif

            return new FlowHandle(taskId, task.Generation);
        }
    }

    private static void RecycleTask(FlowTask task)
    {
        task.ResetInternalState();
        if (_taskPool.Count < _maxPoolSize)
        {
            _taskPool.Push(task);
        }
    }

    #region 动态对象池策略
    private static int _poolTargetSize;
    private const float POOL_GROWTH_FACTOR = 1.5f;
    private const float POOL_SHRINK_FACTOR = 0.7f;

    private static IEnumerator ManagementRoutine()
    {
        var waitForPoolAdjust = new WaitForSecondsRealtime(POOL_ADJUST_INTERVAL);
        var waitForAutoTrim = new WaitForSecondsRealtime(AUTO_TRIM_INTERVAL);

        while (true)
        {
            yield return waitForPoolAdjust;
            AdjustPoolSize();

            yield return waitForAutoTrim;
            TrimPool();
        }
    }

    private static void AdjustPoolSize()
    {
        lock (_lockObject)
        {
            // 基于使用率动态调整池大小
            float usageRatio = (float)TotalTasksCreated / Mathf.Max(1, TotalTasksCompleted);

            if (usageRatio > 1.2f && _poolTargetSize < _maxPoolSize)
            {
                // 使用率增长，扩大池
                _poolTargetSize = Mathf.Min(
                    _maxPoolSize,
                    Mathf.CeilToInt(_poolTargetSize * POOL_GROWTH_FACTOR)
                );
            }
            else if (usageRatio < 0.8f && _poolTargetSize > _initialPoolSize)
            {
                // 使用率下降，缩小池
                _poolTargetSize = Mathf.Max(
                    _initialPoolSize,
                    Mathf.FloorToInt(_poolTargetSize * POOL_SHRINK_FACTOR)
                );
            }

            // 确保实际池大小匹配目标
            while (_taskPool.Count > _poolTargetSize)
            {
                _taskPool.Pop();
            }

            while (_taskPool.Count < _poolTargetSize)
            {
                _taskPool.Push(new FlowTask());
            }
        }
    }

    private static void TrimPool()
    {
        lock (_lockObject)
        {
            int trimCount = Math.Max(0, _taskPool.Count - _poolTargetSize);
            for (int i = 0; i < trimCount; i++)
            {
                _taskPool.Pop();
            }
        }
    }
    #endregion

    #region 增强的任务生命周期跟踪
#if UNITY_EDITOR || DEVELOPMENT_BUILD
    private static readonly List<FlowTask> _taskHistory = new List<FlowTask>(100);
    private const int MAX_HISTORY = 200;

    private static void TrackTaskHistory(FlowTask task)
    {
        lock (_lockObject)
        {
            if (_taskHistory.Count >= MAX_HISTORY)
            {
                _taskHistory.RemoveAt(0);
            }
            _taskHistory.Add(task);
        }
    }

    public static IEnumerable<string> GetTaskHistory(int maxCount = 20)
    {
        lock (_lockObject)
        {
            int count = Math.Min(maxCount, _taskHistory.Count);
            for (int i = _taskHistory.Count - 1; i >= _taskHistory.Count - count; i--)
            {
                var task = _taskHistory[i];
                yield return $"[{task.Id}] {task.Config.Name} - {task.State} ({task.GetElapsedTime():F2}s)";
            }
        }
    }
#endif
    #endregion

    #region 调试API

    public static int ActiveTaskCount
    {
        get
        {
            lock (_lockObject)
            {
                return _activeTasks.Count;
            }
        }
    }

    public static int TaskPoolCount
    {
        get
        {
            lock (_lockObject)
            {
                return _taskPool.Count;
            }
        }
    }

    public static string PerformanceSummary
    {
        get
        {
            return $"Tasks: {TotalTasksCreated} created, {TotalTasksCompleted} completed, {ActiveTaskCount} active\n"
                + $"Peak: {PeakActiveTasks} active tasks\n"
                + $"Pool: {TaskPoolCount}/{_maxPoolSize}";
        }
    }

    public static List<string> GetActiveTaskDetails()
    {
        lock (_lockObject)
        {
            var details = new List<string>();
            foreach (var task in _activeTasks)
            {
                string exceptionInfo =
                    task.Exception != null ? $" | Exception: {task.Exception.Message}" : "";
                details.Add(
                    $"ID: {task.Id} | State: {task.State} | Elapsed: {task.GetElapsedTime():F2}s | "
                        + $"Name: {(string.IsNullOrEmpty(task.Config.Name) ? "N/A" : task.Config.Name)} | "
                        + $"Priority: {task.Config.Priority}{exceptionInfo}"
                );
            }
            return details;
        }
    }

    public static FlowHandle? FindTaskByName(string name)
    {
        if (string.IsNullOrEmpty(name))
            return null;

        lock (_lockObject)
        {
            foreach (var task in _activeTasks)
            {
                if (task.Config.Name == name && task.State != TaskState.Completed)
                {
                    return new FlowHandle(task.Id, task.Generation);
                }
            }
        }
        return null;
    }

    #endregion

    #region 场景持久化配置
    [Serializable]
    public class SchedulerSettings
    {
        public bool DontDestroyOnLoad = true;
        public InjectionPoint InjectionPoint = InjectionPoint.AfterPreLateUpdate;
        public string CustomInjectionPoint;
        public int InitialPoolSize = 50;
        public int MaxPoolSize = 500;
    }

    private static SchedulerSettings _settings;

    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]
    private static void LoadSettings()
    {
        // 在实际项目中，这里可以从配置文件加载设置
        _settings = new SchedulerSettings();

        // 应用设置
        ConfigurePlayerLoopInjection(_settings.InjectionPoint, _settings.CustomInjectionPoint);

        // 更新池大小
        _initialPoolSize = _settings.InitialPoolSize;
        _maxPoolSize = _settings.MaxPoolSize;
        _poolTargetSize = _initialPoolSize;
    }

    public static void Configure(SchedulerSettings settings)
    {
        if (_isPlayerLoopInitialized)
        {
            Debug.LogWarning(
                "[FlowScheduler] Already initialized. Some settings may not take effect."
            );
        }

        _settings = settings;
        LoadSettings();
    }
    #endregion

    #endregion
}


```
