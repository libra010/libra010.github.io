08 网络协议 HeapTimer



HTTP的协议中有 **Keep-Alive** 来允许连接复用，但是长时间不关闭的连接会长期占用服务端资源，所以需要实现一种机制来定时关闭长时间不用的连接。HeapTimer时间堆的目的就是这个。

时间堆的底层是小根堆，保证堆顶元素是最小的。

传统的定时方案是以固定频率调用起搏函数tick，进而执行定时器上的回调函数。而时间堆的做法则是将所有定时器中超时时间最小的一个定时器的超时值作为心搏间隔，当超时时间到达时，处理超时事件，然后再次从剩余定时器中找出超时时间最小的一个，依次反复即可。



## HeapTimer API

主要的对外接口有这些。
```c++
int GetNextTick();
void add(int id, int timeOut, const TimeoutCallBack &cb);
void adjust(int id, int newExpires);
void tick();
void doWork(int id);
```

**GetNextTick**：获取下次超时时间（最小堆的第一个节点的超时时间）。用于服务器的主循环中，配合**epoll_wait**函数，这个函数会在设置的时间超时后立即返回。

**add**：为每个新的连接**新增**定时器，其中TimeoutCallBack参数是回调方法，用于在定时器超时后回调关闭连接。

**adjust**：**调整**定时器，当一个连接有读写活动后，刷新定时器的时间。

**tick**：清理超时节点的主函数，在GetNextTick函数开始时自动调用。

**doWork**：手动清除定时器，有需要的时候外部直接调用。





## HeapTimer 内部实现

```c++
// 回调函数，用于传入关闭连接的lambda函数
typedef std::function<void()> TimeoutCallBack;
// 使用C++11的chrono库
typedef std::chrono::high_resolution_clock Clock;
typedef std::chrono::milliseconds MS;
typedef Clock::time_point TimeStamp;

// 定时器节点
struct TimerNode
{
    int id; // socketfd
    TimeStamp expires; // 时间戳
    TimeoutCallBack cb; // 关闭连接的回调函数
    bool operator<(const TimerNode &t)
    { // 比较时间
        return expires < t.expires;
    }
};

class HeapTimer
{
    // ......
    // 向上调整（堆的「上浮」操作）
    void siftup_(size_t i);
    // 向下调整（堆的「下沉」操作）
    bool siftdown_(size_t index, size_t n);
    // 交换节点
    void SwapNode_(size_t i, size_t j);
    // 使用vector保存时间堆
    std::vector<TimerNode> heap_;
    // 使用map保存socketfd和对应的vector的下标
    std::unordered_map<int, size_t> ref_;
}
```



```c++
while (!isClose_)
{
    int timeout = heapTimer_->GetNextTick();
    
    struct sockaddr_in clientaddr;
    socklen_t clientaddrlen = sizeof(clientaddr);
    int clientfd = accept(listenfd, (struct sockaddr *)&clientaddr, &clientaddrlen);

    if (clientfd > 0)
    {
        // 提交到线程池处理，每个连接一个线程
        threadpool_->AddTask(
            [clientfd, clientaddr, this]()
            {
                HttpConn client;
                client.init(clientfd, clientaddr);
                
                // 为该连接设置超时（例如60秒）
                heapTimer_->add(clientfd, 60000, [clientfd, this]() {
                    // 超时回调：关闭连接
                    close(clientfd);
                    // 其他清理工作（如从连接管理器中移除）
                });
                
                while (client.read(nullptr) > 0 && client.process())
                {
                    // 读写成功，刷新定时器（重置为60秒后超时）
    				heapTimer_->adjust(clientfd, 60000);
                    
                    client.write(nullptr);
                    if (!client.IsKeepAlive())
                        break;
                }
                
                // 循环结束后
				heapTimer_->del_(clientfd);  // 或调用doWork，根据你的接口设计
                close(clientfd);
            });
    }
}
```







END