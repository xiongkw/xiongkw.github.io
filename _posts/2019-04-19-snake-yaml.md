---
layout: post
title: SnakeYAML序列化Java对象
categories: [编程, java]
tags: [snakeyaml]
---


> 一个`java`对象需要以`yaml`格式展示其内容

#### 1. snakeyaml序列化

最简单的代码：

```java
new Yaml().dump(user)
```

结果：
```
!!com.fool.User {address: null, age: 30,
  loginTime: !!timestamp '2019-04-19T02:23:41.971Z', name: Tomcat}
```

> 格式太丑陋，需要美化

#### 2. 美化格式

```java
DumperOptions dumperOptions = new DumperOptions();
dumperOptions.setPrettyFlow(true);
new Yaml(dumperOptions).dump(user)
```

结果：

```
!!com.fool.User {
  address: null,
  age: 30,
  loginTime: !!timestamp '2019-04-19T02:25:27.384Z',
  name: Tomcat
}
```

> 还是不够美观，例如有大括号

#### 3. 去掉大括号

```java
DumperOptions dumperOptions = new DumperOptions();
dumperOptions.setDefaultFlowStyle(DumperOptions.FlowStyle.BLOCK);
dumperOptions.setPrettyFlow(true);
new Yaml(dumperOptions).dump(user)
```

结果：

```
!!com.fool.User
address: null
age: 30
loginTime: 2019-04-19T02:27:42.050Z
name: Tomcat
```

> 类名`!!com.fool.User`应该去掉

#### 4. 去掉class name

```java
DumperOptions dumperOptions = new DumperOptions();
dumperOptions.setDefaultFlowStyle(DumperOptions.FlowStyle.BLOCK);
dumperOptions.setPrettyFlow(true);
new Yaml(dumperOptions).dumpAsMap(user)
```

结果：

```
address: null
age: 30
loginTime: 2019-04-19T02:27:42.050Z
name: Tomcat
```

> `null`属性可以直接省略

#### 5. 忽略null属性

```java
Representer representer = new Representer() {
    @Override
    protected NodeTuple representJavaBeanProperty(Object javaBean, Property property, Object propertyValue, Tag customTag) {
        if (propertyValue == null) {
            return null;
        }
        return super.representJavaBeanProperty(javaBean, property, propertyValue, customTag);
    }
};
DumperOptions dumperOptions = new DumperOptions();
dumperOptions.setDefaultFlowStyle(DumperOptions.FlowStyle.BLOCK);
dumperOptions.setPrettyFlow(true);
new Yaml(representer, dumperOptions).dumpAsMap(user)
```

结果：

```
age: 30
loginTime: 2019-04-19T02:27:42.050Z
name: Tomcat
```

#### 6. utils方法

```java
public class YamlUtils {

    private static Yaml yaml;

    static {
        Representer representer = new Representer() {
            @Override
            protected NodeTuple representJavaBeanProperty(Object javaBean, Property property, Object propertyValue, Tag customTag) {
                if (propertyValue == null) {
                    return null;
                }
                return super.representJavaBeanProperty(javaBean, property, propertyValue, customTag);
            }
        };
        DumperOptions dumperOptions = new DumperOptions();
        dumperOptions.setDefaultFlowStyle(DumperOptions.FlowStyle.BLOCK);
        dumperOptions.setPrettyFlow(true);
        yaml = new Yaml(representer, dumperOptions);
    }

    public static <T> String toYamlString(T obj) {
        if(obj == null){
            return "";
        }
        return yaml.dumpAsMap(obj);
    }

}
```

#### 参考

[SnakeYAML](https://bitbucket.org/asomov/snakeyaml)