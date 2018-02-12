---
layout: post
title: spring-mvc中controller实现自定义参数映射
categories: [编程, java, spring, web]
tags: [spring-mvc, json]
---


> `spring-mvc`中`Get`方法的参数可直接映射到`dto`，`Post`方法中也可通过`jackson`把`json`参数映射到`dto`，但是`Get`方法中却不支持`json`参数的映射

需求：形如`/getJson?dto={"name":"Tom","age":20}`的`HTTP`请求，要求映射到`Get`方法的`DTO`参数中
> 需求比较奇葩，这里只作演示

`rest controller`
```java
@RequestMapping(value = "jsonparam", method = RequestMethod.GET)
public UserDTO jsonParam(@JSONParam UserDTO dto){
    return dto;
}
```

注解:`JSONParam.java`
```java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface JSONParam {

}
```

`GetParamArgumentResolver.java`
```java
public class GetParamArgumentResolver implements HandlerMethodArgumentResolver {

	@Override
	public boolean supportsParameter(MethodParameter parameter) {
		return parameter.hasParameterAnnotation(JSONParam.class);
	}

	@Override
	public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
		String parameterName = parameter.getParameterName();
		String value = webRequest.getParameter(parameterName);
		ObjectMapper mapper = new ObjectMapper();
		return mapper.readValue(value, parameter.getParameterType());
	}

}
```

`spring-mvc.xml`
```xml
<mvc:annotation-driven>
    <mvc:message-converters>
        <ref bean="stringHttpMessageConverter" />
        <ref bean="mappingJacksonHttpMessageConverter" />
    </mvc:message-converters>
    <mvc:argument-resolvers>
        <ref bean="jsonParamArgumentResolver"/>
    </mvc:argument-resolvers>
</mvc:annotation-driven>

<bean id="jsonParamArgumentResolver" class="com.my.GetParamArgumentResolver"/>
```