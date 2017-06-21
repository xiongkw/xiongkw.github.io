---
layout: post
title: Spring load-time-weaver 源码分析
categories: [编程, spring, ltw, java]
tags: [spring, ltw]
---

> spring load-time-weaver 基于[Java Instrumentation]({{ site.url }}/2016/02/06/java-instrumentation/)实现

> 关于`weaver`，中文翻译为`织入`，常见的有编译时织入和类加载时织入，spring-ltw为类加载时织入

### 1. 从spring配置入手
```xml
<!-- 一行配置开启 -->
<context:load-time-weaver/>
```
> `context`是spring-context中定义的一个标签，参考[spring自定义命名空间]({{site.url}}/2011/05/11/spring-custom-namespace/)

根据spring命名空间规范，找到`LoadTimeWeaverBeanDefinitionParser.java`
```java
class LoadTimeWeaverBeanDefinitionParser extends AbstractSingleBeanDefinitionParser {

	@Override
	protected String getBeanClassName(Element element) {
		if (element.hasAttribute(WEAVER_CLASS_ATTRIBUTE)) {
			return element.getAttribute(WEAVER_CLASS_ATTRIBUTE);
		}
		//1. 默认weaver class: org.springframework.context.weaving.DefaultContextLoadTimeWeaver
		return DEFAULT_LOAD_TIME_WEAVER_CLASS_NAME;
	}

	@Override
	protected String resolveId(Element element, AbstractBeanDefinition definition, ParserContext parserContext) {
	    //2. bean id: loadTimeWeaver
		return ConfigurableApplicationContext.LOAD_TIME_WEAVER_BEAN_NAME;
	}

	@Override
	protected void doParse(Element element, ParserContext parserContext, BeanDefinitionBuilder builder) {
		builder.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);

		if (isAspectJWeavingEnabled(element.getAttribute(ASPECTJ_WEAVING_ATTRIBUTE), parserContext)) {
			RootBeanDefinition weavingEnablerDef = new RootBeanDefinition();
			//3. 注册一个org.springframework.context.weaving.AspectJWeavingEnabler bean
			weavingEnablerDef.setBeanClassName(ASPECTJ_WEAVING_ENABLER_CLASS_NAME);
			parserContext.getReaderContext().registerWithGeneratedName(weavingEnablerDef);

			if (isBeanConfigurerAspectEnabled(parserContext.getReaderContext().getBeanClassLoader())) {
				new SpringConfiguredBeanDefinitionParser().parse(element, parserContext);
			}
		}
	}

	protected boolean isAspectJWeavingEnabled(String value, ParserContext parserContext) {
		if ("on".equals(value)) {
			return true;
		}
		else if ("off".equals(value)) {
			return false;
		}
		else {
			//4. 自动模式下，根据classpath是否存在资源文件META-INF/aop.xml决定是否启用AspectJ weaving
			ClassLoader cl = parserContext.getReaderContext().getResourceLoader().getClassLoader();
			return (cl.getResource(AspectJWeavingEnabler.ASPECTJ_AOP_XML_RESOURCE) != null);
		}
	}

}

```

从上面代码看出: `<context:load-time-weaver/>`会定义两个bean(条件充分的情况下)
* DefaultContextLoadTimeWeaver
* AspectJWeavingEnabler

### 2. DefaultContextLoadTimeWeaver
源码：DefaultContextLoadTimeWeaver.java
```java
	public void setBeanClassLoader(ClassLoader classLoader) {
        //1. 根据运行环境获取 LoadTimeWeaver
		LoadTimeWeaver serverSpecificLoadTimeWeaver = createServerSpecificLoadTimeWeaver(classLoader);
		if (serverSpecificLoadTimeWeaver != null) {
			if (logger.isInfoEnabled()) {
				logger.info("Determined server-specific load-time weaver: " +
						serverSpecificLoadTimeWeaver.getClass().getName());
			}
			this.loadTimeWeaver = serverSpecificLoadTimeWeaver;
		}
		//2. 根据运行时容器获取不到，则使用java instrumentation
		else if (InstrumentationLoadTimeWeaver.isInstrumentationAvailable()) {
			logger.info("Found Spring's JVM agent for instrumentation");
			this.loadTimeWeaver = new InstrumentationLoadTimeWeaver(classLoader);
		}
		else {
			try {
			    //3. 还是无法获取LoadTimeWeaver，则使用反射的实现
				this.loadTimeWeaver = new ReflectiveLoadTimeWeaver(classLoader);
				logger.info("Using a reflective load-time weaver for class loader: " +
						this.loadTimeWeaver.getInstrumentableClassLoader().getClass().getName());
			}
			catch (IllegalStateException ex) {
				throw new IllegalStateException(ex.getMessage() + " Specify a custom LoadTimeWeaver or start your " +
						"Java virtual machine with Spring's agent: -javaagent:org.springframework.instrument.jar");
			}
		}
	}
	
	//4. Spring为常用运行容器提供了LoadTimeWeaver实现
    protected LoadTimeWeaver createServerSpecificLoadTimeWeaver(ClassLoader classLoader) {
        String name = classLoader.getClass().getName();
        try {
            if (name.startsWith("weblogic")) {
                return new WebLogicLoadTimeWeaver(classLoader);
            }
            else if (name.startsWith("org.glassfish")) {
                return new GlassFishLoadTimeWeaver(classLoader);
            }
            else if (name.startsWith("org.apache.catalina")) {
                return new TomcatLoadTimeWeaver(classLoader);
            }
            else if (name.startsWith("org.jboss")) {
                return new JBossLoadTimeWeaver(classLoader);
            }
            else if (name.startsWith("com.ibm")) {
                return new WebSphereLoadTimeWeaver(classLoader);
            }
        }
        catch (IllegalStateException ex) {
            logger.info("Could not obtain server-specific LoadTimeWeaver: " + ex.getMessage());
        }
        return null;
    }
    
    public void addTransformer(ClassFileTransformer transformer) {
        //5. addTransformer方法实际是调用了loadTimeWeaver属性的addTransformer方法
        this.loadTimeWeaver.addTransformer(transformer);
    }
```
LoadTimeWeaver反射实现，通过反射获取ClassLoader的addTransformer方法
```java
public ReflectiveLoadTimeWeaver(ClassLoader classLoader) {
    Assert.notNull(classLoader, "ClassLoader must not be null");
    this.classLoader = classLoader;
    this.addTransformerMethod = ClassUtils.getMethodIfAvailable(
            this.classLoader.getClass(), ADD_TRANSFORMER_METHOD_NAME,
            new Class<?>[] {ClassFileTransformer.class});
    if (this.addTransformerMethod == null) {
        throw new IllegalStateException(
                "ClassLoader [" + classLoader.getClass().getName() + "] does NOT provide an " +
                "'addTransformer(ClassFileTransformer)' method.");
    }
    this.getThrowawayClassLoaderMethod = ClassUtils.getMethodIfAvailable(
            this.classLoader.getClass(), GET_THROWAWAY_CLASS_LOADER_METHOD_NAME, new Class<?>[0]);
    // getThrowawayClassLoader method is optional
    if (this.getThrowawayClassLoaderMethod == null) {
        if (logger.isInfoEnabled()) {
            logger.info("The ClassLoader [" + classLoader.getClass().getName() + "] does NOT provide a " +
                    "'getThrowawayClassLoader()' method; SimpleThrowawayClassLoader will be used instead.");
        }
    }
}
```

### 3. AspectJWeavingEnabler
AspectJWeavingEnabler.java
```java
    //1. 实现了BeanFactoryPostProcessor接口，在bean初始化之前把字节码转换器加到loadTimeWeaver(实际根据LoadTimeWeaver不同可能加到了instrumentation或者ClassLoader)
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
		enableAspectJWeaving(this.loadTimeWeaver, this.beanClassLoader);
	}

	public static void enableAspectJWeaving(LoadTimeWeaver weaverToUse, ClassLoader beanClassLoader) {
		if (weaverToUse == null) {
			if (InstrumentationLoadTimeWeaver.isInstrumentationAvailable()) {
				weaverToUse = new InstrumentationLoadTimeWeaver(beanClassLoader);
			}
			else {
				throw new IllegalStateException("No LoadTimeWeaver available");
			}
		}
		weaverToUse.addTransformer(new AspectJClassBypassingClassFileTransformer(
					new ClassPreProcessorAgentAdapter()));
	}
```
AspectJClassBypassingClassFileTransformer.java
```java
public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined,
        ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
    //2. 这里过滤需要转换的类，如果要自定义过滤条件，就要从这里入手了
    if (className.startsWith("org.aspectj") || className.startsWith("org/aspectj")) {
        return classfileBuffer;
    }
    return this.delegate.transform(loader, className, classBeingRedefined, protectionDomain, classfileBuffer);
}
```

### 4. LoadTimeWeaverAwareProcessor和LoadTimeWeaverAware
AbstractApplicationContext.java
```java
	protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        ...
		//1. Detect a LoadTimeWeaver and prepare for weaving, if found.
		if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
            //2. LoadTimeWeaverAware即是在这里回调
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			//3. Set a temporary ClassLoader for type matching.
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}
        ...
	}
	
	protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
        ...
        // Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
        //4. 在其它bean初始化之前就已经完成字节码转换器的初始化
        String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
        for (String weaverAwareName : weaverAwareNames) {
            getBean(weaverAwareName);
        }

        // Stop using the temporary ClassLoader for type matching.
        beanFactory.setTempClassLoader(null);
        ...
    }
```
这里有一个temporary ClassLoader 见beanFactory.setTempClassLoader文档：
> Specify a temporary ClassLoader to use for type matching purposes. Default is none, simply using the standard bean ClassLoader.

### 5.总结：
整个流程串起来就是：

- 通过`<context:load-time-weaver/>`开启load-time-weaver，注册了DefaultContextLoadTimeWeaver和AspectJWeavingEnabler两个bean
- prepareBeanFactory中通过LoadTimeWeaverAwareProcessor给AspectJWeavingEnabler注入loadTimeWeaver，loadTimeWeaver也是在此过程中初始化
- DefaultContextLoadTimeWeaver的setBeanClassLoader中注入spring 的 `BeanClassLoader`，织入功能最终便是通过这个ClassLoader实现
- AspectJWeavingEnabler在postProcessBeanFactory中给loadTimeWeaver注入字节码转换器AspectJClassBypassingClassFileTransformer
- spring容器使用已经增强过的`BeanClassLoader`初始化业务类bean，至此便完成了织入功能

### 6. 附：[Spring load-time-weaver 用法]({{site.url}}/2016/02/04/spring-ltw-useage)中的遗留问题
在如下的写法中，MyService类是不会被织入的
```java
@ContextConfiguration(locations = { "classpath*:aop-weaver.xml" })
@RunWith(SpringJUnit4ClassRunner.class)
public class AopAspectjweaverTest {
	@Autowired
	private MyService service;

	@Test
	public void test() {
		service.doPrivateService();
	}

}
```
原因：MyService class的加载发生在spring load-time-weaver初始化之前，改进方法

1.使用接口声明注入
```java
	@Autowired
	private IService service;

```

2.在spring context初始化完成后再声明
```java
MyService service = context.getBean(MyService.class);
```