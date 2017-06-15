---
title: MyBatis源码解析：数据源和连接池
date: 2017-06-15 15:16:52
tags: MyBatis
---

　　MyBatis是一种ORM框架，作为关系映射一端的数据源自然是重中之重了，下面就简单介绍下MyBatis的数据源模块。

# 1. DataSource的三种类型
MyBatis中默认有三种DataSource实现，分别是不使用连接池的UnpooledDataSource、使用连接池的PooledDataSource和使用JNDI的JndiDataSource。
<!-- more -->

# 2. DataSource的创建过程
![](/images/mybatis_14.png)

## 2.1. 在config.xml中配置environments标签

	<environments default="development1">
	 	<environment id="development1">
	 		<transactionManager type="JDBC"></transactionManager>
	 		<dataSource type="UNPOOLED">
	 			<property name="driver" value="${driver1}"/>
		        <property name="url" value="${url1}"/>
		        <property name="username" value="${username1}"/>
		        <property name="password" value="${password1}"/>
	 		</dataSource>
	 	</environment>
	</environments>

## 2.2. 创建SqlSessionFactory

	SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(is,"development1");

## 2.3. 构造Configuration实例

	public Configuration parse() {
	    if (parsed) {
	      throw new BuilderException("Each XMLConfigBuilder can only be used once.");
	    }
	    parsed = true;
	    //获取config.xml配置文件中的内容，进行组装
	    parseConfiguration(parser.evalNode("/configuration"));
	    return configuration;
	  }
## 2.4. 读取environments标签

	private void environmentsElement(XNode context) throws Exception {
	    if (context != null) {
	      //如果构建SqlSessionFactory时传入了environment参数，则使用指定数据源，否则使用配置的默认数据源
	      if (environment == null) {
	        environment = context.getStringAttribute("default");
	      }
	      //遍历所有配置的environment数据，搜索id与指定数据源匹配的元素
	      for (XNode child : context.getChildren()) {
	        String id = child.getStringAttribute("id");
	        if (isSpecifiedEnvironment(id)) {
	          //事务管理器工厂
	          TransactionFactory txFactory = transactionManagerElement(child.evalNode("transactionManager"));
	          //数据源工厂
	          DataSourceFactory dsFactory = dataSourceElement(child.evalNode("dataSource"));
	          DataSource dataSource = dsFactory.getDataSource();
	          //数据交互环境构造器
	          Environment.Builder environmentBuilder = new Environment.Builder(id)
	              .transactionFactory(txFactory)
	              .dataSource(dataSource);
	          //将数据源放入configuration上下文中
	          configuration.setEnvironment(environmentBuilder.build());
	        }
	      }
	    }
	  }

# 3. 不使用连接池的DataSource
![](/images/mybatis_15.png)

UnpooledDataSource相对PooledDataSource简单一点，每次调用都创建一个数据库连接，执行之后连接即可销毁。相关类主要有以下两个
## 3.1. UnpooledDataSourceFactory
只有两个方法setProperties和getDataSource，一个设置数据源参数，一个获取数据源。

## 3.2. UnpooledDataSource
既然是数据源，那最重要也就是获取数据库连接的方法了，下面看下获取数据库连接的方法：

	private Connection doGetConnection(Properties properties) throws SQLException {
	    //初始化数据库驱动，如果environment中的driver属性配置的数据库驱动没有初始化，则会通过反射加载指定驱动
	    initializeDriver();
	    //获取数据库连接
	    Connection connection = DriverManager.getConnection(url, properties);
	    //配置连接是否自动提交，隔离等级
	    configureConnection(connection);
	    return connection;
	  }

# 4. 使用连接池的DataSource
![](/images/mybatis_16.png)

　　PooledDataSource从源码角度来看实际是在UnpooledDataSource架子上做了一些扩展，在PooledDataSource维护了一个UnpooledDataSource实例，获取数据库连接的依然是UnpooledDataSource的那一套方法，不过既然要使用连接池自然不会如此简单，这就涉及到了PoolState和PooledConnection这两个类。

## 4.1. PoolState
PoolState是连接池的状态上下文，内部维护了很多运行时的状态数据，其中只有两个参与了连接池的操作，(空闲连接队列)idleConnections和(活跃连接队列)activeConnections。

## 4.2. PooledConnection
数据库连接的代理类，说到代理类，那自然就有invoke方法了：

	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
	    String methodName = method.getName();
	    //当执行close方法时，不会真的关闭连接，会根据连接池的状态决定如何处理
	    if (CLOSE.hashCode() == methodName.hashCode() && CLOSE.equals(methodName)) {
	      dataSource.pushConnection(this);
	      return null;
	    } else {
	      try {
	        if (!Object.class.equals(method.getDeclaringClass())) {
	          //检查连接是否可用
	          checkConnection();
	        }
	        return method.invoke(realConnection, args);
	      } catch (Throwable t) {
	        throw ExceptionUtil.unwrapThrowable(t);
	      }
	    }
	  }
有上面源码可知，其实代理主要是为了处理close方法，毕竟连接池中的连接是需要重用的，不然岂不是和不用连接池一样了。

## 4.3. PooledDataSource

	  //连接池状态上下文
	  private final PoolState state = new PoolState(this);
	  //实际获取数据库连接的数据源
	  private final UnpooledDataSource dataSource;
	  //最大活跃连接数
	  protected int poolMaximumActiveConnections = 10;
	  //最大空闲连接数
	  protected int poolMaximumIdleConnections = 5;
	  //如果一次调用超过此时间，则会强制结束调用
	  protected int poolMaximumCheckoutTime = 20000;
	  //再次尝试连接的等待时间
	  protected int poolTimeToWait = 20000;
	  //心跳检测
	  protected String poolPingQuery = "NO PING QUERY SET";
	  //是否使用心跳检测
	  protected boolean poolPingEnabled;
	  //如果一个连接超过此时间没有被使用，尝试ping这个数据源查看是否可用
	  protected int poolPingConnectionsNotUsedFor;
	  //连接的唯一code码
	  private int expectedConnectionTypeCode;

	  /**
	   * 强制关闭所有连接，修改连接池参数或数据源参数时调用
	   */
	  public void forceCloseAll() {
		//因为活跃连接和空闲连接的集合是在PoolState中维护，所以强制关闭时需要锁定state
	    synchronized (state) {
	      //强制关闭所有连接的触发点包括更改数据源信息，所以此处要重新赋值code
	      expectedConnectionTypeCode = assembleConnectionTypeCode(dataSource.getUrl(), dataSource.getUsername(), dataSource.getPassword());
	      //关闭所有活跃连接
	      for (int i = state.activeConnections.size(); i > 0; i--) {
	        try {
	          PooledConnection conn = state.activeConnections.remove(i - 1);
	          //连接状态修改为不可用
	          conn.invalidate();
	          //因为PooledConnection实际上是Connection的代理类，所以关闭的时候需要获取JDBC的连接进行关闭。
	          Connection realConn = conn.getRealConnection();
	          if (!realConn.getAutoCommit()) {
	            realConn.rollback();
	          }
	          realConn.close();
	        } catch (Exception e) {
	        }
	      }
	      //关闭所有空闲连接
	      for (int i = state.idleConnections.size(); i > 0; i--) {
	        try {
	          PooledConnection conn = state.idleConnections.remove(i - 1);
	          //连接状态修改为不可用
	          conn.invalidate();
	          //因为PooledConnection实际上是Connection的代理类，所以关闭的时候需要获取JDBC的连接进行关闭。
	          Connection realConn = conn.getRealConnection();
	          if (!realConn.getAutoCommit()) {
	            realConn.rollback();
	          }
	          realConn.close();
	        } catch (Exception e) {
	        }
	      }
	    }
	    if (log.isDebugEnabled()) {
	      log.debug("PooledDataSource forcefully closed/removed all connections.");
	    }
	  }

通常情况下，当我们使用完一个连接时，会调用close方法释放连接，避免影响性能，连接池中的连接代理实例调用close方法时实际上不会简单的释放掉连接，而是调用pushConnection方法将连接放入空闲队列，等待下次调用。

	  protected void pushConnection(PooledConnection conn) throws SQLException {
	    synchronized (state) {
	      //从活跃连接队列中清除
	      state.activeConnections.remove(conn);
	      //连接是可用状态
	      if (conn.isValid()) {
	    	//当前空闲连接数小于设置的最大空闲连接数且此连接的数据源与连接池相同
	        if (state.idleConnections.size() < poolMaximumIdleConnections && conn.getConnectionTypeCode() == expectedConnectionTypeCode) {
	          state.accumulatedCheckoutTime += conn.getCheckoutTime();
	          //如果连接设置手动提交，则回滚修改的内容，防止强制关闭引起数据错误
	          if (!conn.getRealConnection().getAutoCommit()) {
	            conn.getRealConnection().rollback();
	          }
	          //创建新的代理类，并放入空闲连接队列
	          PooledConnection newConn = new PooledConnection(conn.getRealConnection(), this);
	          state.idleConnections.add(newConn);
	          newConn.setCreatedTimestamp(conn.getCreatedTimestamp());
	          newConn.setLastUsedTimestamp(conn.getLastUsedTimestamp());
	          //废弃连接代理设置为不可用
	          conn.invalidate();
	          if (log.isDebugEnabled()) {
	            log.debug("Returned connection " + newConn.getRealHashCode() + " to pool.");
	          }
	          state.notifyAll();
	        } else {
	          state.accumulatedCheckoutTime += conn.getCheckoutTime();
	          //如果连接设置手动提交，则回滚修改的内容，防止强制关闭引起数据错误
	          if (!conn.getRealConnection().getAutoCommit()) {
	            conn.getRealConnection().rollback();
	          }
	          //关闭连接
	          conn.getRealConnection().close();
	          if (log.isDebugEnabled()) {
	            log.debug("Closed connection " + conn.getRealHashCode() + ".");
	          }
	          //废弃连接代理设置为不可用
	          conn.invalidate();
	        }
	      } else {
	        if (log.isDebugEnabled()) {
	          log.debug("A bad connection (" + conn.getRealHashCode() + ") attempted to return to the pool, discarding connection.");
	        }
	        state.badConnectionCount++;
	      }
	    }
	  }

	/**
	   * 获取连接
	   */
	  private PooledConnection popConnection(String username, String password) throws SQLException {
	    boolean countedWait = false;
	    PooledConnection conn = null;
	    long t = System.currentTimeMillis();
	    int localBadConnectionCount = 0;

	    while (conn == null) {
	      synchronized (state) {
	        if (!state.idleConnections.isEmpty()) {
	          //存在空闲连接时，取队列中的第一个空闲连接
	          conn = state.idleConnections.remove(0);
	          if (log.isDebugEnabled()) {
	            log.debug("Checked out connection " + conn.getRealHashCode() + " from pool.");
	          }
	        } else {
	          //连接池中不存在空闲连接时
	          //当连接池中的活跃连接小于最大的活跃连接数时
	          if (state.activeConnections.size() < poolMaximumActiveConnections) {
	            //创建新的连接代理实例
	            conn = new PooledConnection(dataSource.getConnection(), this);
	            if (log.isDebugEnabled()) {
	              log.debug("Created connection " + conn.getRealHashCode() + ".");
	            }
	          } else {
	            //当连接池中的活跃连接达到阈值时，不能创建新连接
	        	//获取活跃连接队列中执行时间最长的实例（FIFO）
	            PooledConnection oldestActiveConnection = state.activeConnections.get(0);
	            long longestCheckoutTime = oldestActiveConnection.getCheckoutTime();
	            if (longestCheckoutTime > poolMaximumCheckoutTime) {
	              //此连接的checkout时间已经超过了设置的最长checkout时间，此时可以将此连接当前操作作废，移出活跃队列
	              state.claimedOverdueConnectionCount++;
	              state.accumulatedCheckoutTimeOfOverdueConnections += longestCheckoutTime;
	              state.accumulatedCheckoutTime += longestCheckoutTime;
	              state.activeConnections.remove(oldestActiveConnection);
	              //将连接当前操作回滚，防止数据错误
	              if (!oldestActiveConnection.getRealConnection().getAutoCommit()) {
	                try {
	                  oldestActiveConnection.getRealConnection().rollback();
	                } catch (SQLException e) {
	                  log.debug("Bad connection. Could not roll back");
	                }  
	              }
	              //创建新的连接代理实例
	              conn = new PooledConnection(oldestActiveConnection.getRealConnection(), this);
	              conn.setCreatedTimestamp(oldestActiveConnection.getCreatedTimestamp());
	              conn.setLastUsedTimestamp(oldestActiveConnection.getLastUsedTimestamp());
	              //修改旧代理类为不可用
	              oldestActiveConnection.invalidate();
	              if (log.isDebugEnabled()) {
	                log.debug("Claimed overdue connection " + conn.getRealHashCode() + ".");
	              }
	            } else {
	              //当不能创建新连接，活跃连接中也没有超过过期时间的连接时，只能等待了=。=
	              try {
	                if (!countedWait) {
	                  state.hadToWaitCount++;
	                  countedWait = true;
	                }
	                if (log.isDebugEnabled()) {
	                  log.debug("Waiting as long as " + poolTimeToWait + " milliseconds for connection.");
	                }
	                long wt = System.currentTimeMillis();
	                //默认的等待时间和连接过期时间一样是20s，所以如果不做修改的话，20s之后必然会有连接可用，如果短时间内爆发超过处理能力数倍的并发请求的话，就不是能在这里处理的了
	                state.wait(poolTimeToWait);
	                state.accumulatedWaitTime += System.currentTimeMillis() - wt;
	              } catch (InterruptedException e) {
	                break;
	              }
	            }
	          }
	        }
	        if (conn != null) {
	          if (conn.isValid()) {
	        	//当获取的连接可用时，首先将连接rollback，防止有未提交操作
	            if (!conn.getRealConnection().getAutoCommit()) {
	              conn.getRealConnection().rollback();
	            }
	            conn.setConnectionTypeCode(assembleConnectionTypeCode(dataSource.getUrl(), username, password));
	            conn.setCheckoutTimestamp(System.currentTimeMillis());
	            conn.setLastUsedTimestamp(System.currentTimeMillis());
	            //将连接放入活跃队列
	            state.activeConnections.add(conn);
	            state.requestCount++;
	            state.accumulatedRequestTime += System.currentTimeMillis() - t;
	          } else {
	        	//当获取的连接不可用时，增加一个损坏连接数，如果损坏连接数达到阈值，则抛出异常
	            if (log.isDebugEnabled()) {
	              log.debug("A bad connection (" + conn.getRealHashCode() + ") was returned from the pool, getting another connection.");
	            }
	            state.badConnectionCount++;
	            localBadConnectionCount++;
	            conn = null;
	            if (localBadConnectionCount > (poolMaximumIdleConnections + 3)) {
	              if (log.isDebugEnabled()) {
	                log.debug("PooledDataSource: Could not get a good connection to the database.");
	              }
	              throw new SQLException("PooledDataSource: Could not get a good connection to the database.");
	            }
	          }
	        }
	      }
	    }

	    if (conn == null) {
	      if (log.isDebugEnabled()) {
	        log.debug("PooledDataSource: Unknown severe error condition.  The connection pool returned a null connection.");
	      }
	      throw new SQLException("PooledDataSource: Unknown severe error condition.  The connection pool returned a null connection.");
	    }

	    return conn;
	  }

	/**
	   * 检查连接是否可用
	   * @param conn - the connection to check
	   * @return True if the connection is still usable
	   */
	  protected boolean pingConnection(PooledConnection conn) {
	    boolean result = true;

	    try {
	      //1、校验连接是否已关闭
	      result = !conn.getRealConnection().isClosed();
	    } catch (SQLException e) {
	      if (log.isDebugEnabled()) {
	        log.debug("Connection " + conn.getRealHashCode() + " is BAD: " + e.getMessage());
	      }
	      result = false;
	    }

	    if (result) {
	      //开启心跳测试
	      if (poolPingEnabled) {
	    	//连接的空闲时间超过心跳测试的触发时间
	        if (poolPingConnectionsNotUsedFor >= 0 && conn.getTimeElapsedSinceLastUse() > poolPingConnectionsNotUsedFor) {
	          try {
	            if (log.isDebugEnabled()) {
	              log.debug("Testing connection " + conn.getRealHashCode() + " ...");
	            }
	            //尝试执行心跳测试
	            Connection realConn = conn.getRealConnection();
	            Statement statement = realConn.createStatement();
	            ResultSet rs = statement.executeQuery(poolPingQuery);
	            rs.close();
	            statement.close();
	            if (!realConn.getAutoCommit()) {
	              realConn.rollback();
	            }
	            result = true;
	            if (log.isDebugEnabled()) {
	              log.debug("Connection " + conn.getRealHashCode() + " is GOOD!");
	            }
	          } catch (Exception e) {
	            log.warn("Execution of ping query '" + poolPingQuery + "' failed: " + e.getMessage());
	            try {
	              conn.getRealConnection().close();
	            } catch (Exception e2) {
	            }
	            result = false;
	            if (log.isDebugEnabled()) {
	              log.debug("Connection " + conn.getRealHashCode() + " is BAD: " + e.getMessage());
	            }
	          }
	        }
	      }
	    }
	    return result;
	  }

# 5. 使用连接池有什么好处
　　首先我们来看一下创建连接的性能消耗。

	try {
		Class.forName("com.mysql.jdbc.Driver");
		String url = "jdbc:mysql://192.168.1.39:3306/dlb";
		long createConnectionTime = System.currentTimeMillis();
		Connection conn = DriverManager.getConnection(url,"ymt","yimiaotong2015");
		System.out.println("创建连接消耗的时间："+(System.currentTimeMillis() - createConnectionTime));
					
		long executeTime = System.currentTimeMillis();
		PreparedStatement ps = conn.prepareStatement("select phoneNO from users where id = 1");
		ResultSet rs = ps.executeQuery();
		System.out.println("执行sql消耗时间："+(System.currentTimeMillis() - executeTime));
					
	} catch (ClassNotFoundException e) {
		e.printStackTrace();
	} catch (SQLException e) {
		e.printStackTrace();
	}

	![](/images/mybatis_17.png)

创建一个连接需要178ms，如果请求量上来了，那画面太美。恰巧还是一个“不谙世事”的程序员，天天追着老板买新服务器，估计老板砍死你的心都有了。笑
连接池中的连接在执行sql操作之后不会进行close，而是调用pushConnection进行调度，将连接在活跃和空闲队列中来回流转，等待执行任务，这样就节约了每次都要创建连接的开销。

# 6. 何时创建Connection
　　当我们需要执行一个sql语句时，Executor会调用getConnection方法创建一个Connection。

	protected Connection getConnection(Log statementLog) throws SQLException {
	    Connection connection = transaction.getConnection();
	    if (statementLog.isDebugEnabled()) {
	      return ConnectionLogger.newInstance(connection, statementLog, queryStack);
	    } else {
	      return connection;
	    }
	  }
	public Connection getConnection() throws SQLException {
	    if (connection == null) {
	      openConnection();
	    }
	    return connection;
	  }

# 7. JNDI类型DataSource
　　MyBatis定义了一个JndiDataSourceFactory来创建JDNI形式的数据源，具体配置参考各容器的文档。