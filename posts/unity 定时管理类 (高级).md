---
title: "unity 定时管理类 (高级)"
titleColor: "#aaa,#20B2AA"
titleIcon: "asset:unity"
tags: ["Untiy"]
categories: ["Code"]
description: "unity 定时管理类 (高级)"
publishDate: "2023-03-10"
updatedDate: "2023-03-10"
password: "1231234"
---

#### TimeManager 的使用

```csharp
// 延迟：
int _delayId = TimeManager.Instance.CreateDelay(
    10f,
    (id) =>
    {
        TimeManager.Instance.CancelDelay(_repeatId);
    },
    name: "delay"
);
// 重复：
int _repeatId = TimeManager.Instance.CreateDelay(
    1f,
    (id) =>
    {
        Debug.Log("repeat");
    },
    repeat: true,
    maxRepeats: 3,
    name: "repeat"
);
// 帧延迟：
int _frameDelayId = TimeManager.Instance.DelayFrames(10, (id) =>
{
    Debug.Log("10帧后执行");
});
// 暂停任务
TimeManager.Instance.SetPaused(_taskId, true);
// 恢复任务
TimeManager.Instance.SetPaused(_taskId, false);
// 取消任务
TimeManager.Instance.CancelDelay(_taskId);
// 停止所有任务
TimeManager.Instance.StopAllDelays(); // 使用 StopAllDelays()会取消所有任务，包括重复任务
// 获取任务进度
float progress = TimeManager.Instance.GetTaskProgress(_taskId);
if(progress.HasValue)
Debug.Log($"任务进度: {progress.Value * 100}%");
// 获取任务状态
DelayTaskState state = TimeManager.Instance.GetTaskState(_taskId);
if(state.HasValue)
Debug.Log($"任务状态: {state.Value}");
// 获取任务名称
string taskName = TimeManager.Instance.GetTaskName(taskId);
// 获取已用时间
float? elapsed = TimeManager.Instance.GetElapsedTime(taskId);
// 检查是否暂停
bool isPaused = TimeManager.Instance.IsTaskPaused(taskId);
// 获取执行次数
int count = TimeManager.Instance.GetTaskExecutionCount(taskId);
// 当目标对象被销毁时，任务会自动取消：
TimeManager.Instance.DelaySeconds(5.0f, (id) =>
{
    Debug.Log("5秒后执行");
}, target: this.gameObject);

```

#### TimeManager

```csharp
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Linq;
using UnityEngine;

[DefaultExecutionOrder(-100)]
public class TimeManager : MonoBehaviour
{
    // 单例实现
    private static TimeManager _instance;
    public static TimeManager Instance => _instance ??= InitializeInstance();

    private static TimeManager InitializeInstance()
    {
        var existing = FindObjectOfType<TimeManager>();
        if (existing)
            return existing;

        var go = new GameObject("TimeManager");
        var instance = go.AddComponent<TimeManager>();
        DontDestroyOnLoad(go);
        return instance;
    }

    // 任务管理数据结构
    private readonly Dictionary<int, TimeTask> _tasks = new Dictionary<int, TimeTask>();
    private readonly Queue<TimeTask> _pendingAdd = new Queue<TimeTask>();
    private readonly HashSet<int> _pendingRemove = new HashSet<int>();
    private readonly Queue<TimeTask> _taskPool = new Queue<TimeTask>();
    private int _idCounter = 1;

    // 线程安全锁
    private static readonly object _lock = new object();

    // 时间缩放
    [SerializeField, Tooltip("全局时间缩放因子，影响所有延时任务")]
    private float _timeScale = 1f;
    public float TimeScale
    {
        get => _timeScale;
        set => _timeScale = Mathf.Max(0, value);
    }

    // 高精度计时器
    private Stopwatch _stopwatch;
    private const double MAX_TIME = double.MaxValue / 2; // 防止长时间运行溢出

    // 任务池大小限制
    private const int MAX_POOL_SIZE = 1024;

    void Awake()
    {
        if (_instance != null && _instance != this)
        {
            Destroy(gameObject);
            return;
        }

        _instance = this;
        _stopwatch = Stopwatch.StartNew();
    }

    void OnDestroy()
    {
        if (_instance == this)
        {
            StopAllDelays();
            _instance = null;
        }
    }

    void Update()
    {
        ProcessPendingTasks();
        UpdateActiveTasks();
        CleanupRemovedTasks();
    }

    #region 核心任务处理
    private void ProcessPendingTasks()
    {
        lock (_lock)
        {
            while (_pendingAdd.Count > 0)
            {
                var task = _pendingAdd.Dequeue();
                if (!_tasks.ContainsKey(task.Id))
                {
                    _tasks.Add(task.Id, task);
                    task.State = DelayTaskState.Active;
                }
            }
        }
    }

    private void UpdateActiveTasks()
    {
        double currentTime = GetPreciseTime();
        List<TimeTask> tasksToUpdate = new List<TimeTask>(_tasks.Count); // 预分配容量以减少重分配

        lock (_lock)
        {
            // 手动收集活跃任务，避免LINQ开销
            foreach (var task in _tasks.Values)
            {
                if (
                    task.State == DelayTaskState.Active
                    && !task.IsPaused
                    && !_pendingRemove.Contains(task.Id)
                )
                {
                    tasksToUpdate.Add(task);
                }
            }
        }

        foreach (var task in tasksToUpdate)
        {
            if (task.IsFrameBased)
            {
                task.CurrentFrame++;
                if (task.CurrentFrame >= task.FrameDuration)
                {
                    ProcessTaskExecution(task, currentTime);
                }
            }
            else
            {
                double elapsed = (currentTime - task.StartTime) * _timeScale;
                task.ElapsedTime = elapsed;

                if (elapsed >= task.Duration)
                {
                    ProcessTaskExecution(task, currentTime);
                }
            }
        }
    }

    private void CleanupRemovedTasks()
    {
        lock (_lock)
        {
            if (_pendingRemove.Count > 0)
            {
                foreach (var id in _pendingRemove.ToList())
                {
                    if (_tasks.TryGetValue(id, out var task))
                    {
                        ReturnTaskToPool(task);
                        _tasks.Remove(id);
                    }
                    _pendingRemove.Remove(id);
                }
            }
        }
    }

    private void ProcessTaskExecution(TimeTask task, double currentTime)
    {
        if (!IsTargetValid(task.Target))
        {
            UnityEngine.Debug.LogWarning(
                $"任务 {task.Id} ({task.Name}) 的目标对象已被销毁，跳过执行"
            );
            lock (_lock)
            {
                _pendingRemove.Add(task.Id);
            }
            return;
        }

        try
        {
            task.Action?.Invoke(task.Id);
            task.Callback?.Invoke(task.Id, true);
        }
        catch (Exception ex)
        {
            UnityEngine.Debug.LogError(
                $"任务 {task.Id} ({task.Name}) 回调异常: {ex.Message}\n{ex.StackTrace}"
            );
            task.Callback?.Invoke(task.Id, false, ex);
            lock (_lock)
            {
                _pendingRemove.Add(task.Id);
            }
            return;
        }

        task.ExecutionCount++;

        if (task.IsRepeating)
        {
            if (task.MaxRepeats > 0 && task.ExecutionCount >= task.MaxRepeats)
            {
                task.State = DelayTaskState.Completed;
                task.Callback?.Invoke(task.Id, true);
                lock (_lock)
                {
                    _pendingRemove.Add(task.Id);
                }
                return;
            }

            if (task.IsFrameBased)
            {
                task.CurrentFrame = 0;
            }
            else
            {
                // 精确计算下次触发时间（考虑时间缩放）
                double scaledDuration = task.Duration / _timeScale;
                double triggerTime = task.StartTime + scaledDuration;
                task.StartTime = triggerTime;
                task.ElapsedTime = 0;
            }
        }
        else
        {
            task.State = DelayTaskState.Completed;
            task.Callback?.Invoke(task.Id, true);
            lock (_lock)
            {
                _pendingRemove.Add(task.Id);
            }
        }
    }
    #endregion

    #region 公共接口
    public int CreateDelay(
        float duration,
        Action<int> action,
        bool repeat = false,
        int maxRepeats = -1,
        UnityEngine.Object target = null,
        string name = "",
        TaskCallback callback = null,
        bool isFrameBased = false,
        int frameDuration = 0
    )
    {
        if (!isFrameBased && duration <= 0)
        {
            action?.Invoke(-1);
            callback?.Invoke(-1, false);
            return -1;
        }

        if (isFrameBased && frameDuration <= 0)
        {
            action?.Invoke(-1);
            callback?.Invoke(-1, false);
            return -1;
        }

        var task = GetTaskFromPool();
        int newId;

        lock (_lock)
        {
            newId = _idCounter++;
            task.Initialize(
                id: newId,
                duration: duration,
                action: action,
                isRepeating: repeat,
                maxRepeats: maxRepeats,
                target: target,
                startTime: GetPreciseTime(),
                name: name,
                callback: callback,
                isFrameBased: isFrameBased,
                frameDuration: frameDuration
            );
            _pendingAdd.Enqueue(task);
        }

        return newId;
    }

    public bool CancelDelay(int id)
    {
        if (id <= 0)
            return false;

        lock (_lock)
        {
            if (_tasks.TryGetValue(id, out var task) && !_pendingRemove.Contains(id))
            {
                task.State = DelayTaskState.Cancelled;
                task.Callback?.Invoke(id, false);
                _pendingRemove.Add(id);
                return true;
            }
        }
        return false;
    }

    public void StopAllDelays()
    {
        lock (_lock)
        {
            _pendingAdd.Clear();

            foreach (var taskId in _tasks.Keys.ToList())
            {
                if (!_pendingRemove.Contains(taskId))
                {
                    if (_tasks.TryGetValue(taskId, out var task))
                    {
                        task.State = DelayTaskState.Cancelled;
                        task.Callback?.Invoke(taskId, false);
                    }
                    _pendingRemove.Add(taskId);
                }
            }

            UnityEngine.Debug.Log($"已停止所有延时任务 (总数: {_tasks.Count})");
        }
    }

    public bool SetPaused(int id, bool paused)
    {
        if (id <= 0)
            return false;

        lock (_lock)
        {
            if (_tasks.TryGetValue(id, out var task))
            {
                task.IsPaused = paused;
                if (paused)
                {
                    task.State = DelayTaskState.Paused;
                    task.PausedElapsed = task.ElapsedTime;
                    task.PausedFrame = task.CurrentFrame;
                }
                else
                {
                    if (task.IsFrameBased)
                    {
                        task.CurrentFrame = task.PausedFrame;
                    }
                    else
                    {
                        task.StartTime = GetPreciseTime() - (task.PausedElapsed / _timeScale);
                        task.ElapsedTime = task.PausedElapsed;
                    }
                    task.State = DelayTaskState.Active;
                }
                return true;
            }
        }
        return false;
    }

    public int DelayFrames(
        int frames,
        Action<int> action,
        UnityEngine.Object target = null,
        string name = ""
    )
    {
        return CreateDelay(
            duration: 0f,
            action: action,
            repeat: false,
            maxRepeats: -1,
            target: target,
            name: name,
            isFrameBased: true,
            frameDuration: frames
        );
    }

    public int DelaySeconds(
        float seconds,
        Action<int> action,
        UnityEngine.Object target = null,
        string name = ""
    )
    {
        return CreateDelay(duration: seconds, action: action, target: target, name: name);
    }
    #endregion

    #region 查询方法
    public string GetTaskName(int id)
    {
        lock (_lock)
        {
            return _tasks.TryGetValue(id, out var t) ? t.Name : null;
        }
    }

    public float? GetElapsedTime(int id)
    {
        lock (_lock)
        {
            return _tasks.TryGetValue(id, out var t) ? (float)t.ElapsedTime : null;
        }
    }

    public float? GetTaskProgress(int id)
    {
        lock (_lock)
        {
            if (_tasks.TryGetValue(id, out var t))
            {
                if (t.IsFrameBased)
                {
                    return Mathf.Clamp01((float)t.CurrentFrame / t.FrameDuration);
                }
                else
                {
                    return Mathf.Clamp01((float)(t.ElapsedTime / t.Duration));
                }
            }
            return null;
        }
    }

    public float GetTaskDuration(int id)
    {
        lock (_lock)
        {
            return _tasks.TryGetValue(id, out var t) ? (float)t.Duration : 0f;
        }
    }

    public bool IsTaskPaused(int id)
    {
        lock (_lock)
        {
            return _tasks.TryGetValue(id, out var t) && t.IsPaused;
        }
    }

    public bool IsTaskRepeating(int id)
    {
        lock (_lock)
        {
            return _tasks.TryGetValue(id, out var t) && t.IsRepeating;
        }
    }

    public int GetTaskExecutionCount(int id)
    {
        lock (_lock)
        {
            return _tasks.TryGetValue(id, out var t) ? t.ExecutionCount : 0;
        }
    }

    public int GetTaskMaxRepeats(int id)
    {
        lock (_lock)
        {
            return _tasks.TryGetValue(id, out var t) ? t.MaxRepeats : 0;
        }
    }

    public DelayTaskState? GetTaskState(int id)
    {
        lock (_lock)
        {
            return _tasks.TryGetValue(id, out var t) ? t.State : null;
        }
    }

    public int ActiveTaskCount
    {
        get
        {
            lock (_lock)
            {
                return _tasks.Count;
            }
        }
    }

    public int TotalCreatedTasks
    {
        get
        {
            lock (_lock)
            {
                return _idCounter - 1;
            }
        }
    }

    public List<int> GetAllActiveTaskIds()
    {
        lock (_lock)
        {
            return new List<int>(_tasks.Keys);
        }
    }
    #endregion

    #region 辅助方法
    private double GetPreciseTime()
    {
        // 防止长时间运行溢出
        return _stopwatch.Elapsed.TotalSeconds % MAX_TIME;
    }

    private bool IsTargetValid(UnityEngine.Object target)
    {
        // Unity特定的null检查（处理销毁的Unity对象）
        return target == null ? true : target != null;
    }

    private TimeTask GetTaskFromPool()
    {
        lock (_lock)
        {
            return _taskPool.Count > 0 ? _taskPool.Dequeue() : new TimeTask();
        }
    }

    private void ReturnTaskToPool(TimeTask task)
    {
        lock (_lock)
        {
            if (_taskPool.Count < MAX_POOL_SIZE)
            {
                task.Reset();
                _taskPool.Enqueue(task);
            }
            // 池满时直接丢弃，无需警告以减少日志开销
        }
    }
    #endregion
}

public class TimeTask
{
    public int Id { get; private set; }
    public double Duration { get; private set; }
    public Action<int> Action { get; private set; }
    public bool IsRepeating { get; private set; }
    public int MaxRepeats { get; private set; }
    public int ExecutionCount { get; set; }
    public double ElapsedTime { get; set; }
    public double PausedElapsed { get; set; } // 暂停时保存的已用时间
    public int CurrentFrame { get; set; }
    public int FrameDuration { get; private set; }
    public int PausedFrame { get; set; } // 暂停时保存的已用帧数
    public bool IsFrameBased { get; private set; }
    public double StartTime { get; set; }
    public bool IsPaused { get; set; }
    public UnityEngine.Object Target { get; private set; }
    public DelayTaskState State { get; set; } = DelayTaskState.Pending;
    public string Name { get; private set; }
    public TaskCallback Callback { get; private set; }

    public void Initialize(
        int id,
        float duration,
        Action<int> action,
        bool isRepeating,
        int maxRepeats,
        UnityEngine.Object target,
        double startTime,
        string name = "",
        TaskCallback callback = null,
        bool isFrameBased = false,
        int frameDuration = 0
    )
    {
        Id = id;
        Duration = duration;
        Action = action;
        IsRepeating = isRepeating;
        MaxRepeats = maxRepeats;
        Target = target;
        ExecutionCount = 0;
        StartTime = startTime;
        ElapsedTime = 0.0;
        PausedElapsed = 0.0;
        CurrentFrame = 0;
        PausedFrame = 0;
        FrameDuration = frameDuration;
        IsFrameBased = isFrameBased;
        IsPaused = false;
        State = DelayTaskState.Pending;
        Name = name;
        Callback = callback;
    }

    public void Reset()
    {
        Id = 0;
        Duration = 0;
        Action = null;
        IsRepeating = false;
        MaxRepeats = -1;
        Target = null;
        ExecutionCount = 0;
        ElapsedTime = 0.0;
        PausedElapsed = 0.0;
        CurrentFrame = 0;
        PausedFrame = 0;
        FrameDuration = 0;
        IsFrameBased = false;
        StartTime = 0;
        IsPaused = false;
        State = DelayTaskState.Pending;
        Name = "";
        Callback = null;
    }
}

public enum DelayTaskState
{
    Pending,
    Active,
    Paused,
    Completed,
    Cancelled,
}

public delegate void TaskCallback(int taskId, bool success, Exception ex = null);

```

#### TimeManagerWindow

放入 Assets/Editor 目录下，并在菜单栏 Tools/Time Manager Viewer 中打开。

```csharp
#if UNITY_EDITOR
using UnityEditor;
using UnityEngine;
using System;
using System.Linq;

/// <summary>
/// TimeManager 的可视化调试窗口
/// 支持查看任务状态、暂停、恢复、取消、刷新，
/// 自适应窗口宽度。
/// </summary>
public class TimeManagerWindow : EditorWindow
{
    private Vector2 scrollPos;
    private bool autoRefresh = true;
    private double lastRefreshTime;
    private const double refreshInterval = 0.2; // 每0.3秒刷新一次

    [MenuItem("Tools/Time Manager Viewer")]
    public static void OpenWindow()
    {
        var window = GetWindow<TimeManagerWindow>("Time Manager");
        window.minSize = new Vector2(720, 350);
        window.Show();
    }

    private void OnEnable()
    {
        EditorApplication.update += OnEditorUpdate;
    }

    private void OnDisable()
    {
        EditorApplication.update -= OnEditorUpdate;
    }

    private void OnEditorUpdate()
    {
        if (autoRefresh && EditorApplication.timeSinceStartup - lastRefreshTime > refreshInterval)
        {
            Repaint();
            lastRefreshTime = EditorApplication.timeSinceStartup;
        }
    }

    private void OnGUI()
    {
        var mgr = TimeManager.Instance;
        if (mgr == null)
        {
            EditorGUILayout.HelpBox(
                "未找到 TimeManager 实例。请在场景中运行或手动创建。",
                MessageType.Warning
            );
            return;
        }

        DrawHeader(mgr);

        EditorGUILayout.Space();
        EditorGUILayout.LabelField("🕒 当前活动任务", EditorStyles.boldLabel);
        EditorGUILayout.Space();

        var taskIds = mgr.GetAllActiveTaskIds();
        if (taskIds.Count == 0)
        {
            EditorGUILayout.HelpBox("暂无活动任务。", MessageType.Info);
            return;
        }

        scrollPos = EditorGUILayout.BeginScrollView(scrollPos);

        foreach (var id in taskIds.OrderBy(i => i))
        {
            DrawTaskRow(mgr, id);
        }

        EditorGUILayout.EndScrollView();
    }

    private void DrawHeader(TimeManager mgr)
    {
        // 第一行：任务统计信息
        EditorGUILayout.BeginHorizontal(EditorStyles.helpBox);
        {
            GUILayout.Label($"任务总数: {mgr.TotalCreatedTasks}", GUILayout.Width(120));
            GUILayout.Label($"活动任务: {mgr.ActiveTaskCount}", GUILayout.Width(120));
            GUILayout.FlexibleSpace();
        }
        EditorGUILayout.EndHorizontal();

        // 第二行：时间缩放控制
        EditorGUILayout.BeginHorizontal(EditorStyles.helpBox);
        {
            GUILayout.Label("全局时间缩放:", GUILayout.Width(100));
            float newScale = EditorGUILayout.Slider(mgr.TimeScale, 0f, 5f, GUILayout.Width(300));
            if (!Mathf.Approximately(newScale, mgr.TimeScale))
                mgr.TimeScale = newScale;

            GUILayout.FlexibleSpace();
        }
        EditorGUILayout.EndHorizontal();

        // 第三行：控制按钮
        EditorGUILayout.BeginHorizontal(EditorStyles.helpBox);
        {
            autoRefresh = GUILayout.Toggle(autoRefresh, "自动刷新", GUILayout.Width(90));

            GUI.backgroundColor = Color.yellow;
            if (GUILayout.Button("刷新", GUILayout.Width(70)))
                Repaint();
            GUI.backgroundColor = Color.white;

            GUI.backgroundColor = Color.red;
            if (GUILayout.Button("全部停止", GUILayout.Width(90)))
            {
                if (EditorUtility.DisplayDialog("确认", "确定要停止所有延时任务吗？", "是", "否"))
                    mgr.StopAllDelays();
            }
            GUI.backgroundColor = Color.white;

            GUILayout.FlexibleSpace();
        }
        EditorGUILayout.EndHorizontal();
    }

    private void DrawTaskRow(TimeManager mgr, int id)
    {
        var state = mgr.GetTaskState(id);
        var progress = mgr.GetTaskProgress(id) ?? 0f;
        var elapsed = mgr.GetElapsedTime(id) ?? 0f;
        var repeat = mgr.IsTaskRepeating(id);
        var count = mgr.GetTaskExecutionCount(id);
        var max = mgr.GetTaskMaxRepeats(id);
        var paused = mgr.IsTaskPaused(id);
        var duration = mgr.GetTaskDuration(id);
        var name = mgr.GetTaskName(id) ?? ""; // 获取任务名称

        // 计算剩余时间
        float remainingTime = duration - elapsed;
        if (remainingTime < 0)
            remainingTime = 0;

        // 自适应布局 - 根据窗口宽度调整显示方式
        float windowWidth = EditorGUIUtility.currentViewWidth;
        bool isWideScreen = windowWidth > 800;
        bool isMediumScreen = windowWidth > 600;
        bool isNarrowScreen = windowWidth <= 600;

        EditorGUILayout.BeginVertical("box");
        {
            // 第一行：基本信息
            EditorGUILayout.BeginHorizontal();
            {
                // ID和名称
                GUILayout.Label($"ID: {id}", GUILayout.Width(80));

                // 显示任务名称（如果存在）
                if (!string.IsNullOrEmpty(name))
                {
                    GUILayout.Label($"名称: {name}", GUILayout.Width(isWideScreen ? 200 : 120));
                }

                // 状态显示
                string stateText = state switch
                {
                    DelayTaskState.Active => "▶ 运行中",
                    DelayTaskState.Paused => "⏸ 暂停",
                    DelayTaskState.Pending => "⏱ 等待",
                    _ => state.ToString(),
                };
                GUILayout.Label(stateText, GUILayout.Width(80));

                // 任务类型
                if (repeat)
                {
                    string repeatInfo =
                        max < 0 ? $"重复任务 ({count})" : $"重复任务 ({count}/{max})";
                    GUILayout.Label(repeatInfo, GUILayout.Width(isWideScreen ? 150 : 100));
                }
                else
                {
                    GUILayout.Label("单次任务", GUILayout.Width(80));
                }

                GUILayout.FlexibleSpace();

                // 时间信息
                if (isWideScreen)
                {
                    GUILayout.Label($"总时长: {duration:F2}s", GUILayout.Width(100));
                    GUILayout.Label($"已用: {elapsed:F2}s", GUILayout.Width(80));
                    GUILayout.Label($"剩余: {remainingTime:F2}s", GUILayout.Width(80));
                }
                else if (isMediumScreen)
                {
                    GUILayout.Label($"时长: {duration:F2}s", GUILayout.Width(80));
                    GUILayout.Label($"剩余: {remainingTime:F2}s", GUILayout.Width(80));
                }
            }
            EditorGUILayout.EndHorizontal();

            // 第二行：进度条
            if (duration > 0)
            {
                float progressValue = Mathf.Clamp01(elapsed / duration);
                Rect progressRect = EditorGUILayout.GetControlRect();
                EditorGUI.ProgressBar(
                    progressRect,
                    progressValue,
                    $"进度: {progressValue * 100:F1}%"
                );
            }

            // 第三行：控制按钮
            EditorGUILayout.BeginHorizontal();
            {
                GUILayout.FlexibleSpace();

                // 暂停/恢复按钮
                GUI.backgroundColor = paused
                    ? new Color(0.4f, 0.8f, 0.4f)
                    : new Color(1f, 0.9f, 0.3f);
                if (
                    GUILayout.Button(
                        paused ? "▶ 恢复" : "⏸ 暂停",
                        GUILayout.Width(80),
                        GUILayout.Height(20)
                    )
                )
                    mgr.SetPaused(id, !paused);

                // 取消按钮
                GUI.backgroundColor = new Color(1f, 0.4f, 0.4f);
                if (GUILayout.Button("✖ 取消", GUILayout.Width(70), GUILayout.Height(20)))
                    mgr.CancelDelay(id);

                // 详情按钮（仅在宽屏显示）
                if (isWideScreen)
                {
                    GUI.backgroundColor = new Color(0.6f, 0.6f, 1f);
                    if (GUILayout.Button("ℹ 详情", GUILayout.Width(70), GUILayout.Height(20)))
                    {
                        ShowTaskDetails(mgr, id);
                    }
                }

                GUILayout.FlexibleSpace();
            }
            EditorGUILayout.EndHorizontal();
        }
        EditorGUILayout.EndVertical();

        EditorGUILayout.Space(6);
    }

    private void ShowTaskDetails(TimeManager mgr, int id)
    {
        var state = mgr.GetTaskState(id);
        var elapsed = mgr.GetElapsedTime(id) ?? 0f;
        var duration = mgr.GetTaskDuration(id);
        var repeat = mgr.IsTaskRepeating(id);
        var count = mgr.GetTaskExecutionCount(id);
        var max = mgr.GetTaskMaxRepeats(id);
        var paused = mgr.IsTaskPaused(id);
        var name = mgr.GetTaskName(id) ?? ""; // 获取任务名称

        string message =
            $"任务 {id} 详情:\n"
            + $"名称: {name}\n" // 添加名称显示
            + $"状态: {state}\n"
            + $"类型: {(repeat ? "重复任务" : "单次任务")}\n"
            + $"总时长: {duration:F2}秒\n"
            + $"已用时间: {elapsed:F2}秒\n"
            + $"剩余时间: {duration - elapsed:F2}秒\n"
            + $"暂停状态: {paused}\n";

        if (repeat)
        {
            message += $"执行次数: {count}\n";
            message += $"最大重复次数: {(max < 0 ? "无限" : max.ToString())}";
        }

        EditorUtility.DisplayDialog("任务详情", message, "关闭");
    }
}
#endif

```
