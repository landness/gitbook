# 1 背景
鉴权服务的server端app-manage部署在平台侧集群，client端app-auth部署在应用侧集群，在鉴权流程中，server端会向client端回调消息，回调域名事先与应用侧集群ingress 公网LB IP绑定。

在私有化集群中，需要将server和client部署在同一集群。
# 2 现象
私有化集群中鉴权服务的server端app-manage回调在同一集群的client端app-auth失败。
经定位发现在server端app-manage服务所在的pod中```telnet LBIP:PORT```不通，也即存在K8S集群内部不能访问LB的情况
# 3 原因
鉴权server端app-manage为NodePort类型的service，向LoadBalancer类型的ingress-controller service发起telnet，实质上是一个pod与另一service下pod的通信，所以首先了解下K8s网络中pod-to-service-to-pod的包流转过程。
- K8s中pod-service数据包流转过程
pod与service通信可以认为是在pod-to-pod of other node的基础上，加上pod所在Node节点的iptable的过滤。iptables接受到节点网桥转发的数据包后会使用kube-proxy在Node上安装的NAT rules来将数据包的目的地址从Service的IP重写为Service后端特定的Pod IP，这样在数据包到达node的eth0后就变成了pod-to-pod的方式。

在app-manage下的pod中直接telnet对应service下的pod的ClusterIP:port发现是可以通的，也即直接pod-to-pod是可以的，所以问题出现在iptable的NAT转换，可以通过以下命令排查。
```
ip rule show 
ip route show table default
```

查阅资料了解到，**kube-proxy会将LoadBalancer Service的IP同步到节点本地的ipvs/iptables规则中拦截调用**。
使用以下命令验证发现确实本节点的转发规则中同步了公网LB的IP
```
ip route show table default | grep LB-IP
```

这样做的本意是减少网络负载，但是如果LoadBalancer service的外部流量转发策略是local时，会导致意想不到的后果：在某一节点上访问这个LoadBalancer的IP，如果该节点中没有对应SLB service的pod分布，且local模式的SLB Service为了保留源IP，不做二次转发NAT，那么这个包不会被转发给其他节点中的pod，会被丢弃掉，这就导致了开头的集群内部telnet lb不通的现象。

**总结：对于Local类型的LoadBalancer Service，在没有运行其对应的Pod的节点上，访问这个LoadBalancer的IP不通**
# 4 解决办法
集群内部调用可以使用（service名.命名空间.svc.cluster.local）代替域名，利用k8s集群内部自带的dns解析服务
# 5 参考资料
https://zhuanlan.zhihu.com/p/61677445
https://zhuanlan.zhihu.com/p/107703112
https://github.com/kubernetes/kubernetes/issues/66607