---
name: spring-properties
title: Spring 中无处不在的 Properties
date: 2021-02-04 14:14:19
tags: spring-properties
categories: 
---
对 Spring 里面的 Properties 不理解的开发者可能会觉得有点乱，主要是因为配置方式很多种，使用方式也很多种。

本文不是原理分析、源码分析文章，只是希望可以帮助读者更好地理解和使用 Spring Properties。



<!-- toc -->

## Properties 的使用

本文的读者都是使用过 Spring 的，先来看看 Properties 是怎么使用的，Spring 中常用的有以下几种使用方式：

### 1. 在 xml 配置文件中使用

即自动替换 `${}` 里面的值。

```xml
<bean id="xxx" class="com.javadoop.Xxx">
  	<property name="url" value="${javadoop.jdbc.url}" />
</bean>
```

### 2. 通过 @Value 注入使用

```java
@Value("${javadoop.jdbc.url}")
private String url;
```

### 3. 通过 Environment 获取

此法有需要注意的地方。并不是所有的配置方式都支持通过 Environment 接口来获取属性值，亲测只有使用注解 @PropertySource 的时候可以用，否则会得到 **null**，至于怎么配置，下面马上就会说。

```java
@Autowired
private Environment env;

public String getUrl() {
    return env.getProperty("javadoop.jdbc.url");
}
```

> 如果是 Spring Boot 的 application.properties 注册的，那也是可以的。

## Properties 配置

前面我们说了怎么使用我们配置的 Properties，那么该怎么配置呢？Spring 提供了很多种配置方式。

### 1. 通过 xml 配置

下面这个是最常用的配置方式了，很多项目都是这么写的：

```xml
<context:property-placeholder location="classpath:sys.properties" />
```

### 2. 通过 @PropertySource 配置

前面的通过 xml 配置非常常用，但是如果你也有一种要**消灭所有 xml 配置文件**的冲动的话，你应该使用以下方式：

```java
@PropertySource("classpath:sys.properties")
@Configuration
public class JavaDoopConfig {
    
}
```

注意一点，@PropertySource 在这里必须搭配 @Configuration 来使用，具体不展开说了。

### 3. PropertyPlaceholderConfigurer

如果读者见过这个，也不必觉得奇怪，在 Spring 3.1 之前，经常就是这么使用的：

```xml
<bean class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
    <property name="locations">
        <list>
            <value>classpath:sys.properties</value>
        </list>
    </property>
    <property name="ignoreUnresolvablePlaceholders" value="true"/>
  	<!-- 这里可以配置一些属性 -->
</bean>
```

当然，我们也可以用相应的 java configuration 的版本：

```java
@Bean
public PropertyPlaceholderConfigurer propertiess() {
    PropertyPlaceholderConfigurer ppc = new PropertyPlaceholderConfigurer();
    Resource[] resources = new ClassPathResource[]{new ClassPathResource("sys.properties")};
    ppc.setLocations(resources);
    ppc.setIgnoreUnresolvablePlaceholders(true);
    return ppc;
}
```

### 4. PropertySourcesPlaceholderConfigurer

到了 Spring 3.1 的时候，引入了 **PropertySourcesPlaceholderConfigurer**，这是一个新的类，注意看和之前的 PropertyPlaceholderConfigurer 在名字上多了一个 **Sources**，所属的包也不一样，它在 Spring-Context 包中。

在配置上倒是没有什么区别：

```xml
<bean class="org.springframework.context.support.PropertySourcesPlaceholderConfigurer">
    <property name="locations">
        <list>
            <value>classpath:sys.properties</value>
        </list>
    </property>
    <property name="ignoreUnresolvablePlaceholders" value="true"/>
    <!-- 这里可以配置一些属性 -->
</bean>
```

也来一个 java configuration 版本吧：

```java
@Bean
public PropertySourcesPlaceholderConfigurer properties() {
    PropertySourcesPlaceholderConfigurer pspc = new PropertySourcesPlaceholderConfigurer();
    Resource[] resources = new ClassPathResource[]{new ClassPathResource("sys.properties")};
    pspc.setLocations(resources);
    pspc.setIgnoreUnresolvablePlaceholders(true);
    return pspc;
}
```

## Spring Boot 相关

Spring Boot 真的是好东西，开箱即用的感觉实在是太好了。这里简单介绍下相关的内容。

快速生成一个 Spring Boot 项目：https://start.spring.io/

### application.properties

我们每个项目都默认有一个 application.properties 文件，这个配置文件不需要像前面说的那样进行*注册*，Spring Boot 会帮我们**自动注册**。

当然，也许你想换个名字也是可以的，在启动的时候指定你的文件名字就可以了：

```shell
java -Dspring.config.location=classpath:sys.properties -jar app.jar
```

### application-{env}.properties

为了给不同的环境指定不同的配置，我们会用到这个。

比如测试环境和生产环境的数据库连接信息就不一样。

所以，在 application.properties 的基础上，我们还需要新建 application-dev.properties 和 application-prd.properties，用于配置环境相关的信息，然后启动的时候指定环境。

```shell
java -Dspring.profiles.active=prd -jar app.jar
```

结果就是，application.properties 和 application-prd.properties 两个文件中的配置都会注册进去，如果有重复的 key，application-prd.properties 文件中的优先级较高。

### @ConfigurationProperties

这个注解是 Spring Boot 中才有的。

即使大家不使用这个注解，大家也可能会在开源项目中看到这个，这里简单介绍下。

来一个例子直观一些。按照之前说的，在配置文件中填入下面的信息，你可以选择写入 application.properties 也可以用第一节介绍的方法。

```properties
javadoop.database.url=jdbc:mysql:
javadoop.database.username=admin
javadoop.database.password=admin123456
```

java 文件：

```java
@Configuration
@ConfigurationProperties(prefix = "javadoop.database")
public class DataBase {
    String url;
    String username;
    String password;
    // getters and setters
}
```

这样，就在 Spring 的容器中就自动注册了一个类型为 DataBase 的 bean 了，而且属性都已经 set 好了。

### 在启动过程中动态修改属性值

这个我觉得都不需要太多介绍，用 Spring Boot 的应该基本上都知道。

属性配置有个覆盖顺序，也就是当出现相同的 key 的时候，以哪里的值为准。

**启动参数 > application-{env}.properties > application.properties**

**启动参数动态设置属性：**

```shell
java -Djavadoop.database.password=admin4321 -jar app.jar
```

另外，还可以利用系统环境变量设置属性，还可以指定随机数等等，确实很灵活，不过没什么用，就不介绍了。

## 总结

读者如果想要更加深入地了解 Spring 的 Properties，需要去理解 Spring 的 Environment 接口相关的源码。建议感兴趣的读者去翻翻源代码看看

（全文完）