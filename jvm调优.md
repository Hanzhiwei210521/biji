# jvm调优 #

### 首先我们先恶补一下jvm的各种知识： ###

JVM区域总体分为两类，heap（堆）区和非heap区。heap区分为Young(年轻代)和old（老年代）；Yong分为EdenSpace（伊甸园）和survivor（幸存区）。非heap区分为：Code Cache（代码缓存区）、Perm Gen（永久代）、jvm stack（Java虚拟机栈）、Local Method Statck（本地方法栈）。当然这是JDK 1.8之前的分发，从1.8开始，Perm Gen被MetaspaceSize替代。

![](https://github.com/Hanzhiwei210521/loading/blob/master/image/f157091262e858690f68ad993b01e44c.jpg)

* Java内存≈Heap(堆内存)+PermGen(方法区)+Thrend(栈)
* Heap（堆内存）=Young（年轻代）+Old（老年代）
* Young（年轻代）=EdenSpace+From Survivor+To Survivor

设置jvm内存关键字
<table>
    <tr>
        <td>常用配置参数</td>
        <td>功能</td>
    </tr>
    <tr>
        <td>-Xms</td>
        <td>初始堆大小，如：-Xms256m (Core)，如果不指定默认是物理内存的1/64；默认情况下堆内存可用空间小于40%时候，JVM会增加堆内存到最大内存或物理内存的1/4；</td>
    </tr>
    <tr>
        <td>-Xmx</td>
        <td>最大堆大小，如：-Xmx512m，如果不指定默认是无内存的1/4；默认情况下堆内存可用空间大于70%时候，JVM会减小堆内存到最小内存或物理内存的1/64;为了防止垃圾收集器在最小、最大之间收缩堆而产生额外的时间，我们通常把最大、最小设置为相同的值</td>
    </tr>
    <tr>
        <td>-Xmn</td>
        <td>年轻代大小。通常为 Xmx 的 1/3 或 1/4。新生代 = Eden + 2 个 Survivor 空间。实际可用空间为 = Eden + 1 个 Survivor，即 90%</td>
    </tr> 
     <tr>
        <td>-Xss</td>
        <td>JDK1.5+ 每个线程堆栈大小为 1M，一般来说如果栈不是很深的话， 1M 是绝对够用了的。</td>
    </tr>    
    <tr>
        <td>-XX:NewRatio</td>
        <td>新生代与老年代的比例，如 –XX:NewRatio=2，则新生代占整个堆空间的1/3，老年代占2/3</td>
    </tr>    
    <tr>
        <td>-XX:SurvivorRatio</td>
        <td>新生代中 Eden 与 Survivor 的比值。默认值为 8。即 Eden 占新生代空间的 8/10，另外两个 Survivor 各占 1/10</td>
    </tr> 
    <tr>
        <td>-XX:PermSize</td>
        <td>永久代(方法区)的初始大小</td>
    </tr> 
    <tr>
        <td>-XX:MaxPermSize</td>
        <td>永久代(方法区)的最大值</td>
    </tr> 
</table>
注意：PermSize永久代的概念在jdk1.8中已经不存在了，取而代之的是metaspace元空间，当人为执行永久代的初始大小以及最大值是jvm会给出如此下提示： 
Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=30m; support was removed in 8.0 
Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=30m; support was removed in 8.0

### 为什么要分代 ###

分代的垃圾回收策略，是基于这样一个事实：不同的对象的生命周期是不一样的。因此，不同生命周期的对象可以采取不同的收集方式，以便提高回收效率。
 
在Java程序运行的过程中，会产生大量的对象，其中有些对象是与业务信息相关，比如Http请求中的Session对象、线程、Socket连接，这 类对象跟业务直接挂钩，因此生命周期比较长。但是还有一些对象，主要是程序运行过程中生成的临时变量，这些对象生命周期会比较短，比如：String对 象，由于其不变类的特性，系统会产生大量的这些对象，有些对象甚至只用一次即可回收。
 
试想，在不进行对象存活时间区分的情况下，每次垃圾回收都是对整个堆空间进行回收，花费时间相对会长，同时，因为每次回收都需要遍历所有存活对象，但实际上，对于生命周期长的对象而言，这种遍历是没有效果的，因为可能进行了很多次遍历，但是他们依旧存在。因此，分代垃圾回收采用分治的思想，进行代的划 分，把不同生命周期的对象放在不同代上，不同代上采用最适合它的垃圾回收方式进行回收。

#### 年轻代 ####

所有新生成的对象首先都是放在年轻代的。年轻代的目标就是尽可能快速的收集掉那些生命周期短的对象。年轻代分三个区。一个Eden区，两个 Survivor区【一般而言，(From区和to区，To区永远是空的)】。大部分对象在Eden区中生成。当Eden区满时，还存活的对象将被复制到Survivor区（From区），当这个 Survivor区满时，会进行GC，将Eden的存活对象复制到to区，from区的对象会根据年龄值（GC次数）来决定去向，到达阈值的对象会被复制到老年代中，没有达到阈值的对象会被复制到To区域。经过这次GC后，Eden区和From区已经被清空，这个时候，From和To会交换他们的角色，也就是现在的To区变成From区，From区变成To区，不管怎样，都会保证To区是空的。Minor GC会一直重复这样的过程。

### 老年代 ###

老年代存放的都是一些生命周期较长的对象，就像上面所叙述的那样，在新生代中经历了N次垃圾回收后让然存活的对象就会被放到老年代中。此外，老年代的内存也比新生代的大很多（大概比例是1：2），当老年代满时会触发Major GC(Full GC)，老年代对象存活时间比较长，因此Full GC发生的频率比较低。

### 永久代 ###

永久代主要用于存放静态文件，如Java类、方法等。垃圾回收对永久代没有显著影响。有些应用可能动态生成或调用一些class，例如使用反射、动态代理、CGLib等bytecode框架时，在这种时候需要设置一个比较大的永久代来存放这些运行过程中新增的类。

### GC ###

上面频繁提到GC，那么GC又是什么，GC，也就是垃圾回收，分为两种类型,Minor GC 和 Full GC，由于对象进行了分代处理，因此垃圾回收区域、时间也不一样。

* Minor GC：对年轻代进行回收，不会影响到老年代。因为年轻代的Java对象大多死亡频繁，所以Minor GC非常频繁。
* Full GC：也叫Major GC，对整个堆进行回收，包括年轻代和老年代。由于Full GC需要对整个堆进行回收，所以比Minor GC 要慢，因此可以尽可能减少Full GC的次数，导致Full GC的原因包括：老年代被写满、永久代被写满等。


上面恶补了一些jvm知识，下面我们要分析一下Java线上系统问题，那么我们就要查看JVM启动时的一些参数设置，这些参数坑能在启动脚本中明确指明，也可能采用默认值，在系统运行过程中其他人也许动态调整了系统参数。下面我们就实时查看一下正在运行的JVM参数：

#### jinfo ####

jinfo(JVM Configuration info)这个命令作用是实时查看和调整虚拟机运行参数。
之前的jps -v口令只能查看到显示指定的参数，如果想要查看未被显示指定的参数的值就要使用jinfo口令

这是一个在测试机上跑的生产的程序

    [root@bogon ~]# jinfo -flags 69801
    Attaching to process ID 69801, please wait...
    Debugger attached successfully.
    Server compiler detected.
    JVM version is 25.121-b13
    Non-default VM flags: -XX:CICompilerCount=12 -XX:InitialHeapSize=2109734912 -XX:MaxHeapSize=32210157568 -XX:MaxNewSize=10736369664 -XX:MetaspaceSize=104857600 -XX:MinHeapDeltaBytes=524288 -XX:NewSize=703070208 -XX:OldSize=1406664704 -XX:+PrintGC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseFastUnorderedTimeStamps -XX:+UseParallelGC 
    Command line:  -XX:MetaspaceSize=100M -XX:+PrintGC -XX:+PrintGCTimeStamps -XX:+PrintGCDetails
根据上面输出，我么可以分析出：

* -XX:CICompilerCount=12 编译时的编译器线程数
* -XX:InitialHeapSize=2109734912 初始堆内存2012M
* -XX:MaxHeapSize=32210157568 最大堆内存30718M
* -XX:MaxNewSize=10736369664 年轻代最大内存 10239M
* -XX:MetaspaceSize=104857600 元数据内存 100M
* -XX:MinHeapDeltaBytes=524288 未知
* -XX:NewSize=703070208 年轻代 670.5M
* -XX:OldSize=1406664704 老年代 1341.5M
* -XX:+PrintGC 在日志中打印GC信息
* -XX:+PrintGCDetails 打印GC详细信息
* -XX:+PrintGCTimeStamps 打印GC时间戳
* -XX:+UseCompressedClassPointers 压缩类指针（具体作用不明）
* -XX:+UseCompressedOops 压缩对象指针（具体作用不明）
* -XX:+UseFastUnorderedTimeStamps 未知
* -XX:+UseParallelGC 设置并行收集器
* Command line:  -XX:MetaspaceSize=100M -XX:+PrintGC -XX:+PrintGCTimeStamps -XX:+PrintGCDetails 为启动时我们自己设置的参数

使用jmp -head打印heap的概要信息，GC使用的算法，heap的配置及wise heap的使用情况,可以用此来判断内存目前的使用情况以及垃圾回收情况

    [root@bogon ~]# jmap -heap 69801
    Attaching to process ID 69801, please wait...
    Debugger attached successfully.
    Server compiler detected.
    JVM version is 25.121-b13

    using thread-local object allocation.
    Parallel GC with 18 thread(s)

    Heap Configuration:
       MinHeapFreeRatio         = 0
       MaxHeapFreeRatio         = 100
       MaxHeapSize              = 32210157568 (30718.0MB)
       NewSize                  = 703070208 (670.5MB)
       MaxNewSize               = 10736369664 (10239.0MB)
       OldSize                  = 1406664704 (1341.5MB)
       NewRatio                 = 2
       SurvivorRatio            = 8
       MetaspaceSize            = 104857600 (100.0MB)
       CompressedClassSpaceSize = 1073741824 (1024.0MB)
       MaxMetaspaceSize         = 17592186044415 MB
      G1HeapRegionSize         = 0 (0.0MB)

    Heap Usage:
    PS Young Generation
    Eden Space:
       capacity = 2111832064 (2014.0MB)
       used     = 1538455744 (1467.1857299804688MB)
       free     = 573376320 (546.8142700195312MB)
       72.84934111124473% used
    From Space:
       capacity = 87556096 (83.5MB)
       used     = 52909560 (50.45848846435547MB)
       free     = 34646536 (33.04151153564453MB)
       60.429327502222115% used
    To Space:
       capacity = 87556096 (83.5MB)
       used     = 0 (0.0MB)
       free     = 87556096 (83.5MB)
       0.0% used
    PS Old Generation
       capacity = 1406664704 (1341.5MB)
       used     = 147472 (0.1406402587890625MB)
       free     = 1406517232 (1341.359359741211MB)
       0.010483806096836563% used

    24733 interned Strings occupying 3078696 bytes.
    [root@bogon ~]# free -m
                  total        used        free      shared  buff/cache   available
    Mem:         128656        4025      104331        4185       20298      119617
 
对比以上三个命令输出，我们会发现以上数据跟默认设置完全吻合，，初始堆内存为物理内存的1/64,年轻代：老年代=1：2，也就是说JVM没有任何优化，有没有很可怕。。。。

Java程序运行一段时间后，我们会发现Eden和Survivor的空间会小于初始值，这就很让人费解了，我看看个例子如下（还是刚才的测试机）：

    From Space:
       capacity = 29884416 (28.5MB)
       used     = 29434608 (28.071029663085938MB)
       free     = 449808 (0.4289703369140625MB)
       98.49484092310855% used
    To Space:
       capacity = 66584576 (63.5MB)
       used     = 0 (0.0MB)
       free     = 66584576 (63.5MB)
       0.0% used
下面看一个生产上活生生的例子：

    [root@SE1074-V0231 api-gateway]# jinfo -flags 16914
    Attaching to process ID 16914, please wait...
    Debugger attached successfully.
    Server compiler detected.
    JVM version is 25.121-b13
    Non-default VM flags: -XX:CICompilerCount=3 -XX:InitialHeapSize=257949696 -XX:MaxHeapSize=4097835008 -XX:MaxNewSize=1365770240 -XX:MetaspaceSize=104857600 -XX:MinHeapDeltaBytes=524288 -XX:NewSize=85983232 -XX:OldSize=171966464 -XX:+PrintGC -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseParallelGC 
    Command line:  -XX:MetaspaceSize=100M -XX:+PrintGC -XX:+PrintGCTimeStamps -XX:+PrintGCDetails
    ###################################################
    [root@SE1074-V0231 api-gateway]# jmap -heap 16914
    Attaching to process ID 16914, please wait...
    Debugger attached successfully.
    Server compiler detected.
    JVM version is 25.121-b13

    using thread-local object allocation.
    Parallel GC with 4 thread(s)

    Heap Configuration:
       MinHeapFreeRatio         = 0
       MaxHeapFreeRatio         = 100
       MaxHeapSize              = 4097835008 (3908.0MB)
       NewSize                  = 85983232 (82.0MB)
       MaxNewSize               = 1365770240 (1302.5MB)
       OldSize                  = 171966464 (164.0MB)
       NewRatio                 = 2
       SurvivorRatio            = 8
       MetaspaceSize            = 104857600 (100.0MB)
       CompressedClassSpaceSize = 1073741824 (1024.0MB)
       MaxMetaspaceSize         = 17592186044415 MB
       G1HeapRegionSize         = 0 (0.0MB)

    Heap Usage:
    PS Young Generation
    Eden Space:
       capacity = 19398656 (18.5MB)
       used     = 1684128 (1.606109619140625MB)
       free     = 17714528 (16.893890380859375MB)
       8.68167361697635% used
    From Space:
       capacity = 2621440 (2.5MB)
       used     = 2065216 (1.96954345703125MB)
       free     = 556224 (0.53045654296875MB)
       78.78173828125% used
    To Space:
       capacity = 2621440 (2.5MB)
       used     = 0 (0.0MB)
       free     = 2621440 (2.5MB)
       0.0% used
    PS Old Generation
       capacity = 79691776 (76.0MB)
       used     = 76240576 (72.70867919921875MB)
       free     = 3451200 (3.29132080078125MB)
       95.66931473581414% used

    25659 interned Strings occupying 3253856 bytes.

根据以上信息，发现heap初始设置为如下：

* 初始堆内存246M（物理内存16G）
* 最大堆内存 3908M
* 年轻代最大 1302.5M
* 元空间 100M
* 年轻代 82M
* 老年代 164M

然而实际分配的空间却小的可怜：

* Eden为18.5M
* Survivor为2.5M
* old 为76M

经过一番Google，发现原来是JVM 默认参数-XX:+UseAdaptiveSizePolicy在作妖，此选项默认为打开状态，设置此选项后，并行收集器会自动选择Eden大小和相应的Survivor区比例，以达到目标系统规定的最低相应时间或者收集频率等。

Eden和Survivor被设置的如此之小，我们看一下会有什么后果：

    [root@SE1074-V0231 api-gateway]# jstat -gccause 16914
      S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT    LGCC                 GCC                 
      0.00  95.34  55.83  88.83  97.41  95.49 686763 2883.992 22437 1888.471 4772.463 Allocation Failure   No GC  
Minor GC达到了686763次，耗时2883.992s;Full GC也达到了22437次，耗时1888.471s，下面我们看一下日志：

###### Java程序日志输出： ######

    [GC (Allocation Failure) [PSYoungGen: 35600K->560K(34816K)] 104406K->70014K(113664K), 0.0041383 secs] [Times: user=0.01 sys=0.00, real=0.01 secs]
    [Full GC (Ergonomics) [PSYoungGen: 1104K->0K(57856K)] [ParOldGen: 77189K->57047K(80384K)] 78294K->57047K(138240K), [Metaspace: 61467K->61467K(1105920K)], 0.0921226 secs] [Times: user=0.23 sys=0.00, real=0.09 secs]

* [GC (Allocation Failure):说明GC类型，括号中说明发生GC的原因。原因是内存分配失败
* [PSYoungGen: 35600K->560K(34816K)]：新生代：GC前已使用容量-->GC后已使用容量（新生代总容量）
* 104406K->70014K(113664K)：整个Java堆：GC前已使用容量-->GC后已使用容量（Java堆总容量）
* 0.0041383 secs GC耗时，单位s

## JVM调优 ##

* JAVA_OPTS="$JAVA_OPTS -server -XX:+DisableExplicitGC -XX:-UseAdaptiveSizePolicy -XX:-OmitStackTraceInFastThrow -XX:+UseGCLogFileRotation"
* JAVA_OPTS="$JAVA_OPTS -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCApplicationStoppedTime -XX:GCLogFileSize=10M -XX:NumberOfGCLogFiles=20 -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=$CATALINA_BASE/logs/debug/dump -Xloggc:$CATALINA_BASE/logs/debug/heap_trace.txt"
* JAVA_OPTS="$JAVA_OPTS -Xms4096M -Xmx4096M -Xss512k -XX:NewRatio=1 -XX:SurvivorRatio=8 -XX:MaxTenuringThreshold=6"
* JAVA_OPTS="$JAVA_OPTS -XX:+UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=75 -XX:+UseCMSInitiatingOccupancyOnly"
   
参考： 

<https://blog.csdn.net/GYQJN/article/details/49848473>

<http://blog.51cto.com/xinsir/2068032>

<https://blog.csdn.net/justloveyou_/article/details/71216049>

<https://blog.csdn.net/wang379275614/article/details/78471604>

<https://github.com/bingoogolapple/bingoogolapple.github.io/issues/177>

<https://www.cnblogs.com/ityouknow/p/5714703.html> jvm各种命令