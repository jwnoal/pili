---
title: 'Unity 对象池'
titleColor: '#aaa,#5d3bbf'
titleIcon: 'asset:unity'
tags: [ 'Untiy' ]
categories: [ 'Code' ]
description: 'Unity 对象池的使用'
publishDate: '2023-03-10'
updatedDate: '2023-03-10'
---

#### 创建对象池

```c#
public ObjectPool<GameObject> pool; //对象池

private void Awake()
    {
        pool = new ObjectPool<GameObject>(
            createFunc: () =>
            {
                GameObject obj = Instantiate(Prefab);
                obj.transform.SetParent(transform, false);
                return obj;
            },
            actionOnGet: (obj) => obj.SetActive(true),
            actionOnRelease: (obj) => obj.SetActive(false),
            actionOnDestroy: (obj) => Destroy(obj),
            true, 10, 1000);
    }
```

#### 获取对象
```c#
GameObject obj = pool.Get();
```

#### 释放对象
```c#
pool.Release(obj);
```

#### 销毁对象池
```c#
pool.Destroy(); 
//或者
pool = null;
```
