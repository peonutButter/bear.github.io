---
layout:     post                    # 使用的布局（不需要改）
title:      SpringBoot执行流程               # 标题 
subtitle:     #副标题
date:       2019-01-04              # 时间
author:     BY                      # 作者
catalog: true                       # 是否归档
tags:                               #标签
    - Spring Boot
---
# SpringBoot 工作机制
​          **SpringBoot 框架是Spring框架对“约定优先于配置”理念的最佳产物**。

## 1 . @SpringBootApplication

​      @SpringBootApplication 是一个“三体”结构，为一个复合注解：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
```

其中最为主要的三个注解为： 

* @SpringBootConfiguration

* @EnableAutoConfiguration

* @ComponentScan   

  其中@SpringBootConfiguration 本质就是@Configuration，将自己作为一个Spring容器的配置类。

  ```java
  @Target(ElementType.TYPE)
  @Retention(RetentionPolicy.RUNTIME)
  @Documented
  @Configuration
  public @interface SpringBootConfiguration {
  ```

  

## 2. @EnableAutoConfiguration

​	spring框架提供了许多以@Enable 开头的Annotation，我们可以称为激活模式。主要是借助@Import的支持，收集和注册特定场景的Bean定义（后面会讲到）。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
```

其中，最关键的@Import(AutoConfigurationImportSelector.class)，

AutoConfigurationImportSelector帮助Spring将所有符合条件的@Configuration都加载到当前Spring容器中，借助于SpringFactoriesLoader 。

## 3. SpringFactoriesLoader 

   SpringFactoriesLoader 属于Spring框架私有的一种扩展方案，其主要的作用为加载指定的配置文件META-INF/spring.factories中的配置。spring.factories为properties文件。所以，@EnableAutoConfiguration自动配置就是：从classpath中找出所有的META-INF/spring.factories配置文件，并将其中EnableAutoConfiguration对应的配置通过反射实例化为对应的标注了@Configuration的Spring容器配置类。



```java
public abstract class SpringFactoriesLoader {
    ....
// 配置类的位置
public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";
	....
private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
		MultiValueMap<String, String> result = cache.get(classLoader);
		if (result != null) {
			return result;
		}

		try {
			Enumeration<URL> urls = (classLoader != null ?
					classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
					ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
			result = new LinkedMultiValueMap<>();
			while (urls.hasMoreElements()) {
				URL url = urls.nextElement();
				UrlResource resource = new UrlResource(url);
				Properties properties = PropertiesLoaderUtils.loadProperties(resource);
				for (Map.Entry<?, ?> entry : properties.entrySet()) {
					List<String> factoryClassNames = Arrays.asList(
							StringUtils.commaDelimitedListToStringArray((String) entry.getValue()));	// 得到所有配置的全路径类名
					result.addAll((String) entry.getKey(), factoryClassNames);
				}
			}
			cache.put(classLoader, result);
			return result;
		}
		catch (IOException ex) {
			throw new IllegalArgumentException("Unable to load factories from location [" +
					FACTORIES_RESOURCE_LOCATION + "]", ex);
		}
	}
```



​	对于@ComponentScan    这里不做多的讲解，功能为XML配置是相同的。

## 4. SpringApplication执行流程

1) .    在我们启动SpringApplication的run()方法时，首先会创建一个SpringApplication实例，在SpringApplication初始化的时候会做：

* 根据classpath 里面是否存在某个特征类（ConfigurableWebApplicationContext）来决定是否应该创建一个为WEB应用的Application类型。
* 使用SpringFactoriesLoader 在classpath中查找并加载所有可用的ApplicationContexInitializer 
* 使用SpringFactoriesLoader 在classpath中查找并加载所有可用的ApplicationListener
* 推断并设置main方法的定义类



```java
public static ConfigurableApplicationContext run(Class<?>[] primarySources,
      String[] args) {
   return new SpringApplication(primarySources).run(args);
}
```





2) .  开始执行run方法，通过SpringFactoriesLoader加载SpringApplicationRunListener，并调用。

```java
public ConfigurableApplicationContext run(String... args) {
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
		configureHeadlessProperty();
    //  加载SpringApplicationRunListener
		SpringApplicationRunListeners listeners = getRunListeners(args);
    // 调用
		listeners.starting();
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(
					args);
            
            .......
```

```java
private SpringApplicationRunListeners getRunListeners(String[] args) {
		Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
		return new SpringApplicationRunListeners(logger, getSpringFactoriesInstances(
				SpringApplicationRunListener.class, types, this, args));
	}
```

3). 创建并配置当前SpringBoot应用将要使用的Environment（包括配置要使用的PropertySource 以及 Profile）

4). 遍历调用所有SpringApplicationRunListener的environmentPrepared方法

通知SpringBoot     Environment 已经准备好了 

5). 打印Banner

6). 将Environment  设置给创建好的ApplicationContext 。

7). 通过SpringFactoriesLoader加载ApplicationContextInitializer，调用initialize方法

```java
protected void applyInitializers(ConfigurableApplicationContext context) {
		for (ApplicationContextInitializer initializer : getInitializers()) {
			Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(
					initializer.getClass(), ApplicationContextInitializer.class);
			Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
			initializer.initialize(context);
		}
	}
```

8). 调用SpringApplicationRunListeners 的contextPrepared方法 ，通知容器ApplicationContext 准备好了 。

9). 将之前通过@EnableAutoConfiguration 获取的所有配置以及其他形式的配置设置到ApplicationContext 中。

10). 调用SpringApplicationRunListeners 的contextPrepared方法 ，通知容器ApplicationContext 装载好了 。

11). 调用ApplicationContext 的refresh 方法 。 

12) .  查找CommandRunner ， 并执行。

13). 执行 SpringApplicationRunListeners  的 finished 方法 

至此，SpringBoot 启动完毕。



下一节，介绍几个核心的类（ *SpringApplicationRunListeners  ，ApplicationContextInitializer ， ApplicationListener* ）

