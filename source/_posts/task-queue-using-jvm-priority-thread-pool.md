---
title: JVM优先级线程池做任务队列
tags:
  - 线程池
  - 消息队列
categories:
  - 技术
copyright: true
date: 2020-02-21 19:20:16
---

我们都知道 web 服务的工作大多是接受 http 请求，并返回处理后的结果。服务器接受的每一个请求又可以看是一个任务。一般而言这些请求任务会根据请求的先后有序处理，如果请求任务的处理比较耗时，往往就需要排队了。而同时不同的任务直接可能会存在一些优先级的变化，这时候就需要引入任务队列并进行管理了。可以做任务队列的东西有很多，Java 自带的线程池，以及其他的消息中间件都可以。

<!-- more -->

## 同步与异步

这个问题在之前已经提过很多次了，有些任务是需要请求后立即返回结果的，而有的则不需要。设想一下你下单购物的场景，付完钱后，系统只需要返回一个支付成功即可，后续的积分增加、优惠券发放、安排发货等等业务都不需要实时返回给用户的，这些就是异步的任务。大量的异步任务到达我们部署的服务上，由于处理效率的瓶颈，无法达到实时处理，因此与需要用队列将他们暂时保存起来，排队处理。

## 线程池

在 Java 中提到队列，我们除了想到基本的数据结构之外，应该还有线程池。线程池自带一套机制可以实现任务的排队和执行，可以满足单点环境下绝大多数异步化的场景。下面是典型的一个处理流程：

```java
// 注入合适类型的线程池
@Autowired
private final ThreadPoolExecutor asyncPool;
@RequestMapping(value = "/async/someOperate", method = RequestMethod.POST)
public RestResult someOperate(HttpServletRequest request, String params,String callbackUrl {
    // 接受请求后 submit 到线程池排队处理
    asyncPool.submit(new Task(params,callbackUrl);
    return new RestResult(ResultCode.SUCCESS.getCode(), null) {{
        setMsg("successful!" + prop.getShowMsg());
    }};
}

// 异步任务处理
@Slf4j
public class Task extends Callable<RestResult> {
    private String params;
    private String callbackUrl;
    private final IAlgorithmService algorithmService = SpringUtil.getBean(IAlgorithmServiceImpl.class);
    private final ServiceUtils serviceUtils = SpringUtil.getBean(ServiceUtils.class);
    public ImageTask(String params,String callbackUrl) {
        this.params = params;
        this.callbackUrl = callbackUrl;
    }

    @Override
    public RestResult call() {
        try {
            // 业务处理
            CarDamageResult result = algorithmService.someOperate(this.params);
            // 回调
            return serviceUtils.callback(this.callbackUrl, this.caseNum, ResultCode.SUCCESS.getCode(), result, this.isAsync);
        } catch (ServiceException e) {
            return serviceUtils.callback(this.callbackUrl, this.caseNum, e.getCode(), null, this.isAsync);
        }
    }
}
```

对于线程池这里就不具体展开讲了，仅仅简单理了下具体的流程：

1. 收到请求后，参数校验后传入线程池排队。
2. 返回结果：“请求成功，正在处理”。
3. 任务排到后由相应的线程处理，处理完后进行接口回调。

上面的例子描述了一个生产速度远远大于消费速度的模型，普通面向数据库开发的企业级应用，由于数据库的连接池开发的连接数较大，一般不需要这样通过线程池来处理，而一些 GPU 密集型的应用场景，由于显存的瓶颈导致消费速度慢时，就需要队列来作出调整了。

## 带优先级的线程池

更复杂的，例如考虑到任务的优先级，还需要对线程池进行重写，通过 `PriorityBlockingQueue` 来替换默认的阻塞队列。直接上代码。

```java
import lombok.Data;

import java.util.concurrent.Callable;

/**
 * @author Fururur
 * @create 2020-01-14-10:37
 */
@Data
public abstract class PriorityCallable<T> implements Callable<T> {
    private int priority;
}
```

```java
import lombok.Getter;
import lombok.Setter;

import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicLong;

/**
 * 优先级线程池的实现
 *
 * @author Fururur
 * @create 2019-07-23-10:19
 */
public class PriorityThreadPoolExecutor extends ThreadPoolExecutor {

    private ThreadLocal<Integer> local = ThreadLocal.withInitial(() -> 0);

    public PriorityThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit) {
        super(corePoolSize, maximumPoolSize, keepAliveTime, unit, getWorkQueue());
    }

    public PriorityThreadPoolExecutor(int corePoolSize, int maximumPoolSize,
                                      long keepAliveTime, TimeUnit unit, ThreadFactory threadFactory) {
        super(corePoolSize, maximumPoolSize, keepAliveTime, unit, getWorkQueue(), threadFactory);
    }

    public PriorityThreadPoolExecutor(int corePoolSize, int maximumPoolSize,
                                      long keepAliveTime, TimeUnit unit, RejectedExecutionHandler handler) {
        super(corePoolSize, maximumPoolSize, keepAliveTime, unit, getWorkQueue(), handler);
    }

    public PriorityThreadPoolExecutor(int corePoolSize, int maximumPoolSize,
                                      long keepAliveTime, TimeUnit unit, ThreadFactory threadFactory,
                                      RejectedExecutionHandler handler) {
        super(corePoolSize, maximumPoolSize, keepAliveTime, unit, getWorkQueue(), threadFactory, handler);
    }

    private static PriorityBlockingQueue getWorkQueue() {
        return new PriorityBlockingQueue();
    }

    @Override
    public void execute(Runnable command) {
        int priority = local.get();
        try {
            this.execute(command, priority);
        } finally {
            local.set(0);
        }
    }

    public void execute(Runnable command, int priority) {
        super.execute(new PriorityRunnable(command, priority));
    }

    public <T> Future<T> submit(PriorityCallable<T> task) {
        local.set(task.getPriority());
        return super.submit(task);
    }

    public <T> Future<T> submit(Runnable task, T result, int priority) {
        local.set(priority);
        return super.submit(task, result);
    }

    public Future<?> submit(Runnable task, int priority) {
        local.set(priority);
        return super.submit(task);
    }

    @Getter
    @Setter
    protected static class PriorityRunnable implements Runnable, Comparable<PriorityRunnable> {
        private final static AtomicLong seq = new AtomicLong();
        private final long seqNum;
        private Runnable run;

        private int priority;

        PriorityRunnable(Runnable run, int priority) {
            seqNum = seq.getAndIncrement();
            this.run = run;
            this.priority = priority;
        }

        @Override
        public void run() {
            this.run.run();
        }

        @Override
        public int compareTo(PriorityRunnable other) {
            int res = 0;
            if (this.priority == other.priority) {
                if (other.run != this.run) {
                    // ASC
                    res = (seqNum < other.seqNum ? -1 : 1);
                }
            } else {
                // DESC
                res = this.priority > other.priority ? -1 : 1;
            }
            return res;
        }
    }
}
```

要点如下：

1. 替换线程池默认的阻塞队列为 `PriorityBlockingQueue`，响应的传入的线程类需要实现 `Comparable<T>` 才能进行比较。
2. `PriorityBlockingQueue` 的数据结构决定了，优先级相同的任务无法保证 FIFO，需要自己控制顺序。
3. 需要重写线程池的 `execute()` 方法。看过线程池源码的会发现，执行 `submit(task)` 方法后，都会转化成 `RunnableFuture<T>` 再进一步执行，由于传入的 task 虽然实现了 `Comparable<T>` 到，但是内部转换成的 `RunnableFuture<T>` 并未实现，因此直接 `submit` 会抛出 `Caused by: java.lang.ClassCastException: java.util.concurrent.FutureTask cannot be cast to java.lang.Comparable` 这样一个异常，所以需要重写 `execute()` 方法，构造一个 `PriorityRunnable` 作为中转。

## 总结

JVM 线程池是实现异步任务队列最简单最原生的一种方式，本文介绍了基本的使用流程和带有优先队列需求的用法。这种方法可有满足到一些简单的业务场景，但也存在一定的局限性：

- JVM 线程池是单机的，横向扩展多个服务下做负载均衡时，就会存在多个线程池了他们是分开工作的，无法很好的统一和管理，不太适合分布式场景。
- JVM 线程池是基于内存的，一旦服务挂了，会出现任务丢失的情况，可靠性低。
- 缺少作为任务队列的 ack 机制，一旦任务失败不会重新执行，且无法很好地对线程池队列进行监控。

显然简单的 JVM 线程池是无法 handle 到负载的业务场景的，这就需要引入其他中间件了，在接下来的文章中我们会继续探讨。

## 参考文献

- [ThreadPoolExecutor 优先级的线程池](https://www.jianshu.com/p/6c3021779015)

- [implementing PriorityQueue on ThreadPoolExecutor](https://stackoverflow.com/questions/30574777/implementing-priorityqueue-on-threadpoolexecutor)

- [ThreadPoolExecutor 的 PriorityBlockingQueue 类型转化问题](https://www.twblogs.net/a/5b7cdfc42b71770a43dcf6d0/zh-cn)

- [大搜车异步任务队列中间件的建设实践](https://www.infoq.cn/article/uMQb2CFDgRFcDUz9OFD1)