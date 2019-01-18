## 方案介绍

github源代码地址：https://github.com/yu199195/hmily

#### 以dubbo为例：

1. 利用dubbo的filter，在dubbo方法执行前，加上拦截，标明处理的阶段（Trying，Confirm，Cancel）
2. 在要执行事务的方法之前加上注解，在调用该方法之前，拦截，执行Trying方式。
3. 在拦截器中调用preTryXXX之后，在调用point.proceed() 执行事务方法本省
4. 发起方在有异常抛出的时候，再次调用dubbo接口，传入阶段为Cancel
5. 发起方在完成的的时候，再次调用dubbo接口，传入阶段为Confirm
6. 相当于完成一个事务操作需要调两次dubbo接口
7. tcc 每个阶段都操作记录事务日志：trying阶段记录日志，confirm和cancel阶段删除日志。
8. 日志记录通过发布事件的形式disruptor
9. 如果在回滚时候出现问题，可以通过事务日志，人工补偿



#### 总结

1. 记录事务中的每一步操作，对存储的性能要请求高。
2. 对代码有一定的侵入性
3. 感觉tcc是2pc的一个变种
4. 框架相对来说比较简单，只要理解了tcc的原理已经rpc框架的机制，看源代码不难