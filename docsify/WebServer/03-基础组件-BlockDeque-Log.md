BlockDeque 阻塞队列

Log 日志



## BlockDeque 属性

阻塞队列采用 `std::deque` 来实现，另外还需要互斥锁和条件变量。

```c++
// 底层数据结构 双端队列
std::deque<T> deq_;
// 容量
size_t capacity_;
// 互斥锁
std::mutex mtx_;
// 关闭标志
bool isClose_;
// 消费者条件变量
std::condition_variable condConsumer_;
// 生产者条件变量
std::condition_variable condProducer_;
```



## BlockDeque 方法

基于Deque双端队列的方法：push_back（尾部插入），push_front（头部插入），pop_front（删除头部元素），front（头部元素），back（尾部元素），clear（清空元素）。

```c++
void clear();
bool empty();
bool full();
size_t size();
size_t capacity();
T front();
T back();
void push_back(const T &item);
void push_front(const T &item);
bool pop(T &item);
bool pop(T &item, int timeout);
void flush(); // 刷新方法，生产者条件变量通知所有消费者。
void Close(); // 析构时修改isClose_标识
```



```c++
template <class T>
size_t BlockDeque<T>::size()
{
  // 轻量级RAII锁，离开作用域自动解锁
  std::lock_guard<std::mutex> locker(mtx_);
  return deq_.size();
}

template <class T>
void BlockDeque<T>::push_front(const T &item)
{
  // RAII锁，支持条件变量等功能
  std::unique_lock<std::mutex> locker(mtx_);
  while (deq_.size() >= capacity_)
  {
     condProducer_.wait(locker);
  }
  deq_.push_front(item);
  condConsumer_.notify_one();
}
```



## Log 属性

```c++
Buffer buff_;  // 输出的内容 缓冲区
int level_;   // 日志等级
bool isAsync_; // 是否开启异步日志
FILE *fp_;                    // 打开log的文件指针
std::unique_ptr<BlockDeque<std::string>> deque_; // 阻塞队列
std::unique_ptr<std::thread> writeThread_;    // 写线程的指针
std::mutex mtx_;                 // 同步日志必须的互斥量
```



## Log 方法

单例模式和宏定义实现

```c++
// 懒汉模式 局部静态变量法
Log *Log::Instance()
{
  static Log inst;
  return &inst;
}

#define LOG_BASE(level, format, ...)          \
do                         \
{                          \
	Log *log = Log::Instance();           \
	if (log->IsOpen() && log->GetLevel() <= level) \
	{                        \
	log->write(level, format, ##__VA_ARGS__);  \
	log->flush();                \
	}                        \
} while (0);
```

初始化方法，判断异步日志，构造阻塞队列和日志的写入线程（用于异步写入）

```c++
void Log::init()
{
    // ......
    if (!deque_)
    {
        // 创建阻塞队列
        unique_ptr<BlockDeque<std::string>> newDeque(new BlockDeque<std::string>);
        deque_ = move(newDeque);
        // 创建异步写入线程，FlushLogThread方法异步执行写入日志文件
        std::unique_ptr<std::thread> NewThread(new thread(FlushLogThread));
        writeThread_ = move(NewThread);
    }
    // ......
    char fileName[LOG_NAME_LEN] = {0};
    // 安全格式化字符串
    snprintf(fileName, LOG_NAME_LEN - 1, "%s/%04d_%02d_%02d%s",
             path_, t.tm_year + 1900, t.tm_mon + 1, t.tm_mday, suffix_);
    toDay_ = t.tm_mday;
    {
        lock_guard<mutex> locker(mtx_); // 加锁
        buff_.RetrieveAll(); // 清空缓冲区
        if (fp_)
        { // 如果文件已打开，安全关闭
            flush();
            fclose(fp_);
        }

        fp_ = fopen(fileName, "a");
        if (fp_ == nullptr)
        { // 打开失败，尝试创建目录后重新打开
            mkdir(path_, 0777);
            fp_ = fopen(fileName, "a");
        }
        assert(fp_ != nullptr);
    }
}
```



核心的write方法

```c++
void Log::write(int level, const char *format, ...)
{
    // ...... 判断替换新的日志文件
    
    // 在buffer内生成一条对应的日志信息
    {
        // 多线程写日志要先加锁
        unique_lock<mutex> locker(mtx_);
        lineCount_++;
        int n = snprintf(buff_.BeginWrite(), 128, "%d-%02d-%02d %02d:%02d:%02d.%06ld ",
                         t.tm_year + 1900, t.tm_mon + 1, t.tm_mday,
                         t.tm_hour, t.tm_min, t.tm_sec, now.tv_usec);

        // 缓冲区移动写下标
        buff_.HasWritten(n);
        // 缓存区新增日志等级字符串
        AppendLogLevelTitle_(level);

        // vaList可变参数列表，vsnprintf将格式化字符串写入缓冲区
        va_start(vaList, format);
        int m = vsnprintf(buff_.BeginWrite(), buff_.WritableBytes(), format, vaList);
        va_end(vaList);

        buff_.HasWritten(m); // 更新缓存区指针
        buff_.Append("\n\0", 2); // 添加换行符和字符串结束符，完成一条日志

        if (isAsync_ && deque_ && !deque_->full())
        {
            // 异步方式 加入阻塞队列中 等待写线程读取日志信息
            deque_->push_back(buff_.RetrieveAllToStr());
        }
        else
        {
            // 同步方式 直接向日志中写入日志信息
            fputs(buff_.Peek(), fp_);
        }
        buff_.RetrieveAll(); // 清空buffer
    }
}
```







END