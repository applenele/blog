## tcp 流量控制

一个机器向另外一台机器发送数据，接收方来得及处理，对发送方发送数据的速度做了控制。

实现方式：**滑动窗口**



## tcp拥塞控制

整个网络传输的时候发生阻塞，类似马上上车辆堵住了。网络阻塞了，整个网络都无法传输数据。

解决办法：

1. 慢开始( slow-start )、
2. 拥塞避免( congestion avoidance )、
3. 快重传( fast retransmit )
4. 快恢复( fast recovery )



## 参考

<https://www.cnblogs.com/wxgblogs/p/5616829.html>

