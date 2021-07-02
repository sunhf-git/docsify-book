![Java线程池定义指南和用多线程处理任务时的优化策略](/images/code-1.jpg)

# 前言
多线程在生产环境中通常用来处理批量任务和简单的异步访问。这就涉及到使用线程池和线程池如何定义与配置，才能获得最大的处理性能提升。在学习之前需要对一些线程池的基本优化操作有一定了解，下面附带了一些资料可以先看看。
[估算线程池的大小](http://ifeve.com/how-to-calculate-threadpool-size/)
[美团线程池生产环境实战](https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html)

# 需求背景
目标：尽可能少的占用cpu和线程资源达到预期的性能要求
http接口进行时间稍长的删除操作（非数据库）速度较慢， 希望能将删除的动作异步化。

# 设计思路
- 出于对线程池资源开销的考虑，此类线程池用一个共享的就行了，可以使用spring的@Async注解实现。（此处有坑， 当某一个异步方法因为种种原因卡住了或者执行的慢，它会把线程池所有的可用资源都给占了，其他的线程就没法执行了）
- 自定义一个线程池，仅针对自身的业务使用线程池。这个就不举例子了，线程池相信大家都会用（缺点是需要异步的业务太多，创建的线程池太多，也会增加系统的额外开销。）
- 出于对资源隔离和网络IO（http）调用频繁的考虑及其运行时间不稳定的实际情况，最终选择了自定义线程池的策略。

# 线程池调优策略
## io密集型策略
- 尽可能多的同时处理数据，避免过多的io等待，事实上可以不用缓冲队列，需要注意的是控制最大线程数，评估好TPS需求（比如20TPS/s），避免线程资源占用过多。
- 尽量不要使用共享的线程池，尽量根据自定义的线程池来隔离各业务功能的资源，避免由于单条业务线故障使得整个异步执行系统崩溃。
- tps/qps峰值需求需要以实际使用场景来确定，如果没有那么多用户 调太高 也没什么意义。
- 合理的设置线程池线程等待时间，扛过高峰以后应尽快把线程资源释放掉。

## cpu密集型策略
- 比如多线程计算任务，耗费cpu，这种场景应该尽量避免多个线程带来的cpu切换。应该使用缓冲队列且线程数最好不超过cpu数量*2来解决问题。
- 缓冲队列长度的设置应以cpu数量为基准 平均计算时间和内存资源的消耗达到一个平衡的状态。

# 遇到的问题
如何确定线程池的配置，怎样调整才能达到最优的性能。和资源的平衡

# 性能测试步骤
先看一下@Async默认的线程池配置如下。

```java
public class ThreadPoolTaskExecutor extends ExecutorConfigurationSupport
		implements AsyncListenableTaskExecutor, SchedulingTaskExecutor {

	private final Object poolSizeMonitor = new Object();

	private int corePoolSize = 1;

	private int maxPoolSize = Integer.MAX_VALUE; //这种一般认为是无限大了

	private int keepAliveSeconds = 60;

	private int queueCapacity = Integer.MAX_VALUE; //队列大小也是无限大了

	private boolean allowCoreThreadTimeOut = false;

	@Nullable
	private TaskDecorator taskDecorator;

	@Nullable
	private ThreadPoolExecutor threadPoolExecutor;
}
```
来看看默认值直接硬怼的效果,直接oom了
核心代码：
```
 /**
     * 模拟异步的耗时操作
     */
    @Override
    @Async
    public void asyncOperate() {
        try {
            Thread.sleep(1000 * 10);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

/**
     * 单元测试往死里怼
     */
    @Test
    public void test() {
        for(;;) {
            service.asyncOperate();
        }
    }
```
测试结果直接oom， 这个如果在生产环境因为网络或其他问题把任务阻塞了就GG了
```
java.lang.OutOfMemoryError: unable to create new native thread
	at java.lang.Thread.start0(Native Method)
	at java.lang.Thread.start(Thread.java:714)
	at org.springframework.core.task.SimpleAsyncTaskExecutor.doExecute(SimpleAsyncTaskExecutor.java:237)
	at org.springframework.core.task.SimpleAsyncTaskExecutor.execute(SimpleAsyncTaskExecutor.java:195)
	at org.springframework.core.task.SimpleAsyncTaskExecutor.submit(SimpleAsyncTaskExecutor.java:209)
	at org.springframework.aop.interceptor.AsyncExecutionAspectSupport.doSubmit(AsyncExecutionAspectSupport.java:284)
	at org.springframework.aop.interceptor.AsyncExecutionInterceptor.invoke(AsyncExecutionInterceptor.java:129)
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:185)
	at org.springframework.aop.framework.JdkDynamicAopProxy.invoke(JdkDynamicAopProxy.java:212)
	at com.sun.proxy.$Proxy58.asyncOperate(Unknown Source)
	at com.sunhf.security.facade.AsyncThreadPoolFacadeImpl.operate(AsyncThreadPoolFacadeImpl.java:22)
```
所以在生产环境中一定不能用默认的线程池策略，需要自己根据业务场景来定制,比如像下边这样。但是全局的线程池会有一个缺点，多个不同业务的线程，A和B，假如A因为网络问题执行的特别慢，线程一直执行不完。就把B的执行空间给霸占了，如果项目有性能需求的话，还是尽量不要用@Async注解比较好。
```java
/**
     * 自定义@Async的线程池配置
     * @return
     */
    @Bean
    public ThreadPoolTaskExecutor threadPoolTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(Runtime.getRuntime().availableProcessors());
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(1000);
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.AbortPolicy());
        return executor;
    }
```
# 解决方案
最终还是自定义了线程池，直接写在了业务类中。这种方式的优势是也能起到线程资源隔离的作用。用起来还是比较灵活的。 缺点就是资源配比可能比较麻烦，需要根据业务的实际情况进行调整。

# 总结
线程池的使用在生产环境中不是简单的调一下api就行了，如果对线程池本身的机制和默认参数不了解的话流量一上来应用大概率就挂了。 珍爱生命，谨慎使用线程池。

