---
layout: post
title: .NET单变量2G内存限制
category: C#
permalink: /program/
picture_frame: shadow #图片框样式，加阴影
typora-copy-images-to: ..\..\images\_posts
---

## 问题来源

最近在写回归模型，.net环境下使用[Math.NET Numerics](https://numerics.mathdotnet.com/#Math-NET-Numerics)矩阵库。

软硬件环境：WIN10（8G内存）、.NET 4.5、X64编译

以最小二乘法为例，需要求解帽子矩阵：

```c#
var hatmatrix = x_normalize.Multiply(x_normalize.Transpose().Multiply(x_normalize).Inverse()).Multiply(x_normalize.Transpose());
```



当数据量自变量样例达到20000时，hatmatrix达到20000 X 20000 ，单变量大小超过2G，出现内存溢出。



以创建20000 X 20000的双精度数组，

```c#
double[] array = new double[20000*20000];
```

同样出现内存溢出现象。

![donet-outofmemory](https://raw.githubusercontent.com/xwhsky/xwhsky.github.io/master/images/_posts/donet-outofmemory.png)

## 解决方案

这是.net在初始设计是对单变量2G大小的默认限制，与平台和内存大小无关。

这项默认设置已经有10年之久，直到.net 4.5版本的出现，允许用户在X64编译器上通过配置，使用超过2G的变量。

在App.Config文件下，配置如下即可：

```xml
<configuration>
  <runtime>
    <gcAllowVeryLargeObjects enable="true"/>
  </runtime>
</configuration>
```



详细信息，可参考 https://bhrnjica.net/2012/07/22/with-net-4-5-10-years-memory-limit-of-2-gb-is-over/





