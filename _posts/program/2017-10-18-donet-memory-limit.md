---
layout: post
title: .NET单变量2G内存限制
tag: c#
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



## 瓶颈

> 更新于2017-11-2

上述方法，虽然使得单变量的存储突破了2G限制，但对于一个数组来说，其数组的最大长度也依旧有瓶颈。

在.NET 4.5 环境下，字节数组的最大长度不能大于2147483591。对于其他数值类型，长度不能大于2146435071。

这份说明同样在破解2G变量大小限制的MSDN官网有说明，之前没仔细翻到后面。

原文中说到：

> Using this element in your application configuration file enables arrays that are larger than 2 GB in size, but does not change other limits on object size or array size:
>
> - The maximum number of elements in an array is [UInt32.MaxValue](https://docs.microsoft.com/en-us/dotnet/api/system.uint32.maxvalue).
> - The maximum index in any single dimension is 2,147,483,591 (0x7FFFFFC7) for byte arrays and arrays of single-byte structures, and 2,146,435,071 (0X7FEFFFFF) for other types.
> - The maximum size for strings and other non-array objects is unchanged.



[^官网参考]: https://docs.microsoft.com/en-us/dotnet/framework/configure-apps/file-schema/runtime/gcallowverylargeobjects-element

