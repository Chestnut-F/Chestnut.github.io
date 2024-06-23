---
layout:     post
title:      "【AI】3D飞行导航"
subtitle:   "基于稀疏体素八叉树"
date:       2024-06-18 21:14:00 +0800
author:     "Chestnut"
header-img: 'img/bg-stardew-valley.jpg'
tags:
    - 虚幻
    - AI
    - 导航
---

## 前言

参考了[Warframe](http://www.gameaipro.com/GameAIPro3/GameAIPro3_Chapter21_3D_Flight_Navigation_Using_Sparse_Voxel_Octrees.pdf)的文章，据我所知国内的一些仙侠游戏使用的也是完全同款的方案。

具体方案请直接参考文章，本文记录些实现过程中遇到的问题。

[项目代码](https://github.com/Chestnut-F/FlightNavigation)

## SVO

### 体素化

参考Recast，利用了UE自带那个导航八叉树，先找到相关的三角形几何数据，然后判定三角形和体素是否相交。

三角形和AABB的相交检查方案，使用的是[Tomas Akenine-Möller](https://fileadmin.cs.lth.se/cs/Personal/Tomas_Akenine-Moller/code/tribox_tam.pdf)的方案，有[源代码](https://fileadmin.cs.lth.se/cs/Personal/Tomas_Akenine-Moller/code/tribox3.txt)

### 多线程策略

体素化是最耗时的，必须多线程。我的方案是计算好会有多少个LayerOneNode，每个LayerOneNode一个线程。

生不生成线程根据LayerOneNode的Box是否和NavBounds相交决定。

### 八叉树生成

LayerOneNode全有了，逐层向上生成就好了。需要注意的是要补齐8个节点，8节点不剔除的原因论文有讲。

### 生成邻居

生成邻居只生成非叶子节点邻居，只生成层数大于等于自身层数的邻居，这是由于内存有限，存6个邻居已经很多了。（我甚至见过不存邻居的方案，但是运行时性能糟糕太多）

### 拼接（todo）

类似Recast的多Tile拼接，带Streaming的需要，后续看看。

### 动态SVO（废弃）

体素我非常不倾向于做动态，除非整个场景就是体素的。因此这个议题直接标注废弃。

## 寻路

### 投影

投影需要做一下Snap，先把起始点Snap到格子中心，然后就是很标准的八叉树定位，避免边界点导致的奇怪问题。

### 找邻居

由于非叶子节点都保存了6个邻居，所以难点其实就是叶子的找邻居。运行时没办法像离线状态下直接算邻居的莫顿码直接Find，所以存了ParentLink，通过Parent的邻居找邻居。这里会在LeafNode中加一个4字节内存。

### Traversal Cost

可以按文中说的方案优化，使用Unit Cost，效果是有的，会倾向于使用更大空间，但也可能会导致绕路。

### 路径平滑（todo）

### Raycast（todo）

用来做Theta*也好，用来做直线是否可达也好，都是很好用的辅助函数。

八叉树的Raycast方案相当成熟，有空找个论文抄一下。

### 多线程寻路（todo）

不做动态SVO，多线程是比较容易实现的，有空搞一下。

## 胶水

### 序列化

UE这套序列化内容有个很难受的地方在于假如你需要迭代数据结构，反序列化的时候很容易崩溃，因为你的数据存在StartLevel里，开发时连打开引擎的机会都没有。需要Recast搞个序列化Load过程的过滤，假如版本对不上，要跳过Load，然后重新保存。方案参考的Recast。

### Debug

#### SVO的可视化

用了MeshComponent实现，没用DebugDrawComponent的核心原因是，想加个透明材质好看一点。

#### Path的可视化

用UE自带的LineBatchComponent实现。

### NavigationData

FindPath是静态的，需要单独赋值。ConditionalConstructGenerator和GetBounds需要实现。Generator需要实现下面这些函数

```
virtual bool RebuildAll() override;                             // 重新Build
virtual void EnsureBuildCompletion() override;                  // 确保构建完成，编辑器BuildPath会跑这里
virtual void CancelBuild() override;                            // 取消构建
virtual void TickAsyncBuild(float DeltaSeconds) override;       // 编辑器Tick
virtual void OnNavigationBoundsChanged() override;              // 更新NavBounds的回调
virtual int32 GetNumRemaningBuildTasks() const override;        // 剩余构建任务
virtual int32 GetNumRunningBuildTasks() const override;         // 运行中的构建任务
```

## 性能

### 内存（todo）

数据待测试

### 寻路（todo）

这个实测小场景大约百us左右，基本问题不大，后续还可以做多线程优化。
