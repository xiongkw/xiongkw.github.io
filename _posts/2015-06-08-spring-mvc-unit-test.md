---
layout: post
title: Spring mvc中对controller层做单元测试
categories: [编程, java, spring, web]
tags: [unit-test, spring-mvc]
---

> 随着`Restful Web Service`的兴起，对`Web Service`做单元测试也变得非常必要，从`spring-3.2`开始提供了`Spring Web`测试框架

#### 1. 依赖包

加入`maven`依赖

```xml
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>${junit.version}</version>
    <scope>test</scope> 
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-test</artifactId>
    <version>${spring.version}</version>
    <scope>test</scope>
</dependency>
```

#### 2. 编写controller

`controller`

```java
@RestController
public class EchoController {

	@RequestMapping("/echo")
	public Object set(@RequestParam String name) throws IOException {
		return "Echo: "+name;
	}
	
	@RequestMapping("/set")
    public Object set(@RequestParam String name, HttpSession session) throws IOException {
        session.setAttribute("name", name);
        return session.getId();
    }

    @RequestMapping("/get")
    public Object get(HttpSession session) throws IOException {
        return session.getAttribute("name");
    }

}

```

#### 3. 编写单元测试

单元测试代码

```java
@WebAppConfiguration(value = "src/test/webapp")
@ContextHierarchy({ @ContextConfiguration(locations = "classpath:conf/spring-mvc.xml"),
		@ContextConfiguration(locations = "classpath:conf/applicationContext.xml") })
@RunWith(SpringJUnit4ClassRunner.class)
public class EchoControllerTest {
	@Autowired
	private WebApplicationContext context;

	private MockMvc mockMvc;

	@Before
	public void setup() {
		mockMvc = MockMvcBuilders.webAppContextSetup(context).build();
	}

	@Test
	public void testEcho() throws Exception {
	    //测试普通Get请求
		MvcResult result = mockMvc.perform(get("/echo?name=tomcat")).andReturn();
		String content = result.getResponse().getContentAsString();
		assertEquals("tomcat", content);
	}
	
	@Test
	public void testSession() throws Exception {
		MvcResult andReturn = mockMvc.perform(get("/set?name=tomcat")).andReturn();
		//获取Cookie并加入下一次请求，用于测试会话相关的多个请求
        Cookie[] cookies = andReturn.getResponse().getCookies();
        MvcResult result = mockMvc.perform(get("/get").cookie(cookies)).andReturn();
        String content = result.getResponse().getContentAsString();
        assertEquals("tomcat", content);
	}

}
```