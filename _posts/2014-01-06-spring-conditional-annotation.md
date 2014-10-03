---
layout: post
title:  "Spring Framework Tips系列 第一季第二集之 @Condtional"
---

> 先和大家说下@Conditional这个新Annotation的由来吧。Spring在2013年年末发布了4.0.0，Spring Boot作为Spring 4.0的重头戏一下子吸引了很多眼球，而在Spring Boot发布之前，我有幸知道了这个项目并简单的扫过源码，其中autoconfigure模块作为Spring Boot的发动机显得尤为亮眼，而autoconfigure的魔力就在于一套非常优美的Conditional动态加载机制，接下来，咱们先看一下Spring Context中得@Conditional，再来看一下Spring Boot中得给力扩展吧。

@Conditional注解的作用简单来说是让Spring按需加载不同的目标类，而整个Conditonal的核心就是下面的这两个玩意儿啦。

**@Condtional**

这个Annotation可以注解TYPE或者METHOD，接受的参数是实现下面接口的class。

**Interface Condition**

这个接口看方法名就能猜到是用来标识是否匹配，如果这个方法返回true，那么Spring Context就会加载这个Bean啦，一般@Conditional都会和@Configuration或者@Bean配合使用，在Spring Boot中能见到大量的使用场景。

`boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata)`

下面来看一个Example吧。

{% highlight xml %}
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>4.0.0.RELEASE</version>
    </dependency>
{% endhighlight %}

-
{% highlight java %}
@Configuration
@ComponentScan
public class TestConditional {
	public static void main(String[] args) {
		ApplicationContext context = 
		          new AnnotationConfigApplicationContext(TestConditional.class);
		Service service = context.getBean(Service.class);
		System.out.println(service.getClass().getName());
	}
	
	@Bean
	@Conditional(AlwaysFalse.class)
	public Service serviceTrue() {
		return new ServiceTrue();
	}
	
	@Bean
	@Conditional(AlwaysTrue.class)
	public Service serviceFalse() {
		return new ServiceFalse();
	}
	
	private static class AlwaysTrue implements Condition {
		public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
			return true;
		}
	}
	
	private static class AlwaysFalse implements Condition {
		public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
			return false;
		}
	}
}
{% endhighlight %}

运行会输出如下结果：
com.nezhazheng.springtips.conditional.ServiceFalse

可以看到由于serviceFlase()的Condition为true，所以ServiceFalse被加载了。

下面来看下Spring Boot是如何演绎这套动态加载机制的吧

SpringBootCondition是所有Spring Boot中Condition得基类。

`public abstract class SpringBootCondition implements Condition`

SpringBoot会了记录日志Message，提供了下面的这个封装结果，除了match的结果，还有一个log message。

{% highlight java %}
public class ConditionOutcome {

	private final boolean match;

	private final String message;
{% endhighlight %}

Spring Boot对于embedded container提供了两个选择，一个是Jetty还有一个是Emebedded Tomcat，而具体使用哪个就是由Conditional机制来控制的。

{% highlight java %}
@ConditionalOnWebApplication
// 首先需要通过上面的这个Conditional控制这个Servlet Container Configuration被加载
// 然后通过判断Jetty和Tomcat各自特有的类是否存在来判断加载哪个EmbeddedServletContainerFactory
public class EmbeddedServletContainerAutoConfiguration {
	...
	
	/**
	 * Nested configuration for if Tomcat is being used.
	 */	 
	@Configuration
	@ConditionalOnClass({ Servlet.class, Tomcat.class })
	@ConditionalOnMissingBean(value = EmbeddedServletContainerFactory.class, search = SearchStrategy.CURRENT)
	public static class EmbeddedTomcat {

		@Bean
		public TomcatEmbeddedServletContainerFactory tomcatEmbeddedServletContainerFactory() {
			return new TomcatEmbeddedServletContainerFactory();
		}

	}

	/**
	 * Nested configuration if Jetty is being used.
	 */
	@Configuration
	@ConditionalOnClass({ Servlet.class, Server.class, Loader.class })
	@ConditionalOnMissingBean(value = EmbeddedServletContainerFactory.class, search = SearchStrategy.CURRENT)
	public static class EmbeddedJetty {

		@Bean
		public JettyEmbeddedServletContainerFactory jettyEmbeddedServletContainerFactory() {
			return new JettyEmbeddedServletContainerFactory();
		}

	}
	...
}

// Conditional可以通过自定义Annotation的方式更灵活的使用，这样就不用重复的去写Condtion.class了
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnWebApplicationCondition.class)
public @interface ConditionalOnWebApplication {
}

class OnWebApplicationCondition extends SpringBootCondition {
private static final String WEB_CONTEXT_CLASS = "org.springframework.web.context.support.GenericWebApplicationContext";
...
// 通过对ClassLoader是否加载了某些特定类来判断是否需要加载
if (!ClassUtils.isPresent(WEB_CONTEXT_CLASS, context.getClassLoader())) {
			return ConditionOutcome.noMatch("web application classes not found");
		}
...
{% endhighlight %}
