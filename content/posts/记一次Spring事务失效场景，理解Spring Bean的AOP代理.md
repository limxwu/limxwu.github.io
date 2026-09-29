---
title: 记一次Spring事务失效场景，理解Spring Bean的AOP代理
id: 60d97b9c-292e-4316-a059-9459f2b76921
date: 2023-08-24 00:24:41
auther: limxwu
cover: 
excerpt: 在Spring中，我们经常使用@Transactional注解开启事务，在这个注解中有个propagation属性用来控制事务的传播，此次失效场景中使用Propagation.NEVER，Spring中是这样描述的： Execute non-transactionally, throw an exc
permalink: /archives/ji-yi-ci-springshi-wu-shi-xiao-chang-jing-li-jie-spring-beande-aopdai-li
categories:
 - 文章
tags: 
 - java
---

在Spring中，我们经常使用`@Transactional`注解开启事务，在这个注解中有个propagation属性用来控制事务的传播，此次失效场景中使用`Propagation.NEVER`，Spring中是这样描述的：

> Execute non-transactionally, throw an exception if a transaction exists. Analogous to EJB transaction attribute of the same name.

场景复现代码如下：

```java
@Service
public class UserService {

    private final JdbcTemplate jdbcTemplate;

    public UserService(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Transactional
    public User getUser(Long id) {
        jdbcTemplate.execute("insert into db_user(name,sex)value('李四','男') ");
        insertRandomUser();
        return new User() {{
            setId(id);
        }};
    }

    @Transactional(propagation = Propagation.NEVER)
    public void insertRandomUser() {
        jdbcTemplate.execute("insert into db_user(name,sex)value('张三','男') ");
    }
}
```

在`getUser()`方法启用事务，并且在其中调用配置了`Propagation.NEVER`的`insertRandomUser()`方法，按照Spring对`Propagation.NEVER`描述，调用`insertRandomUser()`是应该抛出异常的。
执行程序，输出如下：

```
10:21:03.620 [main] DEBUG org.springframework.jdbc.support.JdbcTransactionManager - Creating new transaction with name [pers.limx.practice.spring.bean.UserService.getUser]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
10:21:04.796 [main] DEBUG org.springframework.jdbc.support.JdbcTransactionManager - Acquired Connection [com.mysql.cj.jdbc.ConnectionImpl@4303b7f0] for JDBC transaction
10:21:04.798 [main] DEBUG org.springframework.jdbc.support.JdbcTransactionManager - Switching JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@4303b7f0] to manual commit
before Service
10:21:04.816 [main] DEBUG org.springframework.jdbc.core.JdbcTemplate - Executing SQL statement [insert into db_user(name,sex)value('李四','男') ]
10:21:04.860 [main] DEBUG org.springframework.jdbc.core.JdbcTemplate - Executing SQL statement [insert into db_user(name,sex)value('张三','男') ]
10:21:04.898 [main] DEBUG org.springframework.jdbc.support.JdbcTransactionManager - Initiating transaction commit
10:21:04.898 [main] DEBUG org.springframework.jdbc.support.JdbcTransactionManager - Committing JDBC transaction on Connection [com.mysql.cj.jdbc.ConnectionImpl@4303b7f0]
10:21:04.926 [main] DEBUG org.springframework.jdbc.support.JdbcTransactionManager - Releasing JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@4303b7f0] after transaction

进程已结束，退出代码为 0
```

可以看到，两条SQL都执行成功，并没有抛出异常，想要理解其中的原因必须要理解Spring AOP是如何代理Bean的。

Spring事务是通过AOP实现的。用上文代码中UserSevice举例，这个类中有需要事务控制的方法，则Spring AOP在原有的UserSevice Bean的基础上，生成其代理对象，在需要注入UserService的地方实际注入是生成的代理对象UserServiceProxy,事务的实现就体现在调用方法前后加入对事务的控制。
伪代码逻辑如下：

```java
public class UserServiceProxy {
    UserService target;//被代理的对象

    public User getUser(Long id) {
        //创建数据库连接con
        //con.setAutoCommit(false);
        doBefore();
        User user = null;
        try {
            user = target.getUser(id);
            doAfter();
            //con.commit();
            return user;
        } catch (RuntimeException e) {
            //con.rollback();
            throw e;
        }
    }
}
```

回到这次事务失效的问题上，在诸如Controller的外部调用事务控制的方法时，调用的是动态代理类的代理方法，由此实现了事务的控制，但是在`getUser()`方法中调用`insertRandomUser()`调用的是UserService target本身的方法，而不是代理对象的`insertRandomUser()`，事务自然也就失效了。

