---
title: MyBatis源码解析：SqlSessionFactory
date: 2017-06-15 15:16:35
tags: MyBatis
---


　　使用MyBatis框架时，首先要使用SqlSessionFactoryBuilder.build方法来实例化一个SqlSessionFactory，SqlSessionFactory相当于一个数据库连接工厂。
SqlSessionFactory接口定义了生成SqlSession实例的几种两种方式，MyBatis中有两个默认实现DefaultSqlSessionFactory和SqlSessionManager。SqlSessionManager貌似已经废弃，此处不再写。
<!-- more -->
　　DefaultSqlSessionFactory内部维护了一个Configuration实例，Configuration中的配置属性是生成SqlSession实例的关键。下面是DefaultSqlSessionFactory的OutLine图：
![](/images/mybatis_5.png)

看似很多种重载方法，但是实际执行的只有两个
* openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit)
* openSessionFromConnection(ExecutorType execType, Connection connection)

　　这两个方法的区别就在于是从Configuration中获取事务管理器工厂创建事务管理器，还是从Connection中获取事务管理器。下面取第一个方法做示例：
 
	  private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
	    Transaction tx = null;
	    try {
	      //获取数据源对象
	      final Environment environment = configuration.getEnvironment();
	      //获取事务工厂，如果数据源不存在或者数据源中没有实例化事务工厂，则创建ManagedTransactionFactory实例作为事务工厂使用
	      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
	      //实例化事务控制器
	      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
	      //实例化Executor执行器
	      final Executor executor = configuration.newExecutor(tx, execType);
	      //实例化SqlSession
	      return new DefaultSqlSession(configuration, executor, autoCommit);
	    } catch (Exception e) {
	      closeTransaction(tx); // may have fetched a connection so lets call close()
	      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
	    } finally {
	      ErrorContext.instance().reset();
	    }
	  }　