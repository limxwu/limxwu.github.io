---
title: Java Spring Service层业务状态的处理
id: 672ffa59-2b88-4452-a66a-c4db8caba63f
date: 2023-08-24 00:10:48
auther: limxwu
permalink: /archives/java-spring-serviceceng-ye-wu-zhuang-tai-de-chu-li
categories:
 - 文章
tags: 
 - java
---



> **soluction 1:**return status
>
> **soluction 2:**throw exception



考虑点如下：

1. 抛出异常能够结合事务实现回滚。

2. 两者的性能，采用抛异常的方式一般而言要损耗更多的性能。

3. 返回状态的方式，势必封装一个Result类，里面包含status以及需要返回的数据模型，则service所有的接口的返回类型都为Result，这是很糟糕的设计。

   假设要写一个查找用户的信息，方法的定义应该如下：

   ```java
   User getUser(long id);
   ```

   如果返回类型为Result，则定义如下：

   ```java
   Result<User> getUser(long id);
   ```

   接口的定义受到了实现方案的约束，违背了依赖倒置原则（抽象不应该依赖于细节，细节应当依赖 于抽象。换言之，要针对接口编程，而不是针对实现编程）。



4. Return status 方式任然需要在上层对status进行判断，也就是通常情况下controller仍需要关系业务status，并对其做出处理。

5. 抛出异常可能被日志采集系统采集到，影响异常治理，这是个小问题，可以不打印业务异常，关键在于被采集到后能不能接受。





综上，在不考虑异常质量的前提下，需要对比两者的性能开销，在可接受的范围内采用抛异常的方式处理业务状态。



根据经验而谈，异常需要更多的性能开销，但是多多少，性能开销在什么地方，能否优化这些问题需要具体实验后才能确定。



**实验代码**如下：



1. 数据类型

```java
 	 /**
     * 用户类
     */
    class User {
    }
    
    private final User user = new User();

    /**
     * 业务异常类
     */
    class BusinessException extends RuntimeException {
        public BusinessException(String message) {
            super(message);
        }
    }

    /**
     * 通用结果类
     * @param <T>
     */
    class Result<T> {
        private final boolean success;

        private final T t;

        public Result(boolean success) {
            this.success = success;
            t = null;
        }

        public Result(boolean success, T t) {
            this.success = success;
            this.t = t;
        }

        public boolean isSuccess() {
            return success;
        }

        public T getT() {
            return t;
        }
    }
```

2. 对比方法

   ```java
   /**
        * return status方式处理业务状态
        * @param num
        * @return
        */
       private  Result<User> getUser1(int num) {
           if (num % 2 == 0) {
               return new Result<>(false);
           } else {
               return new Result<>(true, user);
           }
       }
   
       /**
        * 抛异常的方式处理业务状态
        * @param num
        * @return
        */
       private User getUser2(int num) {
           if (num % 2 == 0) {
               throw new BusinessException("用户号码不符合规定");
           } else {
               return user;
           }
       }
   ```

   _PS：为了产生多个状态，对num的奇偶进行判断_

3. 执行

   ```java
    public void start() {
           int max = 10000000;
           long time1 = System.currentTimeMillis();
           User user1;
           for (int i = 0; i < max; i++) {
               Result<User> userResult = getUser1(i);
               if (userResult.isSuccess()) {
                   user1 = userResult.getT();
               }
           }
           long time2 = System.currentTimeMillis();
           for (int i = 0; i < max; i++) {
               try {
                   user1 = getUser2(i);
               } catch (Exception e) {
                   //在大量循环下打印体验比较糟糕，由get方法替代
   //                e.printStackTrace();
                   e.getStackTrace();
                   e.getMessage();
               }
           }
           long time3 = System.currentTimeMillis();
           System.out.println("getUser1 spend time:" + (time2 - time1));
           System.out.println("getUser2 spend time:" + (time3 - time2));
       }
   ```

**结果：**

   > getUser1 spend time:72
   > getUser2 spend time:10044



究其原因是在默认情况下，创建异常对象时会调用父类`Throwable`的`fillInStackTrace()`方法生成栈追踪信息，JDK中的源码如下：

```java
public synchronized Throwable fillInStackTrace() {
        if (stackTrace != null ||
            backtrace != null /* Out of protocol state */ ) {
            fillInStackTrace(0); // native方法
            stackTrace = UNASSIGNED_STACK;
        }
        return this;
    }
```

异常的创建：

采用单例模式创建异常，`getUser2`方法修改如下

```java
    private BusinessException businessException = new BusinessException();
    /**
     * 抛异常的方式处理业务状态
     * @param num
     * @return
     */
    private User getUser2(int num) {
        if (num % 2 == 0) {
            throw businessException;
        } else {
            return user;
        }
    }
```

>getUser1 spend time:75
>getUser2 spend time:120

可以看到在单例模式下，损耗的性能大幅下降。但是很多情况下我们需要调用`Exception(String message)`方法返回异常的原因，此时单例就显得捉襟见肘了，JDK1.7后`Threadable`类新增了一个构造函数：

```java
protected Throwable(String message, Throwable cause,
                        boolean enableSuppression,
                        boolean writableStackTrace) {
        if (writableStackTrace) {
            fillInStackTrace();
        } else {
            stackTrace = null;
        }
        detailMessage = message;
        this.cause = cause;
        if (!enableSuppression)
            suppressedExceptions = null;
    }
```

> `writableStackTrace`: whether or not the stack trace should be writable

也就是说在业务异常类中，可以通过此构造函数将`writableStackTrace`设置为**false**不去生成异常堆栈信息，毕竟对于业务异常关心的只是其状态及原因而已。

修改业务异常类：

```java
    /**
     * 业务异常类
     */
    class BusinessException extends RuntimeException {
        public BusinessException(String message) {
            super(message, null, false, false);
        }
    }
```

Result

> getUser1 spend time:75
> getUser2 spend time:123

可以看出于单例差距甚微，至此对于Service层业务状态的处理应该采用抛异常的方式去处理更合理与优雅。