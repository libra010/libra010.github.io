06 网络协议 HttpConn



这是HTTP Connection类，主要作用是调用HttpRequest来解析数据，并调用HttpResponse来生成响应。



## HttpConn 属性

最关键的两个属性是一个读缓冲区，一个写缓冲区。另外通过sockaddr_in结构体保存连接数据，通过iovec结构体实现了分散IO，加快性能。

```c++
int fd_;
struct sockaddr_in addr_;
bool isClose_;
int iovCnt_;
struct iovec iov_[2];
Buffer readBuff_;  // 读缓冲区
Buffer writeBuff_; // 写缓冲区
HttpRequest request_;
HttpResponse response_;
```



## HttpConn 连接处理

```c++
bool HttpConn::process(){
    // 初始化HttpRequest 解析请求
    request_.Init();
    if (readBuff_.ReadableBytes() <= 0){
        return false;
    }
    else if (request_.parse(readBuff_)){ // 解析成功
        response_.Init(srcDir, request_.path(), request_.IsKeepAlive(), 200);
    }
    else{ // 解析失败
        response_.Init(srcDir, request_.path(), false, 400);
    }
    
    // 生成响应报文放入writeBuff_中
    response_.MakeResponse(writeBuff_);
    
    // 分散IO
    iov_[0].iov_base = const_cast<char *>(writeBuff_.Peek());
    iov_[0].iov_len = writeBuff_.ReadableBytes();
    iovCnt_ = 1;
    
    // 响应文件
    if (response_.FileLen() > 0 && response_.File()){
        iov_[1].iov_base = response_.File();
        iov_[1].iov_len = response_.FileLen();
        iovCnt_ = 2;
    }
    return true;
}
```



## HttpConn 读写方法

```c++
ssize_t HttpConn::read(int *saveErrno)
{
    ssize_t len = -1;
    do{
        len = readBuff_.ReadFd(fd_, saveErrno);
        if (len <= 0){
            break;
        }
    } while (isET);
    return len;
}

ssize_t HttpConn::write(int *saveErrno)
{
    ssize_t len = -1;
    do{
        // 核心：用writev发送多个数据块
        len = writev(fd_, iov_, iovCnt_);
        
        if (len <= 0){ // 发送错误
            *saveErrno = errno;
            break;
        }
        if (iov_[0].iov_len + iov_[1].iov_len == 0){ // 发送完成
            break;
        }
        // 处理部分发送
        else if (static_cast<size_t>(len) > iov_[0].iov_len)
        { 	// 第一个数据块已经发送完成，剩余第二个数据块
            // 计算第二个数据块已发送的长度：总发送长度 - 第一个数据块的长度
            iov_[1].iov_base = (uint8_t *)iov_[1].iov_base + (len - iov_[0].iov_len);
            // 调整第二个数据块的基地址（跳过已发送部分）和剩余长度
            iov_[1].iov_len -= (len - iov_[0].iov_len);
            // 清空第一个数据块的缓冲区，标记其长度为0（已发完）
            if (iov_[0].iov_len)
            {
                writeBuff_.RetrieveAll(); // 清空响应头缓冲区
                iov_[0].iov_len = 0;
            }
        }
        else 
        { 	// 仅发送了第一个数据块（响应头）的一部分
            // 调整第一个数据块的基地址（跳过已发送部分）和剩余长度
            iov_[0].iov_base = (uint8_t *)iov_[0].iov_base + len;
            iov_[0].iov_len -= len;
            writeBuff_.Retrieve(len);// 从响应头缓冲区中移除已发送的部分
        }
        // 循环条件，isET 如果是边缘触发（ET）模式，需要一次性把所有数据发送完
        // 如果剩余数据超过 10240 字节，即使是水平触发（LT）模式，也继续循环发送
    } while (isET || ToWriteBytes() > 10240);
    return len;
}
```







END