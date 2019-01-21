## 介绍

Martin Fowler在自己网站上写了一篇[LMAX架构](http://ifeve.com/lmax)的文章，在文章中他介绍了LMAX是一种新型零售金融交易平台，它能够以很低的延迟产生大量交易。这个系统是建立在JVM平台上，其核心是一个业务逻辑处理器，它能够在一个线程里每秒处理6百万订单。业务逻辑处理器完全是运行在内存中，使用事件源驱动方式。业务逻辑处理器的核心是Disruptor。

**一个性能很高的内存队列库**



## 学习参考

http://ifeve.com/disruptor/

https://tech.meituan.com/2016/11/18/disruptor.html

