---
title: Gateway源码分析（二）
date: 2021-07-25 11:16:39
categories: 
- 视频
tags:
- 视频
---

# 背景

这一章开始记录我开始Gateway阅读之路，看下究竟是如何实现网关。

# 个人疑问

1. Gateway的网关框架是如何接收请求并转发

2. 如何执行Filter

3. 如何加载路由、过滤器、断言等信息



# 源码分析

## 版本

```
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-gateway</artifactI>
	<version>2.1.0.RELEASE</version>
</dependency>
```

## 架构图

还是一样的图
![Gateway框架](/images/video/gateway网关图.png)

## 组成

gateway-core.jar是gateway的核心包，主要的实现都在里面。阅读代码前最好先知道个个包的主要功能.

![包组成](/images/video/gateway的jar包架构图.png)

- actuate

该包主要是gateway自带的一个控制器GatewayControllerEndpoint，该endpiont提供了关于filter及routes的信息查询以及指定route信息更新的rest api，这给web界面提供管理配置功能提供了极大的便利

- config

该包主要是Gateway的配置实体类，譬如yml上面的配置GatewayProperties、全局的跨域配置GlobalCorsProperties等等。

- discovery

该包主要是实现服务发现的功能。从服务注册中心获取服务注册信息，然后配置相应的路由

- event

该包是一些发布事件的定义。

- filter

该包包含了gateway所有内置的过滤器。

- handler

该包主要包括了所有内置的Predicates断言，RoutePredicateHandlerMapping类是一个实现了将接收请求到转发到filter里面的功能，FilteringWebHandler主要是构造过滤器链。

- route

该包主要是定义路由信息，构造路由等。

- support

该包主要是一些工具方法。用于全局。



## 源码分析

### 问题一：如何转发请求

#### DispatchHandler类

WebFlux请求转发核心类：DispatchHandler

DispatchHandler内部主要的私有字段：

```java
@Nullable
private List<HandlerMapping> handlerMappings;

@Nullable
private List<HandlerAdapter> handlerAdapters;

@Nullable
private List<HandlerResultHandler> resultHandlers;
```

类型 | 解释
---|---
HandlerMapping | 映射请求到一个处理器。该映射是基于一定的标准、细节因不同HandlerMapping而不同。</br>例如有注解控制器, 简单URL匹配映射等等。</br>主要的HandlerMapping实现：</br>1.有RequestMappingHandlerMapping对于注解的@RequestMapping。</br>2.RouterFunctionMapping 对应于函数式端点路由。</br>3.SimpleUrlHandlerMappingURI路径模式的显式注册。</br>4.WebHandler的实例
HandlerAdapter | 帮助DispatcherHandler调用映射的请求的处理器，而不管该处理程序实际上是如何调用的。</br>例如执行一个注解控制器需要解释注解。其主要目的是帮助DispatcherHandler隐藏实现的细节。
HandlerResultHandler | 处理程序调用的结果并最后确定响应。</br>1.ResponseEntityResultHandler：ResponseEntity，处理@Controller实例。</br>2.ServerResponseResultHandler：ServerResponse，处理函数式端点。</br>3.ResponseBodyResultHandler：处理从@ResponseBody方法和@RestController类的返回值。</br>4.ViewResolutionResultHandler：处理成CharSequence,View, Model, Map, Rendering等其他的模型属性。


```java
@Override
public Mono<Void> handle(ServerWebExchange exchange) {
	if (this.handlerMappings == null) {
		return Mono.error(HANDLER_NOT_FOUND_EXCEPTION);
	}
	return Flux.fromIterable(this.handlerMappings)
			.concatMap(mapping -> mapping.getHandler(exchange))
			.next()
			.switchIfEmpty(Mono.error(HANDLER_NOT_FOUND_EXCEPTION))
			.flatMap(handler -> invokeHandler(exchange, handler))
			.flatMap(result -> handleResult(exchange, result));
}
```

核心方法主要做了三个步骤：

1. 匹配每一个不同HandlerMapping，使用首先匹配的那个。
2. 执行器被找到就会找到对应的HandlerAdapter,然后就会将返回结果返回到HandlerResult里。
3. HandlerResult会给出一个合适的处理器去完成直接写到响应里面或者使用View来渲染的处理。

#### RoutePredicateHandlerMapping类

刚说完DispatchHandler的类，就到HandlerMapping了，类RoutePredicateHandlerMapping实现了HandlerMapping。

其核心私有字段分别有：
```java
private final FilteringWebHandler webHandler;

private final RouteLocator routeLocator;

private final Integer managmentPort;
```

Bean类型 | 解释
---|---
webHandler | 构建过滤器的责任链
routeLocator | 路由的定义信息
managmentPort | gateway管理端口


其核心方法：

```java
@Override
protected Mono<?> getHandlerInternal(ServerWebExchange exchange) {
	// don't handle requests on the management port if set
	if (managmentPort != null && exchange.getRequest().getURI().getPort() == managmentPort.intValue()) {
		return Mono.empty();
	}
	exchange.getAttributes().put(GATEWAY_HANDLER_MAPPER_ATTR, getSimpleName());

	return lookupRoute(exchange)
			// .log("route-predicate-handler-mapping", Level.FINER) //name this
			.flatMap((Function<Route, Mono<?>>) r -> {
				exchange.getAttributes().remove(GATEWAY_PREDICATE_ROUTE_ATTR);
				if (logger.isDebugEnabled()) {
					logger.debug("Mapping [" + getExchangeDesc(exchange) + "] to " + r);
				}

				exchange.getAttributes().put(GATEWAY_ROUTE_ATTR, r);
				return Mono.just(webHandler);
			}).switchIfEmpty(Mono.empty().then(Mono.fromRunnable(() -> {
				exchange.getAttributes().remove(GATEWAY_PREDICATE_ROUTE_ATTR);
				if (logger.isTraceEnabled()) {
					logger.trace("No RouteDefinition found for [" + getExchangeDesc(exchange) + "]");
				}
			})));
}
```

该核心方法做了几件事：

1. 找到合适的路由lookupRoute方法。
2. 将路由信息放到ServerWebExchange请求线程的属性里，以便整个运行随时可用。
3. 执行webHandler中的过滤器链。==（后面会介绍如何执行）==


lookupRoute执行过程：
1. 通过routeLocator的获取所有配置好的路由信息。 ==（后面会介绍如何加载路由信息、过滤器、断言）==
2. 匹配每一个路由的断言，是否符合，若符合则返回对应的路由信息。若不符合则next()下一个路由的断言匹配。

```java
protected Mono<Route> lookupRoute(ServerWebExchange exchange) {
	return this.routeLocator
			.getRoutes()
			//individually filter routes so that filterWhen error delaying is not a problem
			.concatMap(route -> Mono
					.just(route)
					.filterWhen(r -> {
						// add the current route we are testing
						exchange.getAttributes().put(GATEWAY_PREDICATE_ROUTE_ATTR, r.getId());
						return r.getPredicate().apply(exchange);
					})
					//instead of immediately stopping main flux due to error, log and swallow it
					.doOnError(e -> logger.error("Error applying predicate for route: "+route.getId(), e))
					.onErrorResume(e -> Mono.empty())
			)
			// .defaultIfEmpty() put a static Route not found
			// or .switchIfEmpty()
			// .switchIfEmpty(Mono.<Route>empty().log("noroute"))
			.next()
			//TODO: error handling
			.map(route -> {
				if (logger.isDebugEnabled()) {
					logger.debug("Route matched: " + route.getId());
				}
				validateRoute(route, exchange);
				return route;
			});
}
```

### 问题二：如何执行Filter


#### 如何执行Filter(FilteringWebHandler类)


执行流程：

1. 初始化时候构造好全局的过滤器集合。
2. 合并路由上配置的过滤器与全局过滤器
3. 排序好所有过滤器传入DefaultGatewayFilterChain的责任链里执行。

```
@Override
public Mono<Void> handle(ServerWebExchange exchange) {
	Route route = exchange.getRequiredAttribute(GATEWAY_ROUTE_ATTR);
	List<GatewayFilter> gatewayFilters = route.getFilters();

	List<GatewayFilter> combined = new ArrayList<>(this.globalFilters);
	combined.addAll(gatewayFilters);
	//TODO: needed or cached?
	AnnotationAwareOrderComparator.sort(combined);

	if (logger.isDebugEnabled()) {
		logger.debug("Sorted gatewayFilterFactories: "+ combined);
	}

	return new DefaultGatewayFilterChain(combined).filter(exchange);
}
```

责任链是一个常用的编程设计模式，它能够将请求与处理步骤解耦，请求操作对链内部的执行透明，而且每个链子都有自己具体实现，能够自由组装复用，不相互影响，使得代码更加简洁。不过责任联在调试方面相对来说比较麻烦，不便于观察等缺点。

看下如何构造一个责任联，内部类DefaultGatewayFilterChain
的filter方法
```java
private static class DefaultGatewayFilterChain implements GatewayFilterChain {

		private final int index;
		private final List<GatewayFilter> filters;

		public DefaultGatewayFilterChain(List<GatewayFilter> filters) {
			this.filters = filters;
			this.index = 0;
		}

		private DefaultGatewayFilterChain(DefaultGatewayFilterChain parent, int index) {
			this.filters = parent.getFilters();
			this.index = index;
		}

		public List<GatewayFilter> getFilters() {
			return filters;
		}

		@Override
		public Mono<Void> filter(ServerWebExchange exchange) {
			return Mono.defer(() -> {
				if (this.index < filters.size()) {
					GatewayFilter filter = filters.get(this.index);
					DefaultGatewayFilterChain chain = new DefaultGatewayFilterChain(this, this.index + 1);
					return filter.filter(exchange, chain);
				} else {
					return Mono.empty(); // complete
				}
			});
		}
	}
```


### 问题三：如何加载路由、过滤器、断言等信息


#### RouteDefinitionRouteLocator类

- 获取路由信息

```
@Override
public Flux<Route> getRoutes() {
	return this.routeDefinitionLocator.getRouteDefinitions()
			.map(this::convertToRoute)
			//TODO: error handling
			.map(route -> {
				if (logger.isDebugEnabled()) {
					logger.debug("RouteDefinition matched: " + route.getId());
				}
				return route;
			});


	/* TODO: trace logging
		if (logger.isTraceEnabled()) {
			logger.trace("RouteDefinition did not match: " + routeDefinition.getId());
		}*/
}
```

==this.routeDefinitionLocator.getRouteDefinitions()== 初始化首次加载获取配置文件中定义的路由信息并存于缓存中以便下次使用
```
public CachingRouteDefinitionLocator(RouteDefinitionLocator delegate) {
		this.delegate = delegate;
		routeDefinitions = CacheFlux.lookup(cache, "routeDefs", RouteDefinition.class)
				.onCacheMissResume(this.delegate::getRouteDefinitions);
}
```

==convertToRoute==则是将配置信息定义的路由信息转变为真正内部使用的路由实体，具体实现如下：

1. 组装断言链表
2. 获取配置信息定义过滤器
3. 组装成真正路由对象Route
```
private Route convertToRoute(RouteDefinition routeDefinition) {
		AsyncPredicate<ServerWebExchange> predicate = combinePredicates(routeDefinition);
		List<GatewayFilter> gatewayFilters = getFilters(routeDefinition);

		return Route.async(routeDefinition)
				.asyncPredicate(predicate)
				.replaceFilters(gatewayFilters)
				.build();
	}
```

其实这段代码使用了一个设计模式就是建造者模式。该模式能将构建和实现分离开来，建造者能逐步细化而不影响其它模块功能。不过建造者对产品会依赖，当产品发生变化，建造者相应也需要改变。所以这种模式建议用在比较简化的建造者依赖类上。

```
return Route.async(routeDefinition)
				.asyncPredicate(predicate)
				.replaceFilters(gatewayFilters)
				.build();
```
![建造者模式](/images/video/Router的结构图.png)



“断言”的功能在我看来实现得是非常巧妙的，所有断言正如过滤器一样都有一个共同的父类AbstractRoutePredicateFactory，实现apply的方法。看个例子：

PathRoutePredicateFactory断言类：作用是匹配请求uri资源。可以看出返回的是一个Predicate的实例，这样的好处就是在下次执行只会执行return返回的这一部分代码功能，不再需要执行配置行的代码。

```
@Override
public Predicate<ServerWebExchange> apply(Config config) {
	final ArrayList<PathPattern> pathPatterns = new ArrayList<>();
	synchronized (this.pathPatternParser) {
		pathPatternParser.setMatchOptionalTrailingSeparator(config.isMatchOptionalTrailingSeparator());
		config.getPatterns().forEach(pattern -> {
			PathPattern pathPattern = this.pathPatternParser.parse(pattern);
			pathPatterns.add(pathPattern);
		});
	}
	return exchange -> {
		PathContainer path = parsePath(exchange.getRequest().getURI().getPath());

		Optional<PathPattern> optionalPathPattern = pathPatterns.stream()
				.filter(pattern -> pattern.matches(path))
				.findFirst();

		if (optionalPathPattern.isPresent()) {
			PathPattern pathPattern = optionalPathPattern.get();
			traceMatch("Pattern", pathPattern.getPatternString(), path, true);
			PathMatchInfo pathMatchInfo = pathPattern.matchAndExtract(path);
			putUriTemplateVariables(exchange, pathMatchInfo.getUriVariables());
			return true;
		} else {
			traceMatch("Pattern", config.getPatterns(), path, false);
			return false;
		}
	};
}
```

然后通过Flux.zip方法连成一条断言链子
```
private AsyncPredicate<ServerWebExchange> combinePredicates(RouteDefinition routeDefinition) {
	List<PredicateDefinition> predicates = routeDefinition.getPredicates();
	AsyncPredicate<ServerWebExchange> predicate = lookup(routeDefinition, predicates.get(0));

	for (PredicateDefinition andPredicate : predicates.subList(1, predicates.size())) {
		AsyncPredicate<ServerWebExchange> found = lookup(routeDefinition, andPredicate);
		predicate = predicate.and(found);
	}

	return predicate;
}

---------------------------------------------------

default AsyncPredicate<T> and(AsyncPredicate<? super T> other) {
	Objects.requireNonNull(other, "other must not be null");

	return t -> Flux.zip(apply(t), other.apply(t))
			.map(tuple -> tuple.getT1() && tuple.getT2());
}
```

# 总结
其实到这里这一章就差不多了，基本上Gateway的主要总体框架功能就这些。这一次的阅读源码能够使我对lambda表达式的用处更深刻，而且，对这种响应式流的理解更进一步了，后续还会持续学习关于netty reactor的用法。