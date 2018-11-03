之前写程序主要是以C/C++，Python为主，而最近Java用的比较多。因为做的是存储服务，程序占用的内存也相对大一些，自然就碰到了Java GC相关的问题。这里把我最近遇到的一些问题和思考记录下来，供自己以后参考。

Java GC调优的首要任务是管理好内存，避免频繁GC，避免Full GC等等；然后才是避免GC STW（Stop The World）造成的影响。关于内存管理，以后有机会再分享。这里我们关注于后者，而且主要是关注内核参数对GC卡顿的影响。

# 理解Java GC
任何一个写Java程序的人都要需要理解Java GC的基本原理，参数设置以及问题排查方法。这里列出几个不错的资源，供大家参考：
* Java GC的科普读物：[Java Garbage Collection Handbook](https://plumbr.io/java-garbage-collection-handbook)
* 分析GC日志的可视化工具：[gceasy.io](http://gceasy.io/)
* G1的科普介绍：[G1 Garbage Collector Details and Tuning](http://presentations2015.s3.amazonaws.com/40_presentation.pdf)
* [G1论文（pdf）](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.63.6386&rep=rep1&type=pdf)

# 线上Java GC参数设置
为了更好的调查GC相关的问题，需要设置一些JVM参数，比如下面的一组参数。其中，日志保存在`/run/log/gc.log`开头的一组文件里。注意到`/run/log`必须是一个内存盘或SSD盘，否则可能因为GC日志导致长时间GC卡顿。具体参考[Eliminating Large JVM GC Pauses Caused by Background IO Traffic](https://engineering.linkedin.com/blog/2016/02/eliminating-large-jvm-gc-pauses-caused-by-background-io-traffic)这篇文章里的分析。
```
-Xloggc:/run/log/gc.log -XX:NumberOfGCLogFiles=5 -XX:+UseGCLogFileRotation -XX:GCLogFileSize=20m -XX:+PrintGC -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+PrintHeapAtGC -XX:+PrintGCCause -XX:+PrintTenuringDistribution -XX:+PrintReferenceGC -XX:+PrintAdaptiveSizePolicy
```

# Young GC卡顿：Sys大于User
基本症状：
1. Young GC有时会导致超过1秒的卡顿
1. CPU Sys使用率有所增高
1. GC日志显示Sys远远大于User

Sys大于User一般是因为在GC时，内核花了大量的时间处理缺页中断（Page Fault），或是不对齐的内存引用等。这方面的科普请参考[GCEasy的文章](https://blog.gceasy.io/2016/12/11/sys-time-greater-than-user-time/)。

我们先仔细看下相关的GC日志：
```
2018-09-06T13:35:40.097+0800: 12198.430: [GC pause (G1 Evacuation Pause) (young), 6.5323256 secs]
   [Parallel Time: 6514.9 ms, GC Workers: 38]
      [GC Worker Start (ms): Min: 12198431.2, Avg: 12198431.6, Max: 12198432.3, Diff: 1.1]
      [Ext Root Scanning (ms): Min: 0.0, Avg: 1.0, Max: 13.2, Diff: 13.2, Sum: 37.8]
      [Update RS (ms): Min: 11.7, Avg: 23.5, Max: 24.8, Diff: 13.1, Sum: 893.4]
         [Processed Buffers: Min: 10, Avg: 19.6, Max: 28, Diff: 18, Sum: 744]
      [Scan RS (ms): Min: 13.1, Avg: 13.9, Max: 14.2, Diff: 1.1, Sum: 527.9]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
      [Object Copy (ms): Min: 6475.3, Avg: 6475.5, Max: 6477.1, Diff: 1.8, Sum: 246069.6]
      [Termination (ms): Min: 0.0, Avg: 0.0, Max: 0.1, Diff: 0.1, Sum: 1.7]
         [Termination Attempts: Min: 1, Avg: 11.3, Max: 17, Diff: 16, Sum: 430]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.3, Max: 0.5, Diff: 0.5, Sum: 10.4]
      [GC Worker Total (ms): Min: 6513.7, Avg: 6514.2, Max: 6514.8, Diff: 1.2, Sum: 247541.0]
      [GC Worker End (ms): Min: 12204945.6, Avg: 12204945.8, Max: 12204946.0, Diff: 0.4]
   [Code Root Fixup: 0.1 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 6.6 ms]
   [Other: 10.7 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 0.9 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.9 ms]
      [Humongous Register: 1.1 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 7.0 ms]
   [Eden: 35.2G(35.2G)->0.0B(2928.0M) Survivors: 112.0M->80.0M Heap: 41.2G(58.9G)->6149.3M(58.9G)]
 [Times: user=1.74 sys=246.18, real=6.53 secs]
```
这里需要注意的是：**Eden区从35.2GB减小到了2.9GB左右**。我们发现只有Eden区大幅度减小的GC才可能出现长时间卡顿。Eden区大幅度减小意味着大量的内存释放。内存大量释放为什么会伴随着Sys CPU使用率高呢？凭借过往的经验，我们猜测可能是透明大页（Transparent HugePage）捣的鬼。

为了验证这个猜测，我们把部分机器的透明大页defrag关闭：
```
# echo never > /sys/kernel/mm/transparent_hugepage/defrag
```
这样一来，由于Young GC导致的卡顿消失了。我们没有再去详细验证（应该做一下！），事实上可以持续对系统进行top，按道理能够发现卡顿时`khugepaged`的Sys CPU使用率高。

# Mixed GC卡顿：Real远大于User和Sys

## 症状和分析
第一个问题解决后，我们持续观察系统，发现几天后出现了更为严重的卡顿。这次所不同是，发生卡顿的是Mixed GC，而且Real远大于User与Sys之和。

通过[GCEasy的这篇文章](https://blog.gceasy.io/2016/12/08/real-time-greater-than-user-and-sys-time/)我们了解到，Real远大于User与Sys之和，可能是因为系统正在做I/O，或是Java GC线程得不到CPU。我们做了如下排查：
* 系统的CPU使用率正常
* Java GC日志的确是在内存盘上

当时系统磁盘的使用率是否有异常呢？果然，系统盘I/O使用率达到了100%（`sar -dp`的输出）：

![iostat output](/assets/images/2018-11-02-iostat.png){:height="290px" width="920px"}

为什么系统盘I/O高？为什么系统盘I/O高会对Java GC有影响？要知道我们的程序只有应用日志打在系统盘上，Java GC日志以及I/O路径并不会触碰系统盘。带着这样的疑问，我们又查看了Page相关的一些指标，发现问题时间段缺页中断显著增加（`sar -B`的输出）：

![iostat output](/assets/images/2018-11-02-pagefault.png){:height="290px" width="920px"}

系统盘I/O使用率高，同时有大量的缺页中断——莫非内核在从swap读取数据？查看一下`sar -s`的输出：

![iostat output](/assets/images/2018-11-02-swap.png){:height="290px" width="920px"}

这里`kbswpcad`在问题时间点明显增高。`sar`的man page里是这样描述kbswpcad的：

```
Amount of cached swap memory in kilobytes.  This is memory that once was swapped out, is swapped back in but still also is in the swap area (if memory is needed it doesn't need to be swapped out again  because it is already in the swap area. This saves I/O).
```

也就是说当时从swap读入到内存的页变多了。看看我们系统的swap相关的配置：

```bash
$ cat /proc/sys/vm/swappiness
0

$ cat /proc/swap
Filename           Type.        Size	Used	Priority
/dev/sda3          partition	6291448	65680	0
```

可以看到swap是开启的，只是把`swappiness`设为0了。原本以为swappiness设为0就能避免swap，事实证明不是这样的。看下内核[vm.txt](https://www.kernel.org/doc/Documentation/sysctl/vm.txt)对swappiness的描述：

```
This control is used to define how aggressive the kernel will swap
memory pages.  Higher values will increase aggressiveness, lower values
decrease the amount of swap.  A value of 0 instructs the kernel not to
initiate swap until the amount of free and file-backed pages is less
than the high water mark in a zone.
```

也就是说swappiness为0并不等于关闭swap，swap事实上还是在起作用，只是回收匿名页的时机更为苛刻了（当一个Zone的内存使用超过high watermark时回收）。

为了证明这个推断，我们选择几台机器并关闭swap（如果要永久关闭，还需要删除`/etc/fstab`中的swap分区)：
```
$ sudo swapoff -a
```
这个改动之后，我们发现虽然还是会有Mixed GC发生，但是卡顿显著降低到了200毫秒左右，符合预期。

## 分析
Java的OldGen内存长时间不被访问，自然是内核swap out的理想对象；当Mixed GC发生时，这些内存大量的被访问，从而导致可观的swap in，也就是从swap分区（sda）大量读取数据，这个过程时非常慢的。

# 小结
本文分享了两例Java GC STW Pause高的案例，卡顿严重都由Java使用内存的方式和内核内存管理共同作用导致的。结论是：
* 需要关闭透明大页的defrag，避免释放大量内存时导致的Sys CPU使用率高。
* 需要完全关闭swap，避免Mixed GC从交换分区swap in原先swap out的OldGen内存。

这里还遗留了两个问题，暂时没有时间调研和分析：
1. Linux swap工作的机制（swappiness, zone watermark）。
2. 为什么频繁发生Mixed GC？
