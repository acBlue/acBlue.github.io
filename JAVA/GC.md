#GC常见问题以及解决
##GC分配对象
1. 空闲列表  
通过额外的存储记录空闲的地址，将随机 IO 变为顺序 IO，但带来了额外的空间消耗。
2. 指针碰撞  
通过一个指针作为分界点，需要分配内存时，仅 需把指针往空闲的一端移动与对象大小相等的距离，分配效率较高，但使用场景有限。
##GC垃圾收集算法
- 引用计数法(Reference Counting)  
对每个对象的引用进行计数，每当有一个地方引用它时计数器 +1、引用失效则 -1，引用的计数放到对象头中，大于 0 的对象被认为是存活对象。
虽然循环引用的问题可通过 Recycler 算法解决，但是在多线程环境下，引用计数变更也要进行昂贵的同步操作，性能较 低，早期的编程语言会采用此算法。
- 可达性分析，又称引用链法(Tracing GC)  
从 GC Root 开始进行对象搜索，可以被搜索到的对象即为可达对象，此时还不足以判断对象是否存活 / 死亡， 需要经过多次标记才能更加准确地确定，
整个连通图之外的对象便可以作为垃 圾被回收掉。目前 Java 中主流的虚拟机均采用此算法。
---
###垃圾回收算法
- Mark-Sweep(标记 - 清除)  
回收过程主要分为两个阶段，第一阶段为追踪 (Tracing)阶段，即从 GC Root 开始遍历对象图，并标记(Mark)所遇到的每个对象，第二阶段为清除(Sweep)阶段，
即回收器检查堆中每一个对象， 并将所有未被标记的对象进行回收，整个过程不会发生对象移动。整个算法在不同的实现中会使用三色抽象(Tricolour Abstraction)、
位图标记(BitMap) 等技术来提高算法的效率，存活对象较多时较高效。
- Mark-Compact(标记 - 整理)  
这个算法的主要目的就是解决在非移动式回 收器中都会存在的碎片化问题，也分为两个阶段，第一阶段与 Mark-Sweep 类似，
第二阶段则会对存活对象按照整理顺序(Compaction Order)进行整 理。主要实现有双指针(Two-Finger)回收算法、滑动回收(Lisp2)算法和 
引线整理(Threaded Compaction)算法等。
- Copying(复制)  
将空间分为两个大小相同的From和To两个半区，同一时 间只会使用其中一个，每次进行回收时将一个半区的存活对象通过复制的方式 转移到另一个半区。
有递归(Robert R. Fenichel 和 Jerome C. Yochelson 提出)和迭代(Cheney 提出)算法，以及解决了前两者递归栈、缓存行等问题的近似优先搜索算法。
复制算法可以通过碰撞指针的方式进行快速地分配内 存，但是也存在着空间利用率不高的缺点，另外就是存活对象比较大时复制的 成本比较高。
---
###GC收集器
- ParNew  
一款多线程的收集器，采用复制算法，主要工作在 Young 区，可以通过 -XX:ParallelGCThreads 参数来控制收集的线程数，整个过程都是STW 的，
常与 CMS 组合使用。
- CMS  
以获取最短回收停顿时间为目标，采用“标记 - 清除”算法，分 4 大步进行垃圾收集，其中初始标记和重新标记会 STW ，
多数应用于互联网站 或者 B/S 系统的服务器端上，JDK9 被标记弃用，JDK14 被删除。
- G1  
一种服务器端的垃圾收集器，应用在多处理器和大容量内存环境中，在实现高吞吐量的同时，尽可能地满足垃圾收集暂停时间的要求。
- ZGC  
DK11 中推出的一款低延迟垃圾回收器，适用于大内存低延迟服务的 内存管理和回收，SPECjbb 2015 基准测试，在 128G 的大堆下，最大停顿时间才 1.68 ms，
停顿时间远胜于 G1 和 CMS。
---
###常用GC命令  

>  jps、jinfo、jstat、jstack、jmap、jcmd、vjtools、arthas、greys
---
####GC常见问题
+ 动态扩容引起的空间震荡  
-   现象     
服务刚刚启动时 GC 次数较多，最大空间剩余很多但是依然发生 GC，这种情况我们 可以通过观察 GC 日志或者通过监控工具来观察堆的空间变化情况即可。
GC Cause 一般为 Allocation Failure，且在 GC 日志中会观察到经历一次 GC ，堆内各个空间 的大小会被调整。
-   原因  
在 JVM 的参数中 -Xms 和 -Xmx 设置的不一致，在初始化时只会初始 -Xms 大小的空间存储信息，每当空间不够用时再向操作系统申请，这样的话必然要进行一次 GC。
另外，如果空间剩余很多时也会进行缩容操作，JVM 通过 -XX:MinHeapFreeRa- tio 和 -XX:MaxHeapFreeRatio 来控制扩容和缩容的比例，
调节这两个值也可以 控制伸缩的时机。
-   策略  
观察 CMS GC 触发时间点 Old/MetaSpace 区的 committed 占比是不是一 个固定的值，或者像上文提到的观察总的内存使用率也可以。
尽量将成对出现的空间大小配置参数设置成固定的，如 -Xms 和 -Xmx，-XX:- MaxNewSize 和 -XX:NewSize，-XX:MetaSpaceSize 和 -XX:MaxMetaSpace- Size 等。

  
+ 显式 GC 的去与留  
-   现象     
除了扩容缩容会触发 CMS GC 之外，还有 Old 区达到回收阈值、MetaSpace 空间 不足、Young 区晋升失败、大对象担保失败等几种触发条件，
如果这些情况都没有 发生却触发了 GC ?这种情况有可能是代码中手动调用了 System.gc 方法，此时可 以找到 GC 日志中的 GC Cause 确认下。
那么这种 GC 到底有没有问题，翻看网上 的一些资料，有人说可以添加 -XX:+DisableExplicitGC 参数来避免这种 GC， 也有人说不能加这个参数，
加了就会影响 Native Memory 的回收。
-   原因  
CMS GC 共分为 Background 和 Foreground 两种模 式，前者就是我们常规理解中的并发收集，可以不影响正常的业务线程运行，
但 Foreground Collector 却有很大的差异，他会进行一次压缩式 GC。此压缩式 GC 使用的是跟 Serial Old GC 一样的 Lisp2 算法，
其使用 Mark-Compact 来做 Full GC，一般称之为 MSC(Mark-Sweep-Compact)，它收集的范围是 Java 堆的 Young 区和 Old 区以及 MetaSpace。
由上面的算法章节中我们知道 compact 的代 价是巨大的，那么使用 Foreground Collector 时将会带来非常长的 STW。
如果在 应用程序中 System.gc 被频繁调用，那就非常危险了。  
如果禁用掉的话就会带来另外一个内存泄漏问题，此时就需要说一下 Direct- ByteBuffer，它有着零拷贝等特点，被 Netty 等各种 NIO 框架使用，
会使用到堆外 内存。堆内存由 JVM 自己管理，堆外内存必须要手动释放，DirectByteBuffer 没有 Finalizer，它的 Native Memory 的清理工作是通过
sun.misc.Cleaner 自动完成的，是一种基于 PhantomReference 的清理工具，比普通的 Finalizer 轻量些。
为 DirectByteBuffer 分配空间过程中会显式调用 System.gc ，希望通过 Full GC 来强迫已经无用的 DirectByteBuffer 对象释放掉它们关联的 Native Memory。
HotSpot VM 只会在 Old GC 的时候才会对 Old 中的对象做 Reference Pro- cessing，而在 Young GC 时只会对 Young 里的对象做 Reference Processing。
Young 中的 DirectByteBuffer 对象会在 Young GC 时被处理，也就是说，做 CMS GC 的话会对 Old 做 Reference Processing，
进而能触发 Cleaner 对已死的 DirectByteBuffer 对象做清理工作。但如果很长一段时间里没做过 GC 或者只做了Young GC 的话则不会在 Old 触发 Cleaner 的工作，
那么就可能让本来已经死亡， 但已经晋升到 Old 的 DirectByteBuffer 关联的 Native Memory 得不到及时释放。 
这几个实现特征使得依赖于 System.gc 触发 GC 来保证 DirectByteMemory 的清 理工作能及时完成。如果打开了 -XX:+DisableExplicitGC，
清理工作就可能得不到及时完成，于是就有发生 Direct Memory 的 OOM。
-   策略  
通过上面的分析看到，无论是保留还是去掉都会有一定的风险点，不过目前互联网 中的 RPC 通信会大量使用 NIO，所以笔者在这里建议保留。
此外 JVM 还提供了 -XX:+ExplicitGCInvokesConcurrent 和 -XX:+ExplicitGCInvokesCon- currentAndUnloadsClasses 参数
来将 System.gc 的触发类型从 Foreground 改为 Background，同时 Background 也会做 Reference Processing，这样的话 就能大幅降低了 STW 开销，
同时也不会发生 NIO Direct Memory OOM。  


+ MetaSpace 区 OOM  
-   现象  
JVM 在启动后或者某个时间点开始，MetaSpace 的已使用大小在持续增长，同时 每次 GC 也无法释放，调大 MetaSpace 空间也无法彻底解决。
-   原因  
为了避免弹性伸缩带来的额外 GC 消耗，我们会将 -XX- :MetaSpaceSize 和 -XX:MaxMetaSpaceSize 两个值设置为固定的，但是这样也 会导致在空间不够的时候无法扩容，
然后频繁地触发 GC，最终 OOM。所以关键原 因就是 ClassLoader 不停地在内存中 load 了新的 Class ，一般这种问题都发生在 动态类加载等情况上。
-   策略  
了解大概什么原因后，如何定位和解决就很简单了，可以 dump 快照之后通过 JProfiler 或 MAT 观察 Classes 的 Histogram(直方图)即可，
或者直接通过命令即可 定位，jcmd 打几次 Histogram 的图，看一下具体是哪个包下的 Class 增加较多就可以 定位了。  

+ 过早晋升  
-   现象  
分配速率接近于晋升速率，对象晋升年龄较小。Full GC 比较频繁，且经历过一次 GC 之后 Old 区的变化比例非常大。
-   原因  
Young/Eden 区过小:过小的直接后果就是 Eden 被装满的时间变短，本应 该回收的对象参与了 GC 并晋升，Young GC 采用的是复制算法，
Young GC 耗时本质上就是 copy 的时间(CMS 扫描 Card Table 或 G1 扫描 Remember Set 出问题的 情况另说)，没来及回收的对象增大了回收的代价，
所以 Young GC 时间增 加，同时又无法快速释放空间，Young GC 次数也跟着增加。
分配速率过大,对象年龄增长过快，本该在Young区死亡的对象进入了Old区。
-   策略  
知道问题原因后我们就有解决的方向，如果是 Young/Eden 区过小，我们可以在总 的 Heap 内存不变的情况下适当增大 Young 区，
具体怎么增加?一般情况下 Old 的 大小应当为活跃对象的 2~3 倍左右，考虑到浮动垃圾问题最好在 3 倍左右，剩下的 都可以分给 Young 区。  

+ CMS Old GC 频繁
-   现象  
Old 区频繁的做 CMS GC，但是每次耗时不是特别长，整体最大 STW 也在可接受 范围内，但由于 GC 太频繁导致吞吐下降比较多。
-   原因  
Young GC 完成之后，负责处理 CMS GC 的一个 后台线程 concurrentMarkSweepThread 会不断地轮询，使用 shouldConcur- rentCollect() 方法
做一次检测，判断是否达到了回收条件。如果达到条件，使用 collect_in_background() 启动一次 Background 模式 GC。  
○ CMS 默认采用 JVM 运行时的统计数据判断是否需要触发 CMS GC，如果需要根据 -XX:CMSInitiatingOccupancyFraction 的值进行判断，需
要设置参数 -XX:+UseCMSInitiatingOccupancyOnly。  
○ 如 果 开 启 了 -XX:UseCMSInitiatingOccupancyOnly 参 数， 判 断 当 前 Old 区使用率是否大于阈值，则触发 CMS GC，该阈值可以通过参数
-XX:CMSInitiatingOccupancyFraction 进行设置，如果没有设置， 默认为 92%。  
○ 如果之前的 Young GC 失败过，或者下次 Young 区执行 Young GC 可能 失败，这两种情况下都需要触发 CMS GC。  
○ CMS 默认不会对 MetaSpace 或 Perm 进行垃圾收集，如果希望对这些区 域进行垃圾收集，需要设置参数 -XX:+CMSClassUnloadingEnabled。  
-   策略  
● 内存 Dump:使用 jmap、arthas 等 dump 堆进行快照时记得摘掉流量，同 时分别在 CMS GC 的发生前后分别 dump 一次。  
● 分析 Top Component:要记得按照对象、类、类加载器、包等多个维度观 察 Histogram，同时使用 outgoing 和 incoming 分析关联的对象，
另外就是 Soft Reference 和 Weak Reference、Finalizer 等也要看一下。  
● 分析 Unreachable:重点看一下这个，关注下 Shallow 和 Retained 的大 小。  

+ 单次CMS Old GC耗时长
-   现象  
CMS GC 单次 STW 最大超过 1000ms，不会频繁发生  
-   原因  
CMS 在回收的过程中，STW 的阶段主要是 Init Mark 和 Final Remark 这两个 阶段，也是导致 CMS Old GC 最多的原因，另外有些情况就是在 STW 
前等待 Mutator 的线程到达 SafePoint 也会导致时间过长，但这种情况较少，我们在此处主 要讨论前者。
-   策略  
知道了两个 STW 过程执行流程，我们分析解决就比较简单了，由于大部分问题都出 在 Final Remark 过程，观察详细 GC 日志，找到出问题时 
Final Remark 日志，分析 下 Reference 处理和元数据处理 real 耗时是否正常，详细信息需要通过 -XX:+PrintReferenceGC 参数开启。
基本在日志里面就能定位到大概是哪 个方向出了问题，耗时超过 10% 的就需要关注。 

+ 内存碎片&收集器退化
-   现象  
并发的 CMS GC 算法，退化为 Foreground 单线程串行 GC 模式，STW 时间超 长，有时会长达十几秒。
其中 CMS 收集器退化后单线程串行 GC 算法有两种:  
● 带压缩动作的算法，称为 MSC，上面我们介绍过，使用标记 - 清理 - 压 缩，单线程全暂停的方式，对整个堆进行垃圾收集，也就是真正意义上的 Full GC，
暂停时间要长于普通 CMS。  
● 不带压缩动作的算法，收集 Old 区，和普通的 CMS 算法比较相似，暂停时间 相对 MSC 算法短一些。
-   原因  
CMS 发生收集器退化主要有以下几种情况:  
1.晋升失败(Promotion Failed)  
顾名思义，晋升失败就是指在进行 Young GC 时，Survivor 放不下，对象只能放 入 Old，但此时 Old 也放不下。  
主要有两种情况，一种是前面说的动态年龄判断导致过早晋升，另一种就是内存碎片导致的 Promotion Failed，Young GC 以为 Old 有足够的空间，
结果到分配时，晋级的大 对象找不到连续的空间存放。  
2.增量收集担保失败  
分配内存失败后，会判断统计得到的 Young GC 晋升到 Old 的平均大小，以及当前 Young 区已使用的大小也就是最大可能晋升的对象大小，是否大于 Old 区
的剩余空 间。只要 CMS 的剩余空间比前两者的任意一者大，CMS 就认为晋升还是安全的， 反之，则代表不安全，不进行 Young GC，直接触发 Full GC。
3.显示GC  
参照前面显示G。  
4.并发模式失败(Concurrent Mode Failure)  
最后一种情况，也是发生概率较高的一种，在 GC 日志中经常能看到 Concurrent Mode Failure 关键字。这种是由于并发 Background CMS GC 正在执行，
同时又 有 Young GC 晋升的对象要放入到了 Old 区中，而此时 Old 区空间不足造成的。  
为什么 CMS GC 正在执行还会导致收集器退化呢?主要是由于 CMS 无法处理浮动 垃圾(Floating Garbage)引起的。CMS 的并发清理阶段，Mutator 还在运行，
因此不断有新的垃圾产生，而这些垃圾不在这次清理标记的范畴里，无法在本次 GC 被 清除掉，这些就是浮动垃圾，除此之外在 Remark 之前那些断开引用脱离了
读写屏障控制的对象也算浮动垃圾。所以 Old 区回收的阈值不能太高，否则预留的内存空间 很可能不够，从而导致 Concurrent Mode Failure 发生。
-   策略  
分析到具体原因后，我们就可以针对性解决了，具体思路还是从根因出发，具体解决 策略:  
1.内存碎片:  
通过配置 -XX:UseCMSCompactAtFullCollection=true 来控制 Full GC 的过程中是否进行空间的整理(默认开启，注意是 Full GC，不是普通CMSGC)，
以及-XX: CMSFullGCsBeforeCompaction=n来控制多少次 Full GC 后进行一次压缩。  
2.增量收集:  
降低触发 CMS GC 的阈值，即参数 -XX:CMSInitiatingOccu- pancyFraction 的值，让 CMS GC 尽早执行，以保证有足够的连续空间， 也减少 Old 区
空间的使用大小，另外需要使用 -XX:+UseCMSInitiatingO- ccupancyOnly 来配合使用，不然 JVM 仅在第一次使用设定值，后续则自动调整。  
3.浮动垃圾:  
视情况控制每次晋升对象的大小，或者缩短每次 CMS GC 的时间， 必要时可调节 NewRatio 的值。另外就是使用 -XX:+CMSScavengeBefore- Remark 
在过程中提前触发一次 Young GC，防止后续晋升过多对象。  

+ 堆外内存 OOM
-   现象  
内存使用率不断上升，甚至开始使用 SWAP 内存，同时可能出现 GC 时间飙升，线 程被 Block 等现象，通过 top 命令发现 Java 进程的 RES 甚至超过了
 -Xmx 的大 小。出现这些现象时，基本可以确定是出现了堆外内存泄漏。  
-   原因  
JVM 的堆外内存泄漏，主要有两种的原因:  
1.通过 UnSafe#allocateMemory，ByteBuffer#allocateDirect 主动申 请了堆外内存而没有释放，常见于 NIO、Netty 等相关组件。  
2.代码中有通过 JNI 调用 Native Code 申请的内存没有释放。  
-   策略  
首先，我们需要确定是哪种原因导致的堆外内存泄漏。这里可以使用 NMT(Native- MemoryTracking)进行分析。
在项目中添加 -XX:NativeMemoryTracking=de- tail JVM 参数后重启项目,(需要注意的是，打开 NMT 会带来 5%~10% 的性能损耗)。
使用命令jcmd pid VM.native_memory detail查看内存分布。重点观察 total 中的 committed，因为 jcmd 命令显示的内存包含堆内内存、Code 区域、
通过 Unsafe.allocateMemory 和 DirectByteBuffer 申请的内存，但是不包含其他 Native Code(C 代码)申请的堆外内存。  
如果 total 中的 committed 和 top 中的 RES 相差不大，则应为主动申请的堆外内存 未释放造成的，如果相差较大，则基本可以确定是 JNI 调用造成的。    
首先可以使用 NMT + jcmd 分析泄漏的堆外内存是哪里申请，确定原因后，使用不 同的手段，进行原因定位。

+ JNI 引发的 GC 问题
-   现象  
在 GC 日志中，出现 GC Cause 为 GCLocker Initiated GC。
-   原因  
JNI(Java Native Interface)意为 Java 本地调用，它允许 Java 代码和其他语言写 的 Native 代码进行交互。  
JNI 如果需要获取 JVM 中的 String 或者数组，有两种方式:  
● 拷贝传递。  
● 共享引用(指针)，性能更高。  
由于 Native 代码直接使用了 JVM 堆区的指针，如果这时发生 GC，就会导致数据 错误。因此，在发生此类 JNI 调用时，禁止 GC 的发生，
同时阻止其他线程进入 JNI 临界区，直到最后一个线程退出临界区时触发一次 GC。
-   策略  
● 添加 -XX+PrintJNIGCStalls 参数，可以打印出发生 JNI 调用时的线程， 进一步分析，找到引发问题的 JNI 调用。  
● JNI 调用需要谨慎，不一定可以提升性能，反而可能造成 GC 问题。  
● 升级 JDK 版本到 14，避免 JDK-8048556 导致的重复 GC。  



