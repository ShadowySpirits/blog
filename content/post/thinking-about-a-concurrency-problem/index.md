---
title: 对一个并发问题的思考
slug: thinking-about-a-concurrency-problem
date: 2019-11-13T22:29:00+08:00
tags:
  - Java
  - Concurrent
categories:
  - tech
---

遇到了一个并发问题，将对锁的设计的思考记录一下：

这个问题的逻辑是：需要根据 id 获取数据库中指定的一行数据，如果这行数据的某个字段为 null 则请求远程接口来获取数据（每次请求接口这个数据都会更新），然后将获取的值写入到数据库中

关键的问题在于如何能在高并发的情况下保证最后数据库中存的是最新获取的数据，显然我们可以用 synchronized 锁来解决。但是这个方法中有三个耗时操作，如果这样粗暴的解决肯定是行不通的，所以要考虑如何减小锁的粒度。

我的解决方案是使用一个 HashMap 来为每个 id 生成一把锁（采用 lazyload 策略，尝试获取锁时再生成）这样只有对多个 id 的连续请求才会阻塞

但是这样随之而来的是另一个新问题：HashMap 会无限地新增锁，如果 id 的数量过多那岂不是得被挤爆内存，所以我这里新开一个线程去删除暂时不用的锁

这个删除逻辑又有两个问题：

1.  如何保证前一个线程执行完下一个线程有机会获取锁
2.  如何保证删除的锁真的是没人用的

第一个问题通过把删除线程睡眠 3s 来解决，第二个问题我通过重新读取 lock 来解决，并且获取 map 的锁来保证没有其他线程能在检测过程中能访问 map 以及获得新 lock、检测是否占用、从 map 中删除这三个操作的原子性

```java
	// private static final ReentrantLock mapLock = new ReentrantLock();
	// private static final HashMap<Long, ReentrantLock> map = new HashMap<>();
	private int checkAndUpdate(int id) {
        mapLock.lock();
        ReentrantLock lock = map.getOrDefault(id, new ReentrantLock());
        mapLock.unlock();
        lock.lock();
        // check and update
        lock.unlock();
        new Thread(() -> {
            try {
                Thread.sleep(3000);
                mapLock.lock();
                    ReentrantLock newLock = map.get(id);
                    if (newLock != null && !newLock.isLocked()) {
                        map.remove(id);
                    }
                mapLock.unlock();
            } catch (InterruptedException e) {

            }
        }).start();
    }
```

可以优化的点：

1.  采用线程池来执行删除逻辑
2.  获取到 lock 后先判断是否已经加锁，如果已经加了则直接放弃更新，否则尝试加锁
    因为如果竞争锁失败那么说明已经有线程在尝试更新这个为 null 的值了，如果更新成功那这个值不再为 null 就不用再次更新了，如果更新失败那显然要提示用户失败，用户会再次尝试。所以还是直接放弃的好
