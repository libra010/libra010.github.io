JVM调优



**JVM调优**，就是通过调整JVM的运行参数，优化Java应用程序的性能、稳定性和资源利用率。让JVM虚拟机在特定的应用场景下运行得更加稳健。

JVM的默认参数是通用平衡的，调优是为了特化以应对特定的需求场景。

JVM（例如 HotSpot）的默认参数设计目标是：在大多数常见场景下够用，平衡启动速度、内存占用、吞吐量、延迟，适合中小应用、开发测试环境。但这些默认值**不是为高性能、高并发、低延迟或资源受限场景设计的**。

所以，**调优的本质就是：根据业务需求，牺牲某些方面的指标，换取另一些方面的极致表现**。





## 不可能三角

**吞吐量（Throughput）、延迟（Latency）和内存占用（Memory Footprint）** 在 JVM 调优中通常构成一个 **不可能三角**。很难同时在三者上都达到最优，优化一个或两个往往会牺牲另一个。

- 吞吐量：单位时间内完成的有效工作量。典型指标是 GC 时间占比低。

- 延迟：单次操作响应的快慢。典型指标是最大暂停时间小于10ms。

- 内存占用：JVM 运行时占用的内存。典型指标是 RSS（Resident Set Size）小，适合容器部署。



在金融交易和实时系统中，优先级是 延迟 >> 吞吐 > 内存 。调优优先选择现代GC以实现极低的GC延迟，比如ZGC和G1，但这些需要更高的内存占用以支持低延迟，强行限制内存会导致GC频率变高，延迟反而更高。

在批处理或后台计算系统中，优先级是 吞吐 > 内存 > 延迟 。调优优先选择Parallel GC和更大的堆内存容量。但是缺点是单次GC的时间会比较长，在这种系统中延迟不太重要。

在微服务容器化环境中，优先级是 内存 ≈ 延迟 > 吞吐 。调优优先选择G1和限制堆内存容量，但是GC可能会比较频繁，吞吐量低。



调优的本质是在吞吐、延迟、内存之间做权衡取舍，没有最优配置，只有最适合当前业务场景的配置。





## 一些调优参数

参数列表官方文档：https://docs.oracle.com/en/java/javase/21/docs/specs/man/java.html

以 -X 开头的参数，区别于标准选项，专用于 HotSpot 虚拟机（Extra Options for Java）。

以 -XX 开头的参数是所谓的高级选项。细分为**运行时行为**（Advanced Runtime Options），**JIT编译行为**（Advanced JIT Compiler Options），**收集系统信息和执行深度调试**（Advanced Serviceability Options），**垃圾回收**（Advanced Garbage Collection）。共四大类。



**Extra Options for Java**


`-Xms` 初始堆大小

`-Xmx` 最大堆大小

> JVM启动时会首先向操作系统申请内存初始化堆内存池，Java程序需要内存时直接从这个内存池中分配。超过内存池容量时，会再次向操作系统申请内存直到达到最大堆大小。这种动态申请内存的操作系统调用会影响程序性能，所以通常建议这两个参数设置为相同值以避免性能波动。

`-Xss` 栈大小



**Advanced Runtime Options for Java** 高级运行时选项

`-XX:+UseLargePages` 启用大页内存

`-XX:+AlwaysPreTouch` JVM 启动时预分配并触碰所有堆页，避免运行时缺页中断



**Advanced JIT Compiler Options** 高级 JIT 编译器选项

`-XX:+TieredCompilation` 启用分层编译（JDK 8+ 默认开启）



**Advanced Serviceability Options** 高级可服务性选项

`-XX:+PrintGCDetails` 打印详细的 GC 日志（重要选项！）

`-XX:+HeapDumpOnOutOfMemoryError` OOM 时自动生成堆转储文件

`-XX:HeapDumpPath=/logs` 指定堆转储文件路径



**Advanced Garbage Collection Options** 高级垃圾回收选项

`-XX:+UseG1GC` 启用 G1 垃圾回收器

`-XX:+UseZGC` 启用 ZGC（JDK 11+，超低延迟）

`-XX:MaxGCPauseMillis=50` 设置 GC 最大暂停时间目标（G1/ZGC 有效）

`-XX:SurvivorRatio=8` Eden区域:S0:S1 = 8:1:1（默认值）

`-XX:NewRatio=2` 老年代:新生代 = 2:1（即新生代占堆 1/3）





END
