---
title: "unity Agent Navigation 使用"
titleColor: "#aaa,#20B2AA"
titleIcon: "asset:unity"
tags: ["Untiy"]
categories: ["Code"]
description: "Agent Navigation组件的使用"
publishDate: "2023-03-10"
updatedDate: "2023-03-10"
password: "1231234"
---

Agent Navigation 是一款导航插件。

#### 添加新的 layer

Project Settings -> Agents Navigation -> 添加新的 layer  
(Element 0 就是 NavigationLayers.Default)

#### 2d 会将 y 轴旋转到目标，导致原本是站立变成了趴着。

Agent： Angular Speed 设置为 0

#### 基础智能体 及 体积

Agent
![](https://cdn.jiangwei.zone/blog/20250813120053033.png)

#### 分离组件

Agent Separation

#### 声纳组件

Agent Sonar
![](https://cdn.jiangwei.zone/blog/20250813115448241.png)

#### 路径规划组件

父级添加 NavMeshSurface 组件，子级可以添加 plane 作为地面，或者添加组件 NavMeshModifier，设置其 override area，然后勾选， 最后 Bake 好地图。

智能体添加 Agent NavMesh Pathing 组件
![](https://cdn.jiangwei.zone/blog/20250813115913728.png)

#### 碰撞组件

Agent Collider

设置碰撞组件后想要不挤压抖动，可以添加 Agent Smart Stop 组件
。

或者用代码实现：分区监测，如果前面有兵则停下。

#### 检测前面有兵：

```csharp
new MoveSlowJob
{
    Spatial = spatial,
    FrameCount = _frameCount,
    TransformLookup = TransformLookup,
}.ScheduleParallel();

// 减速
[BurstCompile]
[WithNone(typeof(SoldierDead))]
[WithNone(typeof(GameObjectGO))]
partial struct MoveSlowJob : IJobEntity
{
    [ReadOnly]
    public AgentSpatialPartitioningSystem.Singleton Spatial; // 空间分区

    [ReadOnly]
    public ComponentLookup<LocalTransform> TransformLookup;

    public int FrameCount;

    [BurstCompile]
    public void Execute(
        [EntityIndexInQuery] int index,
        Entity entity,
        ref AgentLocomotion agentLocomotion,
        ref AgentBody agentBody,
        in Soldier soldier,
        in SoldierState soldierState,
        in LocalTransform transform
    )
    {
        if (soldierState.State != EnumState.Move)
        {
            return;
        }

        // 拥挤时减速
        if (FrameCount == 0)
        {
            NavigationLayers layer = NavigationLayers.Default;

            var forwardAction = new FindForwardTargetAction
            {
                Distance = 0.1f + soldier.Volume * 0.1f,
                Entity = entity,
                Transform = transform,
                Team = soldier.Team,
            };

            if (soldier.Type == EnumType.RedBuBing)
            {
                layer = NavigationLayers.Layer2;
            }
            else { }

            float3 targetPosition;
            if (soldier.Team == EnumTeam.Red)
            {
                targetPosition = new float3(
                    transform.Position.x + 0.1f,
                    transform.Position.y,
                    transform.Position.z
                );
            }
            else
            {
                targetPosition = new float3(
                    transform.Position.x - 0.1f,
                    transform.Position.y,
                    transform.Position.z
                );
            }

            Spatial.QueryCircle(targetPosition, 0.1f, ref forwardAction, layer); // 圆形分区监测

            // 检测面前的队友
            if (forwardAction.Target != Entity.Null)
            {
                agentLocomotion.Speed = 0;
                agentLocomotion.Acceleration = 0;
                agentBody.Stop();
            }
            else
            {
                agentLocomotion.Speed = 1f;
                agentLocomotion.Acceleration = 8;
            }
        }
    }

    [BurstCompile]
    struct FindForwardTargetAction : ISpatialQueryEntity
    {
        public Entity Entity;
        public Entity Target;
        public EnumTeam Team;
        public float Distance;

        public LocalTransform Transform;

        [BurstCompile]
        public void Execute(
            Entity targetEntity,
            AgentBody body,
            AgentShape shape,
            LocalTransform transform
        )
        {
            if (Target != Entity.Null)
            {
                // 已经找到目标
                return;
            }

            if (Entity == targetEntity)
            {
                // Debug.Log($"自己");
                return;
            }

            // 只要有一个目标就满足条件
            float2 targetPosition = new float2(transform.Position.x, transform.Position.y);
            float2 Position = new float2(Transform.Position.x, Transform.Position.y);

            float distanceSq = math.distancesq(Position, targetPosition);

            if (
                Team == EnumTeam.Red
                && transform.Position.x > Transform.Position.x
                && distanceSq < Distance * Distance
                && math.abs(transform.Position.y - Transform.Position.y) < 0.05f
            )
            {
                Target = targetEntity;
            }
            if (
                Team == EnumTeam.Blue
                && transform.Position.x < Transform.Position.x
                && distanceSq < Distance * Distance
                && math.abs(transform.Position.y - Transform.Position.y) < 0.05f
            )
            {
                Target = targetEntity;
            }
        }
    }
}
```
