---
title: MyBatis源码解析：SqlSession
date: 2017-06-15 15:16:35
tags: MyBatis
---
　　顾名思义SqlSession指的就是数据库会话，之前说过SqlSessionFactory如何生成SqlSession实例，本次不在赘述。

　　session中包含了两种执行sql的形式，一种是传统的CRUD，另一种是mapper形式。官方文档推荐使用mapper形式，理由是代码更优雅（笑）。其实主要是mapper形式更加简易，而且通过接口+xml的开发模式代码耦合性很低，更符合开闭原则，而且逻辑看起来更加清晰，配合MyBatis的异常系统定位异常位置简单粗暴（笑哭）。先看下传统的调用方式，下面是outline图：
![](/images/mybatis_6.png)
<!-- more -->
　　这么多方法自然不能一一说明，此处仅取selectList方法进行示例说明：

	 public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
	    try {
	      //在configuration中维护了一个Map<String, MappedStatement>集合，在初始化的时候会读取所有sql配置文件中的信息，
	      //每个sql会生成一个对应的上下文MappedStatement，缓存在这个集合中。
	      MappedStatement ms = configuration.getMappedStatement(statement);
	      //具体操作下放至Executor执行
	      return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
	    } catch (Exception e) {
	      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
	    } finally {
	      ErrorContext.instance().reset();
	    }
	  }
　　源码里写的很简练，第一步获取初始化时创建的sql的对应上下文对象，第二部将具体操作下放到Executor中执行。这里就体现出来SqlSession和Executor的职务差别，SqlSession只负责构造一个sql操作的执行环境，Executor只负责对数据库操作，分工明确。

下面看一下mapper形式是如何执行sql操作的：

	 public <T> T getMapper(Class<T> type) {
	    return configuration.<T>getMapper(type, this);
	  }

很简单，直接甩锅给configuration（笑），而configuration里又有一个MapperRegistry对象，这个对象可是很关键，MapperRegistry在初始化时会缓存所有mapper接口的反射工厂实例，下面看两个关键的方法：

	public <T> void addMapper(Class<T> type) {
	    //mapper类限定只能是接口
	    if (type.isInterface()) {
	      //初始化上下文时会注册所有已检索的mapper接口，所以通常不会出现这个异常，除非配置有问题
	      if (hasMapper(type)) {
	        throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
	      }
	      //用于finally中校验已注册的mapper对象是否是完整对象，如果对象信息不完整则从缓存中删除
	      boolean loadCompleted = false;
	      try {
	    	//knownMappers是一个Map<Class<?>, MapperProxyFactory<?>>集合，
	    	//此处实例化一个mapper接口对应的MapperProxyFactory放入集合中，供之后实例化时调用。
	        knownMappers.put(type, new MapperProxyFactory<T>(type));
	        MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
	        //此处会生成对应的MappedStatement，并且放入Configuration的缓存集合mappedStatements中
	        parser.parse();
	        loadCompleted = true;
	      } finally {
	        if (!loadCompleted) {
	          knownMappers.remove(type);
	        }
	      }
	    }
	  }
	public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
		//首先获取mapper代理工厂实例
	    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
	    if (mapperProxyFactory == null) {
	      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
	    }
	    try {
	      //通过工厂实例创建mapper的代理实例
	      return mapperProxyFactory.newInstance(sqlSession);
	    } catch (Exception e) {
	      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
	    }
	  }

此处可以看到有一个关键的类MapperProxyFactory，顾名思义这是个mapper代理工厂类，而实际的代理类是MapperProxy，下面看一下MapperProxy的invoke方法：

	 public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
	    try {
	      //如果代理的是object类，则直接调用被代理方法
	      if (Object.class.equals(method.getDeclaringClass())) {
	        return method.invoke(this, args);
	      } else if (isDefaultMethod(method)) {
	    	//如果代理执行的方法是接口的默认实现方法（jdk1.8中加入），则创建一个接口实例，执行此方法
	        return invokeDefaultMethod(proxy, method, args);
	      }
	    } catch (Throwable t) {
	      throw ExceptionUtil.unwrapThrowable(t);
	    }
	    //如果执行的方法是属于已注册mapper接口的抽象方法，则执行下面方法。先在methodCache缓存中获取MapperRegistry注册mapper接口时生成
	    //在MapperRegistry注册mapper接口时会创建一个接口方法实例缓存池methodCache，每次执行接口代理方法时会查询缓存池中是否包含此方法的实例，
	    //如果不存在，则创建对应方法实例放入缓存，并执行代理方法。
	    final MapperMethod mapperMethod = cachedMapperMethod(method);
	    return mapperMethod.execute(sqlSession, args);
	  }

这个MapperMethod是MyBatis中的sql操作方法类，暴露给外部调用的也只有execute(SqlSession sqlSession, Object[] args)方法而已，下面看下这个方法的代码：

	public Object execute(SqlSession sqlSession, Object[] args) {
	    Object result;
	    //根据sql类型判断执行的方法
	    switch (command.getType()) {
	      case INSERT: {
	    	Object param = method.convertArgsToSqlCommandParam(args);
	        result = rowCountResult(sqlSession.insert(command.getName(), param));
	        break;
	      }
	      case UPDATE: {
	        Object param = method.convertArgsToSqlCommandParam(args);
	        result = rowCountResult(sqlSession.update(command.getName(), param));
	        break;
	      }
	      case DELETE: {
	        Object param = method.convertArgsToSqlCommandParam(args);
	        result = rowCountResult(sqlSession.delete(command.getName(), param));
	        break;
	      }
	      case SELECT:
	        if (method.returnsVoid() && method.hasResultHandler()) {
	          executeWithResultHandler(sqlSession, args);
	          result = null;
	        } else if (method.returnsMany()) {
	          result = executeForMany(sqlSession, args);
	        } else if (method.returnsMap()) {
	          result = executeForMap(sqlSession, args);
	        } else if (method.returnsCursor()) {
	          result = executeForCursor(sqlSession, args);
	        } else {
	          Object param = method.convertArgsToSqlCommandParam(args);
	          result = sqlSession.selectOne(command.getName(), param);
	        }
	        break;
	      case FLUSH:
	        result = sqlSession.flushStatements();
	        break;
	      default:
	        throw new BindingException("Unknown execution method for: " + command.getName());
	    }
	    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
	      throw new BindingException("Mapper method '" + command.getName() 
	          + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
	    }
	    return result;
	  }
由上可见，其实mapper形式写法最终还是调用SqlSession中的一系列方法，这么看来MyBatis官方文档中说的使用mapper看起来更优雅还真是实至名归（笑）
