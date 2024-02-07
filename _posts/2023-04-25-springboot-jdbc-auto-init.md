---
layout: post
title: Springboot-datasource自动初始化
categories: [编程, java, spring]
tags: []
---

> 

#### 1.maven依赖

springboot的datasource自动初始化在`springboot-autoconfigure`包中

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-autoconfigure</artifactId>
</dependency>
```

#### 2.配置

application.yaml

```yaml
spring:
  datasource:
    initialization-mode: ALWAYS // 每次启动时自动初始化
    continue-on-error=true // 忽略error
    schema: classpath:sql/schema-mysql.sql // ddl
    data: classpath:sql/data-mysql.sql // dml
```

#### 3.sql

src/mainresource/sql/schema-mysql.sql
```
CREATE TABLE IF NOT EXISTS `user` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `username` varchar(128) NOT NULL,
  `password` varchar(128) NOT NULL
);
```

src/mainresource/sql/data-mysql.sql
```
INSERT INTO `user`(`username`, `password`)
SELECT 'admin', '123456'
WHERE NOT EXISTS (SELECT 1 FROM user WHERE username='admin');
```

注意：由于每次服务重启都会执行，所以要求ddl和dml一定是幂等的，并且一定不能包含DROP、DELETE等语句

#### 4.原理

入口在`org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration`

实现在`org.springframework.boot.autoconfigure.jdbc.DataSourceInitializer`

```java
boolean createSchema() {
  List<Resource> scripts = getScripts("spring.datasource.schema", this.properties.getSchema(), "schema");
  if (!scripts.isEmpty()) {
    if (!isEnabled()) {
      logger.debug("Initialization disabled (not running DDL scripts)");
      return false;
    }
    String username = this.properties.getSchemaUsername();
    String password = this.properties.getSchemaPassword();
    runScripts(scripts, username, password);
  }
  return !scripts.isEmpty();
}

void initSchema() {
  List<Resource> scripts = getScripts("spring.datasource.data", this.properties.getData(), "data");
  if (!scripts.isEmpty()) {
    if (!isEnabled()) {
      logger.debug("Initialization disabled (not running data scripts)");
      return;
    }
    String username = this.properties.getDataUsername();
    String password = this.properties.getDataPassword();
    runScripts(scripts, username, password);
  }
}

private boolean isEnabled() {
  DataSourceInitializationMode mode = this.properties.getInitializationMode();
  if (mode == DataSourceInitializationMode.NEVER) {
    return false;
  }
  if (mode == DataSourceInitializationMode.EMBEDDED && !isEmbedded()) {
    return false;
  }
  return true;
}
```