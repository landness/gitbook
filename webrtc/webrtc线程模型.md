# 1 前言
在读源码过程中发现webrtc的各个线程有明确的分工，特定的函数一定要在指定的线程上执行，因此在执行代码时，经常会遇到跨线程调用。同时在学习音视频接收流程时也需要对webrtc底层线程对socketIO的响应机制有了解。为之后的学习做铺垫，从跨线程调用和IO响应这两方面学习和总结了webrtc的线程模型。
# 2 各个线程有明确的分工
- 信令线程（Signal Thread）
一般是工作在 PeerConnection 层，主要是完成控制平面的逻辑，用于和应用层交互。
比如CreateOffer，SetRemoteSession等接口都是通过 Signal thread 完成的。
默认是采用 PeerConnectionFactory初始化线程作为信令线程。
- 工作线程（Worker Thread）
主要是工作在媒体引擎层（media engine），具体工作如下:
音频设备初始化、视频设备初始化、流对象的初始化
从网络线程接收数据，传给解码器线程
从编码器线程接收数据，传给网络线程
- 网络线程（Network thread ）
主要是工作在传输（transport）层，具体工作如下：
Transport 的初始化
从网络接收数据，发送给 Worker thread
从 Worker thread 接收数据，发送到网络

# 3 线程模型
## 3.1 以一个例子开始
应用层想要实现通话首先要创建PeerConnection 如下
```
bool InitializePeerConnection() {
    peer_connection_factory_ = webrtc::CreatePeerConnectionFactory(

      nullptr /* network_thread */, nullptr /* worker_thread */,
      nullptr /* signaling_thread */, nullptr /* default_adm */,
      webrtc::CreateBuiltinAudioEncoderFactory(),
      webrtc::CreateBuiltinAudioDecoderFactory(),
      webrtc::CreateBuiltinVideoEncoderFactory(),
      webrtc::CreateBuiltinVideoDecoderFactory(), nullptr /* audio_mixer */,
      nullptr /* audio_processing */);

      peer_connection_ = peer_connection_factory_->CreatePeerConnection(
      config, nullptr, nullptr, this);
      return peer_connection_ == nullptr;
}
```
CreatePeerConnection 底层调用了peerConnection的Initialize初始化函数（\src\pc\peer_connection.cc ）
```
RTCError PeerConnection::Initialize(
    const PeerConnectionInterface::RTCConfiguration& configuration,
    PeerConnectionDependencies dependencies) {
    // PeerConnection初始化是在signaling_thread
    RTC_DCHECK_RUN_PeerConnectionON(signaling_thread());
    // 创建stun_servers  turn_servers
    cricket::ServerAddresses stun_servers;
    std::vector<cricket::RelayServerConfig> turn_servers;
    // 为stun turn添加配置参数
    RTCErrorType parse_error =
      ParseIceServers(configuration.servers, &stun_servers, &turn_servers);
    // network_thread线程初始化
    network_thread()->Invoke<void>(RTC_FROM_HERE, [this, &stun_servers,
                                                 &turn_servers, &configuration,
                                                 &dependencies] {
    RTC_DCHECK_RUN_ON(network_thread());
    network_thread_safety_ = PendingTaskSafetyFlag::Create();
    InitializePortAllocatorResult pa_result =
        InitializePortAllocator_n(stun_servers, turn_servers, configuration);
    // 填充传输控制参数
    InitializeTransportController_n(configuration, dependencies);
```
可以看到PeerConnection初始化是在signaling_thread中，初始化port和transport时需要调用InitializePortAllocator_n、InitializeTransportController_n， 而这两个函数是要在network thread执行，所以此处进行了跨线程调用。
## 3.2 线程工作大致流程
```mermaid
sequenceDiagram
participant si as signal_thread
participant  net as network_thread <br>  <br>Processmessage
participant ss as SocketServer
si ->> net:invoke msg 并且唤醒目标线程
si->> si:阻塞等待
net ->> net:被唤醒后get msg from message list
net ->> net:不为空，dispatch(msg)
net ->>si :返回跨线程调用结果并唤醒
net ->> ss:list为空且有剩余时间<br>则处理网络IO
ss ->>ss:select/epoll阻塞监听注册的网络IO和msg事件
ss ->> ss:信号触发，若是网络事件则调用OnEvent处理网络事件<br>
ss ->> net:信号触发，若是msg事件返回到Processmessage
net ->> net:dispatch(msg)
net ->>si :返回跨线程调用结果
```
可以看到线程阻塞在select/epoll，select/epoll除了会管理网络IO外，也会管理msg（一般是webrtc内部的跨线程调用），当msg事件被触发，socketServer会将线程控制权归还给Processmessage进行处理，而如果是网络IO，则会直接调用onEvent处理。可以看到，它解决了跨线程函数调用和管理网络IO的问题。详细的分析见3.2跨线程同步调用和3.3 socketserver网络IO响应。
## 3.3 跨线程同步调用

**network_thread()是获取network线程thread对象，注意这里的Invoke()函数是在sigal线程中执行的network线程对象的Invoke()函数。换言之在Invoke函数内的this指针是netword_thread对象，消息队列message_list也是network_thread对象的，而调用TheadManger的方法 CurrentThead()获取的则是singal线程的对象，读代码时需注意，以下的分析都是基于此的：**

**1、Invoke函数（InvokeInternal）**
本质上就是将需要执行的方法封装到消息处理器MessageHandler中，然后执行Send(来源线程，FunctorMessageHandler)
```
void Thread::InvokeInternal(const Location& posted_from,
                            rtc::FunctionView<void()> functor) {
  // FunctorMessageHandler 将函数functor包装成message handler
  class FunctorMessageHandler : public MessageHandler {
   public:
    explicit FunctorMessageHandler(rtc::FunctionView<void()> functor)
        : functor_(functor) {}
    void OnMessage(Message* msg) override { functor_(); }

   private:
    rtc::FunctionView<void()> functor_;
  } handler(functor);

  Send(posted_from, &handler);
}
```
**2、Send函数** 
把入参FunctorMessageHandler包装成msg信息，并且向network_thread线程的message_list投递msg和一些信号量（比如局部变量ready），这里是通过Post函数实现投递的，post函数本身是非阻塞的，所以在Send函数调用post后还需要阻塞当前线程等待结果返回。
这里PostTask实际上向目标线程投递了两个消息，一个是实际的msg处理函数，另一个是msg被消费后的clean函数，由于消息队列是一个FIFO队列，于是msg被消费后会调用clean函数，而clean函数中封装了唤醒信息，会唤醒请求from的线程，如此处的singal thread。这样的做法很巧妙，做到了解耦。
```
void Thread::Send(const Location& posted_from,
                  MessageHandler* phandler,
                  uint32_t id,
                  MessageData* pdata) {
  if (IsQuitting())
    return;
  Message msg;
  msg.posted_from = posted_from;
  msg.phandler = phandler;
  msg.message_id = id;
  msg.pdata = pdata;
  // 如果this == current thread 则在当前线程 直接消费
  if (IsCurrent()) {
    msg.phandler->OnMessage(&msg);
    return;
  }
  // current_thread是singal thread的线程对象
  Thread* current_thread = Thread::Current();
  bool ready = false;
  // PostTask()调用了Post()实现消息投递
  // ToQueuedTask是函数模板 可以将两个lamda函数转化成task
  PostTask(webrtc::ToQueuedTask(
      // 传递msg和处理函数
      [&msg]() mutable { msg.phandler->OnMessage(&msg); },
      // 清理函数 msg被消费后调用 
      [this, &ready, current_thread] {
        CritScope cs(&crit_);
        ready = true;
        current_thread->socketserver()->WakeUp();
      }));
  // 阻塞等待
  bool waited = false;
  // ready当前线程和目标线程都会访问 所以要加同一把锁 即上面的crit_
  // Send阻塞在socketserver的wait上
  crit_.Enter();
  while (!ready) {
    crit_.Leave();
    current_thread->socketserver()->Wait(kForever, false);
    waited = true;
    crit_.Enter();
  }
  crit_.Leave();
  // 当前线程阻塞在wait中时有可能有其他线程向当前线程投递消息 
  // 这时当前线程wakeup但是while循环不能退出（ready变量未置为true） 也即一些唤醒事件被屏蔽了
  // 为了解决这种情况这里使用了waited 在send退出前重新唤醒以及时处理这些事件
  if (waited) {
      current_thread->socketserver()->WakeUp();
  }
```
**3、Post函数**
非阻塞投递消息，本身逻辑比较简单，加锁入队，唤醒目标线程。要注意的就是上面说的这里的thread对象是目标thread，故messagelist是目标thread的，唤醒的也是目标线程。
```
// Post用于投递即时消息
void Thread::Post(const Location& posted_from,
                  MessageHandler* phandler,
                  uint32_t id,
                  MessageData* pdata,
                  bool time_sensitive) {
  if (IsQuitting())
    return;
  {
    // 消息入队加锁 用{}限制cs的作用区和生存期
    CritScope cs(&crit_);
    Message msg;
    msg.posted_from = posted_from;
    msg.phandler = phandler;
    msg.message_id = id;
    msg.pdata = pdata;
    messages_.push_back(msg);
  }
  // 唤醒目标线程network thread
  WakeUpSocketServer();
}
```
**4、消息循环ProcessMessages**
network_thread线程有一个**消息循环ProcessMessages**，线程被唤醒后，会调用Get函数检查其消息队列
```
bool Thread::ProcessMessages(int cmsLoop) {
  while (true) {
    Message msg;
    if (!Get(&msg, cmsNext))
      return !IsQuitting();
    // msg在此处被消费
    Dispatch(&msg);
  }
}
// 为方便分析 默认循环无限期方式执行 也即 cmsLoop=kForever
```
**5、Get函数**
逻辑是取到一个消息就返回 优先取延时消息，再取即时消息，并且会计算超时时间（设置的超时时间和延时队列下一个时间触发的时间取最小值），在超时时间内，处理网络IO（SocketServer），此时get阻塞在socketServer的wait调用上
```
bool Thread::Get(Message* pmsg, int cmsWait, bool process_io) {
  int64_t msStart = TimeMillis();
  int64_t msCurrent = msStart;
  while (true) {
    int64_t cmsDelayNext = kForever;
    bool first_pass = true;
    while (true) {
      // 对消息队列的操作 上锁
      {
        CritScope cs(&crit_);
        // 先处理延迟消息队列
        if (first_pass) {
          first_pass = false;
          // 将延迟消息队列dmsgq_中已经超过触发时间的消息放到即时消息队列msgq_中
          // 计算最近触发时间间隔 作为后续网络IO处理的超时时间
          while (!delayed_messages_.empty()) {
            if (msCurrent < delayed_messages_.top().run_time_ms_) {
              cmsDelayNext =
                  TimeDiff(delayed_messages_.top().run_time_ms_, msCurrent);
              break;
            }
            messages_.push_back(delayed_messages_.top().msg_);
            delayed_messages_.pop();
          }
        }

        // 取即时消息队列头
        if (messages_.empty()) {
            // 若队列为空 直接break 时间交给网络IO
          break;
        } else {
          *pmsg = messages_.front();
          messages_.pop_front();
        }
      }
      return true;
    }
    if (IsQuitting())
      break;
    // 阻塞处理IO多路复用 cmsDelayNext延时队列超时时间 作为IO处理超时时间
    // 阻塞退出的条件是是处理时间耗尽或者有新消息进入循环被唤醒
    {
      if (!ss_->Wait(static_cast<int>(cmsDelayNext), process_io))
        return false;
    }
  }
  return false;
}
// 为方便分析，上面默认get无限期方式执行 也即 cmsWait=kForever
```
**6、Dispatch(&msg)**
会调用msg的OnMessage方法消费消息，也即实现了在指定线程network thread上执行对应的方法。

7、最后，正如上面Send函数中分析的clean函数，会唤醒请求from线程和传递返回值，这样就完成了跨线程调用的整个流程。

## 3.4 socketserver IO响应机制
socketserver可以认为是一个抽象类，它有两个功能wait阻塞监听和wakeup唤醒，而这两个功能实际是在子类PhysicalSocketServer中实现的，所以下面的socketserver实际指的是PhysicalSocketServer。

下面的以linux平台的socketserver实现为例分析。
### 监听及响应机制
**1、wait函数** 
在3.2 get函数分析中可以看到线程阻塞在wait调用中，wait函数核心是通过select/poll/epoll调用实现的阻塞监听。
```
bool PhysicalSocketServer::Wait(int cmsWait, bool process_io) {
  ScopedSetTrue s(&waiting_);
#if defined(WEBRTC_USE_EPOLL)
  // We don't keep a dedicated "epoll" descriptor containing only the non-IO
  // (i.e. signaling) dispatcher, so "poll" will be used instead of the default
  // "select" to support sockets larger than FD_SETSIZE.
  if (!process_io) {
    return WaitPoll(cmsWait, signal_wakeup_);
  } else if (epoll_fd_ != INVALID_SOCKET) {
    return WaitEpoll(cmsWait);
  }
#endif
  return WaitSelect(cmsWait, process_io);
}
```
上面要重点关注下webrtc对这三种模式的选择：当管理文件描述符较少的时候使用select，否则启用epoll来处理网络IO，当调用wait标识本次不处理网络IO时（process_io = false）使用poll，这时poll处理的就是一些唤醒事件，比如上面signal_thread跨线程同步调用send msg之后陷入阻塞，此时的wait就是使用的poll。推测这样的做法是由于epoll的内存开销比较大，需要在内存中维护红黑树和就绪链表，所以每个socketserver不默认维护一个epoll_fd，只有那些处理大量网络IO的才启用epoll。
**2、epoll**
这里epoll的使用比较常见，而且代码比较长这里就不详细写了。
**3、IO响应**
以可读事件为例，底层epoll IO触发后，webrtc是如何一步步通知给上层的？之前见到的做法一般是事先在对应的模块绑定好回调函数，而webrtc是使用了信号与槽机制（\rtc_base\third_party\sigslot\sigslot.h）。

epoll可读事件触发后最终会调用SignalReadEvent(this)，相当于发出信号，会触发注册给到这个信号的槽函数。

举一个简单的例子（\examples\peerconnection\client\peer_connection_client.cc）：
创建一个webrtc客户端，除了开头说的首先要创建PeerConnection之外，还需要有处理IO的能力，比如应用层需要实现与信令服务器交互的PeerConnectionClient::onRead函数，用来解析http包。
```
void PeerConnectionClient::OnRead(rtc::AsyncSocket* socket) {
  size_t content_length = 0;
  if (ReadIntoBuffer(socket, &control_data_, &content_length)) {
    size_t peer_id = 0, eoh = 0;
    bool ok =
        ParseServerResponse(control_data_, content_length, &peer_id, &eoh);
    if (ok) {
      // 登录逻辑...
    }
  }
}
```
然后通过注册到底层socket的read信号上，当control_socket_的可读事件触发时，就会调用到应用层的read
```
control_socket_->SignalReadEvent.connect(this, &PeerConnectionClient::OnRead);
```

### 唤醒机制
在3.2 send函数和get函数分析中可以看到，线程间唤醒是socketserver的WakeUp函数实现的，代码如下：
```
void PhysicalSocketServer::WakeUp() {
  // signal_wakeup 是唤醒类Signaler实例
  signal_wakeup_->Signal();
}
```
通过阅读Signaler类的代码得到，webrtc使用了pipe机制实现线程间的唤醒操作，具体就是通过调用Signal()函数向pipe的写文件描述符 fd[1]写入消息，而读文件描述符fd[0]已事先注册到select或epoll，这样就会触发select或epoll_wait的可读事件进而实现唤醒线程。
```
// 父类Dispatcher 可认为是webrtc包装的socket+event结构
class Signaler : public Dispatcher {
 public:
  Signaler(PhysicalSocketServer* ss, bool& flag_to_clear)
      : ss_(ss),
        // 将arry[1] | arry[0]建立管道
        afd_([] {
          std::array<int, 2> afd = {-1, -1};
          if (pipe(afd.data()) < 0) {}
          return afd;
        }()),
        fSignaled_(false),
        flag_to_clear_(flag_to_clear) {
    // socketserver.add 最终会调用epoll_ctl
    // 将arry[0]注册到epoll
    ss_->Add(this);
  }

  // 唤醒函数
  virtual void Signal() {
    webrtc::MutexLock lock(&mutex_);
    if (!fSignaled_) {
      const uint8_t b[1] = {0};
      // 通过写实现唤醒
      const ssize_t res = write(afd_[1], b, sizeof(b));
      fSignaled_ = true;
    }
  }
  // 继承自Dispatcher
  // arry[0]在epoll触发可读事件后的处理函数
  void OnEvent(uint32_t ff, int err) override {
    webrtc::MutexLock lock(&mutex_);
    if (fSignaled_) {
      uint8_t b[4];
      const ssize_t res = read(afd_[0], b, sizeof(b));
      fSignaled_ = false;
    }
    flag_to_clear_ = false;
  }

  int GetDescriptor() override { return afd_[0]; }

 private:
  PhysicalSocketServer* const ss_;
  const std::array<int, 2> afd_;
  bool fSignaled_ RTC_GUARDED_BY(mutex_);
  webrtc::Mutex mutex_;
  bool& flag_to_clear_;
};
```
## 3.5 相关的类
### ThreadManger
TheadManger是一个全局单例类，主要作用就是通过将一个内核线程与一个thread对象关联（将thread对象的地址存储在线程tls的某个槽位中），管理所有thread对象来管理所有线程。主要的方法有设置当前线程关联的thread对象,获取当前线程关联的thread对象等。

### TaskQueue
TaskQueue是thread类的父类，主要提供了PostTask、PostDelayedTask接口，使得thread类具有向消息队列投递即时消息和延时消息的能力，thread类在此基础上实现了Send,Invoke方法，实现了方法跨线程同步调用的功能。

# 4 整个webrtc的线程结构
上面的分析着眼于signal thread和network thread的交互，从跨线程调用角度分析了webrtc线程间的同步原理，从网络IO角度分析了线程IO的响应原理，着重介绍了webrtc线程模型的底层原理。

webrtc规定了特定的线程执行特定的任务，所以一个线程就可以看成是一个单独的模块，分析线程之间的结构其实就是在分析webrtc整个架构的内部结构，比如内部数据流向，各模块的功能和边界等等，这需要对webrtc的各个模块都有所了解之后才能把握。这里先放一张网上看到的图，比较清楚的展示了线程间的数据流向。
![](../pic/webrtc-线程结构图.jpg)

后续会对视频传输模块做详细分析，即上图中的红框部分，着重关注各个模块的功能和其中的一些qos方法，可以作为这部分的补充。
# 参考资料
https://zhuanlan.zhihu.com/p/136070941
https://zhuanlan.zhihu.com/p/25288799?from_voters_page=true