基础组件 Buffer



在HTTP请求和HTTP响应中，都需要一个缓冲区来存储HTTP请求/响应报文。



## Buffer 属性

我们以 `std::vector<char> buffer_` 作为缓冲区实体，封装一个Buffer类以供使用。

缓冲区既作为读缓冲区，也作为写缓冲区。所以我们维护两个下标，`std::atomic<std::size_t> readPos_` 和 `std::atomic<std::size_t> writePos_` 。atomic原子类型保证多线程下的安全性。

> 可以类比Java中的InputStream和OutputStream的read和write方法，它们内部也各维护了读下标和写下标，这样才能知道调用read/write方法后输出/输入流的当前位置，以供下次读写。

```c++
// 存储实体
std::vector<char> buffer_;
// 读位置下标
std::atomic<std::size_t> readPos_;
// 写位置下标
std::atomic<std::size_t> writePos_;
```





## Buffer 方法

主要的两个方法，读方法 `ReadFd`  ，写方法 `WriteFd` ，分别接收SocketFD来进行，将FD的内容读取到缓冲区，和将缓冲区的内容写入到FD中，两个操作。

读取后，调整 `writePos_` 位置到读取的字节数处。写入后，调整`readPos_`位置到写入的字节数处。

```c++
// 读接口
ssize_t ReadFd(int fd, int *Errno);
// 写接口
ssize_t WriteFd(int fd, int *Errno);
```



读取的时候，为了避免buff无法容纳全部数据，准备一个临时缓冲区，如果读取之后发现缓冲区数量不足，数据就会暂存于临时缓冲区，这时候需要调用 `Append`，来将临时缓存区数据添加到缓冲区之后，必要时调用`MakeSpace_`方法进行缓冲区的扩容处理。

```c++
void Append(const std::string &str);
void Append(const char *str, size_t len);
void Append(const void *data, size_t len);
void Append(const Buffer &buff);

void MakeSpace_(size_t len);
```



获取写指针下标的方法，可以用于读取缓冲区数据。

```c++
const char *BeginWriteConst() const;
char *BeginWrite();
```



还有一系列调整读写下标的检索Retrieve方法。

```c++
void Retrieve(size_t len);
void RetrieveUntil(const char *end);
void RetrieveAll();
std::string RetrieveAllToStr();
```



END