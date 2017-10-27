---
layout: post
title: .NET性能优化的几点建议
category: c#,optimization
---

> 本文引用自赵劼老师2017年QCon演讲的PPT内容

赵劼 @ 2017.10

## 自我介绍

- 赵劼 / 老赵 / 赵姐夫 / Jeffrey Zhao
- 2011年前：互联网
- 2011年起：IBM / JPMorgan Chase & Co.
- 编程语言，代码质量，性能优化……
- 云计算，机器学习，大数据，AI……一窍不通

## 说在前面

- 先评测，再优化
- 专注优化瓶颈
- 重视性能，保持性能导向思维
- 随时优化，持续优化
>**Donald Knuth**:
>
>We should forget about small efficiencies, say about 97% of the time: premature optimization is the root of all evil. *Yet we should not pass up our opportunities in that critical 3%.*

## 零：兵来将挡，水来土掩

### 字符串拼接与StringBuilder

```c#
string Concat(string a, string b, string c, string d) {
  return a + b + c + d;
}
```

```c#
string Concat(string a, string b, string c, string d) {
  return new StringBuilder()
    .Append(a).Append(b)
    .Append(c).Append(d)
    .ToString();
}
```

``` C#
string Concat(int n, string a, string b, string c, string d) {
  var s = "";
  for (var i = 0; i < n; i++) {
    s = s + a + b + c + d;
  }
  return s;
}
```

```c#
string Concat(int n, string a, string b, string c, string d) {
  var sb = new StringBuilder();
  for (var i = 0; i < n; i++) {
    sb.Append(a).Append(b)
      .Append(c).Append(d)
      .ToString();
  }
  return sb.ToString();
}
```

## 一：了解内存布局

### 老生常谈

- 引用类型
  - 分配在托管堆
  - 受GC管理，影响GC性能
  - 自带头数据（如类型信息）
- 值类型
  - 分配在栈上，或为引用类型对象的一部分
  - 分配在栈时不用显示回收
  - 没有头数据（体积紧凑）
- 注意：分配位置（堆/栈）为实现细节

### 获取对象尺寸

```assembly
> !dumpheap -stat
              MT    Count    TotalSize Class Name
...
000007fef5c1fca0        5          120 System.Object
...

> !dumpheap -mt 000007fef5c1fca0        
         Address               MT     Size
0000000002641408 000007fef5c1fca0       24     
00000000026428a8 000007fef5c1fca0       24     
0000000002642f48 000007fef5c1fca0       24     
0000000002642f80 000007fef5c1fca0       24     
0000000002645038 000007fef5c1fca0       24
```

- 优点：细节丰富，不含额外对象。
- 缺点：使用麻烦，不含对齐信息。

```c#
var currentBytes = GC.GetTotalMemory(true);
var obj = new object(); // Or other types
var objSize = GC.GetTotalMemory(true) - currentBytes;

Console.WriteLine(objSize);

// Output:
// 12 in x86
// 24 in x64

GC.KeepAlive(obj);
```

- 优点：使用简单，包含对齐信息。
- 缺点：丢失细节，包含额外对象。

### 引用类型对象布局

```C#
class Person {
  private readonly long _id;
  private readonly string _name;
}
```

![obj-layout](http://zhaojie.me/dotnet-perf/slides/img/obj-layout.svg)

### 引用类型对象尺寸

``` C#
class MyType1 {
  int Field1; // addr+8, 4 bytes
} // 24 bytes

class MyType2 {
  int Field1; // addr+8, 4 bytes
  int Field2; // addr+12, 4 bytes
} // 24 bytes

class MyType3 {
  int Field1;    // addr+8, 4 bytes
  string Field2; // addr+16, 8 bytes (alignment)
} // 32 bytes
```

### 基础类型数组尺寸

```C#
new int[0]; // 24 bytes (8 header + 8 MT + 4 length + 4)
new int[9]; // 64 bytes (24 + 4 * 9 + 4)
new int[10]; // 64 bytes (24 + 4 * 10)
new int[11]; // 72 bytes (24 + 4 * 11 + 4)

new byte[8]; // 32 bytes (24 + 1 * 8)
new byte[9]; // 40 bytes (24 + 1 * 9 + 7)

new bool[1]; // 32 bytes (24 + 1 * 1 + 7)
new bool[2]; // 32 bytes (24 + 1 * 2 + 6)
...
new bool[8]; // 32 bytes (24 + 1 * 8)
new bool[9]; // 40 bytes (24 + 1 * 9 + 7). So bool = byte?
// Consider using BitArray
```

### 自定义值类型数组尺寸

```C#
struct MyStruct {

  bool Field1; // 1 bit

  int Field2;  // 4 bytes

  bool Field3; // 1 bit

}

new MyStruct[3]; // 64 bytes (24 + X * 3) => X = 12?

```

```shell
> !do 0000000002682e38 
...
Fields:
 MT    Field   Offset                 Type ... Name
...  400004a        8       System.Boolean ... Field1
...  400004b        c         System.Int32 ... Field2
...  400004c       10       System.Boolean ... Field3
```



```C#
[StructLayout(LayoutKind.Auto)]
struct MyStruct {
  bool Field1; // 1 bit
  int Field2;  // 4 bytes
  bool Field3; // 1 bit
}

new MyStruct[3]; // 48 bytes (24 + 8 * 3)
```

```assembly
> !do 0000000002932e38 
...
Fields:
 MT    Field   Offset                 Type ... Name
...  400004a        c       System.Boolean ... Field1
...  400004b        8         System.Int32 ... Field2
...  400004c        d       System.Boolean ... Field3
```

### 如何改进？

```c#
class MyItem { }

static IEnumerable<MyItem> GetItems() { 
  // ...
}

// Initialization
var allItems = GetItems().ToArray();

// Iteration
foreach (var item in allItems) {
  // do something with item
}
```

```C#
class MyItem { MyItem Next; }

// Initialization
MyItem head = null;
foreach (var item in GetItems()) {
  head = new Item { Next = head };
}

Reverse(head); // optional

// Iteration
for (var item = head; item != null; item = item.Next) {
  // do something with item
}
```

### 改进后

- 节省内存（可忽略）、无大对象（重要）
- 指令少：少一层间接访问，无数组越界检查
- 布局紧凑：CPU缓存利用得当

### 延伸：双向链表

```C#
abstract class InplaceLinkedListNode<T>
  where T : InplaceLinkedListNode<T>
{
  T Prev;
  T Next;
} // or interface

class InplaceLinkedList<T>
  where T : InplaceLinkedListNode<T> { }

class MyItem : InplaceLinkedListNode<MyItem> { }
```

### 延伸：二叉树

```C#
abstract class InplaceBinaryTreeNode<T>
  where T : InplaceBinaryTreeNode<T>
{
  T Left;
  T Right;
  int Size;
} // or interface

class InplaceAvlTree<T>
  where T : InplaceBinaryTreeNode<T> { }

class MyItem : InplaceBinaryTreeNode<MyItem> { }
```

## 二：迎合GC编程

> **Roslyn Team: **
>
> You might think that building a responsive .NET Framework app is all about algorithms, such as using quick sort instead of bubble sort, but that's not the case. The biggest factor in building a responsive app is allocating memory, especially when your app is very large or processes large amounts of data.																		

### GC in CLR vs. OpenJDK

- 缺点：配置选项少。
  - Workstation GC vs. Server GC
  - Concurrent GC (Why disable?)
- 优点：配置选项少，但对内存控制程度高。
  - 与其调整参数，不如迎合GC编程
  - 拥有更多避免内存分配的特性

### 老生常谈：托管堆

- Gen 0：新对象，短期对象
- Gen 1：Gen 0和Gen 2之间的过渡
- Gen 2：老对象，长期对象
- 回收：扫描，清理，压缩

### 老生常谈：大对象堆（LOH）

- 保存大于85K的对象
- 随着Gen 2回收而回收
- 回收：不压缩，容易产生碎片

``` c#
// Force compact once
GCSettings.LargeObjectHeapCompactionMode = GCLargeObjectHeapCompactionMode.CompactOnce;
```

### 特点

- Gen 0与Gen 1：频率高，速度快
- 大对象堆与Gen 2：频率低，速度慢
  - 与已分配对象数量无关
  - 避免少部分有用对象引用大量无用对象
  - 减少老代对象指向新代对象的引用

### 如何避免垃圾回收

- 避免内存分配
  - 绝大部分垃圾回收由内存分配引起
  - 其他原因：系统资源紧张、主动触发
- 停止垃圾回收

```C#
// Suppress Gen 2 collection (Workstation only)
GCSettings.LatencyMode = GCLatencyMode.LowLatency;
```
```C#
// Suppress full (non-concurrent) Gen 2.
GCSettings.LatencyMode = GCLatencyMode.SustainedLowLatency;
```
```C#
GC.TryStartNoGCRegion(...);
// ... No GC will happen here
GC.EndNoGCRegion();
```

### 例：获取CPU使用率

```c#
// .NET 4.5.2-
var cpuCounter = new PerformanceCounter(
  "Processor",
  "% Processor Time",
  "_Total");

for (var i = 0; i < 10000; i++) {
  cpuCounter.NextValue();
}
```

![img](http://zhaojie.me/dotnet-perf/slides/img/perf-counter-allocations.png)

### 关注分配细节

```C#
void Print(int i) {
  Console.WriteLine("i = " + i); // allocations?
}
```
```C#
void Print(int i) {
  // String.Concat(object o1, object o2)
  Console.WriteLine(String.Concat("i = ", i));
}
```
```C#
void Print(int i) {
  // String.Concat(string s1, string s2)
  Console.WriteLine("i = " + i.ToString()); // avoid boxing
}
```
```c#
enum Color { Red, Green, Yellow }

class Light {
  public readonly Color Color;

  public override int GetHashCode() {
    return Color.GetHashCode(); // allocations?
  }
}
```

```c#
class Light {
  public override int GetHashCode() {
    return ((int)Color).GetHashCode(); // avoid boxing
  }
}
```

### 其他常见内存分配

- 委托（Delegate）
- 匿名函数（Lambda表达式）
- 字符串操作

### 如何优化？

```c#
string Concat(int n, string a, string b, string c, string d) {
  var sb = new StringBuilder();
  
  for (var i = 0; i < n; i++) {
    sb.Append(a).Append(b)
      .Append(c).Append(d)
      .ToString();
  }
  
  return sb.ToString();
}
```

改进：

```c#
string Concat(int n, string a, string b, string c, string d) {
  var length = CalculateLength(n, a, b, c, d);
  var sb = new StringBuilder(length);
  
  for (var i = 0; i < n; i++) {
    sb.Append(a).Append(b)
      .Append(c).Append(d)
      .ToString();
  }
  
  return sb.ToString();
}
```

### Roslyn：重用StringBuilder

```c#
static StringBuilder AcquireBuilder() {
  var result = cachedStringBuilder; // [ThreadStatic]
  if (result == null)  
    return new StringBuilder();

  result.Clear();
  cachedStringBuilder = null;  
  return result;  
}
```

```c#
static string GetStringAndReleaseBuilder(StringBuilder sb) {  
    var result = sb.ToString();  
    cachedStringBuilder = sb;  
    return result;  
}
```

```c#
string Concat(int n, string a, string b, string c, string d) {
  var sb = AcquireBuilder();
  
  for (var i = 0; i < n; i++) {
    sb.Append(a).Append(b)
      .Append(c).Append(d)
      .ToString();
  }
  
  return GetStringAndReleaseBuilder(sb);
}
```

### 复用（大）对象

- 例：StringBuilder，字节数组，对象数组…
- 线程专用，或线程安全栈
- 一次分配到位，避免LOH碎片

### 如何优化？

```c#
void DoSomethingNeedsTempArrayOfLength<T>(int n) {
  var tempArray = new T[n];
  // ...
}
```

```c#
abstract class Buffer<T> : IDisposable {
  // array like operations
  public abstract T this[int i] { get; set; }
}
            
void DoSomethingNeedsTempArrayOfLength<T>(int n) {
  using (var tempBuffer = RequestBuffer<T>(n)) {
    // ...
  }
}
```

改进：

```c#
class StructBuffer<T> : Buffer {
  private readonly T[] _array;
  
  public override T this[int i] {
    get { return _array[i]; }
    set { _array[i] = value; }
  }
}
```

```c#
class ClassBuffer<T> : Buffer {
  private readonly object[] _array;
  
  public override T this[int i] {
    get { return (T)_array[i]; }  
    set { _array[i] = value; }
  }
}
```

### 避免LOH碎片

- 预分配，避免自适应分配，例如
  - StringBuilder
  - MemoryStream
  - List<T>
- 分配统一尺寸的对象，或
- 统一尺寸的整数倍

### 理想中的对象生命周期

- 极短：分配后立即丢弃
  - 不会被GC扫描到
  - 不会被提升至Gen 2


- 极长：分配后永不丢弃
  - 快速提升至Gen 2后永久保留，避免压缩
  - 分配至LOH后永久保留，避免碎片

### 优化案例

```c#
class Order {
  public int OrderId { get; set; }
  public string Side { get; set; }
  public double Price { get; set; }
  // 200+ more fields ...
} // ~1.2KB

// 50000 instances max, ~60M
```

### 优化案例（结果）

```c#
struct OrderData {
  public int OrderId
  public string Side
  public double Price
  // 200+ more fields ...
} // ~1.2KB

// ~60M
static OrderData[] AllOrderData = new OrderData[50000];
```

```c#
struct Order {
  private readonly int _index;
  
  public int OrderId {
    get { return AllOrderData[_index].OrderId; }
    set { AllOrderData[_index].OrderId = value; }
  }
  
  public string Side {
    get { return AllOrderData[_index].Side; }
    set { AllOrderData[_index].Side = value; }
  }
  ...
} // zero in heap, 4 bytes in stack
```

### 优化后

- 优点
  - 更少的对象
  - 更短的GC扫描时间
  - 永不涉及回收的对象


- 缺点
  - 驻留内存较多
  - 重用存储空间带来额外复杂度

### 延伸：双向链表

```c#
var list = new LinkedList<int>();
for (var i = 0; i < 10000; i++) {
  list.AddLast(i);
}
```

```c#
// class LinkedListNode<int> {
//   ... 16 bytes head ...
//   Prev -> 8 bytes
//   Next -> 8 bytes
//   List -> 8 bytes
//   Value -> 4 bytes
//   ... 4 bytes alignment ...
// }
// 48 bytes node to save 4 bytes data

// nodes are everywhere in heap
```



```c#
class ArrayLinkedList<T> {
  private Node[] _nodes;
  
  public struct Node {
    public T Value;
    public int Next;
    public int Prev;
  }
  
  // "index" is node
  public int AddLast(T value) { }
}
```

```c#
// 12 bytes to save 4 bytes data
// nodes are kept closely (probably same cache line) in heap
```

### 延伸：二叉树

```c#
class ArrayAvlTree<T> {
  private Node[] _nodes;

  public struct Node {
    public T Value;
    public int Left;
    public int Right;
    public int Size;
  }
  
  // "index" is node
  public int Add(T value) { }
}
```

## 三、编写不通用的代码

### 代码调用

```c#
public class MyClass {
    [MethodImpl(MethodImplOptions.NoInlining)]
    public void NormalMethod() { }

    public virtual void VirtualMethod() { }
}
```
```c#
void CallNormal(MyClass c) {
    c.NormalMethod();
}

void CallVirtual(MyClass c) {
    c.VirtualMethod();
}
```
### 代码调用（非虚方法）

```c#
void CallNormal(MyClass c) {
    c.NormalMethod();
}
```

```assembly
488bca      mov rcx, rdx
3909        cmp [rcx], ecx ; null check
e8a2a4ffff  call 00007ffe`80c47b40 (MyClass.NormalMethod())
```

### 代码调用（虚方法）

```c#
void CallVirtual(MyClass c) {
    c.VirtualMethod();
}
```

```assembly
488bca    mov rcx, rdx
488b02    mov rax, [rdx]        ; address of method table
488b4040  mov rax, [rax+0x40]   ; address of methods
ff5020    call qword [rax+0x20] ; address of target method
```

### 集合

```c#
static ICollection<T> CreateCollection(IEnumerable<T> items) {
  return name.ToList();
}
```

> **Roslyn Team:**
>
> In Visual Studio and the new compilers, analysis shows that many of the dictionaries contained a single element or were empty. An empty Dictionary has ten fields and occupies 48 bytes on the heap on an x86 machine. Dictionaries are great when you need a mapping or associative data structure with constant time lookup. However, when you have only a few elements, you waste a lot of space by using a Dictionary.

### 集合创建（Roslyn）

```c#
static ICollection<string> CreateReadOnlyMemberNames(
  HashSet<string> names) {
  switch (names.Count) {
    case 0:
      return SpecializedCollections.EmptySet<string>();
    case 1:
      return SpecializedCollections.Singleton(names.First());
    case 2~6:
      return ImmutableArray.CreateRange(names);
    default:
      return SpecializedCollections.ReadOnlySet(names);
  }
  return names.ToList();
}
```

### FrugalList/Map in WPF

```c#
class FrugalListBase<T> { }
class SingleItemList<T> : FrugalListBase<T> { }
class ThreeItemList<T> : FrugalListBase<T> { }
class SixItemList<T> : FrugalListBase<T> { }
class ArrayItemList<T> : FrugalListBase<T> { }
```

```c#
class FrugalMapBase { }
class SingleObjectMap : FrugalMapBase { }
class ThreeObjectMap : FrugalMapBase { }
class SixObjectMap : FrugalMapBase { }
class ArrayObjectMap : FrugalMapBase { }
class SortedObjectMap : FrugalMapBase { }
class HashObjectMap : FrugalMapBase { }
class LargeSortedObjectMap : FrugalMapBase { }
```

### 不通用的迭代器

```c#
class MyCollection<T> : IEnumerable<T> {
  public struct MyEnumerator : IEnumerator<T> {
    // ...
  }
  
  public MyEnumerator GetEnumerator() {
    // ...
  }
  
  IEnumerator<T> IEnumerable<T>.GetEnumerator() {
    return GetEnumerator();
  }
  
  IEnumerator IEnumerable.GetEnumerator() {
    return GetEnumerator();
  }
}
```



```c#
var coll = new MyCollection<int>();

// has allocations:
// 1. ((IEnumerable<int>)coll).Where(...)
// 2. anonymous method
foreach (var i in coll.Where(i => i > 0)) {
  // Do something
}
```

```c#
// no allocation
foreach (var i in coll) {
  if (i > 0) {
    // Do something
  }
}
```

### 简易IDisposable实现

```c#
class MyClass {}
  private void Done(int i) {
    // ...
  }

  public IDisposable DoSomething(int i) {
    // ...
    return Disposables.Create(() => Done(i));
  }
}
```

```c#
// Has allocations
using (myClass.DoSomething(10)) {
  // Do something
}
```

### 不通用的IDisposable实现

```c#
class MyClass {
  public struct DoneToken : IDisposable {
    private readonly MyClass _parent;
    private readonly int _i;
    
    public void Dispose() {
      _parent.Done(_i);
    }
  }
    // no allocation
  public DoneToken DoSomething(int i) {
    // ...
    return new DoneToken(this, i);
  }
}
```

### 使用最新版本的C#/.NET

- ValueTask：轻量级Task
- ValueTuple：轻量级Tuple
- Span<T>：泛化版的String/ArraySegment
- ref return：安全地返回一个地址
- readonly ref：按引用传递只读变量

### 无ref return

```c#
interface IMap<TKey, TValue> {
  TValue Get(TKey key);
  void Set(TKey key, TValue value);
}
```

```c#
void Increase(IMap<TKey, int> map, TKey key, int n) {
  var value = map.Get(key);
  map.Set(key, value + n);
}
```

### ref return

```c#
interface IMap<TKey, TValue> {
  ref TValue GetRef(TKey key);
}
```

```c#
void Increase(IMap<TKey, int> map, TKey key, int n) {
  map.GetRef(key) += n;
}
```

## 五、善用工具

### 常备工具

- WinDBG
- PerfView
- 某商业化CPU Profiler
- 某商业化Memory Profiler
- CLRMD

### CLRMD

```c#
CLRRuntime runtime = ...;
GCHeap heap = runtime.GetHeap();

foreach (ulong obj in heap.EnumerateObjects()) {
    GCHeapType type = heap.GetObjectType(obj);
    
    if (type == null) // heap corruption
      continue;

    ulong size = type.GetSize(obj);
    Console.WriteLine(
      "{0,12:X} {1,8:n0} {2,1:n0} {3}",
      obj, size, heap.GetObjectGeneration(obj), type.Name);
}
```

## 建议回顾

- 兵来将挡，水来土掩
- 了解内存布局
- 迎合GC编程
- 编写不通用代码
- 善用工具

## Q & A

