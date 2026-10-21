Java synchronized锁实现



synchronized同步代码块是通过**监视器（Monitor）**实现的，这是一个用C++实现的同步控制结构。

每个Java对象都有一个**对象头（Object Header）**，其中有一个称为**Mark Word**的结构，里面存储了指向**监视器（Monitor）**的指针。





## 监视器（Monitor）

在 HotSpot 虚拟机中，Monitor 是一个由 C++ 实现的同步控制结构，其具体实现类为 ObjectMonitor。

```c++
// ObjectMonitor类结构的源代码(构造函数)
// github.com/openjdk/jdk/blob/master/src/hotspot/share/runtime/objectMonitor.cpp
ObjectMonitor::ObjectMonitor(oop object) :
  _metadata(0),
  _object(_oop_storage, object),
  _owner(NO_OWNER), // 持有锁的线程（Owner）
  _previous_owner_tid(0),
  _next_om(nullptr),
  _recursions(0), // 锁的计数器（recursion count）
  _entry_list(nullptr), // 等待获取锁的线程队列（Entry Set）
  _entry_list_tail(nullptr),
  _succ(NO_OWNER),
  _SpinDuration(ObjectMonitor::Knob_SpinLimit),
  _contentions(0),
  _wait_set(nullptr), // 调用 wait() 后进入等待的线程队列（Wait Set）
  _waiters(0),
  _wait_set_lock(0),
  _stack_locker(nullptr)
{ }
```

它包含几个关键字段：持有锁的线程（Owner）等待获取锁的线程队列（Entry Set）调用 wait() 后进入等待的线程队列（Wait Set）锁的计数器（recursion count）

这个 ObjectMonitor 就是 Monitor 的具体实现，它存在于 JVM 的堆或 native memory 中，但其引用存储在对象头的 Mark Word 中。





## 对象头（Object Header）

在 64 位 HotSpot JVM 中，每个 Java 对象都有一个对象头（Object Header），其核心部分是 Mark Word（8 字节），用于存储对象的哈希码、GC 分代年龄、锁状态标志，以及指向锁记录或 Monitor 的指针等信息。

Mark Word 的结构会根据对象的锁状态动态变化：

- 当对象处于**重量级锁**状态时，其 64 位数据用于存储指向 `ObjectMonitor` 的指针。
- 当对象处于**轻量级锁**状态时，64 位数据则存储指向当前线程栈帧中 `Lock Record`（锁记录）的指针。

**轻量级锁**：当线程尝试获取锁时，JVM 会在该线程的栈帧中创建一个 `Lock Record`，用于保存对象当前的 Mark Word 值。随后通过 CAS 操作尝试将对象的 Mark Word 更新为指向该 `Lock Record` 的指针。若更新成功，线程获得轻量级锁；若 CAS 失败，说明存在锁竞争，锁将膨胀为重量级锁。

**偏向锁**：其设计目标是消除无竞争场景下的同步开销。当一个线程首次获取偏向锁时，JVM 会通过 CAS 操作将 Mark Word 中的线程 ID 设置为该线程的 ID。（避免了创建 Lock Record 或 ObjectMonitor 等同步控制结构的开销）。此后，该线程再次进入同一锁时，无需进行任何同步操作，直接认为已持有锁。但如果检测到其他线程竞争该锁，偏向锁会升级为轻量级锁（或直接进入锁膨胀流程）。











END