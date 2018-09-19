## Intro

我们要实现一个简单的网络代理，只需要转发HTTP请求事件。网络代理其实就是从客户端到服务器路途中间的一个中间站。网络代理既要像服务器一样接收客户端发送来的数据，也要像客户端一样向服务器发送消息。这么看来，我们弄懂了网络代理，就能同时搞懂客户端和服务器。我估计，这就是为什么网络编程这一章实验的内容，会选为实现网络代理了。我们的实验目标分三步走：一、实现一个单线程的网络代理；二、多线程；三、代理加入缓存功能。

## 知识准备

```
getaddrinfo能够将传入的用String变量代表的 hostnames、 host addresses、 service names和port number等转换成为socket address structure。
该函数是线程安全的，并且是可重入的函数，而且该函数与协议无关，适用IPV4 IPV6 。
```

### 套接字

套接字函数：CSAPP书中为了我们的方便，基于套接字函数实现了两个辅助函数。open_clientfd，输入服务器的ip和端口，返回套接字描述符。open_listenfd输入端口，返回监听描述符

### HTTP

HTTP请求：一个HTTP请求由请求行、请求报头、空行组成。我们的代理要发送给服务器的请求，必须包含这些请求报头。同时如果客户端还发过来了其他请求报头，也要加在下文里面，一起发给客户端。

```
GET /hub/index.html HTTP/1.0
Host: www.cmu.edu：80
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:10.0.3) Gecko/20120305 Firefox/10.0.3
Connection: close
Proxy-Connection: close
空行
```

[HTTP消息头](http://www.cnblogs.com/jacktu/archive/2008/01/16/1041710.html)

### web proxy

思考一下，我们要实现的代理和web服务器的异同是什么：

1. 都需要接收客户端发送来的HTTP请求；都需要解析HTTP请求行。代理解析请求行时，需要得到服务器的ip、端口和资源路径（为了连接服务器和构造请求报头），而不需要关注是动态和静态内容。
2. 我们还需要构造一些固定的请求报头，组成完整的HTTP请求发给服务器。
3. 要接收服务器发回的信息，然后发给客户端。

### 大致框架

我们的大体框架思路就出来了：

- main函数：同TINY Web
- doit:读入请求行（同TINY Web），调用解析请求行函数parse_uri，调用请求报头构造函数build_request_header，再处理和服务器相关事宜。
- parse_uri：解析uri，和TINY Web有差异。
- build_request_header：建立请求报头的函数，需要自己写。

最后得到的框架如下，大部分都和TINY Web里的代码一样。我们只要完成parse_uri函数和build_request_header函数就可以了。

### [Socket描述符](https://www.cnblogs.com/davidzhou11225/archive/2012/05/03/2480347.html)

　　因为套接字API最初是作为UNIX操作系统的一部分而开发的，所以套接字API与系统的其他I/O设备集成在一起。特别是，当应用程序要为因特网通信而创建一个套接字（socket）时，操作系统就返回一个小整数作为描述符（descriptor）来标识这个套接字。然后，应用程序以该描述符作为传递参数，通过调用函数来完成某种操作（例如通过网络传送数据或接收输入的数据）。

　　**要点：当应用程序要创建一个套接字时，操作系统就返回一个小整数作为描述符，应用程序则使用这个描述符来引用该套接字。**

　　int socket(int domain, int type, int protocol);domain指明所使用的协议族，通常为PF_INET，表示互联网协议族（TCP/IP协议族）；type参数指定socket的类型：SOCK_STREAM 或SOCK_DGRAM，Socket接口还定义了原始Socket（SOCK_RAW），允许程序使用低层协议；protocol通常赋值"0"。Socket()调用返回一个整型socket描述符，你可以在后面的调用使用它。 Socket描述符是一个指向内部数据结构的指针，它指向描述符表入口。调用Socket函数时，socket执行体将建立一个Socket，实际上"建立一个Socket"意味着为一个Socket数据结构分配存储空间。 Socket执行体为你管理描述符表。两个网络程序之间的一个网络连接包括五种信息：通信协议、本地协议地址、本地主机端口、远端主机地址和远端协议端口。

　　该函数如果调用成功就返回新创建的套接字的描述符，如果失败就返回INVALID_SOCKET。套接字描述符是一个整数类型的值。每个进程的进程空间里都有一个套接字描述符表，该表中存放着套接字描述符和套接字数据结构的对应关系。该表中有一个字段存放新创建的套接字的描述符，另一个字段存放套接字数据结构的地址，因此根据套接字描述符就可以找到其对应的套接字数据结构。每个进程在自己的进程空间里都有一个套接字描述符表但是套接字数据结构都是在操作系统的内核缓冲里。

**xuef：这里可以看到网络套接字与IO之间的联系了。**



## proxy

### Part 1

要求：实现一个顺序执行的代理，它可以处理GET方法并转发，对于其他方法可以不实现。

命令行调用./proxy < port >来启动代理服务器，其中port可以通过实验包中的工具port-for-user来获取。





## 要注意的点

1. 在proxy打开与server的TCP连接的时候，需要调用`gethostbyname`或者`gethostbyaddr`来通过DNS获取server主机的DNS信息，比如ip地址，别名之类的，返回的是一个struct的指针。但是这个struct是一个静态变量，也就是说这些函数不支持多线程的访问，是线程不安全的。解决方法是定义一个mutex来加锁，任意时刻只能又一个线程在调这些函数。
2. 本书的作者提供了robust IO让我们方便地对socket file descriptor进行读写，不要用C库。
3. 调用 Signal(SIGPIPE, SIG_IGN); 将`SIGPIPE`这个信号忽略掉。如果尝试两次发送数据到一个已经被对方关闭的socket上时，内核会发送一个`SIGPIPE`信号给程序，在默认情况下，会终止当前程序，显然不是我们想要的，所以要忽略它。这里又一个stackoverflow上的[相关问题](http://stackoverflow.com/questions/108183/how-to-prevent-sigpipes-or-handle-them-properly)。还有一点，往broken pipe里写会使errno等于EPIPE，而往broken pipe里读会使errno等于ECONNRESET。
4. HTTP/1.1里默认将connection定义为keep-alive，也就是一条TCP连接可以处理多个请求，不用每次都要重新建立TCP连接。我们的简易proxy还无法提供这样的功能，所以在读client发过来的header的时候，如果是`Connection: keep-alive`或者`Proxy-Connection: keep-alive`，我们都要把它们换成`Connection: close`或`Proxy-Connection: close`。
5. 创建线程以后记得要detach掉，否则这个线程结束后不会释放资源直到有别的线程join了这个线程。
6. 如果header里没有`Content-Length`这一项，怎么确定body的长度？这个问题一直没想过直到现在遇到了这个问题。这个长度写到了body里，这种方式叫做`Transfer Encoding`。因为服务器在处理静态对象时，事先知道对象的大小；而在处理动态对象时，无法事先知道body的长度。实现的时候分两种情况来从sock中读数据。
7. 需要正确关闭所有的文件描述符。系统给一个程序能打开的描述符数量做了一个限制。如果是ubuntu下，可以通过`cat /proc/sys/fs/file-max`来查看最大文件描述符数。在proxy运行一段时间，确保描述符不会持续增加。在ubuntu下查看程序打开的描述符方法：找到程序的pid，然后`cat /proc/$pid/fd`。
8. 记得错误处理，这个一直是个麻烦问题。



