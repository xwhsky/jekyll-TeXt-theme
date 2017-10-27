---
layout: post
title: .NET性能优化的几点建议（2013年）
category: c#,optimization
---

> 转自自赵劼2013年InfoQ的内容
> [原文阅读](http://www.infoq.com/cn/articles/C-sharp-performance-optimization)

性能主要指两个方面：内存消耗和执行速度。性能优化简而言之，就是在不影响系统运行正确性的前提下，使之运行地更快，完成特定功能所需的时间更短。

本文以.NET平台下的控件产品MultiRow为例，描述C#性能优化的实践。

## 性能优化原则

### 理解需求

MultiRow的一个性能需求是：“百万行数据绑定下平滑滚动。”整个MultiRow项目的开发过程一直在考虑这个目标。

### 理解瓶颈

99%的性能消耗是由于1%的代码造成的。大部分性能优化都是针对这1%的瓶颈代码进行的。具体实施也就分为两步：“发现瓶颈”和“消除瓶颈”。

### 切忌过度

性能优化本身是有成本的。这个成本不单单体现在做性能优化所付出的工作量，还包括为性能优化而写出复杂的代码导致额外的维护成本，比如引入新的Bug，额外的内存开销等。性能优化常常需要在收益和成本之间做出权衡。

### 如何发现性能瓶颈

性能优化的第一步是发现性能瓶颈，下面是一些定位性能瓶颈的实践。

### 如何获取内存消耗

以下代码可以获取某个操作的内存消耗。

```c#
long start = GC.GetTotalMemory(true);
// 在这里写需要被测试内存消耗的代码，例如，创建一个GcMultiRow
var gcMulitRow1 = new GcMultiRow();
GC.Collect();
// 确保所有内存都被GC回收
GC.WaitForFullGCComplete();
long end = GC.GetTotalMemory(true);
long useMemory = end - start; 
```

### 如何获取时间消耗

以下代码可以获取某个操作时间消耗。

```c#
System.Diagnostics.Stopwatch watch = new System.Diagnostics.Stopwatch();
watch.Start();
for (int i = 0; i < 1000; i++)
{
	gcMultiRow1.Sort();
}
watch.Stop();
var useTime = (double)watch.ElapsedMilliseconds / 1000;
```

为了获得更加稳定的时间消耗，这里把一个操作循环执行了1000次，取时间消耗的平均值以排除不稳定数据。

### ANTS Performance Profiler

ANTS Performance Profiler是款功能强大的性能检测软件。熟练使用这个工具，我们可以快速准确的定位到有性能问题的代码。这是一款收费软件，会在IL中加入一些钩子用来记录时间，所以在分析时，软件的执行速度会比实际运行慢一些，获得的数据也因此并不是百分之百的准确，还要结合其他技巧来分析程序的性能。

### CodeReview

CodeReview是发现性能问题的最后手段。CodeReview应该对产品的性能瓶颈尽可能多的关注，确保该部分逻辑执行的尽可能的快。

## 性能优化的方法和技巧

定位了性能问题后，解决的办法有很多。下面是一些性能优化的技巧和实践。

### 优化程序结构

在设计时就应该考虑产品结构是否可以达到性能需求。如果后期发现了性能问题，调整结构会带来非常大的开销。

例如：

GcMultiRow要支持100万行数据。假设每行有10列的话，就需要有1000万个单元格，每个单元格上又有很多的属性。如果不做任何优化，大数据量时，一个GcMultiRow软件的内存开销会相当的大。GcMultiRow采用的方案是使用哈希表来存储行数据：只有用户改过的行放到哈希表里，大部分没有改过的行都直接使用模板代替。这就达到了节省内存的目的。

WPF平台和Silverlight平台的画法和Winform平台不同，是通过组合Visual元素的方法实现的。SpreadGrid for WPF产品同样支持百万级的数据量，但是又不能给每个单元格都分配一个View。所以SpreadGrid使用了VirtualizingPanel来实现画法。思路是每一个Visual是一个Cell的展示模块，可以和Cell的数据模块分离，这样就只需要为显示出来的Cell创建Visual。当发生滚动时会有一部分Cell滚出屏幕，有一部分Cell滚入屏幕。这时，让滚出屏幕的Cell和Visual分离，然后再复用这部分Visual给新进入屏幕的Cell。如此循环，就只需要几百个Visual就可以支持很多的Cell。

### 缓存

缓存（Cache）是性能优化中最常用的手段，针对需要频繁的获取一些数据，同时每次获取数据需要的时间比较长的场景。如果使用了缓存的优化方法，需要特别注意缓存数据的同步：如果真实的数据发生了变化，应该及时的清除缓存数据，确保不会因为缓存而使用了错误的数据。

使用缓存的情况比较多。最简单的情况就是缓存到一个Field或临时变量里。

```c#
for（int i = 0; i < gcMultiRow.RowCount; i++）
{ 
// Do something; 
} 
```

以上代码一般情况下是没有问题的，但是，如果GcMultiRow的行数比较大。而RowCount属性的取值又比较慢的时候，就需要使用缓存来做性能优化。

```c#
int rowCount = gcMultiRow.RowCount;
for (int i = 0; i < rowCount; i++)
{
// Do something;
}
```

使用对象池也是一个常见的缓存方案，比使用Field或临时变量稍微复杂一点。例如，在MultiRow中，画边线，画背景，需要用到大量的Brush和Pen。这些GDI对象每次用之前要创建，用完后要销毁。创建和销毁的过程是比较慢的。GcMultiRow使用的方案是创建一个GDIPool。本质上是一些Dictionary，使用颜色做Key。所以只有第一次取的时候需要创建，以后就直接使用以前创建好的。

以下是GDIPool的代码：

```c#
ublic static class GDIPool 
{ 
	Dictionary<Color, Brush > _cacheBrush = new Dictionary<Color, Brush>(); 
	Dictionary<Color, Pen> _cachePen = new Dictionary<Color, Pen>(); 
	public static Pen GetPen(Color color) 
	{ 
		Pen pen; 
		if_cachePen.TryGetValue(color, out pen)) 
		{ 
			return pen; 
		} 
		pen = new Pen(color); 
		_cachePen.Add(color, pen); 
		return pen; 
	} 
}
```

### 懒构造

大多时候，对于创建需要花费较长时间的对象，往往并不是所有的场景下都需要使用。这时，使用懒构造的方法可以有效提高程序启动性能。

举例来说，对象A需要内部创建对象B。对象B的构造时间比较长。 一般做法：

```c#
public class A
{
	public B _b = new B();
}
```

一般做法下，由于构造对象A的同时要构造对象B，导致A的构造速度也变慢了。

优化做法：

```c#
public class A
{
	private B _b;
	public B BProperty
	{
		get
		{
			if(_b == null)
			{
				_b = new B();
			}
			return _b;
		}
	}
}
```

优化后，构造A的时候就不需要创建B对象，有效的提高了A的构造性能。

### 优化算法

优化算法可以有效的提高特定操作的性能。使用一种算法时应该了解算法的适用情况、最好情况和最坏情况。 以GcMultiRow为例，最初MultiRow的排序算法使用了经典的快速排序算法。这看起来是没有问题的。但是，对于表格软件，用户经常的操作是对有序表进行排序，如顺序和倒序之间切换。而经典的快速排序算法的最差情况就是基本有序的情况。所以经典快速排序算法不适合MultiRow。

改进的快速排序算法使用了3个中点来代替经典快排的一个中点的算法，每次交换都是从3个中点中选择中间值。这样，乱序和基本有序的情况都不是这个算法的最坏情况，从而优化了性能。

### 正确的使用既有数据结构

我们现在工作的.NET framework平台有很多现成的数据结构。我们应该了解这些数据结构，提升我们程序的性能。

例如：

1. String的加运算符和StringBuilder： 字符串的操作是我们经常遇到的基本操作之一。 我们经常会写这样的代码 string str = str1 + str2。当操作的字符串很少的时候，这样的操作没有问题。但是如果大量操作的时候（例如文本文件的Save/Load， Asp.net的Render），这样做就会带来严重的性能问题。这时，我们就应该用StringBuilder来代替string的加操作。
2. Dictionary和List: Dictionary和List是最常用的两种集合类。选择正确的集合类可以很大的提升程序的性能。为了做出正确的选择，我们应该对Dictionary和List的各种操作的性能比较了解。 下表中粗略的列出了两种数据结构的性能比较。

|       操作       | List | Dictionary |
| :------------: | :--: | :--------: |
|       索引       |  快   |     慢      |
| Find（Contains） |  慢   |     快      |
|      Add       |  快   |     慢      |
|     Insert     |  慢   |     快      |
|     Remove     |  慢   |     快      |

3. TryGetValue: 对于Dictionary的取值，比较直接的方法是如下代码：

```c#
if(_dic.ContainKey("Key")
{
	return _dic["Key"];
}  
```

当需要大量取值的时候，这样的取法会带来性能问题。优化方法如下：

```c#
object value;
if(_dic.TryGetValue("Key", out value))
{
return value;
}
```

后一种用法要比前一种用法取值性能提高一倍。

4. 为Dictionary选择合适的Key: Dictionary的取值性能很大情况下取决于做Key的对象的Equals和GetHashCode两个方法的性能。如果可以的话，使用Int做Key性能最好。如果是一个自定义的Class做Key的话，最好保证以下两点：1. 不同对象的GetHashCode重复率低。2. GetHashCode和Equals方法简单，效率高。
5. List的Sort和BinarySearch性能很好，如果能满足功能需求，推荐直接使用。

```c#
List<int> list = new List<int>{3, 10, 15};
list.BinarySearch(10); // 对于存在的值，结果是1
list.BinarySearch(8); // 对于不存在的值，会使用负数表示位置，
// 如查找8时，结果是-2， 查找0结果是-1，查找100结果是-4.
```

### 通过异步提升响应时间

#### 多线程

有些操作确实需要花费比较长的时间。在处理的过程中，如果用户进行操作时失去响应，这个用户体验是很差的。使用多线程技术可以解决这个问题。例如，有一个类似Excel的计算引擎，在构造的时候要初始化所有的函数定义。由于函数比较多，初始化时间会比较长。这是如果用到了多线程，在工作线程中做函数定义进行的初始化，就不会影响到UI线程快速响应用户的其他操作了。

代码如下:

```c#
public CalcParser()
{
 if (_functions == null)
  {
   lock (_obtainFunctionLocker)
   {
    if (_functions == null)
    {
      System.Threading.ThreadPool.QueueUserWorkItem((s) => 
      {
       if (_functions == null) 
       { 
         lock (_obtainFunctionLocker) 
         { 
           if (_functions == null) 
           { 
             _functions = EnsureFunctions(); 
            }
           }
         } 
        }); 
       } 
      } 
     } 
    } 
```

这里比较慢的操作就是EnsureFunctions函数，是在另一个线程里执行的，不会影响主线程的响应。当然，使用多线程是一个比较有难度的方案，需要充分考虑跨线程访问和死锁的问题。

#### 加延迟时间

在GcMultiRow实现AutoFilter功能的时候使用了一个类似于延迟执行的方案来提升响应速度。AutoFilter的功能是用户在输入的过程中根据用户的输入更新筛选的结果。数据量大的时候一次筛选需要较长时间，会导致用户输入不流畅，体验不好。使用多线程虽然是个好方案，但是会增加程序的复杂度。MultiRow的解决方案是当接收到用户的键盘输入消息的时候，并不立即出发Filter，而是等待0.3秒。如果用户连续输入，会在这0.3秒内再次收到键盘消息，放弃上一个任务，再等0.3秒，直到连续0.3秒内没有新的键盘消息时再触发Filter。这样就实现了比较流畅的用户体验。

#### Application.Idle事件

在GcMultiRow的Designer里，经常要根据当前的状态刷新ToolBar上按钮的Disable/Enable状态，一次刷新需要较长的时间。这个又一次影响了用户输入的流畅性。GcMultiRow的优化方案是通过系统的Application.Idle事件，仅当系统空闲的时候处理刷新逻辑。接到这个事件时，一般都是用户已经完成了连续的输入，这时就可以从容的刷新按钮的状态了。

#### Refresh, BeginInvoke

平台本身也提供了一些异步方案。例如在WinForm下触发一块区域重画的时候，调用Refresh方法不会导致立即重画，而是设置Invalidate标记，触发异步的刷新。在控件开发中，这个技巧可以有效的提高产品的性能，同时简化实现复杂度。

Control.BeginInvoke方法可以被用来触发异步的自定义行为。

### 进度条，提升用户体验

有时候，以上提到的方案都没有办法快速响应用户操作。进度条、一直转圈圈的图片、提示性文字（如"你的操作可能需要较长时间，请耐心等待"）等，都可以有效的提升用户体验，可以作为最后方案来考虑。