---
title: 在.Net中查找对象的根路径
date: 2016-05-08 21:34:24
categories:
- VirtualMachine
tags:
- .Net
- GC
---
### 背景
[上文](http://blog.wbscan.com/2016/05/08/HowToCatpureAndUseMiniDump/) 提到，Vistual Studio能够在加载了MiniDump文件后，查找特定对象在GC系统中的所有根路径 (关于GC和Root Set的概念，参见[Tracing GC](https://en.wikipedia.org/wiki/Tracing_garbage_collection))。这是非常强大的功能，而第一次注意到调试工具具备这个能力，是上次我在使用[dotMemory](https://www.jetbrains.com/help/dotmemory/10.0/Analyzing_GC_Roots.html?origin=old_help)的时候。
<!-- more -->
幸运的是，没过多久，我在自己用到的开源库中，也发现了演示这个功能的[Demo](https://github.com/Microsoft/dotnetsamples/tree/master/Microsoft.Diagnostics.Runtime/CLRMD/GCRoot)。
***
### clrmd介绍
先介绍下用到的库[clrmd](https://github.com/microsoft/clrmd):

{% blockquote Microsoft https://github.com/microsoft/clrmd Microsoft.Diagnostics.Runtime %}
CLR MD is a C# API used to build diagnostics tools. It gives you the power and flexibility of what the SOS and PSSCOR debugger extensions can do in a simple, fast C# API.

Some features include:

+ Memory Diagnostics
+ Walking the GC Heap.
+ Walking roots in the process.
+ Walking all heaps that CLR owns, such as JIT code heaps, AppDomain heaps, etc.
+ Walk threads in the process to get managed callstacks.
+ Walk AppDomains in the process.
+ Walk COM wrappers in your process (v4.5+ only).
+ And more...
{% endblockquote %}

之前同事用这个库attach到我们生产服的进程中，提取一些实时的CLR状态，后来我接手做一些收尾工作的时候，发现了它本身提供的[Examples](https://github.com/Microsoft/dotnetsamples/tree/master/Microsoft.Diagnostics.Runtime/CLRMD)。出于谨慎起见，我把每个Example给读了下，想看看产品代码里有没什么遗漏，结果发现了这个[Demo](https://github.com/Microsoft/dotnetsamples/tree/master/Microsoft.Diagnostics.Runtime/CLRMD/GCRoot)，它在做基于clrmd的根路径查找。
### 源码分析
我们直接开始分析它的代码。

首先它加载MiniDump：
{% codeblock lang:csharp %}
DataTarget dataTarget = DataTarget.LoadCrashDump(dump);
{% endcodeblock %}

{% blockquote %}
    顺带一说，这个库也是可以attach到live进程的: 
    {% codeblock lang:csharp %}
    public static DataTarget AttachToProcess(int pid, uint msecTimeout`)
    {% endcodeblock %}
{% endblockquote %}

然后枚举根集：
{% codeblock lang:csharp %}
    /// Enumerate the roots of the process.  (That is, all objects which keep other objects alive.)
    public abstract IEnumerable<ClrRoot> EnumerateRoots();
{% endcodeblock %}

我们看下所谓根对象(ClrRoot)是什么：(原谅我擅自去掉了一些无关信息)
{% codeblock lang:csharp %}
  /// Represents a root in the target process.  A root is the base entry to the GC's mark and sweep algorithm.
  public abstract class ClrRoot
  {
    /// A GC Root also has a Kind, which says if it is a strong or weak root
    public abstract GCRootKind Kind { get; }
    /// The name of the root.
    public virtual string Name;
    /// The type of the object this root points to.  That is, ClrHeap.GetObjectType(ClrRoot.Object).
    public abstract ClrType Type { get; }
    /// The object on the GC heap that this root keeps alive.
    public virtual ulong Object { get; protected set; }
    /// The address of the root in the target process.
    public virtual ulong Address { get; protected set; }
    /// Returns true if the root "pins" the object, preventing the GC from relocating it.
    public virtual bool IsPinned;
}
{% endcodeblock %}
这里我们比较感兴趣的是GCRootKind:
{% codeblock lang:csharp %}
  /// The type of GCRoot that a ClrRoot represnts.
  public enum GCRootKind
  {
    StaticVar = 0,
    ThreadStaticVar = 1,
    LocalVar = 2,
    Strong = 3,
    Weak = 4,
    Pinning = 5,
    Finalizer = 6,
    AsyncPinning = 7,
    Max = 7,
  }
{% endcodeblock %}
所以说，这里所谓ClrRoot对象，是指一个静态变量/线程局部变量/局部变量等，而它的值，是一个引用对象。

枚举的根基，被放到一个字典里面：
{% codeblock lang:csharp %}
static Dictionary<ulong, List<ClrRoot>> m_rootDict = new Dictionary<ulong, List<ClrRoot>>();
{% endcodeblock %}
其中，key是ClrRoot的值，value是指向相同引用对象的所有根(或者说局部变量和静态变量)。


查找算法在FindPathToTarget里面：
{% codeblock lang:csharp %}
private static Node FindPathToTarget(ClrHeap heap, ClrRoot root)
{% endcodeblock %}
它尝试判断是否存在从指定根root到目标对象的一条路径，如果有，则返回路径链表的头结点。

在函数体中，需要遍历对象任意对象的子引用，这就是个trace的过程:
{% codeblock lang:csharp %}
curr.Type.EnumerateRefsOfObject(curr.Object, delegate(ulong child, int offset)
{% endcodeblock %}

到这里，你已经可以想到FindPathToTarget的内部是怎么实现了，无非是对对象图的深度优先的搜索。由于这个图没有任何规律，所以搜索算法必须是迭代而不能是递归(否则爆栈)。

图搜索的过程中，一个对象（图节点）一旦被访问，就应该被标记，由于标记本身不能存储在对象内部（clrmd只暴露了只读接口），所以需要一个高效的标记集合表示（对象数可能非常大，考虑你可能是在几十GB的堆中进行搜索）。

为了存储对象标记，这个工具实现了这么个类：
{% codeblock lang:csharp %}
class ObjectSet
{
    public ObjectSet(ClrHeap heap);
    public void Add(ulong value);
    public bool Contains(ulong value);
    public int Count;
}
{% endcodeblock %}
如你所见，从接口上来说，其实就是个HashSet&lt;ulong&gt;，只是内部利用了object地址的特殊性，进行了优化。

具体来说，ClrHeap对象有个字段叫Segments，里面是一个ClrSegment的序列，其中ClrSegment的Start/End字段表示了这个Heap segment的地址范围。从ObjectSet的实现来看，每个对象的object size， 是对齐到指针大小的（IntPtr.Size），所以一个ClrSegment可以看做一系列Word，如果只是为了用作集合、做存在性测试的话，那么，一个Word可以表示为一个bit，于是ClrSegment就对应于一个BitArray，而一个object就是BitArray中一个连续的bit序列。由于对象本身一定不会重叠，因此，我们关心的实际只是对象头所在的bit，因此，判断一个对象是否存在，其实就是判断对象头所对应的bit有没在BitArray中被标记。
基于这个讨论，可以看到Contains是这么实现的：
{% codeblock lang:csharp %}
public bool Contains(ulong value)
{
    if (value == 0) return m_zero;
    int segmentIndex = GetSegmentIndex(value);
    if (segmentIndex == -1) return false;
    int headBitIndex = (value - m_segments[segmentIndex].Low) / IntPtr.Size;
    return m_bitArrays[segmentIndex][headBitIndex];
}
{% endcodeblock %}

可以看到，ObjectSet通过识别ClrSegment集合，来针对地址空间的非连续性做优化；又通过object size是字长对齐的这一特点，以一个bit代表一个字。ObjectSet在这里做的优化，确保了，即使被查找的进程堆很大，搜索进程的内存占用至少在1/64以下（x64的时候）。
### 结语

其实所谓根路径查找的算法很容易理解，难得的是这里一个几百行的小Demo，就演示了怎样对一个产品级的虚拟机进行搜索，挺有意思。

### 其他Example
clrmd的[Examples](https://github.com/Microsoft/dotnetsamples/tree/master/Microsoft.Diagnostics.Runtime/CLRMD)里面还有其他几个例子，我也稍微提一下：

+ DumpHeapLive: 打印所有可达的GC对象。结合上面的ObjectSet，它进行了可达性追踪。
+  DumpDiff: 把两个MiniDump中的对象，以对象类型聚合过后，比较两个聚合结果在相同类型上的对象数、总尺寸的变化。
+ DumpDict: 打印给定地址的中的`System.Collections.Generic.Dictionary`键值对
+ ClrStack: 打印所有线程的调用栈，以及栈上局部变量。