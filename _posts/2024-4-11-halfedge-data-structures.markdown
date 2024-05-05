---
layout:     post
title:      "【几何】Volume的性能问题记录"
subtitle:   "网格体闭合检测"
date:       2024-04-11 23:02:00 +0800
author:     "Chestnut"
header-img: 'img/bg-stardew-valley.jpg'
tags:
    - 笔记
    - 虚幻
    - 性能
---

## 前言

问题的发现是性能测试的时候发现有一个Trigger Volume在Overlap的时候消耗异常的高。经过一阵Debug之后可以确定问题因为Volume不是闭合的。因为没有办法保证投放者对Volume的设计是闭合的，因此决定做一个资产检查工具。

问题来了，如何快速的检查复杂多面体是不是闭合的呢？

## 问题分析

简单的讲，Volume本质是一个流形网格，流形网格的每个边最多被两个面片共用。如果流形网格缺少一个面片，则必然至少存在一个边围绕这个孔洞，也就是说至少存在一个边只属于一个面片，这个边称之为边界边。如果Volume存在一个边界边，则必然是不闭合的。

![halfedge_small](/img/half-edge/halfedge_small.png "Halfedge")

如何才能快速的找到边属于哪个面片呢？需要引入一个数据结构，[Half-Edge 半边数据结构](https://www.flipcode.com/archives/The_Half-Edge_Data_Structure.shtml)。如图所示，一个面是由连续的半边序列围绕而成的。围绕方向既可以是顺时针或逆时针定向，只要保证一致即可。每个半边都会存储其围绕的面、终点处的顶点以及指向其对的指针。大致代码如下：
```
struct HE_edge
{
    HE_vert* vert;   // vertex at the end of the half-edge
    HE_edge* pair;   // oppositely oriented adjacent half-edge 
    HE_face* face;   // face the half-edge borders
    HE_edge* next;   // next half-edge around the face
};

struct HE_vert
{
    float x;
    float y;
    float z;

    HE_edge* edge;  // one of the half-edges emantating from the vertex
};

struct HE_face
{
    HE_edge* edge;  // one of the half-edges bordering the face
};
```

这样，问题的解决思路就非常明确了。
1. 把Volume转化成Halfedge Data Structures
2. 遍历一遍Halfedge，检查有没有HalfEdge的opposite为空

## 解决方案

工作还是效率为主，半边数据结构本身是很有用的数据结构，虚幻引擎有多半现成的，实际一搜确实不少相关，不过没有很方便的可以直接使用的，最后还是要自己写一下。

先检查下虚幻引擎的AVolume，虚幻的AVolume是继承ABrush的，数据则存储在UModel类中，编辑器下UModel大致结构如下：

```
class UModel : public UObject
{
#if WITH_EDITOR
    TObjectPtr<UPolys> Polys;
#endif // WITH_EDITOR
}

class UPolys : public UObject
{
    TArray<FPoly> Element;
}

//
// A general-purpose polygon used by the editor.  An FPoly is a free-standing
// class which exists independently of any particular level, unlike the polys
// associated with Bsp nodes which rely on scads of other objects.  FPolys are
// used in UnrealEd for internal work, such as building the Bsp and performing
// boolean operations.
//
class FPoly
{
    typedef TArray<FVector3f,TInlineAllocator<16>> VerticesArrayType;
    VerticesArrayType Vertices;
}
```

看下来可以轻松的构造一个Halfedge，于是动手实操，测试效果拔群，可喜可贺。

当然这个方案在判定两个点是否为同一个点的时候是存在一点误差的，大致查了下UE自身对FPoly结构的使用，判定是否为同一个点的方案是简单粗暴的PointsAreNear(A, B, 0.5f)，于是直接复用。

接口甩给CI负责人，问题解决。

## 写在最后

一开始确实没有想到什么好的办法，在[CGAL](https://doc.cgal.org/latest/Polyhedron/index.html#Chapter_3D_Polyhedral_Surfaces)上找到了思路。CGAL是真的算法宝库，做游戏的遇到什么几何问题都可以去看看。比较可惜的是CGAL本身是不开源的，不过也足够用了。

## 参考文献

[1] [Halfedge Data Structures](https://www.flipcode.com/archives/The_Half-Edge_Data_Structure.shtml)    
[2] [3D Polyhedral Surface](https://doc.cgal.org/latest/Polyhedron/index.html#Chapter_3D_Polyhedral_Surfaces)