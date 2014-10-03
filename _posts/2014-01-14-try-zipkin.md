---
layout: post
title:  "分布式行为追踪系统-Zipkin"
---

####想象一下下面这两个场景

> 场景1：在一个分布式环境中，出现了一个UI数据展示出错的问题，我们要找到问题到底出现在哪。

> 场景2：在一个分布式环境中，有一个请求执行的非常慢，我们想要快速的找到到底是哪个组件拖慢了整个请求的速度。

对于上面的场景，我们一般的解法可能是从上游到下游逐个的查看整个系统调用链中得各个组件，可能会去查组件日志，可能会去看调用链中得中间件的状态是否正常，更有可能对照着日志直接看代码，总之随着组件的数量增加，整个过程是会越来越痛苦的。

但是程序员都是聪明且乐于分享的，既然是大家都会有的问题，那么咱们的开源社区就肯定会有这么一套系统去尝试解决这个问题，这就是咱们这篇文章的主角，来自Twitter的[Zipkin](http://twitter.github.io/zipkin/index.html)

其实我这么说并不准确，因为Zipkin的出现是源于google的一篇paper-[Dapper](http://research.google.com/pubs/pub36356.html)(尼玛，怎么这么多东西的出现都和Google的论文有关系).

好了，废话不多说，且让咱们看一下Zipkin是怎么来解决上面这两个场景中得问题的，首先大家得对下面的这几个名词有一个基本概念。

- Span 代表了一个唯一的请求，通常就是一个request。

- Trace 一整个调用链，也就是一组Span的集合。

- Annotation 个人就把它理解成了对某一个request在某一个时间点上的注解，包含了timestamp,service和host。

- Client 调用方

- Server 被调用方

然后咱们看一下Zipkin的主要组件架构图。

![Alt text](/assets/media/zipkin-architecture-0.png)

其中Collector负责接收各个service传输的数据，Query负责查询Storage中存储的数据，Scribe作为Transport组件是Optional的，Traced Service可以直接通过Thrift或者HTTP协议传输数据给Collector，Cassandra也是Optional的，默认数据库是SQLite。

上面的架构图看起来很完美，但是要让整个系统运转起来，还漏了很重要的一环，就是数据从哪来？台子Zipkin已经给咱们搭好，就等着咱们唱戏啦。咱们要做的就是构造符合Zipkin Collector规范的trace数据并传输。好在现在咱们已经有了[这些](https://github.com/twitter/zipkin/wiki#external-projects-that-use-zipkin)选择，咱们可以一起把轮子造成火箭，但是可千万别重复造轮子啦。

下面来看一下Java下的Zipkin Library实现-[Brave](https://github.com/kristofa/brave)。

直接上代码

模拟分布式系统互相调用的部分。

{% highlight java %}
@Override
@Path("/a")
@GET
public Response a() throws InterruptedException, ClientProtocolException, IOException {

    final Random random = new Random();
    Thread.sleep(random.nextInt(1000));

    final RestEasyExampleResource client =
        ProxyFactory.create(RestEasyExampleResource.class, "http://localhost:9090/RestEasyTest");
    @SuppressWarnings("unchecked")
    final ClientResponse<Void> response = (ClientResponse<Void>)client.b();
    try {
        return Response.status(response.getStatus()).build();
    } finally {
        response.releaseConnection();
    }
}

@Override
@Path("/b")
@GET
public Response b() throws InterruptedException {
	// 传输Annotation
    serverTracer.submitBinaryAnnotation("int", "22");
    final Random random = new Random();
    Thread.sleep(random.nextInt(1000));

    final RestEasyExampleResource client =
            ProxyFactory.create(RestEasyExampleResource.class, "http://localhost:9090/RestEasyTest");
    @SuppressWarnings("unchecked")
    final ClientResponse<Void> response = (ClientResponse<Void>)client.c();
    long startTime = System.currentTimeMillis();
    Thread.sleep(1010);
    
    // 这个Annotation是包含Duration和Timestamp的
    serverTracer.submitAnnotation("read file", startTime, System.currentTimeMillis());
    
    final RestEasyExampleResource client2 =
            ProxyFactory.create(RestEasyExampleResource.class, "http://localhost:9090/RestEasyTest");
    @SuppressWarnings("unchecked")
    final ClientResponse<Void> response2 = (ClientResponse<Void>)client2.c();
    try {
        return Response.status(response.getStatus()).build();
    } finally {
        response.releaseConnection();
        response2.releaseConnection();
    }
}
}
{% endhighlight %}

Brave给resteasy写的Interceptor，其中submitAnnotation和setXXX，就都是在给Zipkin Collector transport数据啦。

{% highlight java %}
server pre interceptor部分...

public ServerResponse preProcess(final HttpRequest request, final ResourceMethod method) throws Failure,
        WebApplicationException {

        submitEndpoint();

        serverTracer.clearCurrentSpan();
        final TraceData traceData = getTraceData(request);

        if (Boolean.FALSE.equals(traceData.shouldBeTraced())) {
            serverTracer.setStateNoTracing();
            LOGGER.debug("Received indication that we should NOT trace.");
        } else {
            String spanName = getSpanName(request, traceData);
            if (traceData.getTraceId() != null && traceData.getSpanId() != null) {

                LOGGER.debug("Received span information as part of request.");
                serverTracer.setStateCurrentTrace(traceData.getTraceId(), traceData.getSpanId(),
                    traceData.getParentSpanId(), spanName);
            } else {
                LOGGER.debug("Received no span state.");
                serverTracer.setStateUnknown(spanName);
            }
            serverTracer.setServerReceived();
        }
        return null;
    }
...

client 部分，执行调用的部分用注释标识了...

public ClientResponse<?> execute(final ClientExecutionContext ctx) throws Exception {

        final ClientRequest request = ctx.getRequest();
        String spanName = request.getHeaders().getFirst(BraveHttpHeaders.SpanName.getName());

        if (StringUtils.isEmpty(spanName)) {
            final URL url = new URL(request.getUri());
            spanName = url.getPath();
        }

        final SpanId newSpanId = clientTracer.startNewSpan(spanName);
        if (newSpanId != null) {
            LOGGER.debug("Will trace request. Span Id returned from ClientTracer: {}", newSpanId);
            // 往header里加入各种id
            request.header(BraveHttpHeaders.Sampled.getName(), TRUE);
            request.header(BraveHttpHeaders.TraceId.getName(), newSpanId.getTraceId());
            request.header(BraveHttpHeaders.SpanId.getName(), newSpanId.getSpanId());
            if (newSpanId.getParentSpanId() != null) {
                request.header(BraveHttpHeaders.ParentSpanId.getName(), newSpanId.getParentSpanId());
            }
        } else {
            LOGGER.debug("Will not trace request.");
            request.header(BraveHttpHeaders.Sampled.getName(), FALSE);
        }

        clientTracer.submitBinaryAnnotation(REQUEST_ANNOTATION, request.getHttpMethod() + " " + request.getUri());
        clientTracer.setClientSent();
        try {
        	// 此处执行调用，一头一尾分别是记录client sent和client received的状态
            final ClientResponse<?> response = ctx.proceed();
            final int responseStatus = response.getStatus();
            if (responseStatus < 200 || responseStatus > 299) {
                // In this case response will be the error message.
                clientTracer.submitBinaryAnnotation(HTTP_RESPONSE_CODE_ANNOTATION, responseStatus);
                clientTracer.submitAnnotation(FAILURE_ANNOTATION);
            }
            return response;
        } catch (final Exception e) {
            clientTracer.submitAnnotation(FAILURE_ANNOTATION);
            throw e;
        } finally {
            clientTracer.setClientReceived();
        }
    }
	下面的这两行代码用于在Zipkin中设置一个服务，只需执行一次...
	final EndPointSubmitter endPointSubmitter = Brave.getEndPointSubmitter();
	endPointSubmitter.submit("127.0.0.1", 9090, "RestEasyTest");
{% endhighlight %}

这里面有个坑爹的事情，Annotation和BinaryAnnotation的查询目前是不支持模糊查询的，而Brave把时间戳也放到了Annotation的value里，这就导致如果你想在Zipkin的WebUI上通过Annotation定位某个请求基本是办不到的。

效果如下，Zipkin用一种可视化的方式展示系统的所有行为。

![Alt text](/assets/media/zipkin-ui.png)

这些属性是开发Zipkin library的时候需要用到的。

- Trace Id 

- Span Id 

- Optional Parent Span Id 要形成调用链这个肯定少不了

- Sampled boolean 如果false就不去记录这一次调用

类似Brave这些Library做的事情其实就是通过http header记住上面的trace id,span id等各种id，然后通过各种手段把数据传输到Collector，技术难度不是太大，大家有兴趣可以开发一些Zipkin的Library，[这篇文章](http://twitter.github.io/zipkin/instrument.html)详细讲了如果开发一个zipkin library。

*最后大家一定要注意一点*，由于zipkin是twitter开发的，而在sbt和javascript中都引用了GFW外的东西，测试Zipkin时切记始终处于翻墙状态就对了。





