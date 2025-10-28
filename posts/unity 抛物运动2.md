---
title: "unity 抛物运动2"
titleColor: "#aaa,#20B2AA"
titleIcon: "asset:unity"
tags: ["Untiy"]
categories: ["Code"]
description: "unity 抛物运动2"
publishDate: "2023-03-10"
updatedDate: "2023-03-10"
password: "1231234"
---

支持设置目标模式和速度模式，可以设置目标位置和角度，自动计算初速度和总飞行时间。  
支持设置动画速度，可以调整动画播放速度。  
支持设置轨迹点数，可以调整轨迹精度。

```csharp
using UnityEngine;

public class ParabolicMotion : MonoBehaviour
{
    // 速度模式参数
    public Vector3 initialVelocity = new Vector3(5f, 10f, 0f);

    // 模式选择：在Inspector中勾选使用目标模式
    public bool useTargetMode = false; // false: 用initialVelocity；true: 用targetPosition和launchAngle自动计算

    // 目标模式参数
    public Vector3 targetPosition = new Vector3(10f, 0f, 0f);
    public float launchAngle = 45f; // 度，目标模式下使用

    public float gravity = 9.81f;
    public float animationSpeed = 1f;

    public LineRenderer lineRenderer;
    public int trajectoryPoints = 50;

    // 计算或使用的初速度
    private Vector3 calculatedVelocity;

    // 新增：精确的总飞行时间（目标模式下使用）
    private float totalFlightTime = 0f;

    private Vector3 startPosition;
    private float timeElapsed = 0f;
    private bool isMoving = false;

    void Start()
    {
        startPosition = transform.position;

        if (lineRenderer == null)
        {
            lineRenderer = gameObject.AddComponent<LineRenderer>();
            lineRenderer.material = new Material(Shader.Find("Standard"));
            lineRenderer.startColor = Color.red;
            lineRenderer.endColor = Color.blue;
            lineRenderer.startWidth = 0.2f;
            lineRenderer.endWidth = 0.2f;
            lineRenderer.useWorldSpace = true;
        }

        // 初始计算并绘制
        UpdateVelocityAndTrajectory();
    }

    void Update()
    {
        if (Input.GetKeyDown(KeyCode.Space))
        {
            isMoving = true;
            timeElapsed = 0f;
            transform.position = startPosition;
            UpdateVelocityAndTrajectory(); // 重新计算和重绘（以防Inspector变化）
        }

        if (isMoving)
        {
            timeElapsed += Time.deltaTime * animationSpeed;

            // 如果是目标模式，且时间超过总飞行时间，精确停止
            if (useTargetMode && timeElapsed >= totalFlightTime)
            {
                isMoving = false;
                transform.position = targetPosition; // 精确放置到目标
                return;
            }

            float x = calculatedVelocity.x * timeElapsed;
            float y =
                calculatedVelocity.y * timeElapsed - 0.5f * gravity * timeElapsed * timeElapsed;
            float z = calculatedVelocity.z * timeElapsed;

            transform.position = startPosition + new Vector3(x, y, z);

            // 原停止条件作为备份（速度模式或意外）
            if (useTargetMode)
            {
                if (
                    Vector3.Distance(transform.position, targetPosition) < 0.1f
                    || transform.position.y < targetPosition.y
                )
                {
                    isMoving = false;
                    transform.position = targetPosition;
                }
            }
            else
            {
                if (transform.position.y < startPosition.y)
                {
                    isMoving = false;
                }
            }
        }
    }

    // 统一方法：根据模式更新初速度和轨迹
    private void UpdateVelocityAndTrajectory()
    {
        if (useTargetMode)
        {
            CalculateInitialVelocityForTarget();
        }
        else
        {
            calculatedVelocity = initialVelocity;
            // 速度模式总时间
            totalFlightTime = 2f * calculatedVelocity.y / gravity;
        }

        DrawTrajectory();
    }

    // 目标模式：计算初速度和精确总飞行时间
    private void CalculateInitialVelocityForTarget()
    {
        Vector3 direction = targetPosition - startPosition;
        float distance = new Vector2(direction.x, direction.z).magnitude;
        float heightDiff = targetPosition.y - startPosition.y; // 注意：正为目标更高，负为更低

        if (Mathf.Approximately(distance, 0f))
        {
            Debug.LogWarning("目标与起点重合！无运动。");
            calculatedVelocity = Vector3.zero;
            totalFlightTime = 0f;
            return;
        }

        float angleRad = launchAngle * Mathf.Deg2Rad;
        float cosAngle = Mathf.Cos(angleRad);
        float tanAngle = Mathf.Tan(angleRad);

        // 公式：速度大小（处理NaN情况）
        float denominator = 2 * (distance * tanAngle - heightDiff) * cosAngle * cosAngle;
        if (denominator <= 0)
        {
            Debug.LogWarning("不可达目标！调整角度（推荐30-60°）或距离/高度。");
            calculatedVelocity = Vector3.zero;
            totalFlightTime = 0f;
            return;
        }

        float speed = Mathf.Sqrt(gravity * distance * distance / denominator);

        calculatedVelocity.x = speed * cosAngle * (direction.x / distance);
        calculatedVelocity.y = speed * Mathf.Sin(angleRad);
        calculatedVelocity.z = speed * cosAngle * (direction.z / distance);

        Debug.Log("目标模式计算初速度: " + calculatedVelocity);

        // 新增：精确计算总飞行时间（二次方程解）
        float vy = calculatedVelocity.y;
        float dh = startPosition.y - targetPosition.y; // 注意：dh >0 如果目标更低
        float discriminant = vy * vy + 2 * gravity * dh;
        if (discriminant < 0)
        {
            Debug.LogWarning("飞行时间计算失败（不可达）！");
            totalFlightTime = 0f;
            return;
        }
        totalFlightTime = (vy + Mathf.Sqrt(discriminant)) / gravity; // 取大根（落地时间）
        Debug.Log("精确总飞行时间: " + totalFlightTime + " 秒");
    }

    // 绘制轨迹（使用totalFlightTime）
    private void DrawTrajectory()
    {
        if (trajectoryPoints < 2)
            return;

        if (totalFlightTime <= 0)
        {
            Debug.LogWarning("总时间无效！检查参数。");
            lineRenderer.positionCount = 0;
            return;
        }

        lineRenderer.positionCount = trajectoryPoints;

        for (int i = 0; i < trajectoryPoints; i++)
        {
            float t = totalFlightTime * (i / (float)(trajectoryPoints - 1));

            float x = calculatedVelocity.x * t;
            float y = calculatedVelocity.y * t - 0.5f * gravity * t * t;
            float z = calculatedVelocity.z * t;

            lineRenderer.SetPosition(i, startPosition + new Vector3(x, y, z));
        }

        Debug.Log("轨迹绘制完成: " + trajectoryPoints + " 点");
    }
}

```
