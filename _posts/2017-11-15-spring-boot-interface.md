---
layout: post
title: spring-boot中的重要接口和注解
categories: [编程, java, spring]
tags: [spring-boot]
---

@EnableAutoConfiguration
@SpringBootApplication 
@ImportResource
@Import
@ConditionalOnClass
@Conditional
@ConditionalOnMissingBean 
@ConditionalOnProperty 
@ConditionalOnResource 
@ConditionalOnWebApplication 
@ConditionalOnExpression 


SpringApplication 
ApplicationContextInitializer
SpringApplicationRunListener

ApplicationArguments
ConfigurableEnvironment
FailureAnalyzers

ApplicationStartingEvent 
ApplicationEnvironmentPreparedEvent 
ApplicationPreparedEvent 
ApplicationReadyEvent 
ApplicationFailedEvent 

AnnotationConfigEmbeddedWebApplicationContext
AnnotationConfigApplicationContext

junit setWebEnvironment(false)


运行参数--debug

META-INF/spring.factories

PropertySource order

spring.application.json
random

spring.profile.active
application-{profile}.properties

YamlPropertiesFactoryBean 
YamlMapFactoryBean 
YamlPropertySourceLoader 

application.yaml profiles

@ConfigurationProperties(prefix="my")

relaxed binding
ConversionService 
CustomEditorConfigurer 
@ConfigurationPropertiesBinding

logging.file
logging.path
logging.level
loging.config

<springProfile>

WebMvcConfigurerAdapter
WebMvcRegistrationsAdapter 

HttpMessageConverters 
@JsonComponent 
static content

template engines
cors


@ServletComponentScan
@WebServlet
@WebFilter
@WebListener

EmbeddedServletContainerCustomizer 
EmbeddedServletContainerCustomizer 
ConfigurableEmbeddedServletContainer


SecurityAutoConfiguration 
@EnableWebSecurity 
WebSecurityConfigurerAdapter

