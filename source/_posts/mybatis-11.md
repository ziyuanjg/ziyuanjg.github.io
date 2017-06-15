---
title: MyBatis源码解析：Plugin
date: 2017-06-15 15:17:09
tags: MyBatis
---

　　Plugin类实现了InvocationHandler接口，  是MyBatis对java动态代理的一个扩展，用来支持Mybatis自身的plugins插件开发。

# 1. Plugin可以做什么
　　作为一个插件模块，Plugin实际上是一种拦截器，在SqlSession的执行流程中不难发现，其实在创建Executor、StatementHandler、ResultSetHandler、ParameterHandler的实例时都会使用Configuration中的插件队列依次进行代理，结合MetaObject就可以对sql的执行过程做一些定制化的修改，比如常用的分页，分库分表，读写分离，分类缓存等。
<!-- more -->

# 2. Plugin如何工作
![](/images/mybatis_23.png)

　　要使用Plugin首先要在config中进行配置

 	<plugins>
 		<plugin interceptor="interceptor.ExampleInterceptor"></plugin>
 	</plugins>

　　在MyBatis初始化的时候创建了运行环境（Configuration），那Plugin自然也脱离不了这个范畴，在Configuration中有四个触发点会对拦截器进行注册，下面来看一个例子：

	 public ParameterHandler newParameterHandler(MappedStatement mappedStatement, Object parameterObject, BoundSql boundSql) {
	    ParameterHandler parameterHandler = mappedStatement.getLang().createParameterHandler(mappedStatement, parameterObject, boundSql);
	    parameterHandler = (ParameterHandler) interceptorChain.pluginAll(parameterHandler);
	    return parameterHandler;
	  }
　　此处调用了interceptorChain.pluginAll，这个interceptorChain顾名思义就是拦截器链

	public Object pluginAll(Object target) {
	    for (Interceptor interceptor : interceptors) {
	      target = interceptor.plugin(target);
	    }
	    return target;
	  }
　　在pluginAll方法中每次都会将拦截器链中的所有实例依次代理指定类，既每次创建ParameterHandler、ResultSetHandler、StatementHandler、Executor实例的时候都会用所有已注册的拦截器进行代理，性能消耗真心大！

	public Object plugin(Object target) {
			return Plugin.wrap(target, this);
		}
	public static Object wrap(Object target, Interceptor interceptor) {
	    //获取Interceptor的拦截类和方法
	    Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
	    Class<?> type = target.getClass();
	    //匹配传入对象target是否实现Interceptor指定拦截的接口
	    Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
	    if (interfaces.length > 0) {
	      return Proxy.newProxyInstance(
	          type.getClassLoader(),
	          interfaces,
	          new Plugin(target, interceptor, signatureMap));
	    }
	    return target;
	  }

　　下面看下plugin的调用流程，以拦截StatementHandler的prepare方法为例，因为Plugin的存在，首先会执行invoke方法

	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
	    try {
	      Set<Method> methods = signatureMap.get(method.getDeclaringClass());
	      //匹配当前执行方法是否存在对应拦截器
	      if (methods != null && methods.contains(method)) {
	    	//存在则执行拦截器的intercept方法
	        return interceptor.intercept(new Invocation(target, method, args));
	      }
	      return method.invoke(target, args);
	    } catch (Exception e) {
	      throw ExceptionUtil.unwrapThrowable(e);
	    }
	  }

　　如果存在此方法的拦截器，则调用拦截器的intercept方法，下面展示了几种常用变量的获取方式：

	public Object intercept(Invocation invocation) throws Throwable {
			//statementHandler
			MetaObject statementHandler = MetaObject.forObject(invocation.getTarget(), new DefaultObjectFactory(), new DefaultObjectWrapperFactory(), new DefaultReflectorFactory());
			//具体执行的statementHandler，此处是PreparedStatementHandler
			MetaObject ps = statementHandler.metaObjectForProperty("delegate");
			//MappedStatement
			MappedStatement ms = (MappedStatement) ps.getValue("mappedStatement");
			//Configuration
			Configuration config = (Configuration) ps.getValue("configuration");
			//参数对象
			Object parameterObject = statementHandler.getValue("parameterHandler.parameterObject");
			//sql对象
			BoundSql bs = (BoundSql) statementHandler.getValue("boundSql"); 
	  //执行被拦截方法
			return invocation.proceed();
		}

# 3. 多个Plugin的执行顺序
　　上边既然提到了拦截器链，那自然是可以配置多个Plugin的

	<plugins>
	 		<plugin interceptor="interceptor.Interceptor1"></plugin>
	 		<plugin interceptor="interceptor.Interceptor2"></plugin>
	 		<plugin interceptor="interceptor.Interceptor3"></plugin>
	 </plugins>
　　上面配置了三个Plugin，在创建运行环境（Configuration）时会依次将Plugin注册在拦截器中，但是为对象绑定代理时顺序则出现了一些不一样的地方，代理的执行顺序却不一定与绑定的顺序一致。如上面的intercept方法：
　　
	public Object intercept(Invocation invocation) throws Throwable {
			//TODO1
	  invocation.proceed();
			//TODO2
		}
　　如果是这种形式的拦截器，则会先执行TODO1，然后执行下级拦截器，之后再转回TODO2。

# 4. 拦截器的优缺点
　　在漫长的开发过程中，因为人自身的问题导致的程序bug层出不穷，框架就是在这种情况下应运而生，框架简化了编码的操作，极大的拉进了技术人员的差距，但是框架也封装了底层的操作，使得开发人员对程序的把控程度更低了，如果框架的一小部分不适合实际的业务，但是又无法更换框架的时候就需要Plugin这种存在，Plugin是MyBatis主动提供出来的流程修改入口，只要你想，甚至可以改变SqlSession的执行流程。

　　Plugin可以使得开发人员可以根据具体业务对MyBatis的底层做适当的干涉，但是这里就牵涉到第一个问题，人员的问题，如果开发人员自身技术不是很深入，有可能导致整个流程崩溃！一定要慎用！
而且过多的拦截器会极大的消耗服务器资源，导致运行缓慢。

