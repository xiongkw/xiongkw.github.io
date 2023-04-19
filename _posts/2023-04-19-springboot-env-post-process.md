---
layout: post
title: Springboot自动解密配置中的变量值
categories: [java, spring]
tags: [prometheus]
---

> spring的EnvironmentPostProcessor接口提供变量处理的能力

#### 1. maven依赖

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot</artifactId>
  <version>2.7.8</version>
</dependency>
```

#### 2. 实现EnvironmentPostProcessor接口

```java
public class PropertyEncryptProcessor implements EnvironmentPostProcessor {

    public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
        String aesKey = environment.getProperty("encrypt.aes-key", "myaeskey");
        String envPattern = environment.getProperty("encrypt.key-pattern", ".*(password|secret).*");
        Pattern pattern = Pattern.compile(envPattern, Pattern.CASE_INSENSITIVE);
        // 环境变量中的
        Map<String, Object> env = environment.getSystemEnvironment();
        Map<String, Object> plainTextEnv = new HashMap<>();
        env.forEach((k, v) -> {
            if (pattern.matcher(k).matches() && v instanceof String) {
                plainTextEnv.put(k, tryDecrypt((String) v, aesKey));
            }
        });
        if (!plainTextEnv.isEmpty()) {
            environment.getPropertySources().addFirst(new MapPropertySource("plain-text-env", plainTextEnv));
        }
        // 配置文件中的
        Map<String, Object> plainTextMap = new HashMap<>();
        for (PropertySource<?> ps : environment.getPropertySources()) {
            if (ps instanceof OriginTrackedMapPropertySource) {
                OriginTrackedMapPropertySource source = (OriginTrackedMapPropertySource) ps;
                for (String name : source.getPropertyNames()) {
                    String value = environment.getProperty(name);
                    if (pattern.matcher(name).matches() && value != null) {
                        plainTextMap.put(name, tryDecrypt(value, aesKey));
                    }
                }
            }
        }
        if (!plainTextMap.isEmpty()) {
            environment.getPropertySources().addFirst(new MapPropertySource("custom-encrypt", plainTextMap));
        }
    }

    private String tryDecrypt(String v, String aesKey) {
        // ...
    }

}
```

#### 3. 配置spring.factories

resources/META-INF/spring.factories

```
org.springframework.boot.env.EnvironmentPostProcessor=org.example.PropertyEncryptProcessor
```
