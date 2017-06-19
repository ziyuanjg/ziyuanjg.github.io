---
title: MyBatis架构设计总结
date: 2017-06-14 17:33:44
tags: MyBatis
---

## MyBatis的架构
![](/images/mybatis_1.png)

<!-- more -->

### 接口层
接口层提供了与数据库交互的入口，MyBatis支持两种方式的接口调用
* 传统API方式
* mapper接口方式

#### 传统API
![](/images/mybatis_2.png)
　　例：int i = session.selectOne("daoMapper.UserMapper.selectIDByUser", u);
　　传统API的方式虽然很简洁，但是并不方便，不仅要根据sql类型手动选择调用的方法，而且StatementID使用字符串形式的拼写也加大了人为错误的因素，并不符合框架避免人为错误的这一个特点。

### Mapper接口
　　MyBatis支持两种Mapper接口的实现形式一种是java注解，一种是xml配置。
#### 注解
    @Select("select id from users where phoneNO = #{phoneNO}")
	public int selectID(String phoneNO);
注解形式开发比较简单，但是不适合比较复杂的sql。
#### xml配置
	mapper.java:
	public User selectUser(Integer id);

	mapper.xml:
    <select id="selectUser" resultType="dao.User" >
  	select * from policy where id=#{id}
 	</select>

　　xml配置自由度更大，而且支持动态拼接sql，可以根据传入参数. 条件等决定最终生成的sql对象。
　　Mapper接口的调用入口是Configuration.getMapper，MyBatis在初始化运行环境上下文（Configuration）时会读取配置的mapper文件，为mapper中配置的xml文件生成对应的statement，在调用mapper中的方法时，MyBatis会根据方法名和mapper名确定一个唯一的StatementID，值得注意的是mapper形式其实也是调用传统API进行操作的。


## 2.数据处理层
数据处理层可以说是MyBatis的核心精华所在，主要包括以下三个功能：
* 处理参数生成动态SQL
* 数据库操作
* 处理结果集

#### 处理参数生成动态SQL
　　动态生成SQL是一个很优雅的设计，MyBatis通过请求传入的参数对象和Ognl表达式来动态生成需要执行的sql语句，这使得设计sql的时候有很强的灵活性和扩展性。处理参数这部分主要分为两步，首先是查询的时候通过preparedStatement将参数对象与jdbc类型进行映射，其次是查询出结果集时，通过resultSet将查询结果与java类型进行映射。
#### 数据库操作
　　MyBatis底层与数据库的交互依然是使用的JDBC实现，主要通过Executor这个执行者协调各个模块完成请求操作。
#### 处理结果集
　　MyBatis会将数据库返回的结果映射为List<E>的形式，这里要提到ResultMap，这个标签的功能很强大，可以完成多层多类型嵌套，支持多对一或一对多的数据形式。下面是一个例子：

    <resultMap type="com.ymt.config.protectplan.domain.dto.ProtectplanDto" id="protectplanDetail">
		<id property="id" column="protectPlanID"/>
		<result property="protectPlanName" column="protectPlanName"/>
		<result property="protectPlanDesc" column="protectPlanDesc"/>
		<result property="protectplanStatus" column="protectplanStatus"/>
		<result property="createTime" column="protectPlanCreateTime"/>
		<result property="lastUpdate" column="protectPlanLastUpdate"/>
		<collection property="detailList" ofType="java.util.Map" javaType="java.util.ArrayList">
			<id property="id" column="protectPlanContentID"/>
			<result property="protectPlanContName" column="protectPlanContName"/>
			<result property="protectPlanID" column="protectPlanID"/>
			<result property="protectPlanContentType" column="protectPlanContentType"/>
			<result property="priority" column="priority"/>
			<result property="createTime" column="protectPlanContentCreateTime"/>
			<result property="lastUpdate" column="protectPlanContentLastUpdate"/>
			<collection property="insuranceList" ofType="java.util.Map" javaType="java.util.ArrayList">
				<id property="kindID" column="kindID"/>
				<result property="hasFree" column="hasFree"/>
				<result property="isFree" column="isFree"/>
				<result property="mainFlag" column="mainFlag"/>
				<result property="amount" column="detailAmount"/>
				<collection property="amountList" ofType="java.util.Map" javaType="java.util.ArrayList">
					<id  column="customID"/>
					<result property="amount" column="amount"/>
					<result property="amountShow" column="amountShow"/>
					<result property="isDefault" column="isDefault"/>
				</collection>
			</collection>	
		</collection>
	</resultMap>		


### 支撑服务层
　　支撑服务层主要为数据处理层提供额外功能。

#### 元对象模块
　　元对象是MyBatis的一个独立的模块，提供了一种简洁而优雅的访问对象的方式，使得开发时只需要专注于具体的业务，而不需要关注反射细节，并且无需手动处理各种反射异常。其具体实现是依赖与java的反射机制，所以嘛性能是有点损耗的，不过比起方便程度来说也是可以接受的。（不依赖其他模块，可以单独提出使用）

#### 缓存机制
　　在越来越高的并发量的冲击下，不论是服务器还是数据库的压力都越来越大，缓存机制就是缓解数据库这方面的访问压力的。MyBatis设置了两个缓存层级，一级缓存存在于一次SqlSession会话内，一次会话可以不必重复查询相同的数据，二级缓存可以跨SqlSession存在，但是有很大风险。总之，使用MyBatis的缓存要谨慎，不然反而会影响程序的时效性。

#### 拦截器机制
　　拦截器是MyBatis提供的一种可以改变自身运行机制的方式，拦截器仅支持拦截Executor. StatementHandler. ResultSetHandler. ParameterHandler。可以对MyBatis做一些定制化的修改，比如常用的分页，分库分表，读写分离，分类缓存等。

#### 事务管理
　　事务管理是ORM框架的一个必不可少的部分。

#### 连接池机制
　　创建数据库连接占据了一次会话的大部分时间，所以在访问量大的系统中，连接池就至关重要，连接池维护了所有数据库会话，节省了每次创建连接的性能开销。

#### Sql的配置方式
　　MyBatis的传统配置方式是基于xml配置，但是面向接口编程之风越来越盛，而且xml配置也很繁琐，并不适合简单sql，所以就诞生了一种符合面向接口编程思想的方式，而接口调用就引申出了注解这个大杀器，从此以后就可以使用mapper接口+注解的方式来配置sql，但是需要注意的是MyBatis对注解的支持还很简单，很多复杂的功能还是要用xml配置的。

## MyBatis的主要模块
  * SqlSession：数据库会话模块，MyBatis的顶层API，程序的调用入口。
  * Configuration：运行环境上下文，维护了MyBatis的所有配置信息，以及各种缓存区。
  * Executor：MyBatis的执行器，负责调度数据库操作的整个流程，完成各种缓存相关操作。
  * StatementHandler：封装了JDBC的Statement操作，包括设置参数，转换结果为List等。
  * ResultSetHandler：负责将结果集与java类型进行映射，转换为指定的类型。
  * ParameterHandler：负责处理参数对象，将参数与jdbc类型进行映射。
  * TypeHandle：类型处理器，主要负责jdbc与java的类型转换。
  * MapperStatement：维护了CRUD节点的信息。
  * SQLSource：根据传递的参数对象动态生成sql语句，并封装为BoundSql对象。
  * BoundSql：封装了sql相关的参数信息。
  * Plugin：拦截器组件，可以在一定程度上定制MyBatis的执行流程。
  * MetaObject：元对象模块，简洁而优雅的反射工具。

(MyBatis只有这几个模块吗？当然不是，那为什么只写这几个是重点呢？这个嘛，当然是我只看了这几个模块的源码了=。=)
![](/images/mybatis_3.png)

## MyBatis执行流程
此处由MyBatis的查询操作举例
### 开启SqlSession
	InputStream is = Resources.getResourceAsStream("mybatis-config.xml");
	SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(is,"development1");
	SqlSession session = sqlSessionFactory.openSession();
	MyBatis的数据库会话封装在SqlSession中，SqlSession作为顶层API，肩负着增删改查的调度任务。

### SqlSession的查询
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
　　在MyBatis初始化时，会根据加载的配置文件创建Configuration实例，mapper配置文件中的sql会生成对应的MappedStatement实例，Configuration维护了一个Map集合来缓存MappedStatement实例，key为nameSpace.functionName。
　　SqlSession会根据传入的statementID（即上面的key）获取配置缓存区中的MappedStatement实例，然后将操作参数交给Executor执行具体任务。

### Executor执行器
#### BaseExecutor
	public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
	    //创建BoundSql实例
		BoundSql boundSql = ms.getBoundSql(parameter);
		//获取缓存key
	    CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
	    return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
	 }
	public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
	    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
	    //如果已经关闭则抛出异常
	    if (closed) {
	      throw new ExecutorException("Executor was closed.");
	    }
	    //当查询堆栈为0且flushCacheRequired配置为true则刷新一级缓存区
	    if (queryStack == 0 && ms.isFlushCacheRequired()) {
	      clearLocalCache();
	    }
	    List<E> list;
	    try {
	      queryStack++;
	      //根据参数中的resultHandler来判断是否从一级缓存中获取结果集，selectList方法传入的resultHandler默认为null
	      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
	      if (list != null) {//缓存中存在结果集时
	        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
	      } else {
	    	//从数据库获取数据
	        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
	      }
	    } finally {
	      queryStack--;
	    }
	    //当查询堆栈为0时加载延迟加载列表中的所有元素
	    if (queryStack == 0) {
	      for (DeferredLoad deferredLoad : deferredLoads) {
	        deferredLoad.load();
	      }
	      //清空延迟加载列表
	      deferredLoads.clear();
	      //如果设置的本地缓存等级为statement，则清空一级缓存
	      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
	        clearLocalCache();
	      }
	    }
	    return list;
	  }
	private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
	    List<E> list;
	    //执行查询前用占位符在一级缓存中占位
	    localCache.putObject(key, EXECUTION_PLACEHOLDER);
	    try {
	      //具体查询方法，由子类进行差异化实现
	      list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
	    } finally {
	      localCache.removeObject(key);
	    }
	    //将结果集放入一级缓存中
	    localCache.putObject(key, list);
	    //如果statement的类型为CALLABLE，在localOutputParameterCache中放入参数
	    if (ms.getStatementType() == StatementType.CALLABLE) {
	      localOutputParameterCache.putObject(key, parameter);
	    }
	    return list;
	  }
BaseExecutor作为模板类，主要任务就是创建BoundSql实例. 处理一级缓存的内容，以及决定是否需要重新查询数据库取值，具体的数据库操作doQuery由子类进行差异化实现。
#### SimpleExecutor
	public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
	    Statement stmt = null;
	    try {
	      //获取配置上下文
	      Configuration configuration = ms.getConfiguration();
	      //创建StatementHandler实例
	      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
	      //创建statement实例，具体生成过程会放在StatementHandler模块讲解
	      stmt = prepareStatement(handler, ms.getStatementLog());
	      //具体执行操作
	      return handler.<E>query(stmt, resultHandler);
	    } finally {
	      closeStatement(stmt);
	    }
	  }
　　这里可以看出Executor的主要任务就是通过StatementHandler创建Statement实例，此时会将参数绑定到statement的指定位置，然后将任务下方到StatementHandler中，StatementHandler与JDBC进行交互。

### SimpleStatementHandler
	public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
		//获取sql
	    String sql = boundSql.getSql();
	    //执行查询
	    statement.execute(sql);
	    //处理结果集
	    return resultSetHandler.<E>handleResultSets(statement);
	  }
　　SimpleStatementHandler执行了sql语句，并将结果交给ResultSetHandler处理。
### ResultSetHandler
	public List<Object> handleResultSets(Statement stmt) throws SQLException {
	    ErrorContext.instance().activity("handling results").object(mappedStatement.getId());

	    final List<Object> multipleResults = new ArrayList<Object>();
	    
	    //已处理结果行数
	    int resultSetCount = 0;
	    //ResultSetWrapper是ResultSet的包装类
	    ResultSetWrapper rsw = getFirstResultSet(stmt);

	    //获取结果集
	    List<ResultMap> resultMaps = mappedStatement.getResultMaps();
	    int resultMapCount = resultMaps.size();
	    //判断是否有结果集，没有则抛出异常
	    validateResultMapsCount(rsw, resultMapCount);
	    while (rsw != null && resultMapCount > resultSetCount) {
	      ResultMap resultMap = resultMaps.get(resultSetCount);
	      //映射结果集
	      handleResultSet(rsw, resultMap, multipleResults, null);
	      //如果有多个结果集时，此处rsw不为null
	      rsw = getNextResultSet(stmt);
	      //清空缓存池
	      cleanUpAfterHandlingResultSet();
	      resultSetCount++;
	    }

	    String[] resultSets = mappedStatement.getResultSets();
	    if (resultSets != null) {
	      //当存在多个结果集时执行以下逻辑
	      while (rsw != null && resultSetCount < resultSets.length) {
	        ResultMapping parentMapping = nextResultMaps.get(resultSets[resultSetCount]);
	        if (parentMapping != null) {
	          //获取嵌套结果集id
	          String nestedResultMapId = parentMapping.getNestedResultMapId();
	          //从一级缓存中获取结果集
	          ResultMap resultMap = configuration.getResultMap(nestedResultMapId);
	          //绑定子结果集，此处的parentMapping为父级结果集
	          handleResultSet(rsw, resultMap, null, parentMapping);
	        }
	        //获取下一个结果集
	        rsw = getNextResultSet(stmt);
	        //清空缓存池
	        cleanUpAfterHandlingResultSet();
	        resultSetCount++;
	      }
	    }

	    return collapseSingleResultList(multipleResults);
	  }
　　ResultSetHandler会将查询结果转换为List<E>的形式，如果调用的是selectOne方法，则会取List中的第一条数据返回，如果List为空会抛出异常。

上述只是简单描述了MyBatis的查询流程，其实具体细节有很多，无法一一列举，详细内容之后会单独在各个模块的源码讲解中详述。
