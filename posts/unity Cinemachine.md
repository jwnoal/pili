---
title: "unity Cinemachine"
titleColor: "#aaa,#20B2AA"
titleIcon: "asset:unity"
tags: ["Untiy"]
categories: ["Code"]
description: "Unity 虚拟相机的使用"
publishDate: "2023-03-10"
updatedDate: "2023-03-10"
password: "1231234"
---

一般场景中只会有一个 Unity 相机，但可以有多个 Virtual Cameras

### 安装

在 Package Manager 中搜索 Cinemachine，然后安装 Cinemachine 到项目中。

### 使用

#### 在 Main Camra 上挂载组件 CinemachineBrain

![](https://cdn.jiangwei.zone/blog/20250418103034401.png)

#### CinemachineBrain 中的 Live Camera 中需要绑定虚拟相机节点

创建节点 Virtual Camera，添加组件 CinemachineVirtualCamera, Cinemachine Confiner 2D
![](https://cdn.jiangwei.zone/blog/20250418103733789.png)

注意： Body 设置成 Transposer

#### CinemachineVirtualCamera 中的 Follow 绑定跟随的目标物体，Look At 绑定相机的方向。

创建节点 Virtual Camera Follow

#### Cinemachine Confiner 2D 中的 Bounding Shape 2D 可以添加相机可移动区域

创建节点 Virtual Camera Collider 添加 Polygon Collider 2D 组件
![](https://cdn.jiangwei.zone/blog/20250418103539517.png)

Edit Collider 可以设置区域

#### 节点展示

![](https://cdn.jiangwei.zone/blog/20250418104426689.png)

移动 Virtual Camera Follow 节点位置即可移动相机

#### 禁止相机触边反弹

Cinemachine Confiner 2D -> Damping 选项可以设置反弹力度，设置为 0 即可。

移动相机设置战场中心：

在 Follow 节点挂载 CameraAnimation

```csharp
using UnityEngine;

namespace ComPeteSupremacy
{
    public class CameraAnimation : MonoBehaviour
    {
        private Vector3 currentVelocity; // 声明为类的成员变量

        void Update()
        {
            if (GameManager.Instance)
            {
                // 匀速移动
                // Vector3 position = new Vector3(GameManager.Instance.BattlefieldCenter, 0, 0);
                // if (Vector3.Distance(position, transform.position) > 1f)
                // {
                //     var dis = (position - transform.position).normalized;
                //     transform.position += dis * Time.deltaTime * 10;
                // }

                // SmoothDamp（平滑阻尼，逐渐减速）
                Vector3 targetPosition = new Vector3(GameManager.Instance.BattlefieldCenter, 0, 0);
                float smoothTime = 1f; // 调整这个值可以改变到达目标的时间
                if (Vector3.Distance(targetPosition, transform.position) > 1f)
                {
                    transform.position = Vector3.SmoothDamp(
                        transform.position,
                        targetPosition,
                        ref currentVelocity,
                        smoothTime
                    );
                }
            }
        }
    }
}

```

ECS 中获取战场中心：

```csharp
float battlefieldCenter = 0f;
int battfieldCount = 0;

Entities
    .ForEach(
        (Entity entity, in LocalTransform transform) =>
        {
            battlefieldCenter += transform.Position.x;
            battfieldCount += 1;
        }
    )
    .WithAll<SoldierAttacking>()
    .Run();

if (battfieldCount != 0)
{
    GameManager.Instance.BattlefieldCenter = battlefieldCenter / battfieldCount;
}
```
