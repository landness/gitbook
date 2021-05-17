# 前言
p2p连接建立之后，底层的socket已经准备好，接收端要想接收数据，必须能够捕获socket相关的事件。在了解真正的接收流程前有必要先了解webrtc底层线程是如何对IO响应的。[线程模型](./webrtc线程模型)
# 视频接收流程
![](../pic/视频接收流程.jpg)
## 调用关系 QT信号和槽函数
1 connection->SignalReadPacket.connect(this, &P2PTransportChannel::OnReadPacket); 
2 P2PTransportChannel->SignalReadPacket.connect(this, &DtlsTransport::OnReadPacket);
3 DtlsTransport->SignalReadPacket.connect(this, &RtpTransport::OnReadPacket);
4 RtpDemuxer::OnRtpPacket-->VideoChannel这一步的调用
是RtpTransport::RegisterRtpDemuxerSink方法把VideoChannel作为 Sink 加到 RtpTransport 里的 rtp_demuxer_
# 视频发送流程
![](../pic/视频发送流程.jpg)