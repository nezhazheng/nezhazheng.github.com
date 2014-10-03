---
layout: post
title:  "Spring Framework Tips系列 第一季第一集之 如何在Spring MVC 中做request logging"
---

> 由于request body 中得数据在通过request.getInputStream() 读取时，只能读取一次，所以通过简单的Interceptor或者Filter之类的切面技术并不能很好的把request body输出到日志中。

Spring为了解决request logging问题专门写了一组Filter用于输出日志。

![Alt text](/assets/media/FilterGroup.png)

如果是用Log4J 或者 Commons Logging作为日志解决方案，则可以直接采用现成的 CommonsRequestLoggingFilter 或者 Log4jNestedDiagnosticContextFilter 处理。如果是用Logback或者其他Logging Framework的话，则可以自行继承 AbstractRequestLoggingFilter 实现。

{% highlight java %}
 public class RequestLoggingFilter extends AbstractRequestLoggingFilter {

    private static final Logger logger = LoggerFactory.getLogger(RequestLoggingFilter.class);
    
    @Override
    protected void beforeRequest(HttpServletRequest request, String message) {
    }

    @Override
    protected void afterRequest(HttpServletRequest request, String message) {
        logger.info(message);
    }

}
{% endhighlight %}
 
web.xml中得配置如下：

*maxPayloadLength*代表日志中可以输出的最大request body长度，如果给的太小会导致日志输出半截的情况。

{% highlight xml %}
     <filter>
        <filter-name>RequestLoggingFilter</filter-name>
        <filter-class>com.communityhelper.api.RequestLoggingFilter</filter-class>
        <init-param>
            <param-name>includeQueryString</param-name>
            <param-value>true</param-value>
        </init-param>
        <init-param>
            <param-name>maxPayloadLength</param-name>
            <param-value>2000</param-value>
        </init-param>
        <init-param>
            <param-name>includeClientInfo</param-name>
            <param-value>true</param-value>
        </init-param>
        <init-param>
            <param-name>includePayload</param-name>
            <param-value>true</param-value>
        </init-param>
    </filter>
{% endhighlight %}
