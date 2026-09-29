---
title: Java多线程下系统时区全局锁对性能的影响
id: cfbf9d3e-5dcd-4618-8b5c-12932441f384
date: 2023-08-24 00:18:52
auther: limxwu
cover: 
excerpt: 起因 在考量MySql时间格式时，看到有文章提到高并发下timestamp调用系统时区转换格式时会因为全局锁导致线程上下文切换致使cpu使用率暴涨。 测试过程 在查询数据库时，MyBatis或者JPA等框架大都会将timestamp转换为Java中对应的时间戳格式（Timestamp或者Instan
permalink: /archives/javaduo-xian-cheng-xia-xi-tong-shi-qu-quan-ju-suo-dui-xing-neng-de-ying-xiang
categories:
 - 文章
tags: 
 - java
---

## 起因

在考量MySql时间格式时，看到有文章提到高并发下`timestamp`调用系统时区转换格式时会因为全局锁导致线程上下文切换致使cpu使用率暴涨。

## 测试过程

在查询数据库时，MyBatis或者JPA等框架大都会将`timestamp`转换为Java中对应的时间戳格式（Timestamp或者Instant），在这一过程中并不存在时区转换的问题，而在业务代码尤其是返回前端数据时，经常需要将时间戳转换为具体的时间字符串，此时才是应该探究的全局锁对性能影响的关键点。

设计两个对照组：

1. 是否使用系统时区对性能的影响
2. 不同并发量下对性能影响

使用系统时区的Callable：

```java
 private static final Callable<Long> useSystemZoneCallable = () -> {
        long startTime = System.currentTimeMillis();
        for (int k = 0; k < MAX_EX_NUM; k++) {
            ZoneId.systemDefault().toString();
        }
        return System.currentTimeMillis() - startTime;
    };
```

不使用系统时区的Callable:

```java
  private static final Callable<Long> notUseSystemZoneCallable = () -> {
        long startTime = System.currentTimeMillis();
        for (int k = 0; k < MAX_EX_NUM; k++) {
            ZoneOffset.UTC.toString();
        }
        return System.currentTimeMillis() - startTime;
    };
```

其中`MAX_EX_NUM`设置为1000000000

开启不同数量的线程执行上述Callable，并统计每个线程执行时长，计算平均值。

```java
 /**
     * 使用系统时区
     *
     * @param maxThreadNum
     * @throws InterruptedException
     * @throws ExecutionException
     */
    private static void useSystemZone(int maxThreadNum) throws InterruptedException, ExecutionException {
        CountDownLatchFuture<Long>[] futureTasks = new CountDownLatchFuture[maxThreadNum];
        CountDownLatch countDownLatch = new CountDownLatch(maxThreadNum);
        for (int i = 0; i < maxThreadNum; i++) {
            futureTasks[i] = new CountDownLatchFuture<>(useSystemZoneCallable, countDownLatch);
            new Thread(futureTasks[i]).start();
        }
        countDownLatch.await();
        long totalSpendTime = 0;
        for (FutureTask<Long> futureTask : futureTasks) {
            totalSpendTime += futureTask.get();
        }
        System.out.printf("ThreadNum: %s,\tuseSystemZone method take %s milliseconds(average).\n", maxThreadNum,
                (totalSpendTime / maxThreadNum));
    }
```

不使用系统时区的执行方法同上。

>[源码地址](https://github.com/NextWWParty/JavaLesson/blob/master/Practice/src/main/java/pers/limx/practice/thread/SystemTimeZone.java)

## 测试结果

```
ThreadNum: 1,	useSystemZone method take 1024 milliseconds(average).
ThreadNum: 2,	useSystemZone method take 1021 milliseconds(average).
ThreadNum: 3,	useSystemZone method take 655 milliseconds(average).
ThreadNum: 4,	useSystemZone method take 735 milliseconds(average).
ThreadNum: 5,	useSystemZone method take 806 milliseconds(average).
ThreadNum: 6,	useSystemZone method take 837 milliseconds(average).
ThreadNum: 7,	useSystemZone method take 880 milliseconds(average).
ThreadNum: 8,	useSystemZone method take 916 milliseconds(average).
ThreadNum: 9,	useSystemZone method take 969 milliseconds(average).
ThreadNum: 10,	useSystemZone method take 1020 milliseconds(average).
ThreadNum: 11,	useSystemZone method take 1096 milliseconds(average).
ThreadNum: 12,	useSystemZone method take 1195 milliseconds(average).
ThreadNum: 13,	useSystemZone method take 1239 milliseconds(average).
ThreadNum: 14,	useSystemZone method take 1334 milliseconds(average).
ThreadNum: 15,	useSystemZone method take 1374 milliseconds(average).
ThreadNum: 16,	useSystemZone method take 1445 milliseconds(average).
ThreadNum: 17,	useSystemZone method take 1559 milliseconds(average).
ThreadNum: 18,	useSystemZone method take 1640 milliseconds(average).
ThreadNum: 19,	useSystemZone method take 1665 milliseconds(average).
ThreadNum: 20,	useSystemZone method take 1712 milliseconds(average).
ThreadNum: 21,	useSystemZone method take 1760 milliseconds(average).
ThreadNum: 22,	useSystemZone method take 1849 milliseconds(average).
ThreadNum: 23,	useSystemZone method take 2079 milliseconds(average).
ThreadNum: 24,	useSystemZone method take 2247 milliseconds(average).
ThreadNum: 1,	notUseSystemZone method take 4 milliseconds(average).
ThreadNum: 2,	notUseSystemZone method take 4 milliseconds(average).
ThreadNum: 3,	notUseSystemZone method take 0 milliseconds(average).
ThreadNum: 4,	notUseSystemZone method take 0 milliseconds(average).
ThreadNum: 5,	notUseSystemZone method take 0 milliseconds(average).
ThreadNum: 6,	notUseSystemZone method take 0 milliseconds(average).
ThreadNum: 7,	notUseSystemZone method take 0 milliseconds(average).
ThreadNum: 8,	notUseSystemZone method take 0 milliseconds(average).
ThreadNum: 9,	notUseSystemZone method take 0 milliseconds(average).
ThreadNum: 10,	notUseSystemZone method take 0 milliseconds(average).
ThreadNum: 11,	notUseSystemZone method take 0 milliseconds(average).
ThreadNum: 12,	notUseSystemZone method take 0 milliseconds(average).
ThreadNum: 13,	notUseSystemZone method take 0 milliseconds(average).
ThreadNum: 14,	notUseSystemZone method take 0 milliseconds(average).
ThreadNum: 15,	notUseSystemZone method take 0 milliseconds(average).
ThreadNum: 16,	notUseSystemZone method take 0 milliseconds(average).
ThreadNum: 17,	notUseSystemZone method take 0 milliseconds(average).
ThreadNum: 18,	notUseSystemZone method take 0 milliseconds(average).
ake 0 milliseconds(average).
```

## 结论

在高并发环境下，时间戳调用系统时区时，由于全局锁的存在会导致大量线程的上下文切换，容易造成CPU满载的情况。所以在使用时间戳格式时，在情况允许的前提下最好传递给外部系统特定时区（UTC,东八区....)的ISO-8601时间格式，而不是系统默认时区的时间格式。