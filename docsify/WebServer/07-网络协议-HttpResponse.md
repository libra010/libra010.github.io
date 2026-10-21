07 网络协议 HttpResponse



## HTTP 响应报文

**状态行**：协议版本 + 空格 + 状态码 + 空格 + 状态描述符 + 回车换行符

**响应头**：头部字段名 + 冒号 + 值 + 回车换行符

**空行**：回车换行符

**响应正文**：无要求，主要是响应数据

```http
HTTP\1.1 200 OK\r\n
Content-Encoding: gzip\r\n
Content-Type: text/html\r\n
Content-Length: 5\r\n
\r\n
hello
```





## HttpResponse 响应请求

```c++
void HttpResponse::MakeResponse(Buffer &buff)
{
    const std::string fullPath = srcDir_ + path_;
    // stat函数用于获取文件状态，存到mmFileStat_变量里。
    // S_ISDIR 判断是否请求的是文件目录
    if (stat(fullPath.data(), &mmFileStat_) < 0 || S_ISDIR(mmFileStat_.st_mode){
        code_ = 404;
    }
    else if (!(mmFileStat_.st_mode & S_IROTH)){
        // 文件无读取权限
        code_ = 403;
    }
    else if (code_ == -1){
        // 之前未设置状态码，且文件正常
        code_ = 200;
    }
    ErrorHtml_(); // 根据条件生成错误页面
    AddStateLine_(buff); // 添加响应行
    AddHeader_(buff); // 添加响应头
    AddContent_(buff); // 添加响应体
}


// 根据条件生成错误页面
void HttpResponse::ErrorHtml_(){
    // 从Map（状态码与错误页面路径）中查找错误页面
    if (CODE_PATH.count(code_) == 1){
        path_ = CODE_PATH.find(code_)->second;
        stat((srcDir_ + path_).data(), &mmFileStat_);
    }
}
      
// 添加响应行
void HttpResponse::AddStateLine_(Buffer &buff)
{
    string status;
    // 从Map（状态码与响应描述符）中查找响应描述符
    if (CODE_STATUS.count(code_) == 1){
        status = CODE_STATUS.find(code_)->second;
    }
    else{
        code_ = 400;
        status = CODE_STATUS.find(400)->second;
    }
    buff.Append("HTTP/1.1 " + to_string(code_) + " " + status + "\r\n");
}
        
// 添加响应头（Connection与Content-type）
void HttpResponse::AddHeader_(Buffer &buff)
{
    buff.Append("Connection: ");
    if (isKeepAlive_){
        buff.Append("keep-alive\r\n");
        buff.Append("keep-alive: max=6, timeout=120\r\n");
    }
    else{
        buff.Append("close\r\n");
    }
    // 判断文件类响应Content-type
    buff.Append("Content-type: " + GetFileType_() + "\r\n");
}
      
// 添加响应体（如果文件错误，返回ErrorContent函数定义的错误页面）
void HttpResponse::AddContent_(Buffer &buff)
{
    // 以只读方式打开文件
    int srcFd = open((srcDir_ + path_).data(), O_RDONLY);
    if (srcFd < 0){
        ErrorContent(buff, "File NotFound!");
        return;
    }
    // mmap函数（内存映射，零拷贝）：将打开的文件映射到进程的内存地址空间，返回映射区的起始地址
    int *mmRet = (int *)mmap(0, mmFileStat_.st_size, PROT_READ, MAP_PRIVATE, srcFd, 0);
    if (*mmRet == -1){
        ErrorContent(buff, "File NotFound!");
        return;
    }
    // 保存映射地址，关闭文件描述符
    mmFile_ = (char *)mmRet;
    close(srcFd);
    // 添加响应头（Content-length）
    buff.Append("Content-length: " + to_string(mmFileStat_.st_size) + "\r\n\r\n");
}
```







END