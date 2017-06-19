---
title: MyBatis源码解析：Executor
date: 2017-06-15 15:16:38
tags: MyBatis
---
　　前面提到了SqlSession很懒，操作都下放给了Executor这个执行者，Executor是MyBatis的执行器接口，子类如下图：
![](/images/mybatis_7.png)
<!-- more -->
## BaseExecutor
　　BaseExecutor实现了Executor接口的所有方法，采用了模板模式，为下面几个子类提供公共实现。首先看下update方法的源码：

	public int update(MappedStatement ms, Object parameter) throws SQLException {
	    ErrorContext.instance().resource(ms.getResource()).activity("executing an update").object(ms.getId());
	    //如果已经关闭则抛出异常
	    if (closed) {
	      throw new ExecutorException("Executor was closed.");
	    }
	    //执行更新操作会清空一级缓存
	    clearLocalCache();
	    //由子类做差异化实现
	    return doUpdate(ms, parameter);
	  }

由上可见，update的模板方法比较简单，只是清空了以及缓存，外加关闭异常，具体的操作由子类的doUpdate实现决定。下面看下query方法的源码：

	 public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
	   //获取sql对象
	   BoundSql boundSql = ms.getBoundSql(parameter);
	   //获取缓存key
	    CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
	    return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
	 }

此处首先获取MyBatis的sql对象BoundSql和缓存key，然后调用下面方法：

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

由上述方法可见，query的具体实现方法是doQuery，也是由子类进行差异化实现。

	public <E> Cursor<E> queryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds) throws SQLException {
	    //获取sql对象
	    BoundSql boundSql = ms.getBoundSql(parameter);
	    return doQueryCursor(ms, parameter, rowBounds, boundSql);
	  }
cursor是3.4版本新增的特性，用于以游标的方式返回结果，主要用于数据量很大的业务场景，会分批次读取数据，降低每次对数据库的占用时间。

## SimpleExecutor
　　简单sql执行器，不支持添加参数。这里用doQuery方法进行示例：

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
	private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
	    Statement stmt;
	    //获取数据库连接
	    Connection connection = getConnection(statementLog);
	    //创建statement实例
	    stmt = handler.prepare(connection, transaction.getTimeout());
	    //为statement实例绑定参数，（simple类型无方法体）
	    handler.parameterize(stmt);
	    return stmt;
	  }

## ReuseExecutor
　　可重复使用执行器，内部维护了一个Map<String, Statement>缓存池，在一个session生命周期内相同的statement只会实例化一次。与simple的主要差异在于下面两个方法

	public List<BatchResult> doFlushStatements(boolean isRollback) throws SQLException {
	    //遍历statement缓存池，依次关闭
	    for (Statement stmt : statementMap.values()) {
	      closeStatement(stmt);
	    }
	    //清空缓存池
	    statementMap.clear();
	    return Collections.emptyList();
	  }
	 private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
	    Statement stmt;
	    //获取sql对象
	    BoundSql boundSql = handler.getBoundSql();
	    String sql = boundSql.getSql();
	    //判断缓存池中是否有此sql的statement实例
	    if (hasStatementFor(sql)) {
	      //获取缓存池中的statement实例
	      stmt = getStatement(sql);
	      //设置事务过期时间
	      applyTransactionTimeout(stmt);
	    } else {
	      //缓存池中没有此sql的statement实例时，通过StatementHandler创建新实例
	      Connection connection = getConnection(statementLog);
	      stmt = handler.prepare(connection, transaction.getTimeout());
	      //将statement实例放入缓存池
	      putStatement(sql, stmt);
	    }
	    //为statement绑定参数
	    handler.parameterize(stmt);
	    return stmt;
	  }

## BatchExecutor
批处理执行器，BatchExecutor内维护了四个变量，分别是
* statementList：statement执行队列
* batchResultList：结果集队列
* currentSql：上次执行sql
* currentStatement：上次执行MappedStatement

批处理的doQuery方法与simple的差别只在于会清空上述四个变量的值，批处理特性主要体现在doUpdate和doFlushStatements这两个方法中：

	public int doUpdate(MappedStatement ms, Object parameterObject) throws SQLException {
		//获取配置上下文
	    final Configuration configuration = ms.getConfiguration();
	    //创建StatementHandler实例
	    final StatementHandler handler = configuration.newStatementHandler(this, ms, parameterObject, RowBounds.DEFAULT, null, null);
	    //从MappedStatement中获取sql对象
	    final BoundSql boundSql = handler.getBoundSql();
	    final String sql = boundSql.getSql();
	    final Statement stmt;
	    //判断本次执行sql是否与上次执行sql相同
	    if (sql.equals(currentSql) && ms.equals(currentStatement)) {
	      int last = statementList.size() - 1;
	      //获取上次使用的Statement实例
	      stmt = statementList.get(last);
	      //刷新事务过期时间
	      applyTransactionTimeout(stmt);
	      //为Statement绑定新参数
	      handler.parameterize(stmt);
	      //将参数对象添加至BatchResult对象中（BatchResult中维护了一个参数List）
	      BatchResult batchResult = batchResultList.get(last);
	      batchResult.addParameterObject(parameterObject);
	    } else {
	      //获取新的数据库连接实例
	      Connection connection = getConnection(ms.getStatementLog());
	      //创建Statement实例
	      stmt = handler.prepare(connection, transaction.getTimeout());
	      //为Statement绑定参数
	      handler.parameterize(stmt); 
	      //刷新上次执行sql以及Statement，为下次判断做准备
	      currentSql = sql;
	      currentStatement = ms;
	      statementList.add(stmt);
	      //在结果集队列中添加BatchResult对象实例
	      batchResultList.add(new BatchResult(ms, sql, parameterObject));
	    }
	    handler.batch(stmt);
	    //批处理执行器的更新操作不会返回修改行数，而是返回一个固定的数字，此处需要注意！
	    return BATCH_UPDATE_RETURN_VALUE;
	  }
	public List<BatchResult> doFlushStatements(boolean isRollback) throws SQLException {
	    try {
	      List<BatchResult> results = new ArrayList<BatchResult>();
	      //如果rollback则返回空集
	      if (isRollback) {
	        return Collections.emptyList();
	      }
	      for (int i = 0, n = statementList.size(); i < n; i++) {
	    	//从statement队列中获取statement实例
	        Statement stmt = statementList.get(i);
	        //设置事务超时时间
	        applyTransactionTimeout(stmt);
	        //从结果集队列中获取BatchResult实例，结果集队列与上面statement队列是一一对应关系
	        BatchResult batchResult = batchResultList.get(i);
	        try {
	          //设置本次操作修改的行数（int数组形式）,需要注意的是多次执行相同的修改sql时，结果信息会保存在同一个BatchResult实例中
	          batchResult.setUpdateCounts(stmt.executeBatch());
	          MappedStatement ms = batchResult.getMappedStatement();
	          List<Object> parameterObjects = batchResult.getParameterObjects();
	          KeyGenerator keyGenerator = ms.getKeyGenerator();
	          //此处是回写字段逻辑，如果在xml中配置了需要回写的字段，则会调用KeyGenerator进行回写，具体逻辑在KeyGenerator讲解
	          if (Jdbc3KeyGenerator.class.equals(keyGenerator.getClass())) {
	            Jdbc3KeyGenerator jdbc3KeyGenerator = (Jdbc3KeyGenerator) keyGenerator;
	            jdbc3KeyGenerator.processBatch(ms, stmt, parameterObjects);
	          } else if (!NoKeyGenerator.class.equals(keyGenerator.getClass())) { 
	            for (Object parameter : parameterObjects) {
	              keyGenerator.processAfter(this, ms, stmt, parameter);
	            }
	          }
	        } catch (BatchUpdateException e) {
	          StringBuilder message = new StringBuilder();
	          message.append(batchResult.getMappedStatement().getId())
	              .append(" (batch index #")
	              .append(i + 1)
	              .append(")")
	              .append(" failed.");
	          if (i > 0) {
	            message.append(" ")
	                .append(i)
	                .append(" prior sub executor(s) completed successfully, but will be rolled back.");
	          }
	          throw new BatchExecutorException(message.toString(), e, results, batchResult);
	        }
	        results.add(batchResult);
	      }
	      return results;
	    } finally {
	      //清空批处理队列
	      for (Statement stmt : statementList) {
	        closeStatement(stmt);
	      }
	      currentSql = null;
	      statementList.clear();
	      batchResultList.clear();
	    }
	  }
## CachingExecutor
　　缓存执行器，顾名思义这种执行器内部必定维护了一个缓存区，也主要针对于查询语句，具体代码如下：

	//实际的操作执行器
	private Executor delegate;
	//缓存容器，缓存实体TransactionalCache实现了Cache接口
	private TransactionalCacheManager tcm = new TransactionalCacheManager();
	public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
	      throws SQLException {
	    Cache cache = ms.getCache();
	    if (cache != null) {
	      //根据flushCache参数决定是否刷新缓存区
	      flushCacheIfRequired(ms);
	      if (ms.isUseCache() && resultHandler == null) {
	    	//如果执行存储过程，校验boundSql中的绑定参数列表是否包含输出参数，如果包含则抛出异常
	        ensureNoOutParams(ms, parameterObject, boundSql);
	        @SuppressWarnings("unchecked")
	        //从缓存中获取MappedStatement对应的查询结果
	        List<E> list = (List<E>) tcm.getObject(cache, key);
	        if (list == null) {
	          //如果没有缓存实例，则调用delegate执行query，并将结果放入缓存区
	          list = delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
	          tcm.putObject(cache, key, list); // issue #578 and #116
	        }
	        return list;
	      }
	    }
	    //如果MappedStatement配置不使用缓存则直接调用delegate进行查询
	    return delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
	  }