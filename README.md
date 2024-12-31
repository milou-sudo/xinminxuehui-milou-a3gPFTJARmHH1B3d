
## 一：背景


### 1\. 讲故事


准备明年把`.NET高级调试的训练营`的课程进行重构，采用案例引导式，而CPU爆高类有不少是程序员在写代码的时候不注意时间复杂度，在数据量稍微大一点的情况直接幻化成了死循环，时间复杂度这东西是学校的数据结构课程，有心的朋友在写多层循环的时候脑子里面一定要过一遍，今天就给大家带一篇此类案例，也算是继续丰富我的新课程体系。


前些天有位朋友找到我，说他们的网站会有CPU瞬高的情况，在网上找相关资料最终找到我这边，想让我帮忙分析下咋回事？像这种CPU瞬高，比较好的方式就是用procdump自动化的抓取，万不可手工去抓，接下来就上 windbg 分析吧。


## 二：WinDbg分析


### 1\. 为什么会CPU爆高


以终为始，先看看CPU是否真的高，可以用 `!tp` 和 `!cpuid` 命令观察，这里稍微提一下，为什么要用 `!cpuid` 看看CPU的能力呢？这是因为我曾经分析过只有 2core 的CPU。尼玛，只有2个core，还分析个毛线哈，干脆把机器关了，这样CPU就不高了。。。自此以后我就留了一个心眼，输出参考如下：



```

0:033> !tp
CPU utilization: 100%
Worker Thread: Total: 5 Running: 5 Idle: 0 MaxLimit: 32767 MinLimit: 4
Work Request in Queue: 11
    Unknown Function: 00007ffbfaa417d0  Context: 00000283733c3718
    Unknown Function: 00007ffbfaa417d0  Context: 0000027f26f50cb0
    Unknown Function: 00007ffbfaa417d0  Context: 0000028377199f58
    AsyncTimerCallbackCompletion TimerInfo@0000028371c46820
    AsyncTimerCallbackCompletion TimerInfo@0000028371d06800
    Unknown Function: 00007ffbfaa417d0  Context: 00000283756d3248
    Unknown Function: 00007ffbfaa417d0  Context: 0000027f26f63578
    Unknown Function: 00007ffbfaa417d0  Context: 00000283733d0160
    Unknown Function: 00007ffbfaa417d0  Context: 00000283756a72d8
    Unknown Function: 00007ffbfaa417d0  Context: 00000283771a6828
    Unknown Function: 00007ffbfaa417d0  Context: 000002837719d1f8
--------------------------------------
Number of Timers: 0
--------------------------------------
Completion Port Thread:Total: 3 Free: 2 MaxFree: 8 CurrentLimit: 2 MaxLimit: 1000 MinLimit: 4
0:033> !cpuid
CP  F/M/S  Manufacturer     MHz
 0  6,106,6     2800
 1  6,106,6     2800
 2  6,106,6     2800
 3  6,106,6     2800


```

从卦中可以看出当前线程池队列稍有积压，5个托管线程全部被打满，并且当前机器是4个核，看样子是有4个线程在满负荷跑呀。。。


### 2\. 线程都在干什么


为了追踪线程都在干什么？使用 `~*e !clrstack` 观察各个线程的调用栈，结合程序的瞬高特性，捕获了一个相对来说高度可疑的代码，参考如下：



```

OS Thread Id: 0x2f00 (33)
        Child SP               IP Call Site
000000f2a42fd508 00007ffbfd72b0a7 System.String.Equals(System.String, System.String) [f:\dd\ndp\clr\src\BCL\system\string.cs @ 647]
000000f2a42fd510 00007ffba1715a0b xxx.StockAsyncDbTask+c__DisplayClass4_1.b__6(xxx.GoodsInfo)
000000f2a42fd540 00007ffba118c6ca System.Linq.Enumerable.FirstOrDefault[[System.__Canon, mscorlib]](System.Collections.Generic.IEnumerable`1, System.Func`2)
000000f2a42fd5b0 00007ffba1716008 xxx.StockAsyncDbTask+c__DisplayClass4_0.b__2(xxx.GoodsInfo)
000000f2a42fd670 00007ffbfd720505 System.Collections.Generic.List`1[[System.__Canon, mscorlib]].ForEach(System.Action`1) [f:\dd\ndp\clr\src\BCL\system\collections\generic\list.cs @ 553]
000000f2a42fd6c0 00007ffba1349e7e xxx.SaveStockToDb()
000000f2a42fd760 00007ffba13487ed xxx.DoWork()
000000f2a42fd7b0 00007ffba1348631 xxx.QuartzScheduler.QuartzJob.Quartz.IJob.Execute(Quartz.IJobExecutionContext)
000000f2a42fd8b0 00007ffba0f8ca12 Quartz.Core.JobRunShell+d__9.MoveNext()
000000f2a42fdb80 00007ffba0f83150 System.Runtime.CompilerServices.AsyncTaskMethodBuilder.Start[[Quartz.Core.JobRunShell+d__9, Quartz]](d__9 ByRef) [f:\dd\ndp\clr\src\BCL\system\runtime\compilerservices\AsyncMethodBuilder.cs @ 322]
000000f2a42fdc30 00007ffba0f8309d Quartz.Core.JobRunShell.Run(System.Threading.CancellationToken)
000000f2a42fdd30 00007ffba0f829f4 Quartz.Core.QuartzSchedulerThread+c__DisplayClass28_0.b__0()
000000f2a42fdd60 00007ffbfd7abe4e System.Threading.Tasks.Task`1[[System.__Canon, mscorlib]].InnerInvoke() [f:\dd\ndp\clr\src\BCL\system\threading\Tasks\Future.cs @ 680]
000000f2a42fddb0 00007ffbfd7aaf27 System.Threading.Tasks.Task.Execute() [f:\dd\ndp\clr\src\BCL\system\threading\Tasks\Task.cs @ 2498]
000000f2a42fddf0 00007ffbfd73df12 System.Threading.ExecutionContext.RunInternal(System.Threading.ExecutionContext, System.Threading.ContextCallback, System.Object, Boolean) [f:\dd\ndp\clr\src\BCL\system\threading\executioncontext.cs @ 980]
000000f2a42fdec0 00007ffbfd73dd95 System.Threading.ExecutionContext.Run(System.Threading.ExecutionContext, System.Threading.ContextCallback, System.Object, Boolean) [f:\dd\ndp\clr\src\BCL\system\threading\executioncontext.cs @ 928]
000000f2a42fdef0 00007ffbfd7ab1e1 System.Threading.Tasks.Task.ExecuteWithThreadLocal(System.Threading.Tasks.Task ByRef) [f:\dd\ndp\clr\src\BCL\system\threading\Tasks\Task.cs @ 2827]
000000f2a42fdfa0 00007ffbfd7aa8c1 System.Threading.Tasks.Task.ExecuteEntry(Boolean) [f:\dd\ndp\clr\src\BCL\system\threading\Tasks\Task.cs @ 2767]
000000f2a42fdfe0 00007ffbfd708e46 System.Threading.ThreadPoolWorkQueue.Dispatch() [f:\dd\ndp\clr\src\BCL\system\threading\threadpool.cs @ 820]


```

根据卦中的显示找到了问题方法，为了保护客户隐私，这里稍微会模糊一下，主要是看下复杂度的骨架结构。


![](https://img2024.cnblogs.com/blog/214741/202412/214741-20241230114532716-925278951.png)


从卦象看里面至少包含了3层for循环，所以时间复杂度是 O(N3\) 次方，学过数据结构和算法的朋友应该知道，这个复杂度不得了，要逆天了。。。


### 3\. O(N3\) 是祸根吗？


要想知道 O(N3\) 是不是祸根，得要看有没有给它不停的施肥翻土，可以找找相关的集合，使用 `!dso` 命令观察即可。



```

0:033> !dso
OS Thread Id: 0x2f00 (33)
RSP/REG          Object           Name
rbx              0000028227b4be70 xxx.GoodsInfo
000000F2A42FD520 0000028029c02038 System.Collections.Generic.List`1[[xxx.GoodsInfo, xxx.Model]]
...
000000F2A42FD708 0000027f277d3de8 System.Collections.Generic.Dictionary`2[[System.String, mscorlib],[xxx_DataGrab, xxx.Data]]

0:033> !do 0000027f277d3de8
Name:        System.Collections.Generic.Dictionary`2[[System.String, mscorlib],[xxx_DataGrab,xxx.Data]]
MethodTable: 00007ffba12b82f8
EEClass:     00007ffbfd345c10
Size:        80(0x50) bytes
File:        C:\Windows\Microsoft.Net\assembly\GAC_64\mscorlib\v4.0_4.0.0.0__b77a5c561934e089\mscorlib.dll
Fields:
              MT    Field   Offset                 Type VT     Attr            Value Name
00007ffbfd1d8538  4001887        8       System.Int32[]  0 instance 0000027f27d51328 buckets
00007ffbfe422618  4001888       10 ...non, mscorlib]][]  0 instance 0000027f27d51350 entries
00007ffbfd1d85a0  4001889       38         System.Int32  1 instance                1 count
00007ffbfd1d85a0  400188a       3c         System.Int32  1 instance                1 version
00007ffbfd1d85a0  400188b       40         System.Int32  1 instance               -1 freeList
00007ffbfd1d85a0  400188c       44         System.Int32  1 instance                0 freeCount
00007ffbfd1c7790  400188d       18 ...Canon, mscorlib]]  0 instance 00000282274c1978 comparer
00007ffbfd1c57c0  400188e       20 ...Canon, mscorlib]]  0 instance 0000027f27cfc630 keys
00007ffbfd1eef60  400188f       28 ...Canon, mscorlib]]  0 instance 0000000000000000 values
00007ffbfd1d5dd8  4001890       30        System.Object  0 instance 0000000000000000 _syncRoot

0:033> !do 0000028029c02038
Name:        System.Collections.Generic.List`1[[xxx, xxx.Model]]
MethodTable: 00007ffba126e830
EEClass:     00007ffbfd362af8
Size:        40(0x28) bytes
File:        C:\Windows\Microsoft.Net\assembly\GAC_64\mscorlib\v4.0_4.0.0.0__b77a5c561934e089\mscorlib.dll
Fields:
              MT    Field   Offset                 Type VT     Attr            Value Name
00007ffbfd1ee250  40018a0        8     System.__Canon[]  0 instance 00000283278195b0 _items
00007ffbfd1d85a0  40018a1       18         System.Int32  1 instance            21863 _size
00007ffbfd1d85a0  40018a2       1c         System.Int32  1 instance                0 _version
00007ffbfd1d5dd8  40018a3       10        System.Object  0 instance 0000000000000000 _syncRoot
00007ffbfd1ee250  40018a4        8     System.__Canon[]  0   static  


```

从卦中可以看到第一层的dictionary只有1条记录，第二层的 List 有高达 2\.1w 数据，第三层的 dbStocks 在线程栈没有找到，我也懒得找到了，起码发现了第二层的 list 确实比较大，加上数据的佐证，基本上就找到了问题所在，也满足程序的瞬高的现象。


### 4\. 解决方案


知道了复杂度高，优化的方向就是降低时间复杂度，将 O(N3\) 降低到 O(N)，方法就是在深层循环之前提前用 Dictionary 或者 HashSet 来预存数据，将后面的 for 循环变成字段的key查找，而key查找则是 O(1\)。


为了让大家有个宏观概念，我让 chatgpt 给我生成一个 O(N2\) 到 O(N) 的例子，参考代码如下：



```

public class GoodsInfo
{
    public int Spid { get; set; }
    // 其他属性...
}
 
public class Stock
{
    public int Spid { get; set; }
    // 其他属性...
}
 
public class Optimizer
{
    // 原始O(N^2)复杂度的查找方法
    public List FindMatchingGoodsInfoO_N2(List goodsInfos, List stocks, int targetSpid)
    {
        List result = new List();
        foreach (var goodsInfo in goodsInfos)
        {
            foreach (var stock in stocks)
            {
                if (goodsInfo.Spid == stock.Spid && stock.Spid == targetSpid)
                {
                    result.Add(goodsInfo);
                }
            }
        }
        return result;
    }
 
    // 优化后的O(N)复杂度的查找方法
    public List FindMatchingGoodsInfoO_N(List goodsInfos, List stocks, int targetSpid)
    {
        HashSet<int> stockSpids = new HashSet<int>(stocks.Select(s => s.Spid));
        List result = new List();
 
        foreach (var goodsInfo in goodsInfos)
        {
            if (stockSpids.Contains(goodsInfo.Spid) && goodsInfo.Spid == targetSpid)
            {
                result.Add(goodsInfo);
            }
        }
 
        return result;
    }
}


```

可以看到 chatgpt 很聪明，用 HashSet 来化煞。


## 三：总结


说实话像这种生产事故，我以前在公司的项目中也会偶发的遇到，都是赶时间，加班加点写出来的代码，只想把功能写出来早点下班，复杂度高？后面再说吧。。。代码写的太好，容易被老板优化。。。
![图片名称](https://images.cnblogs.com/cnblogs_com/huangxincheng/345039/o_210929020104%E6%9C%80%E6%96%B0%E6%B6%88%E6%81%AF%E4%BC%98%E6%83%A0%E4%BF%83%E9%94%80%E5%85%AC%E4%BC%97%E5%8F%B7%E5%85%B3%E6%B3%A8%E4%BA%8C%E7%BB%B4%E7%A0%81.jpg)


 本博客参考[飞数机场](https://ze16.com)。转载请注明出处！
