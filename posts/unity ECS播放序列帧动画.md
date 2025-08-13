---
title: "Unity ECS播放序列帧动画"
titleColor: "#aaa,#0acee9"
titleIcon: "asset:unity"
tags: ["Untiy"]
categories: ["Code"]
description: "Unity ECS播放序列帧动画"
publishDate: "2023-03-10"
updatedDate: "2023-03-10"
password: "1231234"
---

#### 首先准备贴图

![](https://cdn.jiangwei.zone/blog/20250813122308601.png)

创建新的材质（Materials）并将贴图拖入到贴图属性中，并选择 AnimatedShaderGraph
![](https://cdn.jiangwei.zone/blog/20250813122536635.png)

[AnimatedShaderGraph.shadergraph](https://cdn.jiangwei.zone/blog/AnimatedShaderGraph.shadergraph)

#### 创建 Prefab

右键 3D Object > Quad  
将材质拖入到 Quad 组件中
设置参数：
![](https://cdn.jiangwei.zone/blog/20250813134327606.png)

ECS System: (播放)

```csharp
using Unity.Entities;
using UnityEngine;

namespace ComPeteSupremacy
{
    /// <summary>
    /// System to update the global time shader property. This is required so that when the game pauses, any animated shaders that rely on this property will also pause as this system is part of a group that will be disabled when the game is paused due to level up or pressing the ESC key (<see cref="PauseManager"/>).
    /// </summary>
    /// <remarks>
    /// System updates in Unity's InitializationSystemGroup and after their UpdateWorldTimeSystem to ensure that SystemAPI.Time.ElapsedTime is accurate for the current frame.
    /// </remarks>
    [UpdateInGroup(typeof(InitializationSystemGroup))]
    // [UpdateAfter(typeof(UpdateWorldTimeSystem))]
    public partial class GlobalTimeUpdateSystem : SystemBase
    {
        /// <summary>
        /// Cached integer of the shader property's ID, used for more efficient setting of shader property.
        /// </summary>
        private static int _globalTimeShaderPropertyID;

        protected override void OnCreate()
        {
            _globalTimeShaderPropertyID = Shader.PropertyToID("_GlobalTime");
        }

        protected override void OnUpdate()
        {
            Shader.SetGlobalFloat(_globalTimeShaderPropertyID, (float)SystemAPI.Time.ElapsedTime);
        }
    }
}

```

ECS 更改参数：

```csharp
[MaterialProperty("_AnimationIndex")]
public struct AnimationIndex : IComponentData
{
    public float Value;
}
```

在 entity 添加组件 AnimationIndex 并更改即可

```csharp
using ProjectDawn.Navigation;
using Unity.Burst;
using Unity.Collections;
using Unity.Entities;
using Unity.Mathematics;
using Unity.Transforms;
using UnityEngine;
using static Unity.Entities.SystemAPI;

namespace ComPeteSupremacy
{
    [BurstCompile]
    public partial struct AnimationSystem : ISystem
    {
        [BurstCompile]
        public void OnCreate(ref SystemState state)
        {
            state.RequireForUpdate<Soldier>();
        }

        [BurstCompile]
        public void OnUpdate(ref SystemState state)
        {
            // 调度并行Job
            var ecb = GetSingleton<EndInitializationEntityCommandBufferSystem.Singleton>();

            new AnimationJob
            {
                ElapsedTime = (float)SystemAPI.Time.ElapsedTime,
                Ecb = ecb.CreateCommandBuffer(state.WorldUnmanaged).AsParallelWriter(),
            }.ScheduleParallel();
        }

        [BurstCompile]
        [WithAll(typeof(Soldier))]
        [WithNone(typeof(GameObjectGO))]
        partial struct AnimationJob : IJobEntity
        {
            public EntityCommandBuffer.ParallelWriter Ecb;
            public float ElapsedTime;

            public void Execute(
                [EntityIndexInQuery] int index,
                Entity entity,
                ref SoldierState soldierState,
                ref AnimationIndex animationIndex,
                ref AnimationFrameCount animationFrameCount,
                ref AnimationSpawnTime animationSpawnTime
            )
            {
                if (soldierState.LastState == soldierState.State)
                {
                    return;
                }

                if (soldierState.State == EnumState.Move)
                {
                    animationIndex.Value = (float)EnumState.Move;
                    animationFrameCount.Value = (float)EnumFrame.Move;
                }
                else if (soldierState.State == EnumState.Attack)
                {
                    animationIndex.Value = (float)EnumState.Attack;
                    animationFrameCount.Value = (float)EnumFrame.Attack;
                }
                else
                {
                    animationIndex.Value = (float)EnumState.Dead;
                    animationFrameCount.Value = (float)EnumFrame.Dead;
                }

                animationSpawnTime.Value = -ElapsedTime; // 切换动画时从初始帧开始  *重要

                soldierState.LastState = soldierState.State;
            }
        }
    }
}

```

检测攻击：

```csharp

// 计算当前帧数
int frameCount =
    (int)((ElapsedTime + animationSpawnTime.Value) * animationFramesPerSecond.Value)
    % (int)EnumFrame.Attack;

// 开始攻击
if (
    soldierState.State == EnumState.Attack
    && frameCount == (int)EnumFrame.Attack - 2
)
{
    if (!soldierAttacking.IsAttackStart)
    {
        // 攻击开始
        soldierAttacking.IsAttackStart = true;
    }
    else
    {
        soldierAttacking.IsAttackStart = false;
    }
}

 // 攻击结束
if (
    soldierState.State == EnumState.Attack
    && frameCount == (int)EnumFrame.Attack - 1
)
{
    if (!soldierAttacking.IsAttackFinished)
    {
        soldierAttacking.IsAttackFinished = true;
    }
    else
    {
        soldierAttacking.IsAttackFinished = false;
    }
}
```
