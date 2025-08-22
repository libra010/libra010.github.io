04 基础组件 ThreadPool



WebServer处理请求是多线程的，为了减少线程的创建开销，采用线程池来提高效率。



## ThreadPool 属性

使用一个结构体来封装线程池，内部包含互斥锁、条件变量、是否关闭、任务队列四个字段。

处理任务通过lambda匿名函数存入任务队列，线程从中取出任务并执行。

```c++
struct Pool
{
    std::mutex mtx;
    std::condition_variable cond;
    bool isClosed;
    // 任务队列 函数类型为void()
    std::queue<std::function<void()>> tasks;
};
std::shared_ptr<Pool> pool_;
```



## ThreadPool 构造方法

构造函数接收线程数量作为参数，通过`std::thread`创建数个工作线程，线程不断while循环从任务队列中取出任务并且执行，没有任务时采用条件变量等待唤醒。

```c++
explicit ThreadPool(size_t threadCount = 8) : pool_(std::make_shared<Pool>())
{
    assert(threadCount > 0);
    for (size_t i = 0; i < threadCount; i++)
    {
      std::thread([pool = pool_]
         {
              std::unique_lock<std::mutex> locker(pool->mtx);
              while(true) {
                    // 取任务执行 注意上锁
                    if(!pool->tasks.empty()) {
                      auto task = std::move(pool->tasks.front());
                      pool->tasks.pop();
                      locker.unlock();
                      task();
                      locker.lock();
                    } 
                    else if(pool->isClosed) break;
                    // 等待 如果任务来了就notify
                    else pool->cond.wait(locker);
          	  } 
        }).detach();
    }
}
```



## ThreadPool 添加任务

对外提供AddTask方法作为API添加任务。

```c++
template <class F>
void AddTask(F &&task)
{
	{
		std::lock_guard<std::mutex> locker(pool_->mtx);
		pool_->tasks.emplace(std::forward<F>(task));
	}
	// 唤醒线程处理任务
	pool_->cond.notify_one();
}
```











END