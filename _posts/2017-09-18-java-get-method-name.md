---
layout: post
title: Java 中获取当前方法名
categories: [编程, java]
tags: []
---

> Dao中通过xml配置sql，在调用的时候需要传入sqlId，怎样才能省略sqlId参数，而约定使用方法名作为sqlId呢？

#### 1.原版写法

##### sql.xml
```xml
<!-- xml中定义一个id为selectUserByUserName的sql -->
<sql id="selectUserByUserName">
select * from t_user where username=?
</sql>
```

##### UserDao.java
```java
public User selectUserByUserName(String username){
    //这里需要传入xml定义的sqlId
    return baseDao.getBySqlId("selectUserByUserName", new Object[]{username});
}
```

#### 2.改良版
约定使用dao的方法名作为sqlId，如何在BaseDao.getBySqlId中获取UserDao的方法名？

##### BaseDao
```java
//获取调用方法名称
protected String getInvokerMethodName() {
    return Thread.currentThread().getStackTrace()[3].getMethodName();
}

public T getBySqlId(Object[] args){
    String sqlId = getInvokerMethodName();
    return this.getBySqlId(sqlId, args);
}
```

##### UserDao.java
```java
public User selectUserByUserName(String username){
    //不需要传入sqlId，约定为调用者方法名，即selectUserByUserName
    return baseDao.getBySqlId(new Object[]{username});
}
```

> **注意： 动态获取调用方法名对性能会有一定的影响，所以该方法需要谨慎使用**