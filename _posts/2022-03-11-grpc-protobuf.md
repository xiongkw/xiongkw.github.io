---
layout: post
title: grpc和protobuf
categories: [编程, java]
tags: [grpc, protobuf]
---

>  

#### 1. 概念
ProtoBuf：一种结构数据序列化方法，类似xml、json，Google出品，特点：
* 语言无关、平台无关
* 高效，比 XML 更小（3 ~ 10倍）、更快（20 ~ 100倍），更简单
* 扩展性、兼容性好，可以更新数据结构，而不影响和破坏原有程序

grpc：一个高性能RPC框架，Google出品，特点：
* 可以通过protobuf来定义接口，从而可以有更加严格的接口约束条件
* 通过protobuf可以将数据序列化为二进制编码，这会大幅减少需要传输的数据量，从而大幅提高性能
* 基于HTTP/2协议，可以方便地支持流式通信

#### 2. 手撸

maven工程结构
```
grpc-demo    // 父工程
    |-api    // api模块
    |-client // client模块
    |-server // server模块
```

##### 2.1 定义protobuf

在api模块定义proto文件

src/main/proto/user_dto.proto
```protobuf
//声明protobuf版本为proto3
syntax = "proto3";

//消息实体生成包路径
package com.grpc.demo.api.user.dto;

//如果为true时message会生成多个类
option java_multiple_files = true;

//查询列表请求实体
message SearchUserRequest {
  int32 id = 1;
  string name = 2;
  int32 age =3;
  string address = 4;
}

//返回用户信息实体
message UserResponse {
  int32 id = 1;
  string name = 2;
  int32 age =3;
  string address = 4;
}

//添加用户实体
message AddUserRequest {
  string name = 1;
  int32 age =2;
  string address = 3;
}
```

src/main/proto/user_service.proto
```protobuf
//声明protobuf版本为proto3
syntax = "proto3";

//引入实体定义
import "user_dto.proto";

//如果为true时message会生成多个类
option java_multiple_files = true;

//此处要注意和user_dto.proto相同包，否则找不到实体
package com.grpc.demo.api.user.dto;

//服务定义生成的包
option java_package = "com.grpc.demo.api.user.service";

//指定生成Java的类名，如果没有该字段则根据proto文件名称以驼峰的形式生成类名
option java_outer_classname = "UserProto";

//服务定义
service User {

  //查询用户列表，以流式返回多个对象
  rpc list (SearchUserRequest) returns (stream UserResponse) {}

  //添加用户信息，返回单个对象
  rpc add (AddUserRequest) returns (UserResponse) {}
}
```

##### 2.2 生成java文件

使用Maven Protocol Buffers Plugin插件生成protobuf和grpc的java源码
```xml
<dependencies>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-stub</artifactId>
        <version>1.45.1</version>
    </dependency>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-protobuf</artifactId>
        <version>1.45.1</version>
    </dependency>

</dependencies>

<build>
    <extensions>
        <extension>
            <groupId>kr.motd.maven</groupId>
            <artifactId>os-maven-plugin</artifactId>
            <version>1.6.2</version>
        </extension>
    </extensions>

    <plugins>
        <plugin>
            <groupId>org.xolstice.maven.plugins</groupId>
            <artifactId>protobuf-maven-plugin</artifactId>
            <version>0.6.1</version>
            <configuration>
                <protocArtifact>com.google.protobuf:protoc:3.12.0:exe:${os.detected.classifier}</protocArtifact>
                <pluginId>grpc-java</pluginId>
                <pluginArtifact>io.grpc:protoc-gen-grpc-java:1.31.1:exe:${os.detected.classifier}</pluginArtifact>
                <clearOutputDirectory>false</clearOutputDirectory>
                <outputDirectory>./src/main/java</outputDirectory>
            </configuration>
            <executions>
                <execution>
                    <goals>
                        <goal>compile</goal>
                        <goal>compile-custom</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

执行命令生成java源文件
```
$ mvn protobuf:compile
$ mvn protobuf:compile-custom
```

##### 2.3 编写server

添加maven依赖
```xml
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-netty-shaded</artifactId>
    <version>1.45.1</version>
</dependency>

<dependency>
    <groupId>com.grpc.demo</groupId>
    <artifactId>api</artifactId>
    <version>1.0-SNAPSHOT</version>
</dependency>
```

编写服务端代码，并运行
```java
public class Server {
    public static void main(String[] args) throws Exception {
        io.grpc.Server server = ServerBuilder.forPort(8081)
                .addService(new UserGrpc.UserImplBase() {
                    @Override
                    public void list(SearchUserRequest request, StreamObserver<UserResponse> responseObserver) {
                        System.out.println(request.getName());
                        UserResponse response = UserResponse.newBuilder().setId(1).setName("ZhangSan").setAge(30).build();
                        responseObserver.onNext(response);
                        response = UserResponse.newBuilder().setId(2).setName("LiSi").setAge(30).build();
                        responseObserver.onNext(response);
                        responseObserver.onCompleted();
                    }
                }).build().start();
        server.awaitTermination();
    }
}
```

##### 2.4 编写client
添加maven依赖
```xml
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-netty-shaded</artifactId>
    <version>1.45.1</version>
</dependency>

<dependency>
    <groupId>com.grpc.demo</groupId>
    <artifactId>api</artifactId>
    <version>1.0-SNAPSHOT</version>
</dependency>
```

编写client调用代码，并运行
```java
public class Client {
    public static void main(String[] args) {
        ManagedChannel channel = ManagedChannelBuilder.forTarget("127.0.0.1:8081")
                .usePlaintext().build();
        UserGrpc.UserBlockingStub stub = UserGrpc.newBlockingStub(channel);
        SearchUserRequest request = SearchUserRequest.newBuilder().setName("zhang").build();
        Iterator<UserResponse> response = stub.list(request);
        while (response.hasNext()) {
            UserResponse next = response.next();
            System.out.println(next.getId() + "-" + next.getName());
        }
    }
}
```

#### 3. 集成springboot
使用springboot starter可以很方便的集成到springboot工程

##### 3.1 server
添加maven依赖
```xml
<dependency>
    <groupId>net.devh</groupId>
    <artifactId>grpc-server-spring-boot-starter</artifactId>
    <version>2.13.1.RELEASE</version>
</dependency>

<dependency>
    <groupId>com.grpc.demo</groupId>
    <artifactId>api</artifactId>
    <version>1.0-SNAPSHOT</version>
</dependency>
```

编写service代码
```java
@GrpcService
public class UserServiceGrpcImpl extends UserGrpc.UserImplBase {
    @Override
    public void list(SearchUserRequest request, StreamObserver<UserResponse> responseObserver) {
        System.out.println(request.getName());
        UserResponse response = UserResponse.newBuilder().setId(1).setName("ZhangSan").setAge(30).build();
        responseObserver.onNext(response);
        response = UserResponse.newBuilder().setId(2).setName("LiSi").setAge(30).build();
        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }
}
```

配置grpc服务端口
```yaml
grpc:
  server:
    port: 9091
```

##### 3.2 client
添加maven依赖
```xml
<dependency>
    <groupId>net.devh</groupId>
    <artifactId>grpc-client-spring-boot-starter</artifactId>
    <version>2.13.1.RELEASE</version>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <version>2.6.7</version>
</dependency>
<dependency>
    <groupId>com.grpc.demo</groupId>
    <artifactId>api</artifactId>
    <version>1.0-SNAPSHOT</version>
</dependency>
```

编写controller，并调用rpc服务
```java
@RestController
public class UserController {
    @GrpcClient("userClient")
    private UserGrpc.UserBlockingStub stub;

    @GetMapping("/users")
    public void list() {
        SearchUserRequest request = SearchUserRequest.newBuilder().setName("zhang").build();
        Iterator<UserResponse> response = stub.list(request);
        while (response.hasNext()) {
            UserResponse next = response.next();
            System.out.println(next.getId() + "-" + next.getName());
        }
    }
}
```

配置userClient
```yaml
grpc:
  client:
    userClient:
      negotiationType: PLAINTEXT
      address: static://localhost:9091
```

#### 4. 参考

* [Protocol Buffers](https://github.com/protocolbuffers/protobuf)
* [Introduction to gRPC](https://grpc.io/docs/what-is-grpc/introduction/)
* [Maven Protocol Buffers Plugin](https://www.xolstice.org/protobuf-maven-plugin/index.html)
* [gRPC Spring Boot Starter](https://github.com/yidongnan/grpc-spring-boot-starter)