---
title: MyBatis源码解析：StatementHandler
date: 2017-06-15 15:16:40
tags: MyBatis
---

上文说到Executor是Mybatis的执行者，Mybatis为了防止工作量太大而暴走就准备了几个苦力供他差遣。
* StatementHandler：负责生产statement实例以及监督statement执行sql操作。
* ParameterHandler：负责绑定参数。
* ResultSetHandler：负责处理结果集

<!-- more -->
本次先说说StatementHandler这个苦力。下团是StatementHandler的家族体系：
![](/images/mybatis_8.png)

　　可以看出StatementHandler和他的带头大哥一样是使用的模板模式，下面就先看下这个模板类BaseStatementHandler。
## BaseStatementHandler
　　BaseStatementHandler需要关注的方法只有一个，下面来看下源码：

	public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
	    ErrorContext.instance().sql(boundSql.getSql());
	    Statement statement = null;
	    try {
	      //实例化statement，由子类进行差异化实现
	      statement = instantiateStatement(connection);
	      //设置statement超时时间
	      setStatementTimeout(statement, transactionTimeout);
	      //设置每次返回的结果集大小，取决于xml中的fetchSize值
	      setFetchSize(statement);
	      return statement;
	    } catch (SQLException e) {
	      closeStatement(statement);
	      throw e;
	    } catch (Exception e) {
	      closeStatement(statement);
	      throw new ExecutorException("Error preparing statement.  Cause: " + e, e);
	    }
	  }
## SimpleStatementHandler
　　一看名字就知道了，这是执行简单sql小分队的苦力，正好用来举例，先来看看CRUD方法：

	public int update(Statement statement) throws SQLException {
		//获取sql
	    String sql = boundSql.getSql();
	    //获取参数对象
	    Object parameterObject = boundSql.getParameterObject();
	    //获取主键生产器
	    KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
	    int rows;
	    //根据KeyGenerator的具体类型决定是否返回修改数据的主键，具体逻辑放在KeyGenerator中讲解
	    if (keyGenerator instanceof Jdbc3KeyGenerator) {
	      statement.execute(sql, Statement.RETURN_GENERATED_KEYS);
	      //获取修改行数
	      rows = statement.getUpdateCount();
	      //将返回的主键信息添加到parameterObject中
	      keyGenerator.processAfter(executor, mappedStatement, statement, parameterObject);
	    } else if (keyGenerator instanceof SelectKeyGenerator) {
	      statement.execute(sql);
	      rows = statement.getUpdateCount();
	      keyGenerator.processAfter(executor, mappedStatement, statement, parameterObject);
	    } else {
	      statement.execute(sql);
	      rows = statement.getUpdateCount();
	    }
	    return rows;
	  }
	public void batch(Statement statement) throws SQLException {
		//获取sql
	    String sql = boundSql.getSql();
	    //执行批处理
	    statement.addBatch(sql);
	  }
	public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
		//获取sql
	    String sql = boundSql.getSql();
	    //执行查询
	    statement.execute(sql);
	    //处理结果集
	    return resultSetHandler.<E>handleResultSets(statement);
	  }
	public <E> Cursor<E> queryCursor(Statement statement) throws SQLException {
		//获取sql
	    String sql = boundSql.getSql();
	    //执行查询
	    statement.execute(sql);
	    //处理结果集
	    return resultSetHandler.<E>handleCursorResultSets(statement);
	  }
	  protected Statement instantiateStatement(Connection connection) throws SQLException {
	    if (mappedStatement.getResultSetType() != null) {
		  //创建能返回指定类型的ResultSet的Statement实例
	      return connection.createStatement(mappedStatement.getResultSetType().getValue(), ResultSet.CONCUR_READ_ONLY);
	    } else {
		  //创建普通Statement实例
	      return connection.createStatement();
	    }
	  }
simple不愧simple之名，除了update稍微复杂点都很简单，需要注意的是SimpleStatementHandler中的parameterize方法是没有实现的，即不支持绑定参数。

## PreparedStatementHandler
　　这个是预编译小组的小伙伴，这个小伙伴和simple挺像，甚至update比simple还要简单，真不知道到底他俩谁才更像simple（笑），不过说归说这次的是有很大不同的，下面看下源码：

	protected Statement instantiateStatement(Connection connection) throws SQLException {
	    //获取sql
	    String sql = boundSql.getSql();
	    if (mappedStatement.getKeyGenerator() instanceof Jdbc3KeyGenerator) {
	      //获取xml配置的指定列名
	      String[] keyColumnNames = mappedStatement.getKeyColumns();
	      if (keyColumnNames == null) {
	    	//创建能返回自动生成键的PreparedStatement实例
	        return connection.prepareStatement(sql, PreparedStatement.RETURN_GENERATED_KEYS);
	      } else {
	    	//创建能返回指定数组指定的自动生成键的PreparedStatement实例
	        return connection.prepareStatement(sql, keyColumnNames);
	      }
	    } else if (mappedStatement.getResultSetType() != null) {
	      //创建能返回指定类型的ResultSet的PreparedStatement实例
	      return connection.prepareStatement(sql, mappedStatement.getResultSetType().getValue(), ResultSet.CONCUR_READ_ONLY);
	    } else {
	      //创建普通PreparedStatement实例
	      return connection.prepareStatement(sql);
	    }
	  }
	public int update(Statement statement) throws SQLException {
	    PreparedStatement ps = (PreparedStatement) statement;
	    //执行sql
	    ps.execute();
	    //获取修改的行数
	    int rows = ps.getUpdateCount();
	    //获取参数对象
	    Object parameterObject = boundSql.getParameterObject();
	    //获取主键生成器
	    KeyGenerator keyGenerator = mappedStatement.getKeyGenerator();
	    //将修改行的主键信息反写回parameterObject
	    keyGenerator.processAfter(executor, mappedStatement, ps, parameterObject);
	    return rows;
	  }
	public void parameterize(Statement statement) throws SQLException {
	    parameterHandler.setParameters((PreparedStatement) statement);
	  }

## CallableStatementHandler
　　顾名思义这个就是处理存储过程的小伙伴了，他的修改. 查询方法与上面两位几乎相同就不在赘述了，下面看下不一样的地方：

	protected Statement instantiateStatement(Connection connection) throws SQLException {
	    String sql = boundSql.getSql();
	    if (mappedStatement.getResultSetType() != null) {
	      //创建返回指定ResultSet类型的CallableStatement实例
	      return connection.prepareCall(sql, mappedStatement.getResultSetType().getValue(), ResultSet.CONCUR_READ_ONLY);
	    } else {
	      //创建CallableStatement实例
	      return connection.prepareCall(sql);
	    }
	  }

	public void parameterize(Statement statement) throws SQLException {
		//注册输出参数
	    registerOutputParameters((CallableStatement) statement);
	    //绑定参数
	    parameterHandler.setParameters((CallableStatement) statement);
	  }

	  private void registerOutputParameters(CallableStatement cs) throws SQLException {
		//获取参数集合
	    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
	    for (int i = 0, n = parameterMappings.size(); i < n; i++) {
	      //获取参数对象
	      ParameterMapping parameterMapping = parameterMappings.get(i);
	      //参数类型为输出参数时才会进行绑定
	      if (parameterMapping.getMode() == ParameterMode.OUT || parameterMapping.getMode() == ParameterMode.INOUT) {
	    	//如果参数没有指定jdbc类型则抛出异常
	        if (null == parameterMapping.getJdbcType()) {
	          throw new ExecutorException("The JDBC Type must be specified for output parameter.  Parameter: " + parameterMapping.getProperty());
	        } else {
	          //将指定位置的参数注册为jdbc类型
	          if (parameterMapping.getNumericScale() != null && (parameterMapping.getJdbcType() == JdbcType.NUMERIC || parameterMapping.getJdbcType() == JdbcType.DECIMAL)) {
	            cs.registerOutParameter(i + 1, parameterMapping.getJdbcType().TYPE_CODE, parameterMapping.getNumericScale());
	          } else {
	            if (parameterMapping.getJdbcTypeName() == null) {
	              cs.registerOutParameter(i + 1, parameterMapping.getJdbcType().TYPE_CODE);
	            } else {
	              cs.registerOutParameter(i + 1, parameterMapping.getJdbcType().TYPE_CODE, parameterMapping.getJdbcTypeName());
	            }
	          }
	        }
	      }
	    }
	  }

## RoutingStatementHandler
　　MyBatis中默认实现的StatementHandler就是RoutingStatementHandler，RoutingStatementHandler并没有实现具体的业务逻辑，而是作为一个选择器的角色存在，通过配置中的设置决定实例化哪种子类

	 public RoutingStatementHandler(Executor executor, MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
	    switch (ms.getStatementType()) {
	      case STATEMENT:
	        delegate = new SimpleStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
	        break;
	      case PREPARED:
	        delegate = new PreparedStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
	        break;
	      case CALLABLE:
	        delegate = new CallableStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
	        break;
	      default:
	        throw new ExecutorException("Unknown statement type: " + ms.getStatementType());
	    }
	  }