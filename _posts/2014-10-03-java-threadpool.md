---
layout: post
title:  "ThreadPoolExecutor的那些事"
categories: ["java", "cocurrent"]
---


####重要参数变量解释

* workers，ThreadPool内部的HashSet，里面放得Worker，可以理解为一个线程。

* poolSize，也就是workers的size。

* maximunPoolSize，线程池最大大小。

* corePoolSize，线程池核心大小。

* workQueue，这个queue可以理解成是存放待执行的任务的queue。

* keepAliveTime，worker线程在idle的状态下，可以存活的时间。


####提交任务执行(execute())的处理逻辑

1、首先判断worker数量是否小于corePoolSize，如果小于添加新worker，并启动worker，

2、如果worker数量大于corePoolSize，则尝试把tasks往queue里放，如果成功放到了workQueue中，则直接返回，等到已创建的其他worker来执行这个work。

3、如果往queue里放没有成功（没有成功可能是因为你设置workQueue的大小，或者使用SynchronousQueue），并且poolSize小于maximunPoolSize，则创建worker执行此task

4、如果poolSize大于MaximunPoolSize，则调用RejectHandler，拒绝此任务。


###worker何时退出的问题

ThreadPool中的worker被创建之后，会一直尝试在workQueue中获取待执行的任务，如果在workQueue中获取不到任务，
则会判断worker是否可以结束。具体可以参看下面两段代码。

满足如下3个条件任一一个的，则判断worker可以退出。

#####前置条件：在workQueue内没有拿到task。

1、workQueue中没有任何等待执行的task。

2、`allowCoreThreadTimeOut && poolSize > Math.max(1, corePoolSize)` 至少保留一个worker。

3、线程状态已停止 runState >= STOP

{% highlight java %}
    Runnable getTask() {
        for (;;) {
            try {
                int state = runState;
                if (state > SHUTDOWN)
                    return null;
                Runnable r;
                if (state == SHUTDOWN)  // Help drain queue
                    r = workQueue.poll();
                else if (poolSize > corePoolSize || allowCoreThreadTimeOut)
                    r = workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS);
                else
                    r = workQueue.take();
                if (r != null)
                    return r;
                if (workerCanExit()) {
                    if (runState >= SHUTDOWN) // Wake up others
                        interruptIdleWorkers();
                    return null;
                }
                // Else retry
            } catch (InterruptedException ie) {
                // On interruption, re-check runState
            }
        }
    }
{% endhighlight %}

{% highlight java%}
    private boolean workerCanExit() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        boolean canExit;
        try {
            canExit = runState >= STOP ||
                workQueue.isEmpty() ||
                (allowCoreThreadTimeOut &&
                 poolSize > Math.max(1, corePoolSize));
        } finally {
            mainLock.unlock();
        }
        return canExit;
    }
{% endhighlight %}


###Tips

1、如果你配了RejectHandler，请注意BlockingQueue的选择。选择控制queue大小，或者参见下一条tips。

2、如果使用SynchronousQueue作为workQueue，则代表你希望task会立即被执行，无论是执行失败还是成功，任务不被暂存。

3、如果你不配置BlockingQueue的大小，则MaximunPoolSize相当于是等于corePoolSize的效果，maximunPoolSize不会有任何作用。

