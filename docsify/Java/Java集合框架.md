Java集合框架



官方文档：Collections Framework Overview
https://docs.oracle.com/javase/8/docs/technotes/guides/collections



## 历史

**Java 集合框架（Java Collections Framework）** 是一个统一的架构，用于表示和操作对象集合。该框架是 `java.util` 包的核心组成部分，首次随 **JDK 1.2**（即 Java 2 平台）引入。

在 JDK 1.2 之前，Java 提供的集合类型较为有限，主要包括 `Vector`、`Stack`、`Hashtable`、`Enumeration`、`Properties` 和 `Dictionary` 等类。这些早期类存在设计局限，比如线程同步安全的性能问题。

JDK 1.2 对集合体系进行了全面重构，引入了现代集合框架——**Java 集合框架（JCF）**。新框架基于接口、实现和算法的分层设计，提供了更高的灵活性、可扩展性和性能。原有的集合类（如 `Vector` 和 `Hashtable`）虽被保留并进行了兼容性增强，以支持新的集合接口，但已不推荐使用。





## 接口与实现

接口：**Map**，**Collection**。其中Map的扩展接口**SortedMap**，**NavigableMap**。其中Collection的扩展接口**List**，**Queue（Deque）**，**Set（SortedSet）**。

常用的实现类：Set（HashSet）List（ArrayList）Map（HashMap）Queue（LinkedList）Deque（ArrayDeque）





## 算法 Algorithms

Java 集合框架中的算法操作主要由 **Collections** 工具类提供。这些算法以**静态方法**的形式定义，统一将要操作的集合作为**第一个参数**传入。

**排序（Sorting）**：Collections::sort

**洗牌（Shuffling）**：Collections::shuffle

**常规数据操作（Routine Data Manipulation）**：reverse（反转）fill（填充）copy（复制）swap（交换）addAll（添加所有）

**搜索（Searching）**：Collections::binarySearch

**组合操作（Composition）**：frequency（统计指定元素出现次数）disjoint（判断两个集合是否不相交）

**极值查找（Finding Extreme Values）**：min（最小值）max（最大值）





## 包装器与同步集合

**同步包装器（Synchronization Wrappers）**，可以通过同步包装器为原本非线程安全的集合添加同步控制，这是Java5的引入 JUC 同步框架之前 Java 最常用的同步集合机制。内部实现采用synchronized锁住集合对象本身来实现线程安全。

同步包装器是**Collections**类中的一系列synchronizedXXX静态方法，返回一个线程安全集合，注意必须使用返回的对象，不应该使用之前的对象了。

同步包装器包括Collections#synchronizedList，Collections#synchronizedMap等等。



还有**不可变集合的包装器（Unmodifiable Wrappers）**，用于创建**只读视图**的集合，防止对集合的修改操作。内部实现通过拦截集合的修改方法并抛出异常。

不可变集合包装器包括Collections#unmodifiableList，Collections#unmodifiableMap等等。





## 数组工具

虽然数组本身不属于 Java 集合框架，但 `Arrays` 类自 JDK 1.2 起被引入，并被视为集合框架的**配套工具类**。它与集合框架共用部分基础设施（如集合接口和算法），提供了对数组进行操作的静态方法，例如排序、查找、填充和比较等。





## 接口与实现-附录

除 Map 接口及其子接口外，Java 集合框架中所有集合类型的共同父接口均为 Collection。

#### List接口

`List` 是一个**有序集合**（也称为序列），允许元素重复，并支持按索引访问。常用实现类为 ArrayList 。

#### Map接口

`Map` 表示一个**键值对**（key-value pair）的映射集合，也称关联数组。常用实现类为 HashMap。

#### Set接口

`Set` 是一种**不允许重复元素**的集合。常用实现类为 HashSet。

#### Queue接口

`Queue` 接口自 Java 5 引入，用于表示**队列**数据结构，通常遵循**先进先出**（FIFO）原则。

提供三种操作元素的方法。（每个方法分为抛异常和返回空值两个版本）


| 操作描述             | 抛出异常      | 返回特殊值           |
| -------------------- | ------------- | -------------------- |
| 将元素插入队尾       | `add(E e)`    | `boolean offer(E e)` |
| 获取并移除队首元素   | `E remove()`  | `E poll()`           |
| 获取但不移除队首元素 | `E element()` | `E peek()`           |

`Queue` 有两个重要扩展：

- **`PriorityQueue`**：自 Java 5 引入，基于优先级堆实现的优先级队列。
- **`BlockingQueue`**：自 Java 5 引入，支持阻塞插入和移除操作的线程安全队列。

#### Deque接口

Java 6 引入。双端队列（double ended queue）。同时也是Queue接口的扩展。

作为双端队列使用时，有六种操作元素的方法。（每个方法分为抛异常和返回空值两个版本）

| 操作     | 首元素（头部）  |                 | 尾元素（尾部） |                |
| :------- | :-------------- | :-------------- | :------------- | :------------- |
|          | **抛出异常**    | **返回特殊值**  | **抛出异常**   | **返回特殊值** |
| **插入** | `addFirst(e)`   | `offerFirst(e)` | `addLast(e)`   | `offerLast(e)` |
| **删除** | `removeFirst()` | `pollFirst()`   | `removeLast()` | `pollLast()`   |
| **查看** | `getFirst()`    | `peekFirst()`   | `getLast()`    | `peekLast()`   |

作为**FIFO（先进先出）队列**使用时，提供了兼容Queue接口的方法。

另外，Deque接口同样也可以作为**LIFO（后进先出）栈**来使用，提供了兼容Stack的方法，也是Java推荐的替代原始Stack类的方式。

| 方法描述           |         |
| ------------------ | ------- |
| 把元素压栈         | push(E) |
| 把栈顶的元素弹出   | pop(E)  |
| 取栈顶元素但不弹出 | peek(E) |







END