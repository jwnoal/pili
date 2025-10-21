---
title: "unity 对象池自用版"
titleColor: "#aaa,#20B2AA"
titleIcon: "asset:unity"
tags: ["Untiy"]
categories: ["Code"]
description: "unity 对象池"
publishDate: "2023-03-10"
updatedDate: "2023-03-10"
---

#### ObjectPoolManager 的使用

```csharp
public GameObject prefab; // 在Inspector中赋值

void Start()
{
    // 使用默认配置创建池
    ObjectPoolManager.CreatePool(prefab);

    // 或者使用自定义配置
    var config = new ObjectPoolManager.PoolConfig
    {
        PreloadCount = 10,
        AutoScale = true,
        MaxSize = 50,
        DontDestroyOnLoad = true
    };
    ObjectPoolManager.CreatePool(prefab, config);
}

// 获取GameObject实例
GameObject newObjWithParent = ObjectPoolManager.Get(prefab, position, rotation, parentTransform);

// 回收对象
ObjectPoolManager.Return(gameObjectInstance);

// 清理指定池
string poolKey = ObjectPoolManager.GetPoolKey(prefab);
ObjectPoolManager.ClearPool(poolKey);

// 清理所有池
ObjectPoolManager.ClearAllPools();

// 预加载子弹池
ObjectPoolManager.PreloadPool("Bullet_123456", 20);

```

#### ObjectPoolManager

```csharp
using System;
using System.Collections;
using System.Collections.Concurrent;
using System.Collections.Generic;
using System.Threading.Tasks;
using UnityEngine;
using UnityEngine.Profiling;
using UnityEngine.SceneManagement;
#if UNITY_EDITOR
using UnityEditor;
#endif

/// <summary>
/// Unity 对象池管理器：高效管理 GameObject 和 Component 的复用，支持预加载、自动裁剪、线程安全和异步操作。
/// </summary>
public static class ObjectPoolManager
{
    #region 核心数据结构与配置

    private static readonly ConcurrentDictionary<string, ObjectPool> _pools =
        new ConcurrentDictionary<string, ObjectPool>();
    private static readonly ConcurrentDictionary<GameObject, PooledObjectInfo> _activeObjects =
        new ConcurrentDictionary<GameObject, PooledObjectInfo>();
    private static readonly ConcurrentDictionary<Component, PooledObjectInfo> _activeComponents =
        new ConcurrentDictionary<Component, PooledObjectInfo>();

    private static Transform _poolRoot;
    private static bool _isInitialized = false;
    private static readonly object _lock = new object(); // 保留少量锁，用于非并发集合

    // 默认配置常量
    private const int DEFAULT_PRELOAD_COUNT = 5;
    private const int DEFAULT_MAX_SIZE = 100;
    private const float DEFAULT_CULL_INTERVAL = 30f;
    private const float DEFAULT_CULL_THRESHOLD = 0.2f;
    private const int DEFAULT_CULL_AMOUNT = 5;
    private const float DEFAULT_AUTO_SCALE_FACTOR = 1.2f; // 新增：自动扩展因子
    private const float OBJECT_AGE_CULL_THRESHOLD = 60f; // 新增：对象年龄阈值（秒）

    // 性能计数器
    public static int TotalObjectsCreated { get; private set; }
    public static int TotalObjectsReused { get; private set; }
    public static int TotalObjectsDestroyed { get; private set; }
    public static int PeakActiveObjects { get; private set; }

    // 新增：事件钩子，便于外部监听
    public static event Action<GameObject> OnObjectSpawned;
    public static event Action<GameObject> OnObjectReturned;
    public static event Action<string> OnPoolAutoScaled; // 新增：池自动扩展事件
    #endregion

    #region 初始化与清理

    [RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.SubsystemRegistration)]
    private static void Initialize()
    {
        lock (_lock)
        {
            if (_isInitialized)
                return;

            // 创建池根对象
            var poolRootObj = new GameObject("ObjectPoolManager_Root");
            if (Application.isPlaying)
            {
                poolRootObj.hideFlags = HideFlags.HideInHierarchy;
            }
            UnityEngine.Object.DontDestroyOnLoad(poolRootObj);
            _poolRoot = poolRootObj.transform;

            // 注册事件
            SceneManager.sceneUnloaded += OnSceneUnloaded;
            SceneManager.sceneLoaded += OnSceneLoaded;

            // 启动管理协程
            StartManagementCoroutine();

#if UNITY_EDITOR
            EditorApplication.playModeStateChanged += OnPlayModeStateChanged;
#endif

            _isInitialized = true;

#if UNITY_EDITOR || DEVELOPMENT_BUILD
            Debug.Log("[ObjectPoolManager] Initialized");
#endif
        }
    }

#if UNITY_EDITOR
    private static void OnPlayModeStateChanged(PlayModeStateChange state)
    {
        if (state == PlayModeStateChange.ExitingPlayMode)
        {
            ClearAllPools(); // 彻底清理
        }
    }
#endif

    private static void StartManagementCoroutine()
    {
        var managerObj = new GameObject("ObjectPoolManager_CoroutineRunner");
        managerObj.transform.SetParent(_poolRoot);
        var runner = managerObj.AddComponent<PoolManagerCoroutineRunner>();
        runner.StartCoroutine(ManagementRoutine());
    }

    private static void OnSceneUnloaded(Scene scene)
    {
        foreach (var pair in _pools)
        {
            if (!pair.Value.Config.DontDestroyOnLoad)
            {
                ClearPool(pair.Key);
            }
        }
        DetectLeaks(); // 新增：场景卸载时检测泄漏
    }

    private static void OnSceneLoaded(Scene scene, LoadSceneMode mode)
    {
        if (Application.isPlaying)
        {
            TotalObjectsCreated = 0;
            TotalObjectsReused = 0;
            TotalObjectsDestroyed = 0;
            PeakActiveObjects = 0;
        }
    }

    #endregion

    #region 公共API - 池管理

    /// <summary>
    /// 创建新对象池 (Component版本)。
    /// </summary>
    /// <typeparam name="T">组件类型。</typeparam>
    /// <param name="prefab">预制体。</param>
    /// <param name="config">池配置（可选）。</param>
    public static void CreatePool<T>(T prefab, PoolConfig config = null)
        where T : Component
    {
        if (prefab == null)
        {
#if UNITY_EDITOR || DEVELOPMENT_BUILD
            Debug.LogError("[ObjectPoolManager] Cannot create pool with null prefab");
#endif
            return;
        }

        string poolKey = GetPoolKey(prefab);

        if (_pools.ContainsKey(poolKey))
        {
#if UNITY_EDITOR || DEVELOPMENT_BUILD
            Debug.LogWarning($"[ObjectPoolManager] Pool already exists for {poolKey}");
#endif
            return;
        }

        var pool = new ComponentPool<T>(prefab, config ?? PoolConfig.Default);
        _pools.TryAdd(poolKey, pool);

        if (pool.Config.PreloadCount > 0 && Application.isPlaying)
        {
            PreloadPool(poolKey, pool.Config.PreloadCount);
        }
    }

    /// <summary>
    /// 创建新对象池 (GameObject版本)。
    /// </summary>
    /// <param name="prefab">预制体。</param>
    /// <param name="config">池配置（可选）。</param>
    public static void CreatePool(GameObject prefab, PoolConfig config = null)
    {
        if (prefab == null)
        {
#if UNITY_EDITOR || DEVELOPMENT_BUILD
            Debug.LogError("[ObjectPoolManager] Cannot create pool with null prefab");
#endif
            return;
        }

        string poolKey = GetPoolKey(prefab);

        if (_pools.ContainsKey(poolKey))
        {
#if UNITY_EDITOR || DEVELOPMENT_BUILD
            Debug.LogWarning($"[ObjectPoolManager] Pool already exists for {poolKey}");
#endif
            return;
        }

        var pool = new GameObjectPool(prefab, config ?? PoolConfig.Default);
        _pools.TryAdd(poolKey, pool);

        if (pool.Config.PreloadCount > 0 && Application.isPlaying)
        {
            PreloadPool(poolKey, pool.Config.PreloadCount);
        }
    }

    /// <summary>
    /// 预热对象池。
    /// </summary>
    /// <param name="poolKey">池键。</param>
    /// <param name="count">预加载数量（-1 使用配置默认）。</param>
    public static void PreloadPool(string poolKey, int count = -1)
    {
        if (!_pools.TryGetValue(poolKey, out ObjectPool pool))
        {
#if UNITY_EDITOR || DEVELOPMENT_BUILD
            Debug.LogWarning($"[ObjectPoolManager] Pool not found: {poolKey}");
#endif
            return;
        }

        int preloadCount = count > 0 ? count : pool.Config.PreloadCount;
        preloadCount = Mathf.Min(preloadCount, pool.Config.MaxSize - pool.PooledCount);

        for (int i = 0; i < preloadCount; i++)
        {
            var obj = pool.CreateInstance();
            if (obj != null)
            {
                pool.ReturnInstance(obj);
            }
        }
    }

    /// <summary>
    /// 获取对象实例 (Component版本)。
    /// </summary>
    /// <typeparam name="T">组件类型。</typeparam>
    /// <param name="prefab">预制体。</param>
    /// <param name="position">位置。</param>
    /// <param name="rotation">旋转。</param>
    /// <param name="parent">父级（可选）。</param>
    /// <returns>组件实例。</returns>
    public static T Get<T>(T prefab, Vector3 position, Quaternion rotation, Transform parent = null)
        where T : Component
    {
        AssertMainThread(); // 新增：主线程检查
        return InternalGet(prefab, position, rotation, parent);
    }

    private static T InternalGet<T>(
        T prefab,
        Vector3 position,
        Quaternion rotation,
        Transform parent
    )
        where T : Component
    {
        if (prefab == null)
        {
#if UNITY_EDITOR || DEVELOPMENT_BUILD
            Debug.LogError("[ObjectPoolManager] Cannot get instance with null prefab");
#endif
            return null;
        }

#if UNITY_EDITOR
        if (!Application.isPlaying)
        {
            return UnityEngine.Object.Instantiate(prefab, position, rotation, parent);
        }
#endif

        string poolKey = GetPoolKey(prefab);
        GameObject targetObj = null;
        T component = null;

        Profiler.BeginSample("ObjectPoolManager.Get"); // 新增：Profiler标记

        if (!_pools.TryGetValue(poolKey, out ObjectPool pool))
        {
            CreatePool(prefab);
            pool = _pools[poolKey];
        }

        var instance = pool.GetInstance();
        if (instance == null)
        {
#if UNITY_EDITOR || DEVELOPMENT_BUILD
            Debug.LogError($"[ObjectPoolManager] Failed to get instance for {poolKey}");
#endif
            Profiler.EndSample();
            return null;
        }

        if (!(instance is T tempComponent))
        {
#if UNITY_EDITOR || DEVELOPMENT_BUILD
            Debug.LogError($"[ObjectPoolManager] Instance type mismatch for {poolKey}");
#endif
            Profiler.EndSample();
            return null;
        }

        component = tempComponent;
        targetObj = component.gameObject;
        targetObj.transform.SetPositionAndRotation(position, rotation);
        targetObj.transform.SetParent(parent, false);

        targetObj.SetActive(true);

        var info = new PooledObjectInfo(poolKey, targetObj);
        _activeObjects.TryAdd(targetObj, info);
        _activeComponents.TryAdd(component, info);

        // 更新峰值
        PeakActiveObjects = Mathf.Max(PeakActiveObjects, _activeObjects.Count);

        Profiler.EndSample();

        // 调用回调
        TryInvokeOnSpawned(targetObj);

        return component;
    }

    /// <summary>
    /// 获取对象实例 (GameObject版本)。
    /// </summary>
    /// <param name="prefab">预制体。</param>
    /// <param name="position">位置。</param>
    /// <param name="rotation">旋转。</param>
    /// <param name="parent">父级（可选）。</param>
    /// <returns>GameObject 实例。</returns>
    public static GameObject Get(
        GameObject prefab,
        Vector3 position,
        Quaternion rotation,
        Transform parent = null
    )
    {
        AssertMainThread();
        return InternalGet(prefab, position, rotation, parent);
    }

    private static GameObject InternalGet(
        GameObject prefab,
        Vector3 position,
        Quaternion rotation,
        Transform parent
    )
    {
        if (prefab == null)
        {
#if UNITY_EDITOR || DEVELOPMENT_BUILD
            Debug.LogError("[ObjectPoolManager] Cannot get instance with null prefab");
#endif
            return null;
        }

#if UNITY_EDITOR
        if (!Application.isPlaying)
        {
            return UnityEngine.Object.Instantiate(prefab, position, rotation, parent);
        }
#endif

        string poolKey = GetPoolKey(prefab);
        GameObject targetObj = null;

        Profiler.BeginSample("ObjectPoolManager.Get");

        if (!_pools.TryGetValue(poolKey, out ObjectPool pool))
        {
            CreatePool(prefab);
            pool = _pools[poolKey];
        }

        var instance = pool.GetInstance();
        if (instance == null)
        {
#if UNITY_EDITOR || DEVELOPMENT_BUILD
            Debug.LogError($"[ObjectPoolManager] Failed to get instance for {poolKey}");
#endif
            Profiler.EndSample();
            return null;
        }

        targetObj = instance as GameObject ?? (instance as Component)?.gameObject;

        if (targetObj == null)
        {
#if UNITY_EDITOR || DEVELOPMENT_BUILD
            Debug.LogError($"[ObjectPoolManager] Invalid instance for {poolKey}");
#endif
            Profiler.EndSample();
            return null;
        }

        targetObj.transform.SetPositionAndRotation(position, rotation);
        targetObj.transform.SetParent(parent, false);

        targetObj.SetActive(true);

        var info = new PooledObjectInfo(poolKey, targetObj);
        _activeObjects.TryAdd(targetObj, info);

        if (instance is Component component)
        {
            _activeComponents.TryAdd(component, info);
        }

        // 更新峰值
        PeakActiveObjects = Mathf.Max(PeakActiveObjects, _activeObjects.Count);

        Profiler.EndSample();

        // 调用回调
        TryInvokeOnSpawned(targetObj);

        return targetObj;
    }

    /// <summary>
    /// 异步获取对象实例 (Component版本)。支持 Addressables 等异步加载场景。
    /// </summary>
    /// <typeparam name="T">组件类型。</typeparam>
    /// <param name="prefab">预制体。</param>
    /// <param name="position">位置。</param>
    /// <param name="rotation">旋转。</param>
    /// <param name="parent">父级（可选）。</param>
    /// <returns>Task 返回组件实例。</returns>
    public static async Task<T> GetAsync<T>(
        T prefab,
        Vector3 position,
        Quaternion rotation,
        Transform parent = null
    )
        where T : Component
    {
        AssertMainThread(); // 异步但入口需主线程
        await Task.Yield(); // 允许异步，但实际逻辑同步（可扩展到 Addressables）
        return InternalGet(prefab, position, rotation, parent);
    }

    /// <summary>
    /// 异步获取对象实例 (GameObject版本)。
    /// </summary>
    /// <param name="prefab">预制体。</param>
    /// <param name="position">位置。</param>
    /// <param name="rotation">旋转。</param>
    /// <param name="parent">父级（可选）。</param>
    /// <returns>Task 返回 GameObject 实例。</returns>
    public static async Task<GameObject> GetAsync(
        GameObject prefab,
        Vector3 position,
        Quaternion rotation,
        Transform parent = null
    )
    {
        AssertMainThread();
        await Task.Yield();
        return InternalGet(prefab, position, rotation, parent);
    }

    /// <summary>
    /// 回收对象 (GameObject版本)。
    /// </summary>
    /// <param name="obj">要回收的对象。</param>
    public static void Return(GameObject obj)
    {
        AssertMainThread();

        if (obj == null)
            return; // 修复：早检查 null，避免后续异常
#if UNITY_EDITOR
        if (!Application.isPlaying)
        {
            UnityEngine.Object.Destroy(obj);
            return;
        }
#endif

        if (!obj.activeInHierarchy)
            return;

        Profiler.BeginSample("ObjectPoolManager.Return");

        if (!_activeObjects.TryGetValue(obj, out PooledObjectInfo info))
        {
#if UNITY_EDITOR || DEVELOPMENT_BUILD
            Debug.LogWarning($"[ObjectPoolManager] Object not managed by pool: {obj.name}");
#endif
            UnityEngine.Object.Destroy(obj);
            Profiler.EndSample();
            return;
        }

        if (!_pools.TryGetValue(info.PoolKey, out ObjectPool pool))
        {
#if UNITY_EDITOR || DEVELOPMENT_BUILD
            Debug.LogWarning($"[ObjectPoolManager] Pool not found for object: {obj.name}");
#endif
            UnityEngine.Object.Destroy(obj);
            Profiler.EndSample();
            return;
        }

        obj.SetActive(false);
        obj.transform.SetParent(_poolRoot, false);

        _activeObjects.TryRemove(obj, out _);

        // 移除组件引用
        var componentsToRemove = new List<Component>();
        foreach (var pair in _activeComponents)
        {
            if (pair.Value.GameObject == obj)
            {
                componentsToRemove.Add(pair.Key);
            }
        }
        foreach (var comp in componentsToRemove)
        {
            _activeComponents.TryRemove(comp, out _);
        }

        pool.ReturnInstance(obj);

        Profiler.EndSample();

        TryInvokeOnReturned(obj);
    }

    /// <summary>
    /// 回收组件。
    /// </summary>
    /// <param name="component">要回收的组件。</param>
    public static void Return(Component component)
    {
        AssertMainThread();

        if (component == null)
            return;

#if UNITY_EDITOR
        if (!Application.isPlaying)
        {
            UnityEngine.Object.Destroy(component.gameObject);
            return;
        }
#endif

        if (!component.gameObject.activeInHierarchy)
            return;

        if (!_activeComponents.TryGetValue(component, out PooledObjectInfo info))
        {
#if UNITY_EDITOR || DEVELOPMENT_BUILD
            Debug.LogWarning(
                $"[ObjectPoolManager] Component not managed by pool: {component.name}"
            );
#endif
            UnityEngine.Object.Destroy(component.gameObject);
            return;
        }

        Return(info.GameObject);
    }

    /// <summary>
    /// 清理指定池。
    /// </summary>
    /// <param name="poolKey">池键。</param>
    public static void ClearPool(string poolKey)
    {
        if (!_pools.TryGetValue(poolKey, out ObjectPool pool))
            return;

        var activeObjs = new List<GameObject>();
        foreach (var pair in _activeObjects)
        {
            if (pair.Value.PoolKey == poolKey)
            {
                activeObjs.Add(pair.Key);
            }
        }
        foreach (var obj in activeObjs)
        {
            if (obj != null) // 修复：添加 null 检查
            {
                Return(obj);
            }
            else
            {
                _activeObjects.TryRemove(obj, out _); // 移除无效引用
            }
        }

        pool.Clear();
        _pools.TryRemove(poolKey, out _);
    }

    /// <summary>
    /// 清理所有池。
    /// </summary>
    public static void ClearAllPools()
    {
        foreach (var pool in _pools.Values)
        {
            pool.Clear();
        }
        _pools.Clear();

        var activeObjects = new List<GameObject>(_activeObjects.Keys);
        foreach (var obj in activeObjects)
        {
            if (obj != null)
            {
                UnityEngine.Object.Destroy(obj);
            }
        }
        _activeObjects.Clear();
        _activeComponents.Clear();
    }

    #endregion

    #region 内部数据结构

    /// <summary>
    /// 池配置。
    /// </summary>
    public class PoolConfig
    {
        public int PreloadCount = DEFAULT_PRELOAD_COUNT;
        public int MaxSize = DEFAULT_MAX_SIZE;
        public float CullInterval = DEFAULT_CULL_INTERVAL;
        public float CullThreshold = DEFAULT_CULL_THRESHOLD;
        public int CullAmount = DEFAULT_CULL_AMOUNT;
        public bool AllowSceneActivation = true;
        public bool DontDestroyOnLoad = false;
        public bool AutoScale = false; // 新增：自动扩展池大小

        public static PoolConfig Default => new PoolConfig();

        public PoolConfig Clone()
        {
            return new PoolConfig
            {
                PreloadCount = PreloadCount,
                MaxSize = MaxSize,
                CullInterval = CullInterval,
                CullThreshold = CullThreshold,
                CullAmount = CullAmount,
                AllowSceneActivation = AllowSceneActivation,
                DontDestroyOnLoad = DontDestroyOnLoad,
                AutoScale = AutoScale,
            };
        }
    }

    /// <summary>
    /// 池化对象接口。
    /// </summary>
    public interface IPooledObject
    {
        void OnSpawnedFromPool();
        void OnReturnedToPool();
    }

    private class PooledObjectInfo
    {
        public string PoolKey;
        public GameObject GameObject;
        public float SpawnTime;

        public PooledObjectInfo(string poolKey, GameObject gameObject)
        {
            PoolKey = poolKey;
            GameObject = gameObject;
            SpawnTime = Time.time;
        }
    }

    private abstract class ObjectPool
    {
        public PoolConfig Config { get; private set; }
        public int PooledCount => _poolQueue.Count; // 改为 Queue (FIFO)
        public int ActiveCount { get; protected set; }
        public int TotalCreated { get; protected set; }

        protected Queue<object> _poolQueue = new Queue<object>(DEFAULT_MAX_SIZE); // 新增：用 Queue 替换 Stack，FIFO 更好分布使用
        protected object _prefab;
        protected Transform _poolContainer;

        public ObjectPool(object prefab, PoolConfig config)
        {
            _prefab = prefab;
            Config = config;

            string poolName = $"Pool_{prefab.GetType().Name}_{prefab.GetHashCode()}";
            var container = new GameObject(poolName);
            container.transform.SetParent(_poolRoot);
            _poolContainer = container.transform;

            if (Application.isPlaying)
            {
                container.hideFlags = HideFlags.HideInHierarchy;
            }

            if (Config.DontDestroyOnLoad)
            {
                UnityEngine.Object.DontDestroyOnLoad(container);
            }
        }

        public abstract object CreateInstance();

        public virtual object GetInstance()
        {
            object instanceObj;

            if (_poolQueue.Count > 0)
            {
                instanceObj = _poolQueue.Dequeue();
                if (instanceObj == null)
                {
#if UNITY_EDITOR || DEVELOPMENT_BUILD
                    Debug.LogWarning("[ObjectPoolManager] Found null instance in pool");
#endif
                    instanceObj = CreateInstance();
                }
                else
                {
                    TotalObjectsReused++;
                }
            }
            else
            {
                instanceObj = CreateInstance();
                if (Config.AutoScale && TotalCreated >= Config.MaxSize) // 新增：自动扩展
                {
                    Config.MaxSize = Mathf.CeilToInt(Config.MaxSize * DEFAULT_AUTO_SCALE_FACTOR);
                    OnPoolAutoScaled?.Invoke(_prefab.ToString());
                }
            }

            ActiveCount++;
            return instanceObj;
        }

        public virtual void ReturnInstance(object instance)
        {
            if (_poolQueue.Count < Config.MaxSize)
            {
                _poolQueue.Enqueue(instance);
            }
            else
            {
#if UNITY_EDITOR || DEVELOPMENT_BUILD
                Debug.LogWarning($"[ObjectPoolManager] Pool full, destroying {instance}");
#endif
                DestroyInstance(instance);
            }

            ActiveCount--;
        }

        public void Clear()
        {
            while (_poolQueue.Count > 0)
            {
                var instance = _poolQueue.Dequeue();
                DestroyInstance(instance);
            }
        }

        public void Cull(int amount)
        {
            amount = Mathf.Min(amount, _poolQueue.Count - Config.PreloadCount);
            for (int i = 0; i < amount; i++)
            {
                if (_poolQueue.Count == 0)
                    break;
                var instance = _poolQueue.Dequeue();
                DestroyInstance(instance);
            }
        }

        public void UpdateConfig(PoolConfig newConfig)
        {
            if (newConfig.MaxSize < Config.MaxSize && _poolQueue.Count > newConfig.MaxSize)
            {
                int cullAmount = _poolQueue.Count - newConfig.MaxSize;
                Cull(cullAmount);
            }
            Config = newConfig;
        }

        protected abstract void DestroyInstance(object instance);
    }

    private class GameObjectPool : ObjectPool
    {
        public GameObjectPool(GameObject prefab, PoolConfig config)
            : base(prefab, config) { }

        public override object CreateInstance()
        {
            var prefab = (GameObject)_prefab;
            if (prefab == null)
                return null;

            var instance = UnityEngine.Object.Instantiate(prefab, _poolContainer);
            if (instance == null)
                return null;

            instance.SetActive(false);

            TotalCreated++;
            TotalObjectsCreated++;

            return instance;
        }

        protected override void DestroyInstance(object instance)
        {
            var gameObj = (GameObject)instance;
            if (gameObj != null)
            {
                UnityEngine.Object.Destroy(gameObj);
                TotalObjectsDestroyed++;
            }
        }
    }

    private class ComponentPool<T> : ObjectPool
        where T : Component
    {
        public ComponentPool(T prefab, PoolConfig config)
            : base(prefab, config) { }

        public override object CreateInstance()
        {
            var prefab = (T)_prefab;
            if (prefab == null)
                return null;

            var instance = UnityEngine.Object.Instantiate(prefab, _poolContainer);
            if (instance == null)
                return null;

            instance.gameObject.SetActive(false);

            TotalCreated++;
            TotalObjectsCreated++;

            return instance;
        }

        protected override void DestroyInstance(object instance)
        {
            var component = (T)instance;
            if (component != null && component.gameObject != null)
            {
                UnityEngine.Object.Destroy(component.gameObject);
                TotalObjectsDestroyed++;
            }
        }
    }

    private class PoolManagerCoroutineRunner : MonoBehaviour
    {
        // 空类，用于协程
    }

    #endregion

    #region 内部工具方法

    private static string GetPoolKey(GameObject prefab)
    {
        return $"{prefab.name}_{prefab.GetHashCode()}";
    }

    private static string GetPoolKey(Component prefab)
    {
        return $"{prefab.name}_{prefab.GetHashCode()}";
    }

    private static IEnumerator ManagementRoutine()
    {
        while (true)
        {
            if (_pools.Count == 0)
            {
                yield return new WaitForSecondsRealtime(1f);
                continue;
            }

            float minInterval = float.MaxValue;
            foreach (var pool in _pools.Values)
            {
                minInterval = Mathf.Min(minInterval, pool.Config.CullInterval);
            }

            yield return new WaitForSecondsRealtime(minInterval);

            if (Application.isPlaying)
            {
                CullIdlePools();
            }
        }
    }

    private static void CullIdlePools()
    {
        foreach (var poolEntry in _pools)
        {
            var pool = poolEntry.Value;
            float idleRatio =
                (float)pool.PooledCount / Mathf.Max(1, pool.PooledCount + pool.ActiveCount);

            if (
                idleRatio > pool.Config.CullThreshold
                && pool.PooledCount > pool.Config.PreloadCount
            )
            {
                int cullAmount = Mathf.Min(
                    pool.Config.CullAmount,
                    pool.PooledCount - pool.Config.PreloadCount
                );
                pool.Cull(cullAmount);
            }
        }

        // 新增：年龄基于裁剪（优先销毁旧对象）
        CullOldObjects();
    }

    private static void CullOldObjects()
    {
        float currentTime = Time.time;
        var objectsToCull = new List<GameObject>();
        var invalidKeys = new List<GameObject>(); // 新增：收集无效键

        foreach (var pair in _activeObjects)
        {
            var obj = pair.Key;
            if (obj == null) // 修复：检查已销毁对象
            {
                invalidKeys.Add(obj); // 标记移除
                continue;
            }

            if (currentTime - pair.Value.SpawnTime > OBJECT_AGE_CULL_THRESHOLD)
            {
                objectsToCull.Add(obj);
            }
        }

        // 先移除无效引用
        foreach (var invalidKey in invalidKeys)
        {
            _activeObjects.TryRemove(invalidKey, out _);
            // 同时清理组件引用
            var componentsToRemove = new List<Component>();
            foreach (var compPair in _activeComponents)
            {
                if (compPair.Value.GameObject == invalidKey)
                {
                    componentsToRemove.Add(compPair.Key);
                }
            }
            foreach (var comp in componentsToRemove)
            {
                _activeComponents.TryRemove(comp, out _);
            }

#if UNITY_EDITOR || DEVELOPMENT_BUILD
            Debug.LogWarning(
                "[ObjectPoolManager] Removed invalid (destroyed) reference from active objects."
            );
#endif
        }

        // 再处理裁剪
        foreach (var obj in objectsToCull)
        {
            if (obj != null) // 修复：双重检查，避免访问 name 时异常
            {
#if UNITY_EDITOR || DEVELOPMENT_BUILD
                Debug.LogWarning($"[ObjectPoolManager] Culling old object due to age: {obj.name}");
#endif
                Return(obj);
            }
            else
            {
                _activeObjects.TryRemove(obj, out _); // 如果在循环间被毁，移除
            }
        }
    }

    private static void TryInvokeOnSpawned(GameObject obj)
    {
        if (obj == null)
            return; // 修复：添加 null 检查

        try
        {
            var pooled = obj.GetComponent<IPooledObject>();
            pooled?.OnSpawnedFromPool();
            OnObjectSpawned?.Invoke(obj);
        }
        catch (Exception ex)
        {
#if UNITY_EDITOR || DEVELOPMENT_BUILD
            Debug.LogWarning($"[ObjectPoolManager] OnSpawnedFromPool exception: {ex.Message}");
#endif
        }
    }

    private static void TryInvokeOnReturned(GameObject obj)
    {
        if (obj == null)
            return; // 修复：添加 null 检查

        try
        {
            var pooled = obj.GetComponent<IPooledObject>();
            pooled?.OnReturnedToPool();
            OnObjectReturned?.Invoke(obj);
        }
        catch (Exception ex)
        {
#if UNITY_EDITOR || DEVELOPMENT_BUILD
            Debug.LogWarning($"[ObjectPoolManager] OnReturnedToPool exception: {ex.Message}");
#endif
        }
    }

    private static void AssertMainThread()
    {
        if (System.Threading.Thread.CurrentThread.ManagedThreadId != 1)
        {
            throw new InvalidOperationException(
                "[ObjectPoolManager] Must be called on the main thread."
            );
        }
    }

    private static void DetectLeaks()
    {
        if (_activeObjects.Count > 0)
        {
#if UNITY_EDITOR || DEVELOPMENT_BUILD
            Debug.LogWarning(
                $"[ObjectPoolManager] Potential leak detected: {_activeObjects.Count} active objects not returned."
            );
#endif
            // 可选：自动回收，但这里只警告
        }
    }

    #endregion

    #region 调试与统计API

    /// <summary>
    /// 获取指定池的统计信息。
    /// </summary>
    /// <param name="poolKey">池键。</param>
    /// <returns>统计字符串。</returns>
    public static string GetPoolStats(string poolKey)
    {
        if (_pools.TryGetValue(poolKey, out ObjectPool pool))
        {
            return $"{poolKey}: {pool.ActiveCount} active, {pool.PooledCount} pooled, {pool.TotalCreated} created";
        }
        return $"Pool not found: {poolKey}";
    }

    /// <summary>
    /// 获取所有池的统计信息。
    /// </summary>
    /// <returns>字典：键为池键，值为统计字符串。</returns>
    public static Dictionary<string, string> GetAllPoolStats()
    {
        var stats = new Dictionary<string, string>();
        foreach (var pair in _pools)
        {
            var pool = pair.Value;
            stats[pair.Key] =
                $"{pool.ActiveCount} active, {pool.PooledCount} pooled, {pool.TotalCreated} created";
        }
        return stats;
    }

    /// <summary>
    /// 获取全局统计信息。
    /// </summary>
    public static string GlobalStats =>
        $"Pools: {_pools.Count}\nActive Objects: {_activeObjects.Count}\nPeak Active: {PeakActiveObjects}\nCreated: {TotalObjectsCreated}\nReused: {TotalObjectsReused}\nDestroyed: {TotalObjectsDestroyed}";

    /// <summary>
    /// 获取所有活跃对象列表。
    /// </summary>
    /// <returns>活跃 GameObject 列表。</returns>
    public static List<GameObject> GetActiveObjects()
    {
        return new List<GameObject>(_activeObjects.Keys);
    }

    #endregion

    #region 扩展功能

    /// <summary>
    /// 获取指定池的配置。
    /// </summary>
    /// <param name="poolKey">池键。</param>
    /// <returns>池配置副本。</returns>
    public static PoolConfig GetPoolConfig(string poolKey)
    {
        return _pools.TryGetValue(poolKey, out ObjectPool pool) ? pool.Config.Clone() : null;
    }

    /// <summary>
    /// 更新指定池的配置。
    /// </summary>
    /// <param name="poolKey">池键。</param>
    /// <param name="newConfig">新配置。</param>
    public static void UpdatePoolConfig(string poolKey, PoolConfig newConfig)
    {
        if (_pools.TryGetValue(poolKey, out ObjectPool pool))
        {
            pool.UpdateConfig(newConfig.Clone());
        }
    }

    /// <summary>
    /// 获取指定池的大小（池中对象数）。
    /// </summary>
    /// <param name="poolKey">池键。</param>
    /// <returns>池大小。</returns>
    public static int GetPoolSize(string poolKey)
    {
        return _pools.TryGetValue(poolKey, out ObjectPool pool) ? pool.PooledCount : -1;
    }

    /// <summary>
    /// 设置指定池的大小。
    /// </summary>
    /// <param name="poolKey">池键。</param>
    /// <param name="newSize">新大小。</param>
    public static void SetPoolSize(string poolKey, int newSize)
    {
        if (_pools.TryGetValue(poolKey, out ObjectPool pool))
        {
            var newConfig = pool.Config.Clone();
            newConfig.MaxSize = Mathf.Max(newSize, pool.Config.PreloadCount);
            pool.UpdateConfig(newConfig);
        }
    }

    #endregion

    #region 编辑器工具

#if UNITY_EDITOR
    [MenuItem("Tools/ObjectPool/Clear All Pools")]
    private static void EditorClearAllPools()
    {
        ClearAllPools();
        Debug.Log("[ObjectPoolManager] Cleared all pools");
    }

    [MenuItem("Tools/ObjectPool/Print Stats")]
    private static void PrintPoolStats()
    {
        Debug.Log(GlobalStats);
        foreach (var stat in GetAllPoolStats())
        {
            Debug.Log($"{stat.Key}: {stat.Value}");
        }
    }

    [MenuItem("Tools/ObjectPool/Show Pool Root")]
    private static void ShowPoolRoot()
    {
        if (_poolRoot != null)
        {
            _poolRoot.gameObject.hideFlags = HideFlags.None;
            Debug.Log("[ObjectPoolManager] Pool root shown in hierarchy");
        }
    }
#endif

    #endregion

    #region 测试钩子 (Internal for Unit Tests)

    internal static void ResetCounters() // 新增：测试用重置计数器
    {
        TotalObjectsCreated = 0;
        TotalObjectsReused = 0;
        TotalObjectsDestroyed = 0;
        PeakActiveObjects = 0;
    }

    internal static int GetInternalPoolCount() // 新增：测试用获取池数量
    {
        return _pools.Count;
    }

    #endregion
}


```

#### ObjectPoolEditor

```csharp
using System.Collections.Generic;
using UnityEditor;
using UnityEngine;

/// <summary>
/// ObjectPoolManager 可视化调试工具
/// 实时显示所有对象池的统计信息
/// </summary>
public class ObjectPoolEditor : EditorWindow
{
    private Vector2 scrollPosition;
    private float refreshInterval = 1f; // 刷新间隔，防止每帧都重绘
    private float lastRefreshTime = 0f;

    [MenuItem("Tools/Object Pool/Debugger")]
    public static void ShowWindow()
    {
        GetWindow<ObjectPoolEditor>("Object Pool Debugger");
    }

    private void OnEnable()
    {
        // 订阅 EditorApplication.update 事件，每帧重绘窗口
        EditorApplication.update += Repaint;
    }

    private void OnDisable()
    {
        // 取消订阅，防止内存泄漏
        EditorApplication.update -= Repaint;
    }

    private void OnGUI()
    {
        // 检查是否在播放模式下
        if (!Application.isPlaying)
        {
            EditorGUILayout.HelpBox("请在播放模式下使用此调试工具。", MessageType.Info);
            return;
        }

        // 如果超过刷新间隔，则刷新数据
        if (Time.realtimeSinceStartup - lastRefreshTime > refreshInterval)
        {
            lastRefreshTime = Time.realtimeSinceStartup;
            Repaint();
        }

        // 将所有内容放入一个滚动视图，以防止内容被遮挡
        scrollPosition = EditorGUILayout.BeginScrollView(scrollPosition);

        DrawGlobalStats();
        DrawPoolStats();
        DrawPoolPerformanceChart(); // 添加性能监控部分

        EditorGUILayout.EndScrollView();
    }

    private void DrawGlobalStats()
    {
        GUILayout.Label("全局统计", EditorStyles.boldLabel);

        // 渲染 GlobalStats 字符串，使用多行布局
        string globalStats = ObjectPoolManager.GlobalStats;

        // 使用 TextArea 来显示多行文本，并允许滚动
        EditorGUILayout.TextArea(globalStats, EditorStyles.textArea, GUILayout.Height(150)); // 调整高度以适应更多行

        EditorGUILayout.Space(20);
    }

    private void DrawPoolStats()
    {
        GUILayout.Label("所有对象池详情", EditorStyles.boldLabel);

        // 获取所有池的统计数据
        Dictionary<string, string> allPools = ObjectPoolManager.GetAllPoolStats();

        if (allPools.Count == 0)
        {
            EditorGUILayout.HelpBox("当前没有已创建的对象池。", MessageType.Info);
            return;
        }

        foreach (var pool in allPools)
        {
            EditorGUILayout.BeginVertical(GUI.skin.box);
            // 显示预制体名称 + 实例ID
            GUILayout.Label(
                $"<color=white><b>预制体:</b> (ID:{pool.Key})</color>",
                new GUIStyle(EditorStyles.boldLabel) { richText = true }
            );

            GUILayout.Label(pool.Value);
            EditorGUILayout.EndVertical();
            EditorGUILayout.Space(5);
        }
    }

    #region 添加性能监控与优化功能

    /// <summary>
    /// 通过创建图表或进度条的方式提升数据显示体验
    /// </summary>
    private void DrawPoolPerformanceChart()
    {
        var poolStats = ObjectPoolManager.GetAllPoolStats();
        if (poolStats.Count == 0)
            return;

        foreach (var poolStat in poolStats)
        {
            // 假设每个池的活跃对象数占比是一个性能指标
            string poolName = poolStat.Key;
            string poolData = poolStat.Value;

            // 解析池的活跃对象数量（假设格式为 "x active, y pooled"）
            var activeCount = int.Parse(poolData.Split(new[] { ' ' })[0]); // 假设第一项是 active count

            // 绘制一个进度条来表示池中活跃对象的比例
            EditorGUILayout.LabelField(poolName);

            // 使用 EditorGUI.ProgressBar 来绘制进度条
            float progress = Mathf.Clamp01(activeCount / 100f); // 假设最大池容量是 100
            EditorGUI.ProgressBar(
                EditorGUILayout.GetControlRect(),
                progress,
                $"{poolName}: {activeCount} active"
            );

            EditorGUILayout.Space(5);
        }
    }

    #endregion
}

```
