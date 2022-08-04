---
layout: post
title: MapStruct试用
categories: [Java]
tags: []
---

> 为了减少手写转换的编码工作，常常使用`BeanUtils.copyProperties`来转换PO和DTO。由于`BeanUtils.copyProperties`是基于反射实现，所以性能肯定比不上静态编译的MapStruct

#### 1. 添加依赖

修改pom.xml

```
	<dependency>
		<groupId>org.projectlombok</groupId>
		<artifactId>lombok</artifactId>
		<version>1.18.24</version>
	</dependency>

	<dependency>
		<groupId>org.mapstruct</groupId>
		<artifactId>mapstruct</artifactId>
		<version>1.5.2.Final</version>
	</dependency>
```

#### 2. 编写PO和DTO

UserPO.java

```java
@Getter
@Setter
@ToString
public class UserPO {
    private String name;
    private String sex;
    private int age;
}
```

UserDTO.java

```java
@Getter
@Setter
@ToString
public class UserDTO {
    private String name;
    private String gender;
    private int age;
}
```

#### 3. 编写Mapper接口

UserMapper.java

```java
@Mapper
public interface UserMapper {
    UserMapper INSTANCE = Mappers.getMapper(UserMapper.class);

    @Mapping(source = "sex",target = "gender")
    UserDTO toDTO(UserPO po);
}
```

#### 4. TestCase

```java
public static void main(String[] args) {
	UserPO po = new UserPO();
	po.setName("Tom");
	po.setAge(36);
	po.setSex("male");

	UserDTO dto = UserMapper.INSTANCE.toDTO(po);
	System.out.println(dto);
}
```

#### 5. 配置注解处理器

修改pom.xml

```
<build>
	<plugins>
		<plugin>
			<groupId>org.apache.maven.plugins</groupId>
			<artifactId>maven-compiler-plugin</artifactId>
			<version>3.8.1</version>
			<configuration>
				<source>1.8</source>
				<target>1.8</target>
				<annotationProcessorPaths>
					<path>
						<groupId>org.mapstruct</groupId>
						<artifactId>mapstruct-processor</artifactId>
						<version>1.5.2.Final</version>
					</path>
					<path>
						<groupId>org.projectlombok</groupId>
						<artifactId>lombok</artifactId>
						<version>1.18.24</version>
					</path>
					<path>
						<groupId>org.projectlombok</groupId>
						<artifactId>lombok-mapstruct-binding</artifactId>
						<version>0.2.0</version>
					</path>
				</annotationProcessorPaths>
			</configuration>
		</plugin>
	</plugins>
</build>
```

> 注意同时使用lombok会有冲突，需要配置lombok-mapstruct-binding处理器

#### 6. 参考

* [MapStruct](https://mapstruct.org/)
* [MapStruct 1.5.2.Final Reference Guide](https://mapstruct.org/documentation/stable/reference/html/)