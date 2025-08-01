---
title: "Unity ECS代码片段"
titleColor: "#aaa,#0acee9"
titleIcon: "asset:unity"
tags: ["Untiy"]
categories: ["Code"]
description: "unity ECS代码片段示例"
publishDate: "2023-03-10"
updatedDate: "2023-03-10"
password: "123123"
encrypt:
  description: |
    这是一篇被加密的文章。
---

IComponentData 组件数据

```csharp
public struct Config : IComponentData
{
    public Entity Prefab;
    public int Count;
}

public struct Solider : IComponentData, IEnableableComponent{}

public struct DamageBufferElement : IBufferElementData
{
    public float Damage;
    public float Delay;
}

public class GameObjectGO : IComponentData
{
    public GameObject Value;

    public GameObjectGO(GameObject value)
    {
        Value = value;
    }

    // Every IComponentData class must have a no-arg constructor.
    public GameObjectGO() { }
}
```

Authoring

```csharp
using Unity.Entities;
using UnityEngine;
public class ConfigAuthoring : MonoBehaviour
{
    public GameObject Prefab;
    public int Count;

    class Baker : Baker<ConfigAuthoring>
    {
        public override void Bake(ConfigAuthoring authoring)
        {
            var entity = GetEntity(TransformUsageFlags.None);

            AddComponent(
                entity,
                new Config
                {
                    Prefab = GetEntity(
                        authoring.Prefab,
                        TransformUsageFlags.Dynamic
                    ),
                    Count = authoring.Count,
                }
            );
        }
    }
}

```

ISystem

```csharp
using Unity.Burst;
using Unity.Entities;
public partial struct SpawnSystem : ISystem
{
    [BurstCompile]
    public void OnCreate(ref SystemState state)
    {
        state.RequireForUpdate<Config>();
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        state.Enabled = false;
    }
}
```

SystemBase

```csharp
using Unity.Entities;
public partial class SpawnSystem : SystemBase
{
    protected override void OnCreate()
    {
        RequireForUpdate<Config>();
    }

    protected override void OnUpdate()
    {
    }
}

```

遍历

ISystem

```csharp
using Unity.Burst;
using Unity.Entities;
public partial struct SpawnSystem : ISystem
{
    [BurstCompile]
    public void OnCreate(ref SystemState state)
    {
        state.RequireForUpdate<Config>();
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        float deltaTime = SystemAPI.Time.DeltaTime;
        foreach (var (transform, speed, entity) in
                    SystemAPI.Query<RefRW<LocalTransform>, RefRO<RotationSpeed>>().WithEntityAccess())
        {
            transform.ValueRW = transform.ValueRO.RotateY(
                speed.ValueRO.RadiansPerSecond * deltaTime);
        }
    }
}
```

SystemBase

```csharp
using Unity.Entities;
public partial class SpawnSystem : SystemBase
{
    protected override void OnCreate()
    {
        RequireForUpdate<Config>();
    }

    protected override void OnUpdate()
    {
       Entities
            .ForEach(
                (Entity entity, in LocalTransform transform, GameObjectGO gameObjectGO) =>
                {
                    battlefieldCenter += transform.Position.x;
                }
            )
            .WithAll<SoldierAttacking>()
            .WithoutBurst()
            .Run();
    }
}
```

Aspects

```csharp
using Unity.Burst;
using Unity.Entities;
using Unity.Mathematics;
using Unity.Transforms;

public partial struct RotationSystem : ISystem
{
    [BurstCompile]
    public void OnCreate(ref SystemState state)
    {
        state.RequireForUpdate<ExecuteAspects>();
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        double elapsedTime = SystemAPI.Time.ElapsedTime;

        foreach (var movement in
                    SystemAPI.Query<VerticalMovementAspect>())
        {
            movement.Move(elapsedTime);
        }
    }
}

readonly partial struct VerticalMovementAspect : IAspect
{
    readonly RefRW<LocalTransform> m_Transform;
    readonly RefRO<RotationSpeed> m_Speed;

    public void Move(double elapsedTime)
    {
        m_Transform.ValueRW.Position.y = (float)math.sin(elapsedTime * m_Speed.ValueRO.RadiansPerSecond);
    }
}
```

判断是否有组件

```csharp
if (state.EntityManager.HasComponent<RedSoldier>(entity))
```

获取特定 entity 中的组件

```csharp
public partial struct MoveSystem : ISystem
{
    [ReadOnly]
    public ComponentLookup<LocalTransform> TransformLookup;
    [BurstCompile]
        public void OnCreate(ref SystemState state)
        {
            // 将ComponentLookup标记为只读
            TransformLookup = state.GetComponentLookup<LocalTransform>(true);
            // TransformLookup = GetComponentLookup<LocalTransform>(this); // SystemBase
        }
        [BurstCompile]
        public void OnUpdate(ref SystemState state)
        {
            TransformLookup.Update(ref state);
             if (
                !TransformLookup.TryGetComponent(
                    soldierEnemy.Enemy,
                    out LocalTransform localTransform
                )
            )
            {
            }

            // 立即更改值，不使用ecb
            // AttackTagLookup 不能ReadOnly
            if (
                AttackTagLookup.TryGetComponent(
                    entity,
                    out AttackTag attackTag
                )
            )
            {
                attackTag.Value += 1;
                AttackTagLookup[entity] = attackTag;
            }
        }
}
```

获取实体集合

```csharp
var spinningCubesQuery = SystemAPI.QueryBuilder().WithAll<RotationSpeed>().Build();
spinningCubesQuery.CalculateEntityCount() // 计算实体数量
NativeArray<Entity> poolEntities = query.ToEntityArray(Allocator.TempJob); // 获取实体集合
```

设置组件值

```csharp
var Soldier = state.EntityManager.GetComponentData<Soldier>(entity);
Soldier.Destination = new float3(0, 0, 1f);
state.EntityManager.SetComponentData(entity, Soldier);
// IsComponentEnabled
// SetComponentEnabled
```

ECB

```csharp
// ISystem
var EcbSingleton = GetSingleton<EndInitializationEntityCommandBufferSystem.Singleton>();
var Ecb = EcbSingleton.CreateCommandBuffer(state.WorldUnmanaged); // .AsParallelWriter()

Ecb.AddComponent(
    index,
    entity,
    new SoldierAttacking
    {
        TargetEntity = soldierEnemy.Enemy,
    }
);

if (AttackTagNumLookup.TryGetComponent(entity, out AttackTagNum _attackTagNum))
{
    _attackTagNum.Value = action.AttackTagNum;
    Ecb.SetComponent(index, entity, _attackTagNum);
}
```

```csharp
// SystemBase
public partial class AnimationManagedSystem : SystemBase
{
    private EndSimulationEntityCommandBufferSystem EcbSystem;
    protected override void OnCreate()
    {
        EcbSystem = World.GetOrCreateSystemManaged<EndSimulationEntityCommandBufferSystem>();
    }
    protected override void OnUpdate()
    {
        EntityCommandBuffer Ecb = EcbSystem.CreateCommandBuffer(); // 不需要手动关闭

        // EntityCommandBuffer Ecb = new EntityCommandBuffer(Allocator.TempJob); // 需要手动关闭 Ecb.Dispose()
    }
}
```

注解

```csharp
[BurstCompile] // 编译优化
[RequireMatchingQueriesForUpdate] // 优化更新
[UpdateInGroup(typeof(FixedStepSimulationSystemGroup))] // 固定步长更新
[UpdateAfter(typeof(AgentSystemGroup))] // 更新在AgentSystemGroup之后
```

与常规 GameObject 交互

ecs 调用 Mono

```csharp
// 使用SystemBase 和 GameManager可以交互
using Unity.Entities;

public partial class CountSystem : SystemBase
{
    EntityQuery red_Query;
    EntityQuery blue_Query;

    protected override void OnCreate()
    {
        RequireForUpdate<Config>();
        RequireForUpdate<Demo3_collider>();
        red_Query = GetEntityQuery(typeof(RedSoldier));
        blue_Query = GetEntityQuery(typeof(BlueSoldier));
    }

    protected override void OnUpdate()
    {
        if (GameManager.Instance == null)
        {
            return;
        }

        GameManager.Instance.RedSoldierCount = red_Query.CalculateEntityCount();
        GameManager.Instance.BlueSoldierCount = blue_Query.CalculateEntityCount();
    }
}

```

Mono 更改 ecs

```csharp
using UnityEngine;
using Unity.Entities;
using Unity.Collections;

public class SpeedManager : MonoBehaviour
{
    public float global_speed;

    void Update()
    {
        var Manager = World.DefaultGameObjectInjectionWorld.EntityManager;
        var entities = Manager.CreateEntityQuery(typeof(Speed)).ToEntityArray(Allocator.Temp);
        foreach (var entity in entities)
        {
            var speed = Manager.GetComponentData<Speed>(entity);
            speed.value = global_speed;
            Manager.SetComponentData(entity, speed);
        }
        entities.Dispose();

        using (
            var query = new EntityQueryBuilder(Allocator.Temp).WithAll<Config>().Build(Manager)
        )
        {
            if (query.IsEmpty)
                throw new System.Exception("Missing Config entity");

            var config = query.GetSingleton<Config>();
        }
    }
}
```

Mono Ecs 互相绑定

```csharp
// 将 GameObject 绑定到 ECS
manager.AddComponentData(entity, new GameObjectGO(gameObject));
// go.Value.GetComponent<SeniorSoldier>();

// 从 ECS 绑定到 GameObject
var go = GameObject.Instantiate(prefab);
SeniorSoldier seniorSoldierData = go.GetComponent<SeniorSoldier>();
```

创建实体

```csharp
using Unity.Burst;
using Unity.Collections;
using Unity.Entities;
using Unity.Mathematics;
using Unity.Transforms;
using UnityEngine;

namespace Demo2
{
    public partial struct SpawnSystem1 : ISystem
    {

        [BurstCompile]
        public void OnCreate(ref SystemState state)
        {
            state.RequireForUpdate<Spawner>();
        }

        [BurstCompile]
        public void OnUpdate(ref SystemState state)
        {
            if (Input.GetKeyDown(KeyCode.Q))
            {
                SpawnSoldiers(ref state, SystemAPI.GetSingleton<Spawner>().RedSoldierPrefab, 22f, -20f);
            }
        }

        [BurstCompile]
        private void SpawnSoldiers(ref SystemState state, Entity prefab, float xPosition, float xPosition2)
        {
            var instances = state.EntityManager.Instantiate(prefab, 10, Allocator.TempJob); // 直接创建多个而不是循环创建一个

            for (int i = 0; i < instances.Length; i++)
            {
                var instance = instances[i];
                var transform = SystemAPI.GetComponentRW<LocalTransform>(instance);
                transform.ValueRW.Position = new float3(xPosition, 0, -8 + i * 1.6f);

                var Soldier = state.EntityManager.GetComponentData<Soldier>(instance);
                Soldier.Destination = new float3(xPosition2, 0, -8 + i * 1.6f);
                SystemAPI.SetComponentData(instance, Soldier); //SystemAPI性能更高
                // state.EntityManager.SetComponentData(instance, Soldier);
            }

            instances.Dispose();
        }
    }
}
```

DynamicBuffer

```csharp
AddBuffer<DamageBufferElement>(entity); // Authoring

// Ecb.SetBuffer<DamageBufferElement>(entity, new NativeArray<DamageBufferElement>(new DamageBufferElement[10], Allocator.Temp);

Ecb.AppendToBuffer(
    index,
    ConfigEntity,
    new DamageBufferElement
    {
        Damage = soldier.Attack,
        Delay = delay,
        AttackUid = soldier.Uid,
        Type = target.Type,
    }
);

// damageBuffer.Clear();
// damageBuffer.Add(dmg);

protected override void OnUpdate()
{
    // 等待句柄完成
    Dependency.Complete();

    var AttackTagBufferElement = SystemAPI.GetSingletonBuffer<AttackTagBufferElement>();
    foreach (var delta in AttackTagBufferElement)
    {
        if (
            AttackTagNum2Lookup.TryGetComponent(
                delta.Target,
                out AttackTagNum2 attackTagNum2
            )
        )
        {
            attackTagNum2.Value += delta.Value;
            AttackTagNum2Lookup[delta.Target] = attackTagNum2;
        }
    }
    AttackTagBufferElement.Clear();
}
```

Input.GetKeyDown 不能使用 burst 编译，所以不能用[BurstCompile]注解。

```csharp
using Unity.Burst;
using Unity.Entities;
using UnityEngine;

namespace Demo4
{
    [UpdateInGroup(typeof(InitializationSystemGroup), OrderFirst = true)]
    public partial struct InputGatheringSystem : ISystem
    {
        private Entity _inputEntity;

        // 初始化时创建单例实体
        public void OnCreate(ref SystemState state)
        {
            // 确保单例唯一性
            if (!SystemAPI.TryGetSingletonEntity<InputData>(out _inputEntity))
            {
                _inputEntity = state.EntityManager.CreateEntity(typeof(InputData));
                state.EntityManager.SetName(_inputEntity, "InputSingleton");
            }
        }

        // 必须禁用Burst以访问UnityEngine.Input
        [BurstDiscard]
        public void OnUpdate(ref SystemState state)
        {
            // 更新单例组件
            SystemAPI.SetSingleton(new InputData
            {
                QPressed = Input.GetKeyDown(KeyCode.Q),
                WPressed = Input.GetKeyDown(KeyCode.W)
            });
        }
    }

    // 输入数据结构（单例）
    public struct InputData : IComponentData
    {
        public bool QPressed;
        public bool WPressed;
    }

}

```

​IJobFor：适合大批量数据并行处理，当数量足够大时，分摊 Job 调度开销。但对于少量数据（如 10 个），每个批次的开销可能超过实际处理时间，导致性能下降。

​IJob：单线程 Job，适合处理小批量数据，调度开销较低。虽然无法并行，但对于 小于 100 个实体来说，单线程处理可能更快，因为避免了并行调度的开销。

```csharp
using Unity.Burst;
using Unity.Entities;
using Unity.Jobs;
using Unity.Mathematics;
using Unity.Transforms;
using UnityEngine;
using static Unity.Entities.SystemAPI;

namespace Demo4
{
    [BurstCompile]
    public partial struct SpawnSystem : ISystem
    {
        [BurstCompile]
        public void OnCreate(ref SystemState state)
        {
            state.RequireForUpdate<InputData>();
            state.RequireForUpdate<Config>();
        }

        [BurstCompile]
        public void OnUpdate(ref SystemState state)
        {
            var inputData = GetSingleton<InputData>();
            if (!inputData.QPressed && !inputData.WPressed) return;

            var config = GetSingleton<Config>();
            var ecbSingleton = GetSingleton<EndInitializationEntityCommandBufferSystem.Singleton>();

            JobHandle handle = state.Dependency;

            if (inputData.QPressed)
            {
                var ecb = ecbSingleton.CreateCommandBuffer(state.WorldUnmanaged);

                if (config.SoldierCount < 100)
                {
                    handle = new SpawnerSingleJob
                    {
                        Ecb = ecb,
                        Prefab = config.RedSoldierPrefab,
                        SoldierCount = config.SoldierCount
                    }.Schedule(handle);
                }
                else
                {
                    int optimalBatchSize = math.max(64, config.SoldierCount / (SystemInfo.processorCount * 2));
                    handle = new SpawnerJob
                    {
                        ParallelWriter = ecb.AsParallelWriter(),
                        Prefab = config.RedSoldierPrefab,
                        SoldierCount = config.SoldierCount,
                    }.ScheduleParallel(config.SoldierCount, optimalBatchSize, handle);
                }

            }

            state.Dependency = handle;
        }

        [BurstCompile]
        public struct SpawnerSingleJob : IJob
        {
            public EntityCommandBuffer Ecb;
            public Entity Prefab;
            public int SoldierCount;

            public void Execute()
            {
                int side = (int)math.ceil(math.sqrt(SoldierCount));

                for (int i = 0; i < SoldierCount; i++)
                {
                    Entity instance = Ecb.Instantiate(Prefab);

                    // 优化计算：减少重复运算
                    int row = i / side;
                    int col = i - (row * side); // 替代取模运算

                    Ecb.SetComponent(instance, new LocalTransform
                    {
                        Position = new float3(col * 0.3f, 0, row * 0.3f),
                        Scale = 1,
                        Rotation = quaternion.identity,
                    });
                }
            }
        }

        [BurstCompile]
        public struct SpawnerJob : IJobFor
        {
            public EntityCommandBuffer.ParallelWriter Ecb;
            public Entity Prefab;
            public int SoldierCount;

            public void Execute(int index)
            {
                Entity instance = Ecb.Instantiate(index, Prefab);

                // 优化后的行列计算
                int side = (int)math.ceil(math.sqrt(SoldierCount));
                int row = index / side;
                int col = index % side;

                Ecb.SetComponent(index, instance, new LocalTransform
                {
                    Position = new float3(col * 0.3f, 0, row * 0.3f),
                    Scale = 1,
                    Rotation = quaternion.identity,
                });
            }
        }
    }
}

```

IJobEntity

```csharp
var EcbSingleton = GetSingleton<EndInitializationEntityCommandBufferSystem.Singleton>();
new MoveJob
{
    DeltaTime = SystemAPI.Time.DeltaTime,
    TransformLookup = TransformLookup,
    Ecb = EcbSingleton.CreateCommandBuffer(state.WorldUnmanaged).AsParallelWriter(),
}.ScheduleParallel();

[BurstCompile]
[WithAll(typeof(Soldier))]
[WithNone(typeof(SoldierDead))]
[UpdateAfter(typeof(MoveSystem))]
partial struct MoveJob : IJobEntity
{
    [ReadOnly]
    public ComponentLookup<LocalTransform> TransformLookup;

    public EntityCommandBuffer.ParallelWriter Ecb;

    public float DeltaTime;

    [BurstCompile]
    public void Execute(
        [EntityIndexInQuery] int index,
        Entity entity,
        ref SoldierState soldierState,
        in Soldier soldier,
        in LocalTransform transform
    )
    {}
}
```
