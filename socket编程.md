[toc]



# socket编程铺垫

## 什么是套接字

socket 的原意是“插座”，在计算机通信领域，socket 被翻译为“套接字”，它是计算机之间进行通信的**一种约定**或一种方式。也可以理解为网络进程通信的一端.

主要是UDP和TCP,UDP是不可靠的,TCP是可靠的.

- 网络里可靠是一个中性词,因为可靠意味着要做更多的事,不可靠意味着简单,所以各有利弊

传输时不确定就无脑TCP,因为可靠,以后学的多了可以肯定用UDP,比如可以接受丢包的时候就选择UDP

- 套接字很多套,不同的通信时得用不同的标准,linux设置的结构保证统一接口(sockaddr),接口里没有void* ,因为当时c语言还没有void*

## 端口和IP

socket通信本质是进程间通信,所以需要快速定位进程,这时就有了端口,端口标识进程的唯一性

简单来说,IP唯一标识一台主机,端口唯一标识主机上的一个进程.所以IP+端口能在多台主机里唯一确定一个进程.

>  端口也是标识进程的,和PID的标识场景不同
>
> 协议确定,端口也会确定,协议是一种"约定",比如http协议,就确定了端口就是80,https协议就是443

## 网络字节序

机器有大小端,跨主机通信时得保证大端机传给小端机不出问题,反过来也一样,所以要解决这个问题.

最直接的就是订协议,但是我们并没有采用这种方案.比如在协议字段里加一个字段表示大小端,但是这也是一个数据,如果这个数据发送到另一台主机上,两台主机的大小端不同,那怎么去解析出这个字段呢?这就是蛋生鸡还是鸡生蛋的问题了.当然要解决肯定也是能解决的,比较协议的规则是我们定的.

最后采用的是,TCP/IP规定的网络字节序:**全部按大端发,小端要发就先转成大端再发,这就是网络字节序**.主机序列转网络序列的函数就是解决这种问题,保证发过去的数据的准确性.

## 绑定端口和IP

linux里有个函数叫做bind,将一个套接字和一些属性绑定,这些属性在sockaddr这个结构里,下面的bind接口的介绍里有详细的信息.

bind的本质是将端口私有化的过程,一个进程调用bind绑定一个端口,那这个端口就是这个进程的了

- 服务器(server)要绑定吗?要不要明确绑定?

要绑定,也需要明确绑定.服务器和客户端是1:n的关系,即一个服务器连接多个客户端,所以服务器应该尽可能的"暴露"自己的端口让客户端来连,而且服务器是稳定的,这个稳定指的是端口一般不怎么改变

- 客户端(client)要绑定吗?要不要明确绑定?

客户端不是必须是哪一个端口,只需要有端口就行了







所以客户端一般不自己bind,而是交给了操作系统,操作系统帮我们查找端口,主要也是因为服务器需要尽可能把自己暴露出去,因为很多人找一个服务器,而客户端不需要,哪有别人的浏览器来访问你的浏览器

如果自己明确绑定一个明确端口,如果有多个进程,进程间端口冲突那客户端直接启动不了了.

两者的区别是因为一个频繁启动,一个启动了基本不会关机,很多服务器从通电开始会一直用到报废(24小时不间断的跑)

> 域名和端口是稳定的,一般不怎么变化







read读键盘读到什么结束?是否会读入'\n'   (cout以空格为间隔符,所以读一段可能要读多次,read一次读完,效率跟高)



# sockaddr结构

虽然套接字有很多种,但是linux的socket设计者想设计一种统一的接口,所以设计了sockaddr这种数据结构

判断这个结构的前十六个字节就知道是哪种套接字,用C语言的方式实现了

> 设计这套网络接口的时候是没有void*的,所以是历史原因的.



# SOCK常见接口使用

## socket

创建套接字

```c
NAME
       socket - create an endpoint for communication

SYNOPSIS
       #include <sys/types.h>          /* See NOTES */
       #include <sys/socket.h>

       int socket(int domain, int type, int protocol);
RETURN VALUE
       On  success,  a  file descriptor for the new socket is returned.  On error, -1 is returned, and errno is set	appropriately.
```

参数:

- domain:协议域,或者说协议家族即你想使用哪种协议,一般是AF_INET,表明底层用的是IPV4的协议
- type:套接字种类,常用的是SOCK_STREAM和SOCK_DGRAM
- protocol:指定哪种协议,比如是UDP还是TCP,一般默认0,默认的0会采用与type类型对应的协议.

返回值:

- 成功返回一个文件描述符,失败返回一个-1.

例子:

```cpp
#include <iostream>
#include<sys/types.h>
#include<sys/socket.h>


int main()
{
  //创建套接字
  int sock=socket(AF_INET,SOCK_DGRAM,0);
  if(sock==-1)
  {
    std::cout<<"socker err\n";
    return 1;
  }
  
  std::cout<<"socket: "<<sock<<"\n";
  return 0;
}
```

![image-20220926195624443](https://pic-1304888003.cos.ap-guangzhou.myqcloud.com/img/image-20220926195624443.png)

## bind

让socket创建的套接字与IP和端口强相关,也可以说关联IP和端口

```c
NAME
       bind - bind a name to a socket

SYNOPSIS
       #include <sys/types.h>          /* See NOTES */
       #include <sys/socket.h>

       int bind(int sockfd, const struct sockaddr *addr,
                socklen_t addrlen);
RETURN VALUE
       On success, zero is returned.  On error, -1 is returned, and errno is set appropriately.
```

参数:

- sockfd:创建的套接字的文件描述符
- addr:输入性参数,填入sockaddr结构

```c
struct sockaddr_in {
  __kernel_sa_family_t	sin_family;	/* Address family		*/
  __be16		sin_port;	/* Port number			*/
  struct in_addr	sin_addr;	/* Internet address		*/

  /* Pad to size of `struct sockaddr'. */
  unsigned char		__pad[__SOCK_SIZE__ - sizeof(short int) -
			sizeof(unsigned short int) - sizeof(struct in_addr)];
};
struct in_addr {
	__be32	s_addr;
};
```

上面的sin_family表示协议家族,sin_port表示端口,sin_addr表示IP, struct in_addr里的s_addr是一个__be32类型,是一个无符号的32位整数

> ![image-20220927100647414](https://pic-1304888003.cos.ap-guangzhou.myqcloud.com/img/image-20220927100647414.png)

- addrlen:

返回值:

- 成功返回0

> 普通模式下q+:查看vim敲过的历史命令

例子:

```cpp
#include <iostream>
#include<sys/types.h>
#include<sys/socket.h>
#include<unistd.h>
#include <cstring>
#include <netinet/in.h> // 使用struct sockaddr_in所需的头文件



#define PORT 8081
int main()
{
  //创建套接字
  int sock=socket(AF_INET,SOCK_DGRAM,0);
  if(sock==-1)
  {
    std::cout<<"socker err\n";
    return 1;
  }
  
  std::cout<<"socket: "<<sock<<"\n";

  //bind绑定端口
  struct sockaddr_in local;
  memset(&local,0,sizeof(local));
  local.sin_family=AF_INET;
  //网络端口会以源端口的方式发送给对面,8081是主机序列,发过去得是网络序列,所以得转换一下
  local.sin_port=htons(PORT);
  //IP端口可以用4个字节保存,[0-255][0-255][0-255][0-255],点分十进制
  //云服务器建议不要绑定一个固定的IP,因为你看到的服务器的IP可能是腾讯模拟出来给你的,INADDR_ANY
  local.sin_addr.s_addr=htonl(INADDR_ANY);//绑定你机器上的所有IP,凡是发到我这个端口上的我全接受

  if(bind(sock,(struct sockaddr*)&local,sizeof(local))<0)
  {
    std::cout<<"bind err\n";
    return 2;
  }

  //for(;;)//服务器启动后要做事了
  //{
  //
  //}

  return 0;
}
```



## recvfrom

服务器跑起来后要接收数据,这个就是接收数据的接口

```c
NAME
       recv, recvfrom, recvmsg - receive a message from a socket

SYNOPSIS
       #include <sys/types.h>
       #include <sys/socket.h>

       ssize_t recv(int sockfd, void *buf, size_t len, int flags);

       ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags,
                        struct sockaddr *src_addr, socklen_t *addrlen);//这个接口

       ssize_t recvmsg(int sockfd, struct msghdr *msg, int flags);
RETURN VALUE
       These  calls  return the number of bytes received, or -1 if an error occurred.  In the event of an error, errno is set to indicate the error.  The return value will be 0 when the peer has performed an orderly shutdown.
```

参数:

- sockfd:从哪个套接字读数据
- buf:读的数据放在哪
- len:一次读多少(buf和len这两个参数其实就是用户缓冲区)
- flags:默认设为0表示阻塞读取.
- src_addr:输入输出型参数.输入时是获取对端socket信息的缓冲区,输出时是对端socket信息
- addrlen:输入输出型参数,输入时是长度缓冲区,输出时是对端socket信息的长度

返回值:

- 成功返回收到的字节数,失败返回-1.

> recv(int sockfd, void *buf, size_t len, int flags)==recvfrom(int sockfd, void *buf, size_t len, int flags,nullptr,nullptr);

例子:

```cc
//这里已经创建了套接字,并且绑定了端口  
// 服务器接收和发送信息 
char message[1024];
for(;;)
{
    struct sockaddr_in peer;
    socklen_t len=sizeof(peer);
    ssize_t s=recvfrom(sock,message,sizeof(message)-1,0,(struct sockaddr*)&peer,&len);//接收数据
    if(s>0)
    {
        message[s]='\0';
        std::cout<<"client#: "<<message<<"\n";
        sendto(sock,message,strlen(message),0,(struct sockaddr*)&peer,len);//把收到的数据往回写
    }
    else 
    {
        //TODO
    }

}
```

## sendto

上面的recvfrom是收数据,这个接口是把数据写回去

```c
NAME
       send, sendto, sendmsg - send a message on a socket

SYNOPSIS
       #include <sys/types.h>
       #include <sys/socket.h>

       ssize_t send(int sockfd, const void *buf, size_t len, int flags);

       ssize_t sendto(int sockfd, const void *buf, size_t len, int flags,
                      const struct sockaddr *dest_addr, socklen_t addrlen);

       ssize_t sendmsg(int sockfd, const struct msghdr *msg, int flags);
RETURN VALUE
       On success, these calls return the number of characters sent.  On error, -1 is returned, and errno is set appropriately.
```

参数:

- sockfd:通过哪个套接字发
- buf:发的数据的起始地址
- len:发的数据的长度(单位是字节)
- flags:调用方式的标志位,一般设为0,表示阻塞
- dest_addr:这个解决的是发给谁的问题,这个指针指向的结构体则包含了对方的信息,比如端口号和IP
- addrlen:dest_addr的长度

返回值:

- 成功返回发送的字节数,失败返回-1

例子:

```cpp
//这里已经创建了套接字,并且绑定了端口  
// 服务器接收和发送信息 
//下面的sock指的是服务器创建的套接字
char message[1024];
for(;;)
{
    struct sockaddr_in peer;
    socklen_t len=sizeof(peer);
    ssize_t s=recvfrom(sock,message,sizeof(message)-1,0,(struct sockaddr*)&peer,&len);//接收数据
    if(s>0)
    {
        message[s]='\0';
        std::cout<<"client#: "<<message<<"\n";
        sendto(sock,message,strlen(message),0,(struct sockaddr*)&peer,len);//把收到的数据往回写
    }
    else 
    {
        //TODO
    }

}
```

## inet_addr

把一个字符串风格的点分十进制IP地址转成大端发出去,解决大小端的问题.这里做了两件事:**转换点分十进制的IP字符串和把转换后的序列自动转成网络序列**

```c
NAME
       inet_aton,  inet_addr, inet_network, inet_ntoa, inet_makeaddr, inet_lnaof, inet_netof - Internet address manipula‐
       tion routines

SYNOPSIS
       #include <sys/socket.h>
       #include <netinet/in.h>
       #include <arpa/inet.h>

       in_addr_t inet_addr(const char *cp);

```

参数:

- cp:点分十进制的IP地址字符串

返回值:

- 返回值是一个无符号整数,IP地址大端形式的十进制数字

返回值的验证:

```cpp
#include<iostream>
#include<sys/types.h>
#include<sys/socket.h>
#include <arpa/inet.h>
#include<typeinfo>
using namespace std;


int main()
{
  auto e=inet_addr("192.168.0.1");
  cout<<e<<endl;
  std::cout<<typeid(e).name()<<"\n";


  return 0;
}
```

运行结果:

![image-20220927204759561](https://pic-1304888003.cos.ap-guangzhou.myqcloud.com/img/image-20220927204759561.png)

运行解释:

![image-20220927204608211](https://pic-1304888003.cos.ap-guangzhou.myqcloud.com/img/image-20220927204608211.png)

## inet_ntoa

传入struct in_addr这个类型,返回对应的字符串类型的ip

```c
NAME
       inet_aton,  inet_addr, inet_network, inet_ntoa, inet_makeaddr, inet_lnaof, inet_netof - Internet address manipula?
       tion routines

SYNOPSIS
       #include <sys/socket.h>
       #include <netinet/in.h>
       #include <arpa/inet.h>


        char *inet_ntoa(struct in_addr in);
```

参数:

-  in:struct sockaddr_in里有个属性是sin_addr,sin_addr也是个结构体,sin_addr里有个元素叫s_addr,s_addr的类型就是struct in_addr

返回值:

返回 in对应的IP的字符串,且是点分十进制的形式



## htonl

和inet_addr一样,也是解决大小端问题

```c
NAME
       htonl, htons, ntohl, ntohs - convert values between host and network byte order

SYNOPSIS
       #include <arpa/inet.h>

       uint32_t htonl(uint32_t hostlong);
```

h:host,本地主机

to:转

n:net网络

l:unsigned long,合起来就是本地主机转网络.

保证主机发到另一台主机数据的是大端形式,即准确性,而不是"倒过来"的.

参数:

- hostlong:一个无符号的32位整数

返回值:

传入的无符号32位整数的网络序列(大端形式的序列)的十进制形式

验证:

```cpp
#include<iostream>
#include<sys/types.h>
#include<sys/socket.h>
#include <arpa/inet.h>
#include<typeinfo>
using namespace std;


int main()
{
  //auto e=inet_addr("192.168.0.1");
  auto e=htonl(0x00000001);
  cout<<e<<endl;
  std::cout<<typeid(e).name()<<"\n";

  return 0;
}
```

效果:

![image-20220927210644000](https://pic-1304888003.cos.ap-guangzhou.myqcloud.com/img/image-20220927210644000.png)

解释:

![image-20220927210447825](https://pic-1304888003.cos.ap-guangzhou.myqcloud.com/img/image-20220927210447825.png)

## listen

监听套接字,让这个套接字从主动链接别人变为被链接

```c
NAME
       listen - listen for connections on a socket

SYNOPSIS
       #include <sys/types.h>          /* See NOTES */
       #include <sys/socket.h>

       int listen(int sockfd, int backlog);NAME
       socket - create an endpoint for communication
RETURN VALUE
       On success, zero is returned.  On error, -1 is returned, and errno is set appropri‐ately.
```

参数:

- sockfd:套接字
- backlog:暂不作解释

返回值:

- 成功返回0,失败返回-1

## accept

获取新链接,返回值是新链接的文件描述符

```c
NAME
       accept, accept4 - accept a connection on a socket

SYNOPSIS
       #include <sys/types.h>          /* See NOTES */
       #include <sys/socket.h>

       int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
RETURN VALUE
       On success, these system calls return a nonnegative integer that  is  a  descriptor for the accepted socket.  On error, -1 is returned, and errno is set appropriately.

```

参数:

- sockfd:被监听的套接字
- addr:对端套接字的地址(输入输出型参数,调用后会填充相关信息,我们就知道是谁连的服务器,也就知道了客户端的IP和端口号等)
- addrlen:addr(缓冲区)的长度

返回值:

- 成功获得一个套接字,反之返回-1

## connect

客户端通过connect链接对端服务器

```c
NAME
       connect - initiate a connection on a socket

SYNOPSIS
       #include <sys/types.h>          /* See NOTES */
       #include <sys/socket.h>

       int connect(int sockfd, const struct sockaddr *addr,
                   socklen_t addrlen);

RETURN VALUE
       If the connection or binding succeeds, zero is returned.  On error, -1  is  returned,  and errno is set appropriately.

```

参数:

- sockfd:套接字
- addr:指向的结构体里填写对端服务器的IP,端口等信息
- addrlen:addr的长度

返回值:

- 调用成功返回0,失败返回-1.

##  bzero

把s指向的那块空间置0.类似于memset

```c
NAME
       bzero - write zero-valued bytes

SYNOPSIS
       #include <strings.h>

       void bzero(void *s, size_t n);
```

参数:

- s:指向要置0空间的指针
- n:空间的大小,即从s开始后面n个字节都会被置0

返回值:

空

# UDP(demo)

## 铺垫

- bind绑定就是将主机的IP,端口,协议家族等写入特定的fd标定的文件中

- 客户端要绑定吗?

要绑定,但不用明确绑定,不需要主动bind,sendto的时候OS会帮我们绑定

- 为什么服务器要明确bind?

client:server=n:1,服务器需要尽可能暴露自己(IP和PORT,PORT一般被隐藏)给客户端,且是稳定的,即不轻易改变,服务器端口一旦改变客户端可能请求不到服务器,所以要求服务器"稳定".

- 为什么客户端不用主动绑定?

客户端不对外提供服务,不用被外界所熟知,不用被熟知,就表示如果我们手动bind,bind就占据了一个固定的端口号,可能导致别的客户端因端口号被占用启动不了.又因为客户端一般是多个且客户端会频繁启动,所以启动次数比较多,启动失败的概率就增加了.

-  客户端需要端口吗?

client(客户端)也必须有端口,不然服务器不知道具体找谁,  客户端不必须绑定某一个端口,有端口就行,所以客户端的端口由OS帮我们随机查找和分配

> bind的本质是将port私有化,上面的原因是由客户端和服务端的应用特性决定的

## 代码

### Makefile文件

一次性生成多个可执行文件用all

```makefile
.PHONY:all
all:udp_client udp_server #一次生成多个文件

udp_client:udp_client.cc
	g++ -o  $@ $^ -std=c++11

udp_server:udp_server.cc 
	g++ -o $@ $^ -std=c++11


.PHONY:clean
clean:
	rm -f udp_client udp_server
```

### udp_client.cc

![image-20221001002444896](https://pic-1304888003.cos.ap-guangzhou.myqcloud.com/img/image-20221001002444896.png)

> 这几个过程比较简略,实际上这个demo更加侧重于熟悉接口,比如recvfrom和sendto

```cc
#include<iostream>
#include<sys/types.h>
#include<sys/socket.h>
#include <unistd.h>
#include<cstring>
#include<string>
#include<netinet/in.h>
#include<arpa/inet.h>

//Usage:
//    ./*** desc_ip desc_port

void Usage(std::string proc)
{
  std::cout<<"Usage: \n\t"<<proc<<" desc_ip desc_port\n";
}

int main(int argc,char* argv[])
{
  if(argc!=3)
  {
    Usage(argv[0]);
    return -1;
  }
  int sock=socket(AF_INET,SOCK_DGRAM,0);
  if(sock<0)
  {
    std::cout<<"socket err\n";
    return 1;
  }
  
  // 客户端不需要手动bind,sendto的时候os会自动随机的给客户端绑定端口号
  
  struct sockaddr_in desc;
  memset(&desc,0,sizeof(desc));
  desc.sin_family=AF_INET;
  desc.sin_port=htons(atoi(argv[2]));//端口需要经过网络传输给到对方,所以需要保证是大端发过去的,简单说就是解决两台主机大小端不同的问题
  desc.sin_addr.s_addr=inet_addr(argv[1]);//与上面同理
  //填入目标主机的信息
  char buffer[1024];
  for(;;)
  {
    std::cout<<"Please Enter# ";
    fflush(stdout);
    buffer[0]=0;//C语言方式的初始化字符串
    ssize_t size=read(0,buffer,sizeof(buffer)-1);
    if(size>0)
    {
      buffer[size-1]='\0';
      //std::cout<<"echo# "<<buffer<<"\n";
      sendto(sock,buffer,strlen(buffer),0,(struct sockaddr*)&desc,sizeof(desc));
 
      struct sockaddr_in peer;
      socklen_t len=sizeof(peer);//注意其类型
      //ssize_t s=recvfrom(sock,buffer,sizeof(buffer)-1,0,(struct sockaddr*)&peer,&len);//peer和len暂时不用
      ssize_t s=recvfrom(sock,buffer,sizeof(buffer)-1,0,(struct sockaddr*)&peer,&len);
      if(s>0)
      {
        buffer[s]='\0';
        std::cout<<"echo# "<<buffer<<"\n";
      }
    }
  }
  
  close(sock);
  return 0;
}

```

### udp_server.cc

![image-20221001002332411](https://pic-1304888003.cos.ap-guangzhou.myqcloud.com/img/image-20221001002332411.png)

```cc
#include <iostream>
#include<sys/types.h>
#include<sys/socket.h>
#include<unistd.h>
#include <cstring>
#include <netinet/in.h> // 使用struct sockaddr_in所需的头文件


void Usage(std::string  proc)
{
  std::cout<<"Usage\n\t"<<proc<<" local_port\n";
}

//#define PORT 8081
int main(int argc,char* argv[])
{
  if(argc!=2)
  {
    Usage(argv[0]);
    return -1;
  }
  //创建套接字
  int sock=socket(AF_INET,SOCK_DGRAM,0);
  if(sock==-1)
  {
    std::cout<<"socker err\n";
    return 1;
  }
  
  std::cout<<"socket: "<<sock<<"\n";


  //bind绑定端口
  struct sockaddr_in local;
  memset(&local,0,sizeof(local));
  local.sin_family=AF_INET;
  //网络端口会以源端口的方式发送给对面,8081是主机序列,发过去得是网络序列,所以得转换一下
  local.sin_port=htons(atoi(argv[1]));
  //IP端口可以用4个字节保存,[0-255][0-255][0-255][0-255],点分十进制
  //云服务器建议不要绑定一个固定的IP,因为你看到的服务器的IP可能是腾讯模拟出来给你的,INADDR_ANY
  local.sin_addr.s_addr=htonl(INADDR_ANY);//绑定你机器上的所有IP,凡是发到我这个端口上的我全接受


  if(bind(sock,(struct sockaddr*)&local,sizeof(local))<0)
  {
    std::cout<<"bind err\n";
    return 2;
  }

  // 服务器接收和发送信息 
  char message[1024];
  for(;;)
  {
    struct sockaddr_in peer;
    socklen_t len=sizeof(peer);
    ssize_t s=recvfrom(sock,message,sizeof(message)-1,0,(struct sockaddr*)&peer,&len);
    if(s>0)
    {
        message[s]='\0';
        std::cout<<"client#: "<<message<<"\n";
        std::string echo_message=message;
        echo_message+=" [server]";
        sendto(sock,echo_message.c_str(),echo_message.size(),0,(struct sockaddr*)&peer,len); // 前面recv已经知道了谁发的
    }
    else
    {
      //TODO
    }
  
  }

  return 0;
}
```



## 运行效果

![服务器和客户端发消息](https://pic-1304888003.cos.ap-guangzhou.myqcloud.com/img/%E6%9C%8D%E5%8A%A1%E5%99%A8%E5%92%8C%E5%AE%A2%E6%88%B7%E7%AB%AF%E5%8F%91%E6%B6%88%E6%81%AF.gif)

## netstat -nlup查看当前的udp服务

n表示尽可能用数字显示

l表示list

u表示udp
p表示process,即进程

```c
netstat -nlup
```

这条命令可以查看当前的udp服务,在这可以看我们的服务器是否启动



![image-20220927213331014](https://pic-1304888003.cos.ap-guangzhou.myqcloud.com/img/image-20220927213331014.png)



## lszrz工具

如果我们要把写好的可执行文件发给别人怎么办?这时就需要通过lszrz工具了

```shell
sz -E udp_client #上传udp_client到本地
rz   # 上传本地的文件到云服务器
```

## 模拟简易XShell(玩具版)

在上面的代码中稍作修改即可.

上面实现的是客户端发数据然后服务器收数据再回写给客户端,想改变其功能只需更改服务器处理数据的操作,收到的数据是一个字符串.现在我们规定客户端发给服务器端的字符串是一条命令,服务器端收到字符串后当成命令处理,如执行这个命令,执行命令后打印的信息回显给客户端,这样就实现了客户端通过命令访问服务器上的数据,并且得到的结果会显示在客户端.这不就是个玩具版的XShell.

实现这个借助popen函数和C的文件读写



### udp_client.cc

```cpp
#include<iostream>
#include<sys/types.h>
#include<sys/socket.h>
#include <unistd.h>
#include<cstring>
#include<string>
#include<netinet/in.h>
#include<arpa/inet.h>

//Usage:
//    ./*** desc_ip desc_port

void Usage(std::string proc)
{
  std::cout<<"Usage: \n\t"<<proc<<" desc_ip desc_port\n";
}

int main(int argc,char* argv[])
{
  if(argc!=3)
  {
    Usage(argv[0]);
    return -1;
  }
  int sock=socket(AF_INET,SOCK_DGRAM,0);
  if(sock<0)
  {
    std::cout<<"socket err\n";
    return 1;
  }
  
  // 客户端不需要手动bind,sendto的时候os会自动随机的给客户端绑定端口号
  
  struct sockaddr_in desc;
  memset(&desc,0,sizeof(desc));
  desc.sin_family=AF_INET;
  desc.sin_port=htons(atoi(argv[2]));//端口需要经过网络传输给到对方,所以需要保证是大端发过去的,简单说就是解决两台主机大小端不同的问题
  desc.sin_addr.s_addr=inet_addr(argv[1]);//与上面同理
  //填入目标主机的信息
  char buffer[1024];
  for(;;)
  {
    std::cout<<"Please Enter# ";
    fflush(stdout);
    buffer[0]=0;//C语言方式的初始化字符串
    ssize_t size=read(0,buffer,sizeof(buffer)-1);//如果要处理特殊字符如/b就得一个字符一个字符判断
    if(size>0)
    {
      //buffer[size-1]='\0';
      buffer[size]='\0';
      //std::cout<<"echo# "<<buffer<<"\n";
      sendto(sock,buffer,strlen(buffer),0,(struct sockaddr*)&desc,sizeof(desc));
 
      struct sockaddr_in peer;
      socklen_t len=sizeof(peer);//注意其类型
      //ssize_t s=recvfrom(sock,buffer,sizeof(buffer)-1,0,(struct sockaddr*)&peer,&len);//peer和len暂时不用
      ssize_t s=0;
      
      std::cout<<"echo# ";
      while(s=recvfrom(sock,buffer,sizeof(buffer)-1,0,(struct sockaddr*)&peer,&len))//循环读取
      {
        if(s==0)
        {
          break;
        }
        buffer[s]='\0';
        std::cout<<buffer;
        //std::cout<<"debug:"<<s<<"\n";  
      }
    }
  
  }
  close(sock);
  return 0;
}
```



### udp_server.cc

```cpp
#include <iostream>
#include<sys/types.h>
#include<sys/socket.h>
#include<unistd.h>
#include <cstring>
#include <netinet/in.h> // 使用struct sockaddr_in所需的头文件


void Usage(std::string  proc)
{
  std::cout<<"Usage\n\t"<<proc<<" local_port\n";
}

//#define PORT 8081
int main(int argc,char* argv[])
{
  if(argc!=2)
  {
    Usage(argv[0]);
    return -1;
  }
  //创建套接字
  int sock=socket(AF_INET,SOCK_DGRAM,0);
  if(sock==-1)
  {
    std::cout<<"socker err\n";
    return 1;
  }
  
  std::cout<<"socket: "<<sock<<"\n";


  //bind绑定端口
  struct sockaddr_in local;
  memset(&local,0,sizeof(local));
  local.sin_family=AF_INET;
  //网络端口会以源端口的方式发送给对面,8081是主机序列,发过去得是网络序列,所以得转换一下
  local.sin_port=htons(atoi(argv[1]));
  //IP端口可以用4个字节保存,[0-255][0-255][0-255][0-255],点分十进制
  //云服务器建议不要绑定一个固定的IP,因为你看到的服务器的IP可能是腾讯模拟出来给你的,所以采用INADDR_ANY
  local.sin_addr.s_addr=htonl(INADDR_ANY);//绑定你机器上的所有IP,凡是发到我这个端口上的我全接受


  if(bind(sock,(struct sockaddr*)&local,sizeof(local))<0)
  {
    std::cout<<"bind err\n";
    return 2;
  }

  // 服务器接收和发送信息 
  char message[1024];
  for(;;)
  {
    struct sockaddr_in peer;
    socklen_t len=sizeof(peer);
    ssize_t s=recvfrom(sock,message,sizeof(message)-1,0,(struct sockaddr*)&peer,&len);
    if(s>0)
    {
        FILE* in=popen(message,"r");//执行命令
        if(in==nullptr)
        {
          continue;
        }
        std::string echo_string;
        char buffer[128];
        //char* tmp=nullptr;
        while(fgets(buffer,sizeof(buffer)-1,in))//
        {
          //std::cout<<"debug:"<<tmp<<"\n";
         // echo_string+=buffer;
        // sendto(sock,buffer,sizeof(buffer)-1,0,(struct sockaddr*)&peer,len); // 前面recv已经知道了谁发的
          //std::cout<<"debug:"<<strlen(buffer)<<"\n";
         sendto(sock,buffer,strlen(buffer),0,(struct sockaddr*)&peer,len); // 循环发数据,保证数据不会出现截断(丢失)
        }
        //std::cout<<"debug:读完了\n";
        //这边读完文件后需要再发送一个空让对端的循环退出(不然对端一直阻塞在那)
        sendto(sock,echo_string.c_str(),echo_string.size(),0,(struct sockaddr*)&peer,len); // 前面recv已经知道了谁发的
        pclose(in);
      //拿到了数据就得要处理数据了
      //下面的是第一个版本,采用创建子进程+进程替换的方式实现玩具XShell,但是有一个bug:发送的数据大于几千字节后会出现数据截断问题
      //message[s]='\0';
      //char* command[64];
      //int index=0;
      //command[index]=strtok(message," ");
      //while(1)
      //{
      //  index++;
      //  command[index]=strtok(NULL," ");
      //  if(command[index]==NULL)
      //  {
      //    break;
      //  }
      //}
      //if(fork()==0)
      //{
      //  execvp(command[0],command);
      //  std::cerr<<"client# "<<command<<"\n";
      //  exit(3);
      //}

      //message[s]='\0';
      //std::cout<<"client#: "<<message<<"\n";
      //std::string echo_message=message;
      //echo_message+=" [server]";
      //sendto(sock,echo_message.c_str(),echo_message.size(),0,(struct sockaddr*)&peer,len); // 前面recv已经知道了谁发的
      //sendto(sock,command[0],strlen(command[0]),0,(struct sockaddr*)&peer,len); // 前面recv已经知道了谁发的
    }   
    else
    {
      //TODO
    }
  
  }

  return 0;
}
```

### 运行效果

在客户端访问了服务器的数据

![](https://ccl-1304888003.cos.ap-guangzhou.myqcloud.com/img/%E6%9C%8D%E5%8A%A1%E5%99%A8%E5%92%8C%E5%AE%A2%E6%88%B7%E7%AB%AF%E5%AE%9E%E7%8E%B0%E7%AE%80%E6%98%93XShell.gif)



## 小结

服务器创建套接字->绑定端口->然后收数据->再把数据处理完发回客户端.这个过程前面几步都是套路化的,最后一步处理数据是关键,毕竟收到一个数据再发回去没有意义...所以根据场景不同数据的处理也不同.

客户端创建套接字,不用手动绑定,OS会帮我们绑定,直接收发数据.

# TCP(demo)

> 小技巧:
>
> sed可在命令行里往文件中插入内容
>
> vim:w,e,b 让单词按光标往后移动
>
> vim->      :bn打开下一个文件    :bp打开上一个文件    :ls列出当前所有的文件
>
> vim->    ')'和'('用来快速跳出函数,%找配对括号也可以

## 铺垫

下面写了7个文件,代码量也比较大,所以在这做一些注释

| 文件名         | 作用                                                         |
| -------------- | ------------------------------------------------------------ |
| Makefile       | 自动化编译                                                   |
| tcp_server.hpp | 服务端创建套接字,绑定,创建链接,获取链接,再通过链接处理业务逻辑 |
| tcp_client.hpp | 客户端创建套接字,链接服务端,处理业务逻辑                     |
| handler.hpp    | 处理业务逻辑,里面包含四个函数(版本),分别是单进程,多进程,多线程,线程池版本,server.cc通过函数指针调用这四个版本 |
| ThreadPool.hpp | 单例模式的线程池                                             |
| server.cc      | main函数,调用上面的函数                                      |
| client.cc      | main函数,调用上面的函数                                      |

注意:业务逻辑有四个版本,理论上在server.cc更改函数指针就可以实现不同版本的业务逻辑,但是写的时候封装的没有那么好,所以直接替换函数指针可能有一点问题,比如现在写好的是版本4,要是想换成版本3,就得注意在哪关闭套接字这种细节.

## Makefile文件

一次编译多个文件用all

```makefile
.PHONY:all
all:tcp_client tcp_server 

tcp_client:client.cc
	g++ -o  $@ $^ -std=c++11 -g

tcp_server:server.cc 
	g++ -o $@ $^ -std=c++11 -lpthread -g


.PHONY:clean
clean:
	rm -f tcp_client tcp_server
```

## tcp_server.hpp

![image-20221005143210079](https://ccl-1304888003.cos.ap-guangzhou.myqcloud.com/img/image-20221005143210079.png)

- 代码里的listen_sock和fd,这两个都是套接字,举个例子,你去外面吃饭,餐馆门口有个叫做李三的小二给你带进店里,店里的小二李四给你提供服务,所以实际提供服务的是这里的李四(店里的小二).这里的listen_sock是为了创建链接创建的套接字,而fd是真正业务处理时用到的套接字,所以读写数据都是通过fd,又因为tcp读写数据都是流式的,所以通过write和read读写文件即可,不需要像udp那样借助recvfrom和sendto函数.
- read和write这两个函数和套接字扯上关系时会变成阻塞的,如果没处理好可能导致一端一直阻塞

```cpp
#pragma once
#include <iostream>
#include<unistd.h>
#include<sys/types.h>
#include<sys/socket.h>
#include<netinet/in.h>
#include <strings.h>
#include<arpa/inet.h> //inet_ntoa


namespace ns_tcpserver
{
  typedef void(*handler_t)(int);//函数指针,作回调函数

  const int backlog=5;
  class TcpServer
  {
    private:
      uint16_t port;
      int listen_sock;
    public:
      TcpServer(uint16_t _port)
        :port(_port)
        ,listen_sock(-1)//获取新链接
       {}
      void InitServer()
      {
        //创建套接字
        listen_sock=socket(AF_INET,SOCK_STREAM,0);
        if(listen_sock==-1)
        {
          std::cerr<<"listen_sock err\n";
          exit(1);
        }
         
        
        struct sockaddr_in local;
        bzero(&local,sizeof(local));//类似于memset清0
        local.sin_family=AF_INET;
        local.sin_port=htons(port);//这里必须要主机转网络,不然的话可能传入的是8081端口,绑到别的端口上去了,然后用telnet测试的时候一直报错....
        local.sin_addr.s_addr=htonl(INADDR_ANY);//主机转网络
        
        //bind绑定
        if(bind(listen_sock,(struct sockaddr*)&local,sizeof(local))<0)
        {
          
          std::cerr<<"bind err\n";
          exit(2);
        }
        //监听,创建链接
        int lt=listen(listen_sock,backlog);//监听这个listen_sock,或者说让这个进程从主动链接变为被链接
       
        if(lt<0)
        {
          std::cerr<<"listen err\n";
          exit(3);
        }
        
        
        
      }
      //回调机制
      void Loop(handler_t handler)
      {
        while(true)
        {
          struct sockaddr_in peer;
          socklen_t len=sizeof(peer);
          //获取链接
          //accept负责把listen_sock带入餐馆,实际上提供服务的是这个返回值fd(餐馆里吃饭之类的服务由fd提供)
          int fd=accept(listen_sock,(struct sockaddr*)&peer,&len);
          if(fd<0)
          {
            std::cerr<<"accept err\n";
            //exit(4);获取链接失败最多是个警告,不可能直接让服务器退出
          }
          //debug验证fd和端口号等信息
          std::cout<<"debug:sock->"<<fd<<std::endl;
          uint16_t peer_port=ntohs(peer.sin_port);//网络转主机
          std::string peer_ip=inet_ntoa(peer.sin_addr);//四字节IP转为点分十进制风格的IP
          std::cout<<"debug: "<<peer_ip<<":"<<peer_port<<"\n";

          //处理链接
          handler(fd);
          
          //关闭链接
         // close(fd);
        }

      }
      ~TcpServer()
      {
        if(listen_sock>=0)
        {
          close(listen_sock);
        }
      }
  };
}
```

## tcp_client.hpp

![image-20221005143939589](https://ccl-1304888003.cos.ap-guangzhou.myqcloud.com/img/image-20221005143939589.png)

- 客户端要不要我们手动绑定?不用,OS会帮我们绑定.
- inet_addr这个函数是负责把点分十进制的IP字符串转成网络序列发出去,所以转换后的序列不要再使用htonl(~~改这个bug改了半天~~)

```cpp
#pragma once
#include <iostream>
#include <string>
#include <sys/types.h>
#include<sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <strings.h>
#include <string.h>


namespace ns_tcpclient
{
  class TcpClient
  {
    private:
      int sock;
      std::string desc_ip;//要访问的服务器IP
      uint16_t desc_port;//要访问的服务器的端口
      

    public:
      TcpClient(std::string _ip,uint16_t _port)
        :sock(-1)
        ,desc_ip(_ip)
        ,desc_port(_port)
      {}
      void InitTcpClient()
      {
        //创建套接字
        sock=socket(AF_INET,SOCK_STREAM,0);
        if(sock<0)
        {
          std::cerr<<"sock err\n";
          exit(1);
        }
        //client要不要bind?不用,OS会自动绑定
        
        //client要不要listen?不用处于监听状态(不需要等待别人来连)
        //client要不要accept?不需要(listen没有建立链接自然不用获取链接)
        
      }


      //client虽然不用accept获取链接,但需要connect链接服务器
      void Start()
      {
        //填写对端服务器的信息
        struct sockaddr_in svr;
        svr.sin_family=AF_INET;
        svr.sin_port=htons(desc_port);//别忘了主机转网络
        svr.sin_addr.s_addr=inet_addr(desc_ip.c_str());//这里不需要主机再转网络了,已经转好了,否则会出错
        socklen_t len=sizeof(svr);
        //发起链接请求
        if(connect(sock,(struct sockaddr*)&svr,len)==0)
        {
          std::cout<<"connect success...\n";
        }
        else 
        {
          std::cout<<"connect failed...\n";
        }

        //完成业务逻辑
        while(true)
        {
            char buffer[1024]={0};
            std::cout<<"请你输入# :";
            fflush(stdout);
            ssize_t s=read(0,buffer,sizeof(buffer)-1);
            //std::cout<<"s:"<<s<<"\n";//debug
            if(s>0)
            {
              buffer[s-1]='\0';//最后一个回车符
              //write(sock,buffer,sizeof(buffer));
              write(sock,buffer,strlen(buffer));
              //std::cout<<"已经write\n";//debug
              int rs=read(sock,buffer,sizeof(buffer)-1);
              //std::cout<<"rs:"<<rs<<"\n";//debug
              if(rs>0)
              {
                buffer[rs]='\0';
                std::cout<<buffer<<"\n";
              }
              else //那边循环一旦退出就读不到了这时提示服务器close
              {
                std::cout<<"server close...\n";
                break;
              }
            }
        }
      }

      ~TcpClient()
      {
        if(sock>=0)
        {
          close(sock);
        }
      }

  };

}
```

## handler.hpp

业务逻辑就是简单的"发消息",简单的读写文件,没加别的

- HandlerSock_V1:单进程的业务处理,缺陷:只能有一个客户端链接服务端,多个客户端虽然能链接上服务端,但是发消息会发生阻塞的情况

- HandlerSock_V2:多进程的业务处理,采用创建子进程的方式去完成业务处理,但是会出现一个问题,子进程处理业务时父进程要等待,此时父进程就不能做别的事了,所以我们得让父进程不等待子进程同时不会发生僵尸进程的问题.当然可以通过信号的方式实现父进程不等待进程.

  这里采用另外一种方式不等待子进程:父进程创建子进程,子进程创建孙子进程,创建完孙子进程后子进程直接退出,孙子进程被创建出来完成业务处理,因为孙子进程的父亲在孙子进程处理业务时退出了所以孙子进程变成了孤儿进程,会被1号进程领养,之后OS会帮我们回收这个孤儿进程.而子进程创建孙子进程后直接退出父进程直接就可以等待到子进程.这就保证了子进程被父进程等待,孙子进程完成业务后被OS处理,同时父进程也不会等待很久.

- HandlerSock_V3:多线程,创建进程的成本还是挺高的,所以采用多线程的方式,但是每个线程都能看到同一个套接字,所以我们在线程函数routine里面关闭套接字,同时因为套接字是形参且在栈上,可能线程还没启动,传进pthread_create的sock就被销毁了,导致传参的sock出现问题,所以得拷贝一个sock且开在堆上,保证传参的准确性.此外使用线程分离让主线程不必等待业务处理的线程

- HandlerSock_V4:线程池的业务处理,这个是效率最高的,因为多线程和多进程会出现一个问题:如果有成千上万个客户端那不是得创建成千上万个进程或线程,这显然是不合理的,所以采用线程池,保证限定创建的进程和线程个数.

```cpp
#pragma once 

#include<sys/wait.h>
#include "tcp_server.hpp"
#include <unistd.h>
#include <pthread.h>
#include "ThreadPool.hpp"

namespace ns_handler
{

  using namespace ns_tcpserver;

#define SIZE 1024
  void HandlerHelper(int sock)
  {
    while(true)
    {
      
      //std::cout<<"回调函数被调用,debug: "<<sock<<"\n";
      char buffer[SIZE];
      ssize_t s=read(sock,buffer,sizeof(buffer)-1);
     // std::cout<<"server已经读了\n";//debug
      if(s>0)
      {
        buffer[s]='\0';
        std::cout<<"client# "<<buffer<<"\n";
        std::string echo_string=buffer;
        if(echo_string=="quit")//客户端发过来的数据是quit直接退出
        {
          break;
        }
        echo_string+="[server say]";//用telnet测试时实际上的数据是\nbuffer\n[server],因为回车符也会被读进去,而不是想象中的buffer[server]
        write(sock,echo_string.c_str(),echo_string.size());

      }
      else if(s==0)//读完了数据
      {
        //读完了退出即可
        std::cout<<"client quit...\n";
        break;
      }
      else //出错了
      {
        std::cerr<<"read err\n";
        exit(5);
      }

   }

  }

  void HandlerSock_V1(int sock)
  {
    HandlerHelper(sock);
  }

  //多进程版
  void HandlerSock_V2(int sock)
  {
    if(fork()==0)
    {
      if(fork()==0)
      {
        //grandson
        HandlerHelper(sock);
      }
      //son
      exit(0);
    }
    //father
    waitpid(-1,nullptr,0);

  }

  void* routine(void* args)
  {
    pthread_detach(pthread_self());
    int sock=*(int*)args;
    HandlerHelper(sock);

    close(sock);
    return nullptr;
  }

  void HandlerSock_V3(int sock)
  {
    pthread_t tid;
    int* p=new int(sock);
    //pthread_create(&tid,nullptr,routine,&sock);//sock是局部变量,传地址出去处理业务时主线程直接退出局部变量也被销毁了,所以导致传参有问题
    pthread_create(&tid,nullptr,routine,p);
  }

  class Task
  {
    private:
      int sock;
    public:
      Task(){} //不能省
      Task(int _sock)
        :sock(_sock)
      {
      }
      void operator()()
      {
        std::cout<<"开始处理任务..."<<endl;
        HandlerHelper(sock);
        close(sock);
      }
      ~Task()
      {}

  };

  void HandlerSock_V4(int sock)
  {
       ThreadPool<Task>* instance=ThreadPool<Task>::get_instance(5);
       instance->PushTask(Task(sock));//客户端一链接马上获取链接调用这个函数处理数据,即会立即插入一个task,然后调用业务处理函数,所以一链接就会打印有任务,而不是我们发送消息再提示有任务

  }
}
```

## ThreadPool.hpp

单例模式的线程池,采用的 是条件变量+队列+单例模式.

值得注意的是routine是执行任务的函数,因为写在类里,所以必须是静态的,来消除this指针保证routine函数里只能有一个void*参数,多个参数在pthread_create调用时会调用失败.由于routine函数是静态的,导致只能调用静态的方法,所以得借助pthread_create传入this指针来调用相关接口,比如上锁,直接调用锁的接口会因为接口不是静态的报错,而采用this指针调用就没问题了.

```cpp
#pragma once 

#include<iostream>
#include<unistd.h>
#include<pthread.h>
#include<stdlib.h>
#include <time.h>
#include<queue>
using namespace std;

template<class T>
class ThreadPool
{
private:
  ThreadPool()
  {
    pthread_cond_init(&_cond,nullptr);
    pthread_mutex_init(&_lock,nullptr);
  }
  ThreadPool(const ThreadPool<T>&)=delete ;
  ThreadPool<T>& operator=(ThreadPool<T>&)=delete ;
  static ThreadPool<T>* instance;
public:
  static ThreadPool<T>* get_instance(int num)
  {
    
    static pthread_mutex_t mtx;
    pthread_mutex_init(&mtx,nullptr);
    if(instance==nullptr)
    {
      pthread_mutex_lock(&mtx);
      if(instance==nullptr)//双检查,经典单例
      {
        instance=new ThreadPool<T>();
        instance->ThreadPoolInit(num);
      }
      pthread_mutex_unlock(&mtx);
    }
    return instance;//别忘了返回,当时忘了报段错误找了半天
  }
  void Lock()
  {
    pthread_mutex_lock(&_lock);
  }
  void UnLock()
  {
    pthread_mutex_unlock(&_lock);
  }
  void Wait()
  {
    pthread_cond_wait(&_cond,&_lock);
  }
  void WakeUp()
  {
    pthread_cond_signal(&_cond);
  }
  bool IsEmpty()
  {
    return _tq.size()==0;
  }
  void PopTask(T* out)
  {
    *out=_tq.front();
    _tq.pop();
  }
  void PushTask(const T& t)
  {
    Lock();
    _tq.push(t);
    WakeUp();
    UnLock();
  }
  size_t Tqsize()
  {
    return _tq.size();
  }
  static void* Routine(void* args)//非static函数会有隐含的参数this指针,所以用static
  {
    
    pthread_detach(pthread_self());
      //因为static函数只能调用static方法或数据,但是我们需要访问lock这些所以需要this指针,此外lock相关操作也要进行封装
      ThreadPool* threadPool=(ThreadPool*)args;
      //看队列里有没有待处理的任务,没有的话就进行等待,有的话直接给处理了
      //_tq表示任务队列,由于不是线程安全的,得加锁
      while(true)
      {
      //这里必须一直在跑,不然创建的线程只会处理一次任务
          threadPool->Lock();
          while(threadPool->IsEmpty())//这里不能用if进行判断
          {
            cout<<"队列为空开始等待"<<endl;
            threadPool->Wait();
          }
          //走到这说明任务队列不为空
          T t;
          threadPool->PopTask(&t);//把任务取出来
          threadPool->UnLock();
          t();//不用在锁里面处理了,相当于线程的局部变量,被一个线程独有
 
      }
  }

 void ThreadPoolInit(int num)
  {
    for(int i=0;i<num;i++)
    {
      pthread_t tid;
      pthread_create(&tid,nullptr,Routine,(void*)this);
    }
  }
  ~ThreadPool()
  {
    pthread_cond_destroy(&_cond);
    pthread_mutex_destroy(&_lock);
  }


private:
  pthread_cond_t _cond;
  pthread_mutex_t _lock;
  queue<T> _tq;


};

template<class T>
ThreadPool<T>* ThreadPool<T>::instance=nullptr;
```



## server.cc

调用相关函数

```c++
#include "tcp_server.hpp" //提供网络链接功能
#include "handler.hpp" //提供网络sock的处理功能

void Usage(char* proc)
{
  std::cout<<"Usage\n\t"<<proc<<" port\n";
}



// ./server port 
int main(int argc,char* argv[])
{
  if(argc!=2)
  {
    Usage(argv[0]);
    return 0;
  }
  uint16_t port=atoi(argv[1]);
  ns_tcpserver::TcpServer* svr=new ns_tcpserver::TcpServer(port);
  
  svr->InitServer();
  svr->Loop(ns_handler::HandlerSock_V4);//传函数指针
  return 0;
}
```

## client.cc

调用相关函数

```cpp
#include "tcp_client.hpp"

//./tcp_client peer_ip peer_port

void Usage(char* proc)
{
  std::cout<<"Usage:\n\t"<<proc<<" svr_ip svr_port\n";
}
int main(int argc,char* argv[])
{
  if(argc!=3)
  {
    Usage(argv[0]);
    return 1;
  }

  std::string ip=argv[1];
  uint16_t port=atoi(argv[2]);
  ns_tcpclient::TcpClient cli(ip,port);

  cli.InitTcpClient();
  cli.Start();

  return 0;
}
```

## 运行结果

这里采用的是V4版本.

<img src="https://ccl-1304888003.cos.ap-guangzhou.myqcloud.com/img/tcp%E6%B6%88%E6%81%AF%E4%BA%92%E5%8F%91.gif" alt="tcp消息互发" style="zoom:67%;" />





## telnet监听服务器端口

> [( Linux书签（03）用linux telnet命令监测服务器端口号_](https://blog.csdn.net/itanping/article/details/96481668)

如果我们只写完了服务器,又想测试服务器的某个端口是否正常开启,可采用telnet工具进行测试.从下图可以看出我们利用telnet工具成功链接上了服务器,端口也是正常开启的.

<img src="https://ccl-1304888003.cos.ap-guangzhou.myqcloud.com/img/tcp%E6%B6%88%E6%81%AF%E4%BA%92%E5%8F%91_telnet.gif" alt="tcp消息互发_telnet" style="zoom: 50%;" />



## 小结

服务器创建套接字->服务器绑定端口->创建链接->获取链接->开始业务处理,这里就是收数据然后回写给客户端

客户端创建套接字->链接服务器->业务处理,这里的业务处理就是把键盘输入的数据发给服务器.

TCP读写数据是流式的,直接用write和read读写数据即可,也可以用recv和send接口.





















