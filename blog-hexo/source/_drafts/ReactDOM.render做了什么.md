---
title: react暇谈系列一——当我们再谈论render时，我们在谈论什么
date: 2019-10-02 19:00:03
tags:
  - react
  - 前端
  - react暇谈
---

- 创建 ReactRoot 对象，并将其挂载到目标 DOM 的 _reactRootContainer 属性上
- 创建 ReactRoot 对象的同时，清空目标 DOM 的内部结构
- ReactRoot 创建了 FiberRoot 对象，挂载到了自身 _internalRoot 属性上
- FiberRoot 创建了 RootFiber   current 身上，同时将自身挂到 RootFiber.stateNode 上
- - ReactRoot 不是 Fiber 对象，不属于 Fiber 一类的，不要搞混了
- - ReactDOM 有各种 type 的 Fiber 对象，RootFiber 是一个 type 为 HostRoot 的 Fiber 对象
- - - 其特殊点在于一般 fiber 对象的 stateNode 属性是真实的 DOM 节点对象，而 RootFiber 的 stateNode 是 FiberRoot
- 调用 ReactRoot 的 render 方法
- 计算了下这次 render 任务的过期时间，计算后得到本次任务优先级是最高的，属于 Sync
- 准备调度
- 生成这次任务对应的 update，推入 updateQueue
- - 由于是第一次任务，RootFiber 上并不存在 updateQueue，创建一个
- 开始调度
- 找到本次更新所在的 FiberRoot，加入到调度队列中
- - 一个网页可能存在多个 FiberRoot，在多次调用 render 方法后
- - 因此调度的起始点是 FiberRoot，即使大部分情况下，网页只存在一个 FiberRoot 对象
- 因此第一次 render 的任务的优先级是 Sync，所以立即开始更新