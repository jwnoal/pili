---
title: "unity 抛物运动"
titleColor: "#aaa,#20B2AA"
titleIcon: "asset:unity"
tags: ["Untiy"]
categories: ["Code"]
description: "unity 抛物运动"
publishDate: "2023-03-10"
updatedDate: "2023-03-10"
password: "1231234"
---

#### MonoBehaviour 模式

```csharp
using System.Collections;
using System.Collections.Generic;
using Unity.Mathematics;
using UnityEngine;

public class ArrowPosAnimation : MonoBehaviour
{
    public GameObject prefab;

    [Header("属性")]
    public float Power = 100; //发射的速度、力度
    public float Angle = 20; //发射角度
    public float Gravity = -200f; //重力加速度
    public bool Isreverse; //反向
    public bool IsAngle; //释放转向

    private Vector3 MoveSpeed = Vector3.zero; //初始速度向量
    private Vector3 GritySpeed = Vector3.zero; // 重力速度向量
    private Vector3 CurrentAngle; //当前角度
    private float Dtime; //已经过去的时间

    private Vector3 EnemyVector; //目标位置

    void FixedUpdate()
    {
        if (MoveSpeed == Vector3.zero)
            return;

        GritySpeed.y = Gravity * (Dtime += Time.fixedDeltaTime);
        //模拟轨迹
        transform.position += (MoveSpeed + GritySpeed) * Time.fixedDeltaTime;
        CurrentAngle.z = Mathf.Atan((MoveSpeed.y + GritySpeed.y) / MoveSpeed.x) * Mathf.Rad2Deg;
        if (IsAngle)
            transform.eulerAngles = CurrentAngle;
        float distance = math.distancesq(
            new float3(transform.position.x, 0, 0),
            new float3(EnemyVector.x, 0, 0)
        );
        if (
            transform.position.y < EnemyVector.y && distance < 0.2f * 0.2f
            || transform.position.y < -2.9f
        )
        {
            MoveSpeed = Vector3.zero;
            if (prefab != null)
            {
                Instantiate(prefab, transform);
                Invoke("DisGameObject", 1.1f);
            }
            var an = transform.Find("An");
            if (an != null)
            {
                an.gameObject.SetActive(false);
            }
            return;
        }
    }

    public void ArrowPosition(Vector3 vector)
    {
        EnemyVector = vector;

        Gravity = -18f;
        MoveSpeed = CalculateInitialVelocity(transform.position, EnemyVector, Gravity);
    }

    private void DisGameObject()
    {
        Destroy(gameObject);
    }

    private static Vector3 CalculateInitialVelocity(
        Vector3 startPos,
        Vector3 targetPos,
        float gravity
    )
    {
        float g = gravity;
        float dx = targetPos.x - startPos.x;
        float dy = targetPos.y - startPos.y;
        float dz = targetPos.z - startPos.z; // 添加Z轴差

        // 水平飞行时间（秒）
        float flightTime = math.max(0.5f, math.abs(dx) / 8f);

        float vx = dx / flightTime;
        float vy = (dy - 0.5f * g * flightTime * flightTime) / flightTime;
        float vz = dz / flightTime; // 添加Z轴速度

        // 最小垂直速度确保正常弧线
        if (math.abs(dx) < 0.5f)
            vy = math.max(vy, 5f);

        return new Vector3(vx, vy, vz); // 返回包含vz的速度
    }
}


```

调用：

```csharp
var anim = node.GetComponent<ArrowPosAnimation>();
if (anim != null)
{
    anim.ArrowPosition(EnemyVector);
}
```

#### ECS 模式

```csharp
using Unity.Burst;
using Unity.Collections;
using Unity.Entities;
using Unity.Mathematics;
using Unity.Transforms;
using UnityEngine;

namespace ComPeteSupremacy
{
    [BurstCompile]
    public partial struct UnitThrowSystem : ISystem
    {
        [ReadOnly]
        public ComponentLookup<Soldier> SoldierLookup;

        [BurstCompile]
        public void OnCreate(ref SystemState state)
        {
            state.RequireForUpdate<Config>();
            SoldierLookup = state.GetComponentLookup<Soldier>(true);
        }

        [BurstCompile]
        public void OnUpdate(ref SystemState state)
        {
            SoldierLookup.Update(ref state);
            var config = SystemAPI.GetSingleton<Config>();
            var ecb = SystemAPI.GetSingleton<EndSimulationEntityCommandBufferSystem.Singleton>();
            new CannonBowJob
            {
                SoldierLookup = SoldierLookup,
                Config = config,
                Ecb = ecb.CreateCommandBuffer(state.WorldUnmanaged).AsParallelWriter(),
                DeltaTime = SystemAPI.Time.DeltaTime,
            }.ScheduleParallel();
        }

        [BurstCompile]
        [UpdateInGroup(typeof(FixedStepSimulationSystemGroup))]
        public partial struct CannonBowJob : IJobEntity
        {
            public EntityCommandBuffer.ParallelWriter Ecb;

            [ReadOnly]
            public ComponentLookup<Soldier> SoldierLookup;

            public float DeltaTime;
            public Config Config;

            void Execute(
                [EntityIndexInQuery] int index,
                Entity entity,
                ref ThrowBow throwBow,
                ref LocalTransform transform
            )
            {

                float distance = math.distancesq(
                    new float3(transform.Position.x, 0, 0),
                    new float3(throwBow.EndPosition.x, 0, 0)
                );
                if (
                    throwBow.EndY != 9999f && transform.Position.y < throwBow.EndY
                    || throwBow.EndY == 9999f
                        && (
                            transform.Position.y < throwBow.EndPosition.y && distance < 0.2f * 0.2f
                            || transform.Position.y < -2.9f
                        )
                )
                {
                    Ecb.DestroyEntity(index, entity);

                    // Debug.Log(entity + "xiaoshi");
                    // 炮弹消失的时候出现爆炸
                    if (throwBow.Type == 1)
                    {
                        Entity instance = Ecb.Instantiate(index, Config.PaoDanBom);
                        Ecb.SetComponent(
                            index,
                            instance,
                            new LocalTransform
                            {
                                Position = transform.Position,
                                Scale = 1,
                                Rotation = quaternion.identity,
                            }
                        );
                        Ecb.AddComponent(index, instance, new PaoDanBom { Time = 0, First = true });
                    }
                    return;
                }

                throwBow.DTime += DeltaTime;
                throwBow.GritySpeed.y = throwBow.Gravity * throwBow.DTime;
                transform.Position += (throwBow.MoveSpeed + throwBow.GritySpeed) * DeltaTime;

                transform.Position += new float3(0, 0f, -0.01f); //层级

                // 士兵死亡时需要调整角度
                if (SoldierLookup.HasComponent(entity))
                {
                    float angleInRadians = Mathf.Atan(
                        (throwBow.MoveSpeed.y + throwBow.GritySpeed.y) / throwBow.MoveSpeed.x
                    );

                    int a = throwBow.MoveSpeed.x > 0 ? 1 : -1;

                    throwBow.CurrentAngle.z = -90f * a + angleInRadians * Mathf.Rad2Deg;
                }
                else
                {
                    throwBow.CurrentAngle.z =
                        Mathf.Atan(
                            (throwBow.MoveSpeed.y + throwBow.GritySpeed.y) / throwBow.MoveSpeed.x
                        ) * Mathf.Rad2Deg;
                }

                transform.Rotation = Quaternion.Euler(throwBow.CurrentAngle);
            }
        }
    }
}

```

调用：

```csharp
var initialVelocity = CalculateInitialVelocity(
    transform.Position,
    EndPosition,
    gravity
);

Ecb.AddComponent(
    index,
    instance,
    new ThrowBow
    {
        Gravity = gravity,
        MoveSpeed = initialVelocity,
        EndPosition = EndPosition,
    }
);

// 抛物线 计算达到目标点所需的初始速度
private static float3 CalculateInitialVelocity(
    float3 startPos,
    float3 targetPos,
    float gravity
)
{
    float g = gravity;
    float dx = targetPos.x - startPos.x;
    float dy = targetPos.y - startPos.y;
    float dz = targetPos.z - startPos.z; // 添加Z轴差

    // 水平飞行时间（秒）
    float flightTime = math.max(0.5f, math.abs(dx) / 8f);

    float vx = dx / flightTime;
    float vy = (dy - 0.5f * g * flightTime * flightTime) / flightTime;
    float vz = dz / flightTime; // 添加Z轴速度

    // 最小垂直速度确保正常弧线
    if (math.abs(dx) < 0.5f)
        vy = math.max(vy, 5f);

    return new float3(vx, vy, vz); // 返回包含vz的速度
}
```
