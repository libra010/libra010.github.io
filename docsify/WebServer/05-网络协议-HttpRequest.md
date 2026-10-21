HttpRequest

HttpRequest的主要作用是解析HTTP请求，采用有限状态机实现。



## HTTP请求报文

主要分为请求行，请求头，空行，请求体四部分。

**请求行**：请求方法 + 空格 + URL + 空格 + 协议版本 + 回车换行符

**请求头**：头部字段名 + 冒号 + 值 + 回车换行符

**空行**：回车换行符

**请求体**：无要求，主要是请求数据

```http
GET /HEELO HTTP/1.1\r\n
Host: 127.0.0.1:1234\r\n
Connection: Keep-alive\r\n
Content-Length: 12\r\n
\r\n
hello world;
```





## 有限状态机

有限状态机，用明确的规则管理状态转换，让复杂逻辑变得可预测和可维护。

1、状态驱动行为：有限状态机在不同的状态下执行对应的行为。

2、条件转移：当执行行为的时候遇到了特定的状态就会切换下一个状态。

3、循环迭代：下一个状态再继续执行对应的行为，直到遇到条件，再次切换。如此循环直到错误状态或者终止。



HTTP请求报文的解析过程符合这个特点，有特定的状态（解析请求行 -> 解析请求头 -> 解析请求体），有明确的条件转移规则（遇到回车换行符切换到下个状态）。





## HttpRequest 解析方法

```c++
bool HttpRequest::parse(Buffer &buff){
    // ......
    const char CRLF[] = "\r\n";
    while (buff.ReadableBytes() && state_ != FINISH){
        const char *lineEnd = search(buff.Peek(), buff.BeginWriteConst(), CRLF, CRLF + 2);
        std::string line(buff.Peek(), lineEnd); // 获取读缓冲区的所有内容
        switch (state_) // 有限状态机
        {
            case REQUEST_LINE: // 当前状态为解析请求行
                if (!ParseRequestLine_(line)){ // 解析请求行
                    return false;
                }
                ParsePath_(); // 解析请求行的路径
                break;
            case HEADERS: // 当前状态为解析请求头
                ParseHeader_(line);
                if (buff.ReadableBytes() <= 2)
                { // 若缓冲区剩余数据≤2（可能只剩最后的CRLF），说明头部解析完毕
                    state_ = FINISH; // 切换到"解析完成"状态
                }
                break;
            case BODY:
                ParseBody_(line); // 解析请求体（如POST请求中的表单数据）
                break;
            default:
                break;
        }
    }
}

// 解析请求行
bool HttpRequest::ParseRequestLine_(const string &line)
{
    regex patten("^([^ ]*) ([^ ]*) HTTP/([^ ]*)$");
    smatch subMatch;
    if (regex_match(line, subMatch, patten))
    {  // 采用正则表达式，将匹配到的HTTP方法，路径，HTTP版本保存下来
        method_ = subMatch[1];
        path_ = subMatch[2];
        version_ = subMatch[3];
        state_ = HEADERS; // 切换有限状态机的状态
        return true;
    }
    LOG_ERROR("RequestLine Error");
    return false;
}

// 解析请求头
void HttpRequest::ParseHeader_(const string &line)
{
    regex patten("^([^:]*): ?(.*)$");
    smatch subMatch;
    if (regex_match(line, subMatch, patten))
    { // 将匹配到的所有请求头保存到Map
        header_[subMatch[1]] = subMatch[2];
    }
    else
    {
        state_ = BODY; // 切换有限状态机的状态
    }
}

// 解析请求体
void HttpRequest::ParseBody_(const string &line)
{
    body_ = line;// 将匹配到的所有请求体保存起来p
    ParsePost_();
    state_ = FINISH; // 切换有限状态机的状态
    LOG_DEBUG("Body:%s, len:%d", line.c_str(), line.size());
}

// 处理Post请求
void HttpRequest::ParsePost_()
{
    if (method_ == "POST" && header_["Content-Type"] == "application/x-www-form-urlencoded")
    { // 解析POST提交的登录表单数据
        ParseFromUrlencoded_();
        if (DEFAULT_HTML_TAG.count(path_))
        {
            int tag = DEFAULT_HTML_TAG.find(path_)->second;
            LOG_DEBUG("Tag:%d", tag);
            if (tag == 0 || tag == 1)
            {
                path_ = "/error.html";

                // bool isLogin = (tag == 1);
                // if (UserVerify(post_["username"], post_["password"], isLogin))
                // {
                //     path_ = "/welcome.html";
                // }
                // else
                // {
                //     path_ = "/error.html";
                // }
            }
        }
    }
}
```











END