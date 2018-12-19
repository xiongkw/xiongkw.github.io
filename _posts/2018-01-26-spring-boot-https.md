---
layout: post
title: 使用spring-boot搭建https服务
categories: [编程, java, spring]
tags: [spring-boot, https]
---

> 使用`spring-boot`开发的`web`项目，直接使用内嵌的`tomcat`运行

#### 1. keystore生成
```
keytool -genkeypair -keyalg RSA -keysize 2048 -sigalg SHA1withRSA -validity 3650 -alias myapp -keystore myapp.keystore

输入密钥库口令:
再次输入新口令:
您的名字与姓氏是什么?
  [Unknown]:  www.myapp.com
您的组织单位名称是什么?
  [Unknown]:  xx
您的组织名称是什么?
  [Unknown]:  xx
您所在的城市或区域名称是什么?
  [Unknown]:  xx
您所在的省/市/自治区名称是什么?
  [Unknown]:  xx
该单位的双字母国家/地区代码是什么?
  [Unknown]:  xx
CN=www.myapp.com, OU=xx, O=xx, L=xx, ST=xx, C=xx是否正确?
  [否]:  y

输入 <myapp> 的密钥口令
        (如果和密钥库口令相同, 按回车):
```

#### 2. spring-boot配置

`application.yml`

```yaml
server:
  port: 8443
  ssl:
    key-alias: myapp
    key-store: 'classpath:myapp.keystore'
    key-store-password: 123456
```

> 为方便，这里把`myapp.keystore`放到`resources`目录下

#### 3. 访问

运行`spring-boot`项目并访问

```
https://www.myapp.com:8443
```

#### 4. 同时启用https和http端口

application.yml
```yaml
server:
    port: 8080
https:
    port: 8443
    ssl:
          key-store: classpath://myapp.keystore
          key-store-password: 1234
          key-store-type: PKCS12
          key-alias: tomcat
```

HttpsConfigurer.java
```java
@Value("${https.ssl.key-store}")
private Resource keystore;

@Value("${https.ssl.key-store-password}")
private String keystorePassword;

@Bean
public EmbeddedServletContainerFactory servletContainer() {
    TomcatEmbeddedServletContainerFactory tomcat = new TomcatEmbeddedServletContainerFactory();
    tomcat.addAdditionalTomcatConnectors(createSslConnector());
    return tomcat;
}

private Connector createSslConnector() {
    Connector connector = new Connector("org.apache.coyote.http11.Http11NioProtocol");
    Http11NioProtocol protocol = (Http11NioProtocol) connector.getProtocolHandler();
    try {
        File keystore = keystore.getFile();
        connector.setScheme("https");
        connector.setSecure(true);
        connector.setPort(port);
        protocol.setSSLEnabled(true);
        protocol.setKeystoreFile(keystore.getAbsolutePath());
        protocol.setKeystorePass(keystorePassword);
        return connector;
    }
    catch (IOException ex) {
        throw new IllegalStateException("can't access keystore: [" + "keystore"
                + "] or truststore: [" + "keystore" + "]", ex);
    }
}
```

#### 5. http端口重定向到https端口

application.yml
```yaml
server:
    port: 8443
    ssl: 
      key-store: keystore.p12
      key-store-password: hefeng1234
      key-store-type: PKCS12
      key-alias: tomcat
```

HttpsConfigurer.java
```java
@Bean
public EmbeddedServletContainerFactory servletContainer(){
  TomcatEmbeddedServletContainerFactory tomcat=new TomcatEmbeddedServletContainerFactory(){
      @Override
      protected void postProcessContext(Context context) {
          SecurityConstraint securityConstraint=new SecurityConstraint();
          securityConstraint.setUserConstraint("CONFIDENTIAL");
          SecurityCollection collection=new SecurityCollection();
          collection.addPattern("/*");
          securityConstraint.addCollection(collection);
          context.addConstraint(securityConstraint);
      }
  };
  tomcat.addAdditionalTomcatConnectors(httpConnector());
  return tomcat;
}

@Bean
public Connector httpConnector(){
  Connector connector=new Connector("org.apache.coyote.http11.Http11NioProtocol");
  connector.setScheme("http");
  connector.setPort(8080);
  connector.setSecure(false);
  connector.setRedirectPort(8443);
  return connector;
}
```