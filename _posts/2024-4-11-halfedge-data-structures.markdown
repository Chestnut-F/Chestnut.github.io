---
layout:     post
title:      "如何检查Volume是否闭合？"
date:       2024-04-11 23:02:00 +0800
author:     "Chestnut"
background: '/img/posts/06.jpg'
tags:
    - 笔记
---

## 前言

问题的发现是性能测试的时候发现有一个Trigger Volume在Overlap的时候消耗异常的高。经过一阵Debug之后可以确定问题因为Volume不是闭合的。因为没有办法保证投放者对Volume的设计是闭合的，因此决定做一个资产检查工具。

问题来了，如何快速的检查复杂多面体是不是闭合的呢？

## 问题分析

![halfedge_small](/img/half-edge/halfedge_small.png "Halfedge")

闭合的Volume本质是复杂的多面体，面是由一个halfedge序列围成的。如果多面体缺少一个面，则必然至少存在一个边围绕这个孔洞，称之为border halfedge。如果Volume存在一个halfedge是border halfedge，则必然不是闭合的。

这样，问题的解决思路就非常明确了。
1. 搞定Halfedge Data Structures
2. 遍历一遍Halfedge，检查有没有边界边

## 解决方案

Halfedge Data Structures本身是很有用的数据结构，虚幻引擎有多半现成的，实际一搜确实不少相关，不过没有很方便的可以直接使用的，最后还是要自己写一下。

工作还是效率为主，于是重新想下这个问题，虚幻引擎的AVolume的本质是ABrush，数据则存储在UModel类中，编辑器下UModel大致代码如下：

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

看下来完全可以遍历每个Poly中的Vertex找到所有的边，然后检查下是否有边界边就好了，没有必要真的造一个Halfedge。由于Vertices储存的点一定是按固定方向环绕的，边界边的检查就是检查边AB有没有对应的边BA，如果没有就是边界边。于是动手实操，测试效果拔群，可喜可贺。

当然这个方案在判断两个点是否为同一个点的时候是存在一定误差的，这里只能寄希望于设计师们不要把两个顶点靠的太近了。

问题解决。

## 写在最后

一开始确实没有想到什么好的办法，在[CGAL](https://www.cgal.org/)上找到了思路。CGAL是真的算法宝库，做游戏的遇到什么几何问题都可以去看看。

## 参考文献

[1] [Halfedge Data Structures](https://doc.cgal.org/latest/HalfedgeDS/index.html#Chapter_Halfedge_Data_Structures)  
[2] [3D Polyhedral Surface](https://doc.cgal.org/latest/Polyhedron/index.html#Chapter_3D_Polyhedral_Surfaces)