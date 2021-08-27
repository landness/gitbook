# 总纲
启发：结合场景（达到的目的）说明约束和适用范围 然后展开说各个技术
qoe和qos结合起来说 qoe是现象层目的层 qos是原理层 有qoe->qos1、qos2->qoe
https://km.woa.com/group/29800/articles/show/431656
# BWE（GCC2.0）
基于丢包率和延时
C语言实现
https://github.com/yuanrongxi/razor
码率计算大致流程 （需细化）
https://km.woa.com/articles/show/494425?kmref=search&from_page=1&no=4
流控思路 
反馈+控制。抛开公式，总结其流控思路，以下三点比较重要：

1)       流控参数明确：用滤波方式取出主要决策量——排队延时，其实基于“延时”的流控算法具体来说，就是指的基于“排队延时”，其它延时成分如传输延时，网络抖动应设法去除；此外，丢包率将在发送端流控中发挥指导作用；

2)       调节有序：码率控制应定时、定事件控制，而不是一直在调节，因为有可能编码器不能实时响应下发的编码参数，导致编码器失调，即期望与实际编码出来的相差很大；

3)       升降有度：发送码率上升、下降到何处需参加长期统计出来的“参考带宽”，避免“一降到底”和“一升就升”。
**数据流向图（需重画）及调用栈**
https://toutiao.io/posts/cxav78/
gcc拥塞控制原理及webrtc代码分析
https://www.jianshu.com/p/5259a8659112
C语言实现
https://github.com/yuanrongxi/razor
# 包组时间差 滤波
相关代码分析 （需细化）
https://blog.jianchihu.net/webrtc-research-interarrival.html

# 拥塞控制发展
1、TCP经典拥塞控制：基于丢包
reno: 慢启动 拥塞避免 拥塞发生、快速恢复
缺点：
- 基于丢包，慢增长快减少，但无法区分是由于网络拥塞导致的丢包还是网络传输错误，只要产生丢包就会导致窗口减半，如果是大窗口环境（比如中间路由器的缓存区很大），带来的是带宽利用率低
- 一定会出现丢包 直到要填满第二类缓存（路由器缓存）丢包才判断拥塞
- 没有规定一个窗口内数据的发送速率，如果全部打入网络，会导致路由器缓冲队列堆积，进而导致rtt增大 测不准
rtt = 物理链路延时+路由器排队时间
水管问题？类比？？

2、BBR
https://km.woa.com/articles/show/337431?kmref=search&from_page=1&no=6
放弃了 丢包或RTT作为拥塞指标 的方案
引入BDP管道容量来衡量链路传输水平 将BDP作为发送窗口
BtlBW：最大带宽 瓶颈链路的带宽
RtProp：物理链路延迟（可以认为是固有延迟 近似链路上没有任何排队和其他时延时的RTT）
BDP：管道容量，BDP=BtlBW * RtProp
BBR 相对于传统TCP拥塞控制算法的优势：
基于BDP pace-rate
https://www.zhihu.com/question/53559433
bbr存在问题
1. 升得快，降得慢，猜测是期望基于丢包的拥塞控制算法会降窗，这就可以用他们腾出来的带宽。但如果路由器缓存不够深可能会造成大量丢包。甚至在深队列，等不到基于丢包的拥塞控制算法降窗而不断降低发送速率，吞吐率下降。且当大家发现可以自己接管丢包的降窗行为时，都采用激进做法，导致网络更加拥塞。

BWE模块的输出值：
```
  struct Result {
    Result();
    ~Result() = default;
    bool updated;
    bool probe;
    DataRate target_bitrate = DataRate::Zero();
    bool recovered_from_overuse;
    bool backoff_in_alr;
  };
```
核心就是目标码率target_bitrate 
入口函数 IncomingPacketFeedbackVector
**接合处**
每次接收rtcp报文后 被调用
调用链路:https://blog.csdn.net/CrystalShaw/article/details/111573233

## inter arrival

包组理解：
https://blog.jianchihu.net/webrtc-research-interarrival.html
流程：
https://blog.csdn.net/MeRcy_PM/article/details/72629279