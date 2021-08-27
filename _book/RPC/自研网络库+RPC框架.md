## 基础知识
### 框架设计理念（转）
好的基础架构的方式

1) 底层的组件都是开源的,不是自研的,例如RPC组件,日志组件,监控组件等,最好不要自研

2) 基础架构系统整体是自研的,即把RPC组件,日志组件和监控组件等组合在一起的系统是自研的,运维的主要工作在这个系统上

说下这样的好处

1) 基础架构的主要矛盾是可持续性和可迭代性,底层组件开源在很大程度上缓解了这个问题

2) 底层组件开源,对于新人和老人的融入几乎就是无缝的,可以很快衔接进来,可以极大的降低人才融入成本,这部分隐层的成本降低非常有价值

3) 基础架构在实际使用中的主要问题,也基本都是开源组件的使用和配置问题,这个在开源世界中几乎都有答案,这样就避免了闭源系统的1对多的难题,会极大缓解运维的压力

总体的设计原则是：在稳定的前提下，首先保证易用性，然后尽可能地提高性能。

### 基于 Go 的 I/O 多路复用 netpoller
Go 基于 I/O multiplexing 和 goroutine scheduler 构建了一个简洁而高性能的原生网络模型(基于 Go 的 I/O 多路复用 netpoller )，提供了 goroutine-per-connection 这样简单的网络编程模式。实现了让 开发者使用的是同步的模式去编写异步的逻辑
比如
- 基本并发模式 是：goroutine-per-connection
- 多线程同步使用的是：nonblockingPipe 没有使用eventfd (可能是为了兼容老版本)
- 底层用的是epoll + 非阻塞IO +ET
文章外网开源地址：[https://strikefreedom.top/go-netpoll-io-multiplexing-reactor](https://strikefreedom.top/go-netpoll-io-multiplexing-reactor)
[https://cloud.tencent.com/developer/article/1536759](https://cloud.tencent.com/developer/article/1536759)

###

### edu-mod 大致流程图
![https://www.processon.com/view/link/5f1671f45653bb7fd245a441](https://www.processon.com/view/link/5f1671f45653bb7fd245a441)
部门的框架是edu-mod + api-gateway 支撑起来的 属于大网关架构
### 理解trpc设计
#### trpc-cpp 网络层模型
##### 概述
在线程运行上，使用绑核减少线程切换和cache miss；网络收发包使用epoll ET和使用writev合并写减少系统调用；内存管理上使用tcmalloc/内存池/对象池，减少内存频繁分配/释放的性能开销；线程交互上使用无锁队列，减少通知消耗和锁损耗；协议操作避免大对象的内存拷贝等，以及一些关键路径（字符串操作、优化获取系统时间同步改异步、减少map查找
##### 经典的线程模型
Thread-Per-Connection：一个线程专门监听与客户端建立连接之后，启动一个专门的线程去处理
Pre-Thread：基于”Thread-Per-Connection“改进，不同之处在于，预先创建一个线程池（Connection-Thread-Pool），以提高响应速度。
Process-Per-Connection：将图中的线程换成进程即是。
Pre-Fork：基于“Process-Per-Connection”改进，与Pre-Thread同理采用进程池。
这类模型有个显著的优点，就是在Connection-Thread中可以随意阻塞，业务编程非常方便，如Apache Http Server中的PHP编程
长连接+连接复用 
one loop per thread : 基于epoll的多路复用 带来的问题是异步编程
##### RPC框架——与服务器编程不同 需同时支持客户端和服务端
客户端服务端分离模式：![](./客户端服务端分离模式.png)
客户端服务端合并模式：![](./客户端服务端合并模式.png)
异步操作的Continuation始终运行在当前的Reactor线程，从而一个异步任务的挂起和恢复始终在同一个线程中。在很多场景可以做到无锁编程。
各个工作线程（Reactor）地位对等，只需要根据CPU核数，配置其数量即可。可以充分利用CPU（在192核机器上测试，CPU可以满负载运行
优点：
消除了线程间的通知唤醒，避免了大量的上下文切换。
一个任务始终在一个线程中运行，可以避免线程调度带来的延迟。
工作线程和CPU核一一对应，可以减少大量的CPU切换，从而减少了Cache Miss
#### trpc-go 网络层——基于goroutinue协程
实现与golang中的net/http里的server方式一样，每个连接创建一个server协程
trpc对每个service都起一个service协程 会根据service注册transport 有同步和异步处理模式
同步：service协程是一个for循环 一个请求处理之后才会处理下一个
异步：每接收到一个请求就会启动一个新的handle协程异步处理该请求
#### 性能对比
很明显的可以看到trpc-cpp的性能要高于trpc-go 
![trpc性能对比](./trpc性能对比.png)
#### 协议层抽象
##### 协议识别
一个trpc-go服务可以包含多个service，每个service监听一个端口，使用一种协议,因此trpc不需要进行协议识别
##### 协议编解码
为实现类似HTTP2的多路复用功能 帧头加入streamID
trpc协议设计：![](./trpc数据帧设计.png)
参考协议设计：![](./P.可参考的rpc数据帧协议设计.png)
codec是业务协议打解包接口 只解析出二进制流数据 具体的业务body结构体通过serializer来处理
常用的几种序列化方式：
http:multipart/form-data application/x-www-form-urlencoded application/json
其他：
protobuf thrift jce
![](./trpc协议层抽象.png)
#### trpc启动流程以及请求处理流程 
![](./trpc-go服务端主流程.png)

## 自研框架的一些思考点
### 网络库技术选型：
性能杀手：系统调用、上下文切换/io操作/内存拷贝/锁/
Epoll ET + NIO + Reactor + one loop(eventloop epoll_wait) per thread(thread pool) 
#### 1、线程还是协程?
如果是 I/O 密集型，且 I/O 请求比较耗时的话，使用协程。
如果是 I/O 密集型，且 I/O 请求比较快的话，使用多线程。
如果是 计算 密集型，考虑可以使用多核 CPU，使用多进程。
- 线程与协程的关系：goroutine调度
简单来说，Go程序可以利用少量的内核级线程来支撑大量Goroutine的并发。对于单核CPU来说，多个Goroutine通过用户级别的上下文切换来共享内核单个线程M的计算资源。
goroutine 协程的优势:
- goroutine相比线程更加轻量，主要体现在
  * 上下文切换：保存现场+拉起另外一个线程
    + 线程：触发CPU中断，涉及用户态切换到内核态(极端情况会跨核切换而导致缓存失效 损失更多的时间去从主存中捞数据)，涉及16个寄存器的刷新，CPU切换延迟大约在50-100ns
    + goroutine：用户态切换且只涉及PC SP DX三个寄存器的值刷新
  * 内存占用少：
    + 线程：单个线程的栈空间通常是2M
    + goroutine：单个goroutine栈空间最小2K(可变)
####

#### 2、并发模式
- 技术背景：
网络库的特点：IO密集型任务
使用多线程或者是多协程可以避免IO阻塞提高性能
对于多线程来说直接使用one connection per thread模式，会有上下问切换问题带来的性能瓶颈（Apache类似 进程级切换），所以一般使用异步非阻塞的开发模型，即one loop per thread模式，每一个eventloop（事件驱动） 使用epoll多路复用管理IO
对于多协程来说没有线程切换问题，协程之间的切换比较轻量，所以go net包直接使用 one goroutine-per-connection 
（对于计算密集型任务 采用多线程或者是多协程不一定能提高性能 甚至由于线程切换导致程序性能下降 这种情况一般直接使用进程）
所以可选择的有两种并发模式：
  * one loop per thread 
  * one goroutine-per-connection 
    + 缺点：虽然goroutine能够避免线程切换，但是避免不了线程创建、销毁的消耗，虽然Go scheduler内部做了的goroutine缓存链表(相当于线程池)，但是对于海量连接的瞬间暴涨的长连接场景无能为力
    goroutine 虽然非常轻量，它的自定义栈内存初始值仅为 2KB，后面按需扩容；海量连接的业务场景下， goroutine-per-connection ，此时 goroutine 数量以及消耗的资源就会呈线性趋势暴涨，虽然 Go scheduler 内部做了 g 的缓存链表，可以一定程度上缓解高频创建销毁 goroutine 的压力，但是对于瞬时性暴涨的长连接场景就无能为力了，大量的 goroutines 会被不断创建出来，从而对 Go runtime scheduler 造成极大的调度压力和侵占系统资源，然后资源被侵占又反过来影响 Go scheduler 的调度，进而导致性能下降。
####

#### 技术选型:
由于这里的网络库不仅要支持服务端还要支持客户端，而对于客户端来说无法实现epoll管理多连接，所以必须要使用多线程（线程池）无法避免线程的切换开销，所以还是使用协程实现比较好
  * 设想 ：服务端 one loop per goroutine 客户端 goroutine连接池
    + 优点：使用goroutine会使各个eventloop切换开销更小（只涉及几个eventloop切换 优势不会很明显）
    + 缺点：epoll异步回调机制 + goroutine 丧失了后者one goroutine-per-connection编程上直观性的优势
###

### 微服务框架技术选型
微服务框架应有的能力：
RPC框架的基础能力：callID映射（服务调用方式） 协议编解码  网络传输
中间件：服务注册发现、调用链跟踪、认证鉴权、动态路由、负载均衡、集中式日志、故障熔断、健康检查、访问限流、监控告警、分布式配置
#### 服务调用方式：
1、直接使用go结构体对象+反射获取服务方法 2、通过描述文件（proto）+ 脚本protoc生成代码
这两种方式各有利弊，第一种方式比较符合 go 用户的使用习惯，我只需要定义 go 结构体就可以注册服务，也不需要安装和执行额外的代码生成工具，比较简单方便。第二种方式提前生成了服务的描述信息和静态处理代码，性能更高，避免了第一种方式由于 go 反射带来的性能损耗。社区内这两种方式也都有框架在用，比较用得多的是第二种，类似 grpc、go-micro 使用 proto 作为描述文件，thrift 使用自身的 IDL 描述文件，第一种也有框架在用，比如 rpcx。
#### 网络模型：
详见自研网络库技术选型
#### 协议编解码：这里的协议是数据传输的协议，位于第五层，底层还是TCP/UDP
使用自定义协议 学习cap协议 或者公用协议（http1.1 http2.0等等）
公有协议 or 自定义协议？
公有协议往往具有通用性、易于理解等特点。比较熟知的典型代表就是 HTTP 协议。但是一般来说，协议为了支持通用性，往往会有一些性能牺牲。比如，文本协议的 http1.x 的性能是非常低下的。文本协议容易理解，但是会占用大量字节，二进制协议不易理解，但是占用字节数少，所以通信效率要远远高于文本型协议。所以 http2.0 吸取 SPDY 协议的经验，使用二进制帧的形式传输数据，极大提高了通信效率。
#### 序列化与反序列化组件：协议里的报文的二进制编解码方式
性能：gogoprotobuf > msgpack > flatbuffers > thrift > protobuf > json
其中只有msgpack json能够对原生的go结构体序列化 其他的必须使用脚本生成最终代码

在计算机网络中，数据都是以二进制形式进行传输的。将对象转换成可以传输的二进制数据的过程叫做序列化。将二进制的数据转换成对象的过程叫做反序列化。
社区上序列化的组件非常多，主要包括 protobuf、gogoprotobuf、thrift、msgpack、flatbuffers、json 等等

从编码后的字节长度来看，Jce和Protobuf的压缩效果接近。
从编码速度来看，Protobuf的吞吐量是Jce的2倍以上，这是我之前没有预料到的！
从解码速度来看，Jce比Protobuf快约10%。
##### proto编码方法
###### Varint编码 uint32 int64 int32 uint64 sint32 sint64 bool enum
部分源码：
private void writeVarint32(int n) {                                                                                    
  int idx = 0;  
  while (true) {  
    if ((n & ~0x7F) == 0) {  
      i32buf[idx++] = (byte)n;  
      break;  
    } else {  
      i32buf[idx++] = (byte)((n & 0x7F) | 0x80);  
      // 步骤1：取出字节串末7位
      // 对于上述取出的7位：在最高位添加1构成一个字节
      // 如果是最后一次取出，则在最高位添加0构成1个字节

      n >>>= 7;  
      // 步骤2：通过将字节串整体往右移7位，继续从字节串的末尾选取7位，直到取完为止。
    }  
  }  
  trans_.write(i32buf, 0, idx); 
      // 步骤3： 将上述形成的每个字节 按序拼接 成一个字节串
      // 即该字节串就是经过Varint编码后的字节
}
<font color='red'>取一个字节串的后7位的方法 (n & 0x7F) | 0x00 </font>
pb中Varint 编码：
int32 a = 8;
补码：0000 … 0000 1000
Varint 编码为 0000 1000（低位靠前）
Varint 编码后只使用一个字节就可以了，而正常的 int32 编码一般需要 4 个字节
Varint 编码的本质是：每一个字节都牺牲了一个bit位，来表示是否已经结束，msb实质上起到了length的作用，摆脱了无论数字大小都必须分配四个字节，所以他的优点就是对于比较小的数字可以用更少的字节表示，减少了序列化的体积。但是对于大数字来说，数字大于2^28,采用Varint 编码将导致分配5个字节 原本只要四个字节
由于负数的二进制补码的最高位是1，所以负数一定会占满所有字节，起不到数据压缩的作用，对于负数来说用Varint 编码不太好，一般用ZigZag编码

##### ZigZag编码
不重要 主要是辅助Varint编码 有时间再了解
##### 64（fixed64，sfixed64，double）/32bit（fixd32，sfixed64，float）编码
##### length-delimited (唯一采用T-L-V的编码形式 上面的都是T-V编码形式)
其中T L采用的是VarInt编码方式
V中会根据不同的数据类型采用不同的编码方式：
string类型 utf-8编码 
message嵌套类型会根据里面字段的数据类型选择 具体见图片：![protobuf message类型编码格式](./protobuf message类型编码格式.png)
##### proto序列化和反序列化过程
- 序列化的过程简单来说主要有下面两步
  * 判断每个字段是否有设置值，有值才进行编码
  * 根据 tag 中的 wire_type 确定该字段采用什么类型的编码方案进行编码即可
- Protobuf 反序列化过程简单来说也主要有下面两步：
  * 调用消息类的 parseFrom(input) 解析从输入流读入的二进制字节数据流
  * 将解析出来的数据按照指定的格式读取到相应语言的结构类型中
##### 写proto文件可以优化的几个点：
- 如果某个字段有负数 类型使用sint32/sint64 这样就会使用ZigZag编码
- 字段的标识号 尽量只使用1-15，因为如果标识符大于4位（msb1bit+标识符4bit+wire_type3bit），tag就会占用多个字节
  * 标识符尽量连续递增（原因不太清楚）
## 
##### TODO：需对比json得到优势 对比gogprotobuf得到劣势 对比维度（编解码 序列化流程）

###

#### 中间件
##### 分布式链路追踪
起源于google论文 Dapper, a Large-Scale Distributed Systems Tracing Infrastructure https://storage.googleapis.com/pub-tools-public-publication-data/pdf/36356.pdf
现在发展为opentracing规范
常见组件jaeger（go开发 上手难） appdash(使用内存实现 不适合大型系统)