JDK开发工具 bin目录



OpenJDK 的 bin 目录包含了大量用于开发、调试、监控和管理 Java 应用程序的实用工具。



## 编译、运行与打包

javac：编译器

java：启动程序

jar：打包工具

javap：反编译字节码

javadoc：生成HTML格式的API文档



## 监控、运维与管理

jps：列出当前系统中所有正在运行的 Java 进程。（类似于Linux中的ps）

jstat：监控 JVM 的各种运行时状态信息，提供有关垃圾回收、类加载、JIT 编译等运行数据。

jinfo：用于在不重启应用的情况下，调整虚拟机的各项参数，或者输出 Java 进程的详细信息。

jstack：用于打印出 JVM 中某个进程或远程调试服务的线程堆栈信息（一般称为 threaddump 或者 javacore 文件）。它常用于诊断应用程序中的线程问题，比如线程死锁、死循环或长时间等待。

jmap：生成堆内存的快照（heap dump），用于分析内存使用情况。

jcmd：向正在运行的 JVM 发送诊断命令。功能非常强大且集成了多种其他工具的功能（如 jps, jinfo, jstack, jmap -heap 等）。





## JConsole程序

JConsole（Java Monitoring and Management Console），是一款基于 JMX（Java Manage-ment Extensions）的可视化监视管理工具。

JConsole 可以用来监视 Java 应用程序的运行状态，包括内存使用、线程状态、类加载、GC 等，还可以进行一些基本的性能分析。





## 内存泄漏排查

TODO









参考：

https://docs.oracle.com/en/java/javase/21/docs/specs/man/index.html

https://javabetter.cn/jvm/console-tools.html









END