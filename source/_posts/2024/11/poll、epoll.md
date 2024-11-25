---
title: poll、epoll
tags: [C语言]
categories: [C语言]
permalink: 
poster:
  topic: null
  headline: null
  caption: null
  color: null
date: 2024-11-25 22:47:23
topic: c
description:
cover:
banner:
references:
---
### 一、poll介绍

poll与select一样，只负责IO的等的过程，只不过一次可以等待多个文件描述符，他的作用是让read和write不再阻塞。

是用来监视多个文件描述符的状态变化的
程序会停在poll这里等待，直到被监视的文件描述符有一个或多个发生了状态改变

### 二、poll的接口

poll的接口如下，比select要轻量化很多，只有三个参数
![](https://i-blog.csdnimg.cn/direct/899a62e984c7418f8e6c3ed248b59676.png)

```c
参数1：struct pollfd *fds，pollfd数组首元素地址，

	pollfd是操作系统给我们提供的结构体，主要成员如下
	fd：文件描述符
	events：用户告诉内核，需要关心的fd，上面的事件
	revents：poll返回，内核告诉用户，关心的fd，那些事件就绪

参数2：nfds_t nfds，数组元素个数

参数3：int timeout，毫秒级的等待时间
timeout > 0 等待timeout毫秒或者有fd就绪再返回。
timeout == 0 非阻塞轮询。
timeout == -1 阻塞等待，直到有fd就绪。

返回值：
ret  >  0 ：poll等待的多个fd中，已经就需要的fd个数
ret == 0 ：poll超时返回
ret  <  0 ：poll出错
```

poll的事件如下，这些值是bit位，可以通过  |（或运算）  的方式写入到events中，我们着重学习POLLIN和POLLOUT

![](https://i-blog.csdnimg.cn/direct/a866b6a4129643debaa03017a1f5fb47.png)

![](https://i-blog.csdnimg.cn/direct/3480236feb334bfcaaae5a49c04bb9aa.png)

### 三、poll使用例子

Log.hpp

```c
#pragma once
 
#include <iostream>
#include <cstdarg>
#include <unistd.h>
#include <sys/stat.h>
#include <sys/types.h>
#include <pthread.h>
using namespace std;
 
enum
{
    Debug = 0,
    Info,
    Warning,
    Error,
    Fatal
};
 
enum
{
    Screen = 10,
    OneFile,
    ClassFile
};
 
string LevelToString(int level)
{
    switch (level)
    {
    case Debug:
        return "Debug";
    case Info:
        return "Info";
    case Warning:
        return "Warning";
    case Error:
        return "Error";
    case Fatal:
        return "Fatal";
 
    default:
        return "Unknown";
    }
}
 
const int default_style = Screen;
const string default_filename = "Log.";
const string logdir = "log";
 
class Log
{
public:
    Log(int style = default_style, string filename = default_filename)
        : _style(style), _filename(filename)
    {
        if (_style != Screen)
            mkdir(logdir.c_str(), 0775);
    }
 
    // 更改打印方式
    void Enable(int style)
    {
        _style = style;
        if (_style != Screen)
            mkdir(logdir.c_str(), 0775);
    }
 
    // 时间戳转化为年月日时分秒
    string GetTime()
    {
        time_t currtime = time(nullptr);
        struct tm *curr = localtime(&currtime);
        char time_buffer[128];
        snprintf(time_buffer, sizeof(time_buffer), "%d-%d-%d %d:%d:%d",
                 curr->tm_year + 1900, curr->tm_mon + 1, curr->tm_mday, curr->tm_hour, curr->tm_min, curr->tm_sec);
        return time_buffer;
    }
 
    // 写入到文件中
    void WriteLogToOneFile(const string &logname, const string &message)
    {
        FILE *fp = fopen(logname.c_str(), "a");
        if (fp == nullptr)
        {
            perror("fopen failed");
            exit(-1);
        }
        fprintf(fp, "%s\n", message.c_str());
 
        fclose(fp);
    }
 
    // 打印日志
    void WriteLogToClassFile(const string &levelstr, const string &message)
    {
        string logname = logdir;
        logname += "/";
        logname += _filename;
        logname += levelstr;
        WriteLogToOneFile(logname, message);
    }
 
    pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;
    void WriteLog(const string &levelstr, const string &message)
    {
        pthread_mutex_lock(&lock);
        switch (_style)
        {
        case Screen:
            cout << message << endl; // 打印到屏幕中
            break;
        case OneFile:
            WriteLogToClassFile("all", message); // 给定all，直接写到all里
            break;
        case ClassFile:
            WriteLogToClassFile(levelstr, message); // 写入levelstr里
            break;
        default:
            break;
        }
        pthread_mutex_unlock(&lock);
    }
 
    // 提供接口给运算符重载使用
    void _LogMessage(int level, const char *file, int line, char *rightbuffer)
    {
        char leftbuffer[1024];
        string levelstr = LevelToString(level);
        string currtime = GetTime();
        string  idstr = to_string(getpid());
 
        snprintf(leftbuffer, sizeof(leftbuffer), "[%s][%s][%s][%s:%d]", levelstr.c_str(), currtime.c_str(), idstr.c_str(), file, line);
 
        string messages = leftbuffer;
        messages += rightbuffer;
        WriteLog(levelstr, messages);
    }
 
    // 运算符重载
    void operator()(int level, const char *file, int line, const char *format, ...)
    {
        char rightbuffer[1024];
        va_list args;                                              // va_list 是指针
        va_start(args, format);                                    // 初始化va_list对象，format是最后一个确定的参数
        vsnprintf(rightbuffer, sizeof(rightbuffer), format, args); // 写入到rightbuffer中
        va_end(args);
        _LogMessage(level, file, line, rightbuffer);
    }
 
    ~Log()
    {
    }
 
private:
    int _style;
    string _filename;
};
 
Log lg;
 
class Conf
{
public:
    Conf()
    {
        lg.Enable(Screen);
    }
    ~Conf()
    {
    }
};
 
Conf conf;
 
// 辅助宏
#define lg(level, format, ...) lg(level, __FILE__, __LINE__, format, ##__VA_ARGS__)
```

Socket.hpp

```c
#pragma once
 
#include <iostream>
#include <string>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <cstring>
#include <unistd.h>
using namespace std;
namespace Net_Work
{
    static const int default_backlog = 5;
    static const int default_sockfd = -1;
    using namespace std;
 
    enum
    {
        SocketError = 1,
        BindError,
        ListenError,
        ConnectError,
    };
 
    // 封装套接字接口基类
    class Socket
    {
    public:
        // 封装了socket相关方法
        virtual ~Socket() {}
        virtual void CreateSocket() = 0;
        virtual void BindSocket(uint16_t port) = 0;
        virtual void ListenSocket(int backlog) = 0;
        virtual bool ConnectSocket(string &serverip, uint16_t serverport) = 0;
        virtual int AcceptSocket(string *peerip, uint16_t *peerport) = 0;
        virtual int GetSockFd() = 0;
        virtual void SetSockFd(int sockfd) = 0;
        virtual void CloseSocket() = 0;
        virtual bool Recv(string *buff, int size) = 0;
        virtual void Send(string &send_string) = 0;
 
        // 方法的集中在一起使用
    public:
        void BuildListenSocket(uint16_t port, int backlog = default_backlog)
        {
            CreateSocket();
            BindSocket(port);
            ListenSocket(backlog);
        }
 
        bool BuildConnectSocket(string &serverip, uint16_t serverport)
        {
            CreateSocket();
            return ConnectSocket(serverip, serverport);
        }
 
        void BuildNormalSocket(int sockfd)
        {
            SetSockFd(sockfd);
        }
    };
 
    class TcpSocket : public Socket
    {
    public:
        TcpSocket(int sockfd = default_sockfd)
            : _sockfd(sockfd)
        {
        }
        ~TcpSocket() {}
 
        void CreateSocket() override
        {
            _sockfd = socket(AF_INET, SOCK_STREAM, 0);
            if (_sockfd < 0)
                exit(SocketError);
        }
        void BindSocket(uint16_t port) override
        {
            int opt = 1;
            setsockopt(_sockfd, SOL_SOCKET, SO_REUSEADDR | SO_REUSEPORT, &opt, sizeof(opt));
 
            struct sockaddr_in local;
            memset(&local, 0, sizeof(local));
            local.sin_family = AF_INET;
            local.sin_port = htons(port);
            local.sin_addr.s_addr = INADDR_ANY;
 
            int n = bind(_sockfd, (struct sockaddr *)&local, sizeof(local));
            if (n < 0)
                exit(BindError);
        }
        void ListenSocket(int backlog) override
        {
            int n = listen(_sockfd, backlog);
            if (n < 0)
                exit(ListenError);
        }
        bool ConnectSocket(string &serverip, uint16_t serverport) override
        {
            struct sockaddr_in addr;
            memset(&addr, 0, sizeof(addr));
            addr.sin_family = AF_INET;
            addr.sin_port = htons(serverport);
            // addr.sin_addr.s_addr = inet_addr(serverip.c_str());
            inet_pton(AF_INET, serverip.c_str(), &addr.sin_addr);
            int n = connect(_sockfd, (sockaddr *)&addr, sizeof(addr));
 
            if (n == 0)
                return true;
            return false;
        }
        int AcceptSocket(string *peerip, uint16_t *peerport) override
        {
            struct sockaddr_in addr;
            socklen_t len = sizeof(addr);
            int newsockfd = accept(_sockfd, (sockaddr *)&addr, &len);
            if (newsockfd < 0)
                return -1;
 
            // *peerip = inet_ntoa(addr.sin_addr);
 
            // INET_ADDRSTRLEN 是一个定义在头文件中的宏，表示 IPv4 地址的最大长度
            char ip_str[INET_ADDRSTRLEN];
            inet_ntop(AF_INET, &addr.sin_addr, ip_str, INET_ADDRSTRLEN);
            *peerip = ip_str;
 
            *peerport = ntohs(addr.sin_port);
            return newsockfd;
        }
        int GetSockFd() override
        {
            return _sockfd;
        }
        void SetSockFd(int sockfd) override
        {
            _sockfd = sockfd;
        }
        void CloseSocket() override
        {
            if (_sockfd > default_sockfd)
                close(_sockfd);
        }
 
        bool Recv(string *buff, int size) override
        {
            char inbuffer[size];
            ssize_t n = recv(_sockfd, inbuffer, size - 1, 0);
            if (n > 0)
            {
                inbuffer[n] = 0;
                *buff += inbuffer;
                return true;
            }
            else
                return false;
        }
 
        void Send(string &send_string) override
        {
            send(_sockfd, send_string.c_str(),send_string.size(),0);
        }
 
    private:
        int _sockfd;
        string _ip;
        uint16_t _port;
    };
}
```

PollServer.hpp

```c
#pragma once
#include <iostream>
#include <string>
#include <poll.h>
#include <memory>
#include "Log.hpp"
#include "Socket.hpp"
 
using namespace Net_Work;
const static int gdefaultport = 8888;
const static int gbacklog = 8;
const static int gnum = 1024;
class PollServer
{
public:
    PollServer(int port) : _port(port), _num(gnum), _listensock(new TcpSocket())
    {
    }
    void HandlerEvent()
    {
        for (int i = 0; i < _num; i++)
        {
            if (_rfds[i].fd == -1)
                continue;
 
            int fd = _rfds[i].fd;
            short revents = _rfds[i].revents;
            // 判断事件是否就绪
            if (revents & POLLIN)
            {
                // 读事件分两类，一类是新链接到来，一类是新数据到来
                if (fd == _listensock->GetSockFd())
                {
                    // 新链接到来
                    lg(Info, "get a new link");
                    // 获取连接
                    std::string clientip;
                    uint16_t clientport;
                    int sockfd = _listensock->AcceptSocket(&clientip, &clientport);
                    if (sockfd == -1)
                    {
                        lg(Error, "accept error");
                        continue;
                    }
                    lg(Info, "get a client,client info is# %s:%d,fd: %d", clientip.c_str(), clientport, sockfd);
                    // 此时获取连接成功了，但是不能直接read write,sockfd仍需要交给poll托管 -- 添加到数组_rfds中
                    int pos = 0;
                    for (; pos < _num; pos++)
                    {
                        if (_rfds[pos].fd == -1)
                        {
                            _rfds[pos].fd = sockfd;
                            _rfds[pos].events = POLLIN;
                            lg(Info, "get a new link, fd is : %d", sockfd);
                            break;
                        }
                    }
                    if (pos == _num)
                    {
                        // 1.扩容
                        // 2.关闭
                        close(sockfd);
                        lg(Warning, "server is full, be carefull...");
                    }
                }
                else
                {
                    // 普通的读事件就绪
                    char buffer[1024];
                    ssize_t n = recv(fd, buffer, sizeof(buffer-1), 0);
                    if (n > 0)
                    {
                        buffer[n] = 0;
                        lg(Info, "client say# %s", buffer);
                        std::string message = "你好,同志";
                        message += buffer;
                        send(fd, message.c_str(), message.size(), 0);
                    }
                    else
                    {
                        lg(Warning, "client quit ,maybe close or error,close fd: %d", fd);
                        close(fd);
                        // 还要取消poll的关心
                        _rfds[i].fd = -1;
                        _rfds[i].events = 0;
                        _rfds[i].revents = 0;
                    }
                }
            }
        }
    }
    void InitServer()
    {
        _listensock->BuildListenSocket(_port, gbacklog);
        _rfds = new struct pollfd[_num];
        for (int i = 0; i < _num; i++)
        {
            _rfds[i].fd = -1;
            _rfds[i].events = 0;
            _rfds[i].revents = 0;
        }
        // 最开始的时候，只有一个文件描述符，Listensock
        _rfds[0].fd = _listensock->GetSockFd();
        _rfds[0].events |= POLLIN;
    }
 
    void Loop()
    {
        _isrunning = true;
        // 循环重置select需要的rfds
        while (_isrunning)
        {
            // 定义时间
            int timeout = 1000;
 
            //PrintDebug();
 
            // rfds是输入输出型参数，rfds是在select调用返回时，不断被修改，所以每次需要重置rfds
            int n = poll(_rfds, _num, timeout);
            switch (n)
            {
            case 0:
                lg(Info, "select timeout...");
                break;
            case -1:
                lg(Error, "select error!!!");
            default:
                // 正常就绪的fd
                lg(Info, "select success,begin event handler");
                HandlerEvent();
                break;
            }
        }
        _isrunning = false;
    }
 
    void Stop()
    {
        _isrunning = false;
    }
 
    void PrintDebug()
    {
        // std::cout << "current select rfds list is :";
        // for (int i = 0; i < num; i++)
        // {
        //     if (_rfds_array[i] == nullptr)
        //         continue;
        //     else
        //         std::cout << _rfds_array[i]->GetSockFd() << " ";
        // }
        // std::cout << std::endl;
    }
 
private:
    std::unique_ptr<Socket> _listensock;
    int _port;
    bool _isrunning;
 
    struct pollfd *_rfds;
    int _num;
};
```

 Main.cc

```cpp
#include <iostream>
#include <memory>
#include "PollServer.hpp"
 
void Usage(char* argv)
{
  
    std::cout<<"Usage: \n\t"<<argv<<" port\n"<<std::endl;
}
// ./select_server 8080
int main(int argc,char* argv[])
{
    // std::cout<<num<<std::endl;       1024
    if(argc!=2)
    {
        Usage(argv[0]);
        return -1;
    }
    uint16_t localport = std::stoi(argv[1]);
    std::unique_ptr<PollServer> svr = std::make_unique<PollServer>(localport);
    svr->InitServer();
    svr->Loop();
 
    return 0;
}
```

运行结果如下，由于我们poll第三个参数设置的是1000ms，因此每一秒poll都会返回，当发现有新链接的时候，就回去执行函数，在函数中调用write或者read变不会再阻塞了。 

![](https://raw.githubusercontent.com/mengchao666/picture/main/blog03c25829c18e4621add56de57fd2a1aa.png)