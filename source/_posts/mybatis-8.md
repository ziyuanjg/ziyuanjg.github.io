---
title: MyBatis源码解析：Mybatis的缓存机制
date: 2017-06-15 15:16:50
tags: MyBatis
---

　　俗话说不搞缓存的ORM不是好框架（笑），咱大MyBatis自然也不能免俗了，这就提供了一级缓存和二级缓存来应个景。
# 1. 缓存的作用
　　在实际的业务场景里，通常查询操作比修改操作多很多，当并发量很大的时候就会让服务器不堪重负，但是老板又不给批款购买新机器，这可怎么办呢？只能发挥程序员的奇思妙想提升程序的性能了，既然与数据库的会话会消耗很多性能，那就将使用最多的查询功能提取出来，将常用查询语句的结果放在内存中缓存，这样每次先在内存中查询是否有缓存数据，如果没有才会创建一个数据库会话，是不是就快很多了。但是这也有一个新的问题，就是缓存的命中率，如果架构设计的时候设计不当，导致常用的语句没设置缓存，不常用的却设置了缓存，反而会降低系统性能。
<!-- more -->
# 2. Cache
　　MyBatis的缓存基础是PerpetualCache，PerpetualCache实现了Cache接口，内部其实是维护了一个HashMap对象进行数据的缓存，下面看下Cache的家族树：
![](/images/mybatis_9.png)

乍一看哎呦喂，挺多实现类啊，其实这里使用了装饰者模式，其他的实现类都是装饰了PerpetualCache类。

# 3. 一级缓存
　　MyBatis的一级缓存默认就是开启的（想关都关不掉），我们能做的就是设置一下他的生命周期，配置如下：

	<settings>
	 	<setting name="localCacheScope" value="SESSION|STATEMENT" />
	 </settings>
如果配置为Session，就是以一次数据库会话为周期，在实际开发中通常一次请求只创建一次会话，所以也可以理解为以一次请求为周期的缓存。
如果配置为Statement，则仅作用于语句的执行上，不会影响同一会话的其他调用。

下面我们来看下MyBatis的一级缓存是如何工作的：
![](/images/mybatis_10.png)

首先创建一个新的数据库会话，此时会实例化一个Executor

	//实例化Executor执行器
	final Executor executor = configuration.newExecutor(tx, execType);
	在BaseExecutor维护了一个PerpetualCache，这个就是所说的一级缓存（也叫本地缓存）了
	//一级缓存（本地缓存）
	protected PerpetualCache localCache;
	在查询的时候会首先在这个一级缓存中查询是否有值
	list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
	if (list != null) {
	     //缓存中存在结果集时
	     handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
	} else {
	     //从数据库获取数据，此处会将结果保存在localCache中
	     list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
	}

　　正确的使用缓存确实能提高不少性能，但是这里就引出了一个新问题，如果我缓存的数据过期了怎么办？对于一些时效性不强的模块中有一些脏数据或许不会影响使用，但是对于要求数据时效性很强的系统中就不能这么玩了，所以MyBatis提供了手动配置刷新时机的方法。

一共有五个触发方法会刷新一级缓存：
* commit
* rollback
* update
* query（如果配置了flushCache属性为true或localCacheScope设置为Statement）

只有query中的两个触发点是需要手动配置，其余三个都会自动清空一级缓存区。
![](/images/mybatis_11.png)
![](/images/mybatis_12.png)

　　既然是大量操作的时候才会用到缓存，那么缓存的数据量想必是很大的，如何确定每次操作要请求的缓存数据对象就是一个问题了，在MyBatis中有一个叫做cacheKey的东西，这个就是用来确定缓存唯一性的key了，下面看一下生成cacheKey的方法：

	public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
	    if (closed) {
	      throw new ExecutorException("Executor was closed.");
	    }
	    CacheKey cacheKey = new CacheKey();
	    cacheKey.update(ms.getId());
	    cacheKey.update(rowBounds.getOffset());
	    cacheKey.update(rowBounds.getLimit());
	    cacheKey.update(boundSql.getSql());
	    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
	    TypeHandlerRegistry typeHandlerRegistry = ms.getConfiguration().getTypeHandlerRegistry();
	    // mimic DefaultParameterHandler logic
	    for (ParameterMapping parameterMapping : parameterMappings) {
	      if (parameterMapping.getMode() != ParameterMode.OUT) {
	        Object value;
	        String propertyName = parameterMapping.getProperty();
	        if (boundSql.hasAdditionalParameter(propertyName)) {
	          value = boundSql.getAdditionalParameter(propertyName);
	        } else if (parameterObject == null) {
	          value = null;
	        } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
	          value = parameterObject;
	        } else {
	          MetaObject metaObject = configuration.newMetaObject(parameterObject);
	          value = metaObject.getValue(propertyName);
	        }
	        cacheKey.update(value);
	      }
	    }
	    if (configuration.getEnvironment() != null) {
	      // issue #176
	      cacheKey.update(configuration.getEnvironment().getId());
	    }
	    return cacheKey;
	  }
这里不难看出实际上MyBatis采用的是符合型key值，key的组合值有以下六项：
* MappedStatementID
* offset
* limit
* sql
* param value
* environmentID

通过这六个组合参数可以看出MyBatis对于相同查询的定义：
* MappedStatementID必须相同，在构建Configuration时会为每个mapper方法创建一个唯一MappedStatement，所以这是第一个约束。
* 查询范围必须相同，RowBounds中的offset和limit共同约束了本次查询的范围。这种分页是基于查询结果进行再次过滤，并不是数据库的物理分页。
* 传递给JDBC的sql必须相同（BoundSql.getSql()）。由于MyBatis底层也是依赖于JDBC实现，对于JDBC来说只要传入的sql和参数相同就视为相同查询。
* 查询语句的参数必须相同，这里有一个盲点，参数相同指的是sql有用到的参数必须相同，比如传入一个user对象，如果sql没有用到user.name这个属性，那么不论name的值是否相同都不会影响判断。
* 数据源必须相同，这个就不用说了。

一级缓存本质上是维护了一个HashMap，那么作为key的CacheKey就需要有一个统一的hash算法进行支持了，下面看下CacheKey的核心算法:

	public void update(Object object) {
	    if (object != null && object.getClass().isArray()) {
	      int length = Array.getLength(object);
	      for (int i = 0; i < length; i++) {
	        Object element = Array.get(object, i);
	        doUpdate(element);
	      }
	    } else {
	      doUpdate(object);
	    }
	  }
	  private void doUpdate(Object object) {
	    //得到对象的hashCode
	    int baseHashCode = object == null ? 1 : object.hashCode();
	    //计数器+1
	    count++;
	    //累加CacheKey组合参数的hashCode
	    checksum += baseHashCode;
	    //扩大对象的hashCode为count倍
	    baseHashCode *= count;
	    //CacheKey的hashCode变为原hashCode*扩展系数（默认37）+此对象的hashCode
	    hashcode = multiplier * hashcode + baseHashCode;
	    updateList.add(object);
	  }
　　说了这么多，那么一级缓存的性能如何呢？
　　从空间角度来看，一级缓存只是简单的使用的HashMap进行维护，并且没有对Has和Map的容量进行限制，这时候可能就有小伙伴会问了，这岂不是会占用很多内存吗，哪个渣渣写的代码，拖出去斩了=。=。MyBatis确实没在空间方面做硬性限制，但是呢在操作方式上其实还是有软限制的，一级缓存的生存周期是一个SqlSession会话，通常情况下每个request会新建一个SqlSession，一个request能执行多长时间（如果您的项目对SqlSession使用单例实现，请默默点击右上角的叉叉，笑），而且呢就算是这个SqlSession生活节奏比较慢，就是拖着不让回收也没问题，每次执行update操作的时候MyBatis会清空一次一级缓存，更极端点就算是这个死拖着不走的SqlSession全是查询操作，不还是可以手动清理缓存区的么（笑）。
从时间的角度看，MyBatis的一级缓存是粗粒度的，没有缓存过期的概念，也就是不会主动请求数据源对缓存进行更新，
由上面两点来看呢，一级缓存的一点不泛用性就展现出来了，对于数据变化的频率高，且对数据时效性要求高的业务来说就要控制每个SqlSession的生存时间了，时间越长缓存中的数据的准确性就越低了。

# 4. 二级缓存
　　一级缓存是在一个SqlSession会话的周期内有效，那么不同的SqlSession想一起玩怎么办呢，这时候就用到了二级缓存。
![](/images/mybatis_13.png)

二级缓存简单来说就是在Configuration中维护的一个HashMap集合，MyBatis默认使用的和一级缓存一样是PerpetualCache这个实现（只不过使用TransactionalCache装饰了一下），与一级缓存不同，二级缓存是需要手动开启的，要使用二级缓存需要以下两个步骤：
* 配置config.xml：

	<setting name="cacheEnabled" value="true"/>

* 配置mapper.xml：

	<cache></cache><cache-ref namespace="daoMapper.UserMapper"/>

由上面可知，mapper有两种配置方式，其中cache标签是声明此mapper使用二级缓存，具体参数如下：
* eviction：缓存刷新机制，默认可用的回收策略有四种，LRU（清理最近最少使用的缓存）、FIFO（先进先出）、SOFT（ 软引用:移除基于垃圾回收器状态和软引用规则的对象）和WEAK（更积极地移除基于垃圾收集器状态和弱引用规则的对象。）
* flushInterval：刷新间隔，可以被设置为任意的正整数,而且它们代表一个合理的毫秒 形式的时间段。默认情况是不设置,也就是没有刷新间隔,缓存仅仅调用语句时刷新。
* size：引用数目，可以被设置为任意正整数,要记住你缓存的对象数目和你运行环境的 可用内存资源数目。默认值是 1024。
* readOnly：只读属性，可以被设置为 true 或 false。只读的缓存会给所有调用者返回缓 存对象的相同实例。因此这些对象不能被修改。这提供了很重要的性能优势。可读写的缓存 会返回缓存对象的拷贝(通过序列化) 。这会慢一些,但是安全,因此默认是 false。

一级缓存是在BaseExecutor中进行操作，那二级缓存想要在一级之前调用自然就要在上一级了，MyBatis采用的是装饰器模式，当cacheEnabled设置为true时，默认创建的Executor实例都会用cacheEnabled进行装饰，下面看下cacheEnabled的查询源码：

	 public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
	      throws SQLException {
	    Cache cache = ms.getCache();
	    if (cache != null) {
	      //根据flushCache参数决定是否刷新缓存区
	      flushCacheIfRequired(ms);
	      if (ms.isUseCache() && resultHandler == null) {
	    	//校验boundSql中的绑定参数列表是否包含输出参数，如果包含则抛出异常，Mybatis不支持存储过程结果缓存
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

这里的List<E> list = (List<E>) tcm.getObject(cache, key);是重点，画一下，考试要考，咳咳。
此处可以看出cache其实是从MappedStatement中取出的，正是因为MappedStatement在MyBatis初始化时就全部创建完毕，所以不会和SqlSession的生命周期挂钩。下面看下之后的方法：

	public Object getObject(Cache cache, CacheKey key) {
	    return getTransactionalCache(cache).getObject(key);
	  }
	private TransactionalCache getTransactionalCache(Cache cache) {
	    TransactionalCache txCache = transactionalCaches.get(cache);
	    //如果此SqlSession的缓存集合中不包含cache，则创建cache的装饰类TransactionalCache并放入transactionalCaches中
	    if (txCache == null) {
	      txCache = new TransactionalCache(cache);
	      transactionalCaches.put(cache, txCache);
	    }
	    return txCache;
	  }
	  public Object getObject(Object key) {
	    // 此处实际调用的是默认实现PerpetualCache的getObject方法
	    Object object = delegate.getObject(key);
	    if (object == null) {
	      //此处是将缓存缓存的key放入缓存队列中，并没有直接放入二级缓存，只有调用commit方法才会将队列中的key依次放入二级缓存
	      entriesMissedInCache.add(key);
	    }
	    // issue #146
	    if (clearOnCommit) {
	      return null;
	    } else {
	      return object;
	    }
	  }

上面说了二级缓存是从MappedStatement取出的，那么现在就来找一下二级缓存是如何创建的，之前讲创建SqlSessionFactory的过程时有提到过解析xml配置文件的方法，既然二级缓存的开关是在mapper.xml文件中，那么创建二级缓存的入口自然要落在解析mapper的方法中，下面来看下源码：

	private void cacheElement(XNode context) throws Exception {
	    if (context != null) {
	      //自定义缓存类型
	      String type = context.getStringAttribute("type", "PERPETUAL");
	      Class<? extends Cache> typeClass = typeAliasRegistry.resolveAlias(type);
	      //缓存过期类型装饰类
	      String eviction = context.getStringAttribute("eviction", "LRU");
	      Class<? extends Cache> evictionClass = typeAliasRegistry.resolveAlias(eviction);
	      //缓存刷新间隔
	      Long flushInterval = context.getLongAttribute("flushInterval");
	      //引用数量
	      Integer size = context.getIntAttribute("size");
	      //是否只读
	      boolean readWrite = !context.getBooleanAttribute("readOnly", false);
	      boolean blocking = context.getBooleanAttribute("blocking", false);
	      //cache标签张配置的propertie属性
	      Properties props = context.getChildrenAsProperties();
	      //创建缓存对象
	      builderAssistant.useNewCache(typeClass, evictionClass, flushInterval, size, readWrite, blocking, props);
	    }
	  }
	public Cache useNewCache(Class<? extends Cache> typeClass,
	      Class<? extends Cache> evictionClass,
	      Long flushInterval,
	      Integer size,
	      boolean readWrite,
	      boolean blocking,
	      Properties props) {
		//创建缓存对象
	    Cache cache = new CacheBuilder(currentNamespace)
	        .implementation(valueOrDefault(typeClass, PerpetualCache.class))
	        .addDecorator(valueOrDefault(evictionClass, LruCache.class))
	        .clearInterval(flushInterval)
	        .size(size)
	        .readWrite(readWrite)
	        .blocking(blocking)
	        .properties(props)
	        .build();
	    //将缓存对象添加至caches
	    configuration.addCache(cache);
	    //此处留待之后创建MappedStatement使用
	    currentCache = cache;
	    return cache;
	  }
	public Cache build() {
		//设置默认实现类PerpetualCache和装饰类LruCache
	    setDefaultImplementations();
	    //创建PerpetualCache实例
	    Cache cache = newBaseCacheInstance(implementation, id);
	    //设置propertie属性
	    setCacheProperties(cache);
	    if (PerpetualCache.class.equals(cache.getClass())) {
	      //装饰者
	      for (Class<? extends Cache> decorator : decorators) {
	        cache = newCacheDecoratorInstance(decorator, cache);
	        setCacheProperties(cache);
	      }
	      //根据flushInterval，readOnly，blocking属性装饰缓存
	      cache = setStandardDecorators(cache);
	    } else if (!LoggingCache.class.isAssignableFrom(cache.getClass())) {
	      cache = new LoggingCache(cache);
	    }
	    return cache;
	  }
　　如果不想使用MyBatis默认的二级缓存实现，也可以接入第三方缓存，如redis等，或者自定义cache的实现类。
　　二级缓存有很多优点，但是也存在很大的局限性，那就是二级缓存只适合单表，而且这个表的操作都在同一个mapper中定义，因为如果多个mapper都可以操作同一张表，那么缓存中的数据的准确性就无
法保证，多表查询的缓存也是同理。
　　如果实在是想用二级缓存处理多表的数据也不是没有办法，使用拦截器将涉及的所有mapper的缓存全部刷新就可以保证数据的准确性，不过编写这样一个插件还要保证插件的兼容性、可扩展性、稳定性都
达到上线的水准还不如在设计的时候就避免这种坑了。
