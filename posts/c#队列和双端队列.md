---
title: 'c# 队列和双端队列'
titleColor: '#aaa,#0ae9ad'
titleIcon: 'asset:markdown'
tags: [ 'c#' ]
categories: [ 'Code' ]
description: '队列无法插队，所以需要双端队列'
---

#### 队列 Queue

```c#
private Queue barrageQueue;

barrageQueue = new Queue();

// barrageQueue.Count

barrageQueue.Enqueue("Hello"); // 入队

barrageQueue.Dequeue(); // 出队

barrageQueue.Peek(); // 看队首元素

barrageQueue.Clear() // 清空队列
```

#### 双端队列 Deque

使用 LinkedList 实现。

```c#
private LinkedList<string> barrageQueue;

LinkedList<string> deque = new LinkedList<string>();

// 添加元素
    barrageQueue.AddLast(1);     // 尾部添加
    barrageQueue.AddFirst(2);    // 头部添加

    // 删除元素
    barrageQueue.RemoveFirst();  // 删除头部
    barrageQueue.RemoveLast();   // 删除尾部

    // 访问元素
    Console.WriteLine("头部元素: " + barrageQueue.First.Value); // 输出 2
    Console.WriteLine("尾部元素: " + barrageQueue.Last.Value);  // 输出 1
```
