---
title: Netty之NioEventLoop流程简单分析
date: 2020-02-19 12:43:38
categories: Netty
tags: 
   - Java 
   - Netty 
   - 并发
---

#### Netty之NioEventLoop流程简单分析（4.1.30.Final-SNAPSHOT）

```java
NioEventLoop类核心代码块:
@Override
protected void run() {
    for (;;) {
        try {
            switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
                case SelectStrategy.CONTINUE:
                    continue;
                case SelectStrategy.SELECT:
                    // 轮询IO事件，等待事件的发生
                    select(wakenUp.getAndSet(false));

                    if (wakenUp.get()) {
                        selector.wakeup();
                    }
                    // fall through
                default:
            }

            cancelledKeys = 0;
            needsToSelectAgain = false;
            final int ioRatio = this.ioRatio;
            // 如果ioRatio==100 就调用第一个 processSelectedKeys();  否则就调用第二个
            if (ioRatio == 100) {
                try {
                    // 处理感兴趣事件
                    processSelectedKeys();
                } finally {
                    // Ensure we always run tasks.
                    // 用于处理本eventLoop外的线程 扔到taskQueue中的任务
                    runAllTasks();
                }
            } else { // 因为ioRatio默认是50 , 所以第一次走else
                final long ioStartTime = System.nanoTime();
                try {
                    // 处理IO事件
                    processSelectedKeys();
                } finally {
                    // Ensure we always run tasks.
                    // 根据处理IO事件耗时 ,控制 下面的runAllTasks执行任务不能超过 ioTime 时间
                    final long ioTime = System.nanoTime() - ioStartTime;
                    // 这里面有聚合任务的逻辑
                    runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                }
            }
        } catch (Throwable t) {
            handleLoopException(t);
        }
        // Always handle shutdown even if the loop processing threw an exception.
        try {
            if (isShuttingDown()) {
                closeAll();
                if (confirmShutdown()) {
                    return;
                }
            }
        } catch (Throwable t) {
            handleLoopException(t);
        }
    }
}

```

```java
/**
 * 基于deadline的任务穿插处理逻辑
 *
 * 根据当前时间计算出本次for()的最迟截止时间, 也就是他的deadline
 * 判断1 如果超过了 截止时间,selector.selectNow(); 直接退出
 * 判断2 如果任务队列中出现了新的任务 selector.selectNow(); 直接退出
 * 经过了上面12两次判断后, netty 进行阻塞式select(time) ,默认是1秒这时可会会出现空轮询的Bug
 * 判断3 如果经过阻塞式的轮询之后,出现的感兴趣的事件,或者任务队列又有新任务了,或者定时任务中有新任务了,或者被外部线程唤醒了 都直接退出循环
 * 如果前面都没出问题,最后检验是否出现了JDK空轮询的BUG
 */
private void select(boolean oldWakenUp) throws IOException {
    Selector selector = this.selector;
    try {
        int selectCnt = 0;
        long currentTimeNanos = System.nanoTime();
        // todo 计算出估算的截止时间,  意思是, select()操作不能超过selectDeadLineNanos这个时间, 不让它一直耗着,外面也可能有任务等着当前线程处理
        long selectDeadLineNanos = currentTimeNanos + delayNanos(currentTimeNanos);

        for (;;) {
            // todo 计算超时时间
            long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L;
            if (timeoutMillis <= 0) { // todo 如果超时了 , 并且selectCnt==0 , 就进行非阻塞的 select() , break, 跳出for循环
                if (selectCnt == 0) {
                    selector.selectNow();
                    selectCnt = 1;
                }
                break;
            }
            // todo 判断任务队列中时候还有别的任务, 如果有任务的话, 进入代码块, 非阻塞的select() 并且 break; 跳出循环
            // todo 通过cas 线程安全的把 wakenUp设置成true表示退出select()方法, 已进入时,我们设置oldWakenUp是false
            if (hasTasks() && wakenUp.compareAndSet(false, true)) {
                selector.selectNow();
                selectCnt = 1;
                break;
            }

            ///todo ----------- 如上部分代码, 是 select()的deadLine及任务穿插处理逻辑----------
            ///todo ----------- 如下, 是 阻塞式的select() ---------------------------------

            // todo  上面设置的超时时间没到,而且任务为空,进行阻塞式的 select() , timeoutMillis 默认1
            // todo netty任务,现在可以放心大胆的 阻塞1秒去轮询 channel连接上是否发生的 selector感性的事件
            int selectedKeys = selector.select(timeoutMillis);

            // todo 表示当前已经轮询了SelectCnt次了
            selectCnt ++;

            // todo  阻塞完成轮询后,马上进一步判断 只要满足下面的任意一条. 也将退出无限for循环, select()
            // todo  selectedKeys != 0      表示轮询到了事件
            // todo  oldWakenUp             当前的操作是否需要唤醒
            // todo  wakenUp.get()          可能被外部线程唤醒
            // todo  hasTasks()             任务队列中又有新任务了
            // todo   hasScheduledTasks()   当时定时任务队列里面也有任务
            if (selectedKeys != 0 || oldWakenUp || wakenUp.get() || hasTasks() || hasScheduledTasks()) {
                // - Selected something,
                // - waken up by user, or
                // - the task queue has a pending task.
                // - a scheduled task is ready for processing
                break;
            }

            // todo ------------ 如上, 是 阻塞式的select() -----------------------------------------------------

            if (Thread.interrupted()) {
                if (logger.isDebugEnabled()) {
                    logger.debug("Selector.select() returned prematurely because " +
                            "Thread.currentThread().interrupt() was called. Use " +
                            "NioEventLoop.shutdownGracefully() to shutdown the NioEventLoop.");
                }
                selectCnt = 1;
                break;
            }

            // todo 每次执行到这里就说明,已经进行了一次阻塞式操作 ,并且还没有监听到任何感兴趣的事件,也没有新的任务添加到队列, 记录当前的时间
            long time = System.nanoTime();

            // todo 如果  当前的时间 - 超时时间 >= 开始时间   把 selectCnt设置为1 , 表明已经进行了一次阻塞式操作
            // todo 每次for循环都会判断, 当前时间 currentTimeNanos 不能超过预订的超时时间 timeoutMillis
            // todo 但是,现在的情况是, 虽然已经进行了一次 时长为timeoutMillis时间的阻塞式select了,
            // todo 然而, 我执行到当前代码的 时间 - 开始的时间 >= 超时的时间

            // todo 但是,如果 当前时间- 超时时间< 开始时间, 也就是说,并没有阻塞select, 而是立即返回了, 就表明这是一次空轮询
            // todo 而每次轮询   selectCnt ++;  于是有了下面的判断,
            if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
                // timeoutMillis elapsed without anything selected.
                selectCnt = 1;
            } else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
                    // todo  selectCnt如果大于 512 表示cpu确实在空轮询, 于是rebuild Selector
                    selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
                // The selector returned prematurely many times in a row.
                // Rebuild the selector to work around the problem.
                logger.warn(
                        "Selector.select() returned prematurely {} times in a row; rebuilding Selector {}.",
                        selectCnt, selector);

                // todo 它的逻辑创建一个新的selectKey , 把老的Selector上面的key注册进这个新的selector上面 , 进入查看
                rebuildSelector();
                selector = this.selector;

                // Select again to populate selectedKeys.
                // todo 解决了Select空轮询的bug
                selector.selectNow();
                selectCnt = 1;
                break;
            }

            currentTimeNanos = time;
        }

        if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS) {
            if (logger.isDebugEnabled()) {
                logger.debug("Selector.select() returned prematurely {} times in a row for Selector {}.",
                        selectCnt - 1, selector);
            }
        }
    } catch (CancelledKeyException e) {
        if (logger.isDebugEnabled()) {
            logger.debug(CancelledKeyException.class.getSimpleName() + " raised by a Selector {} - JDK bug?",
                    selector, e);
        }
        // Harmless exception - log anyway
    }
}
```



NioEventLoop主要做三件事：

- 轮询IO事件
- 处理IO事件
- 处理非IO任务

#### 什么是Jdk的Selector空轮询

我们可以看到,上面的`run()`方法,经过两次判断后进入了指定时长的阻塞式轮询,而我们常说的空轮询bug,指的就是本来该阻塞住轮询,但是却直接返回了, 在这个死循环中,它的畅通执行很可能使得CPU的使用率飙升, 于是把这种情况说是jdk的selector的空轮询的bug。

在NIO中通过Selector的轮询当前是否有IO事件，根据JDK NIO api描述，Selector的select方法会一直阻塞，直到IO事件达到或超时，但是在Linux平台上这里有时会出现问题，在某些场景下select方法会直接返回，即使没有超时并且也没有IO事件到达，这就是著名的epoll bug，这是一个比较严重的bug，它会导致线程陷入死循环，会让CPU飙到100%，极大地影响系统的可靠性，到目前为止，JDK都没有完全解决这个问题。

但是Netty有效的规避了这个问题，经过实践证明，epoll bug已Netty框架解决，Netty的处理方式是这样的：

记录select空转的次数，定义一个阀值，这个阀值默认是512，可以在应用层通过设置系统属性io.netty.selectorAutoRebuildThreshold传入，当空转的次数超过了这个阀值，重新构建新Selector，将老Selector上注册的Channel转移到新建的Selector上，关闭老Selector，用新的Selector代替老Selector。

#### Netty 如何解决了Jdk的Selector空轮询bug?

一个分支语句 `if(){}else{}` , 首先他记录下,现在执行判断时的时间, 然后用下面的公式判断

```java
当前的时间t1 - 预订的deadLine截止时间t2  >= 开始进入for循环的时间t3
```

我们想, 如果说,上面的阻塞式`select(t2)`没出现任何问题,那么 我现在来检验是否出现了空轮询是时间t1 = t2+执行其他代码的时间, 如果是这样, 上面的等式肯定是成立的, 等式成立说没bug, 顺道把`selectCnt = 1;`

但是如果出现了空轮询,`select(t2)` 并没有阻塞,而是之间返回了, 那么现在的时间 t1 = 0+执行其他代码的时间, 这时的t1相对于上一个没有bug的大小,明显少了一个t2, 这时再用t1-t2 都可能是一个负数, 等式不成立,就进入了else的代码块, netty接着判断,是否是真的在空轮询, 如果说循环的次数达到了512次, netty就确定真的出现了空轮询, 于是netty`rebuild()`Selector ,从新开启一个Selector, 循环老的Selector上面的上面的注册的时间,重新注册进新的 Selector上,用这个中替换Selector的方法,解决了空轮询的bug。