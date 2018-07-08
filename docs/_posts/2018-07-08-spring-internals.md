---
layout: post
title: Spring internals
---
# Spring internals

## Simple lifecycle

- Initialization
  - Spring Beans are created
  - Dependency Injection occurs
- Usage
  -Beans are available for use in the application
- Destruction
  -Beans are released for Garbage Collection

## Initialization

- **Load Bean Definition**
  - `@Configuration` classes processed
  - `@Components` annotation scanned for
  - XML files parsed
  - bean definitions added to `BeanFactory` - indexed by id and type
- **Post Process Bean Definition**
  - `BeanFactoryPostProcessor` beans invoced on bean definitions (able to **update definitions**)
    - `void	postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory)` method signature
    - declre by `@Component` class annotation or `@Bean` **static** method annotation
    - spring implementations reads properties, registaring scopes...
- for each bean
  - Find and create dependences
  - Instattiate Bean (constructor dependenced injection)
  - Call Setters (injection)
  - **Bean Post Processors**
    - BPP - before init
      - `public interface BeanPostProcessor` mathod `public Object postProcessBeforeInitialization(Object bean, String beanName);`
    - Initializer
    - BPP - after init
      - `public interface BeanPostProcessor` mathod `public Object postProcessAfterInitialization(Object bean, String beanName);`
    - Example: `CommonAnnotationBeanPostProcessor` enables `@PostConstruct`, `@Resource`
    - bean **proxy** is added here
- Bean ready

## `LoggingBeanPostProcessor`

```java
@Component
public class LoggingBeanPostProcessor implements BeanPostProcessor {
    Logger logger = Logger.getLogger(LoggingBeanPostProcessor.class);

    public Object postProcessBeforeInitialization(Object bean, String beanName) { 
        return bean; // Remember to return your bean or you'll lose it!
    }

    public Object postProcessAfterInitialization(Object bean,String beanName) { 
        logger.info(beanName + ": " + bean.getClass() );
        return bean; // Remember to return your bean or you'll lose it!
   }
}
```

## Proxy

JDK Proxy | CGLib Proxy
--- | ---
Also called dynamic proxies     | NOT built into JDK
API is built into the JDK       | Included in Spring jars
Requirements: Java interface(s) | Used when interface not available
All interfaces proxied          | **Cannot** be applied to **final** classes or methods

[https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop-pfb-proxy-types]