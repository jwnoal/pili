---
title: 'Unity 弹幕'
titleColor: '#aaa,#0ae9ad'
titleIcon: 'asset:unity'
tags: [ 'Untiy' ]
categories: [ 'Code' ]
description: 'Unity 弹幕的实现'
publishDate: '2023-03-10'
updatedDate: '2023-03-10'
password: 'pilipal'
---

#### 创建容器和prefab
![](https://cdn.jiangwei.zone/blog/20250310165946068.png)
![](https://cdn.jiangwei.zone/blog/20250310170204760.png)

#### 创建脚本
Barrage.cs
```c#
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Pool;

public class Barrage : MonoBehaviour
{

    private static Barrage _instance;
    public static Barrage Instance
    {
        get
        {
            return _instance;
        }
    }

    public GameObject barrageItemPrefab;
    public int lineNums = 3;

    public ObjectPool<GameObject> pool; //对象池

    private LinkedList<BulletScreen> barrageQueue;
    private bool[] lineStatus;

    // Start is called before the first frame update
    private void Awake()
    {
        _instance = this;

        barrageQueue = new LinkedList<BulletScreen>();

        lineStatus = new bool[lineNums];
        for (int i = 0; i < lineStatus.Length; i++)
        {
            lineStatus[i] = true; // 初始化为 true
        }

        pool = new ObjectPool<GameObject>(
            createFunc: () =>
            {
                GameObject obj = Instantiate(barrageItemPrefab);
                obj.transform.SetParent(transform, false);
                return obj;
            },
            actionOnGet: (obj) => obj.SetActive(true),
            actionOnRelease: (obj) => obj.SetActive(false),
            actionOnDestroy: (obj) => Destroy(obj),
            true, 10, 1000);
    }

    public void AddBarrage(BulletScreen barrageData)
    {
        barrageQueue.AddLast(barrageData);
    }

     public void AddBarrageFirst(BulletScreen barrageData)
    {
        barrageQueue.AddFirst(barrageData);
    }

    private void FixedUpdate()
    {
        if (barrageQueue.Count > 0)
        {
            int index = GetFirstAvailableLineIndex();
            if (index > -1)
            {
                SetLineStatus(index, false);

                BulletScreen barrageData = barrageQueue.First.Value;
                barrageQueue.RemoveFirst();

                GameObject barrageItem = pool.Get();
                barrageItem.GetComponent<BarrageItem>().SetData(
                    new BulletScreen()
                    {
                        nick = barrageData.nick,
                        content = barrageData.content,
                        avatar = barrageData.avatar,
                        lineIndex = index,
                    }
                    );
            }

        }
    }

    // 获取首个可用的行号
    public int GetFirstAvailableLineIndex()
    {
        for (int i = 0; i < lineStatus.Length; i++)
        {
            if (lineStatus[i])
            {
                return i;
            }
        }
        return -1; // 表示没有找到可用的行
    }

    // 设置行号的状态
    public void SetLineStatus(int index, bool status)
    {
        if (index >= 0 && index < lineStatus.Length)
        {
            lineStatus[index] = status;
        }
    }
}

public class BulletScreen
{
    public string nick;
    public string content;
    public string avatar;
    public int lineIndex;
}

```

BarrageItem.cs
```c#

using TMPro;
using UnityEngine;
using UnityEngine.Pool;
using UnityEngine.UI;
using DG.Tweening;
using System.Collections;
using System;
using System.Text.RegularExpressions;
using UnityEngine.Networking;

public class BarrageItem : MonoBehaviour
{
    public Image avatar;
    public GameObject nickname;
    public GameObject msg;

    public float bridgeSpace = 20f; // 弹幕间距
    public float rowHeight = 40f; // 弹幕行高
    public float duration = 16f; // 弹幕运动时间
    public float moveDistance = 500f; // 弹幕离开屏幕后继续走的距离，防止被长句子追上

    private float canvasWidth = 0f;
    private RectTransform myRectTransform;
    private int lineIndex;
    private bool status = false;
    private float visibleWidth = 0f;



    void Start()
    {
    }

    private void FixedUpdate()
    {
        if (status)
        {
            if (transform.localPosition.x < visibleWidth)
            {
                // 允许下个弹幕出现
                Barrage.Instance.SetLineStatus(lineIndex, true);
                status = false;
            }
        }

    }
    public void SetData(BulletScreen barrageData)
    {
        // 设置名字
        nickname.GetComponent<TextMeshProUGUI>().text = barrageData.nick;
        float width = nickname.GetComponent<TextMeshProUGUI>().preferredWidth;
        nickname.GetComponent<RectTransform>().sizeDelta = new Vector2(width > 110 ? 110 : width, 15);

        // 设置内容
        msg.GetComponent<TextMeshProUGUI>().text = barrageData.content;
        // msg.GetComponent<TextMeshProUGUI>().color = new Color(255f / 255f, 210f / 255f, 83f / 255f); // 黄色

        // 设置初始位置
        RectTransform parentRectTransform = transform.parent.GetComponent<RectTransform>();
        canvasWidth = parentRectTransform.rect.width;
        lineIndex = barrageData.lineIndex;
        transform.localPosition = new Vector3(canvasWidth / 2, -(rowHeight * (lineIndex - 1)), 0);

        // 加载头像
        StartCoroutine(LoadTextureFormFile(barrageData.avatar, (avatarSprite) =>
        {
            avatar.sprite = avatarSprite;
        }));

        // 使用协程确保在一帧后获取宽度
        StartCoroutine(UpdateLayoutAndGetWidth());

    }

    private IEnumerator UpdateLayoutAndGetWidth()
    {
        // 等待一帧
        yield return null;

        myRectTransform = GetComponent<RectTransform>();

        // 强制重建布局
        LayoutRebuilder.ForceRebuildLayoutImmediate(myRectTransform);

        float targetX = transform.localPosition.x - myRectTransform.rect.width - canvasWidth - moveDistance; // 整句走完后仍走500的距离，防止被长句子追上

        // 继续其他操作，例如移动等
        transform.DOLocalMoveX(targetX, duration)
            .SetEase(Ease.Linear) // 匀速运动
            .OnComplete(() => { Barrage.Instance.pool.Release(gameObject); });

        visibleWidth = canvasWidth / 2 - myRectTransform.rect.width - bridgeSpace; // 同行弹幕间距为bridgeSpace
        status = true;
    }

    public IEnumerator LoadTextureFormFile(string path, Action<Sprite> callBack)
    {
        Sprite sprite = null;
        UnityWebRequest request = UnityWebRequestTexture.GetTexture(path);
        request.timeout = 10;
        yield return request.SendWebRequest();
        if (request.result == UnityWebRequest.Result.Success)
        {
            var texture = DownloadHandlerTexture.GetContent(request);
            if (texture != null)
            {
                texture.Compress(true);
                sprite = Sprite.Create(texture, new Rect(0, 0, texture.width, texture.height), new Vector2(0.5f, 0.5f));
                sprite.texture.filterMode = FilterMode.Bilinear;
            }
        }
        callBack(sprite);
    }
}

```

BarrageManager.cs
```c#
using UnityEngine;

public class BarrageManager : MonoBehaviour
{
    // Start is called before the first frame update
    void Start()
    {
        for (int i = 0; i < 5; i++)
        {
            Barrage.Instance.AddBarrage(new BulletScreen()
            {
                nick = "小明" + i,
                content = "这是一条弹幕",
                avatar = "https://gips2.baidu.com/it/u=1651586290,17201034&fm=3028&app=3028&f=JPEG&fmt=auto&q=100&size=f600_800",
            });
        }

        Invoke("Add", 2);
    }

    void Add()
    {
        Barrage.Instance.AddBarrageFirst(new BulletScreen()
        {
            nick = "小明" + 11,
            content = "这是一条弹幕",
            avatar = "https://gips2.baidu.com/it/u=1651586290,17201034&fm=3028&app=3028&f=JPEG&fmt=auto&q=100&size=f600_800",
        });
    }

    // Update is called once per frame
    void Update()
    {

    }
}

```

将脚本挂载到节点上。