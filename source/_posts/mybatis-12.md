---
title: MyBatis源码解析：动态SQL
date: 2017-06-15 15:17:11
tags: MyBatis
---

## MyBatis中和动态SQl相关的类
### BaseBuilder
![](/images/mybatis_24.png)
<!-- more -->

BaseBuilder主要负责解析xml配置文件，以及构建各种相关的实例：
* XMLConfigBuilder：解析全局xml配置文件
* XMLMapperBuilder：解析mapper节点配置
* XMLStatementBuilder：解析CRUD节点配置
* XMLScriptBuilder：解析节点中的sql
* MapperBuilderAssistant：辅助构建工具类
* SqlSourceBuilder：sql源构建类

### SqlNode
![](/images/mybatis_25.png)

SqlNode对应的就是动态sql的各种标签了：
* ChooseSqlNode：对应choose标签，类似于switch功能
* ForEachSqlNode：对应forEach标签
* IfSqlNode：对应if标签和when标签
* MixedSqlNode：SqlNode集合类，内部维护了一个SqlNode集合，储存一个Statement中的所有动态sql节点元素
* StaticTextSqlNode：静态sql节点
* TextSqlNode：文本型节点的默认类型
* TrimSqlNode：过滤sql前后缀，支持set和where类型
* SetSqlNode：TrimSqlNode的set调用
* WhereSqlNode：TrimSqlNode的where调用
* VarDeclSqlNode：对应bind标签，用于解析Ognl表达式

### SqlSource
![](/images/mybatis_26.png)
SqlSource是处理xml配置文件中的sql部分的工具：
* DynamicSqlSource：创建动态sqlSourse
* ProviderSqlSource：处理通用mapper
* RawSqlSource：处理静态sql
* StaticSqlSource：创建静态sqlSourse

### LanguageDriver
　　LanguageDriver主要实现类是XMLLanguageDriver，XMLLanguageDriver内部使用XMLScriptBuilder解析sql节点，用于生成SqlSource。

## 动态Sql创建流程
![](/images/mybatis_26.png)

　　mapper的路径既然是在全局xml中配置的，那么入口自然也要落在创建运行环境上下文中

	private void parseConfiguration(XNode root) {
	    try {
	      //TODO
		  //此处进入解析mappers节点流程
	      mapperElement(root.evalNode("mappers"));
	    } catch (Exception e) {
	      throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
	    }
	  }
	private void mapperElement(XNode parent) throws Exception {
	    if (parent != null) {
	      for (XNode child : parent.getChildren()) {
	        if ("package".equals(child.getName())) {
	          //如果配置的是package，会遍历包下所有的mapper进行注册
	          String mapperPackage = child.getStringAttribute("name");
	          configuration.addMappers(mapperPackage);
	        } else {
	          //三种配置方式由县级为Resource>url>class
	          String resource = child.getStringAttribute("resource");
	          String url = child.getStringAttribute("url");
	          String mapperClass = child.getStringAttribute("class");
	          if (resource != null && url == null && mapperClass == null) {
	            ErrorContext.instance().resource(resource);
	            InputStream inputStream = Resources.getResourceAsStream(resource);
	            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
	            //处理内部mapper节点
	            mapperParser.parse();
	          } else if (resource == null && url != null && mapperClass == null) {
	            ErrorContext.instance().resource(url);
	            InputStream inputStream = Resources.getUrlAsStream(url);
	            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
	            mapperParser.parse();
	          } else if (resource == null && url == null && mapperClass != null) {
	            Class<?> mapperInterface = Resources.classForName(mapperClass);
	            configuration.addMapper(mapperInterface);
	          } else {
	            throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");
	          }
	        }
	      }
	    }
	  }
　　下面就进入了XMLMapperBuilder中：

	public void parse() {
	    if (!configuration.isResourceLoaded(resource)) {
	      //避免重复加载同一mapper
	      //解析mapper节点
	      configurationElement(parser.evalNode("/mapper"));
	      //在缓存中添加mapper标识，避免重复加载
	      configuration.addLoadedResource(resource);
	      //绑定命名空间
	      bindMapperForNamespace();
	    }

	    parsePendingResultMaps();
	    parsePendingChacheRefs();
	    parsePendingStatements();
	  }

	private void configurationElement(XNode context) {
	    try {
	      String namespace = context.getStringAttribute("namespace");
	      if (namespace == null || namespace.equals("")) {
	        throw new BuilderException("Mapper's namespace cannot be empty");
	      }
	      builderAssistant.setCurrentNamespace(namespace);
	      //处理缓存标签，cache会覆盖cache-ref
	      cacheRefElement(context.evalNode("cache-ref"));
	      cacheElement(context.evalNode("cache"));
	      //已废弃，不推荐使用
	      parameterMapElement(context.evalNodes("/mapper/parameterMap"));
	      //处理外部resultMap引用
	      resultMapElements(context.evalNodes("/mapper/resultMap"));
	      //处理sql片段
	      sqlElement(context.evalNodes("/mapper/sql"));
	      //处理CRUD，也是动态sql的关键
	      buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
	    } catch (Exception e) {
	      throw new BuilderException("Error parsing Mapper XML. Cause: " + e, e);
	    }
	  }

	  private void buildStatementFromContext(List<XNode> list) {
	    if (configuration.getDatabaseId() != null) {
	      buildStatementFromContext(list, configuration.getDatabaseId());
	    }
	    buildStatementFromContext(list, null);
	  }

	  private void buildStatementFromContext(List<XNode> list, String requiredDatabaseId) {
	    for (XNode context : list) {
	      final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, builderAssistant, context, requiredDatabaseId);
	      try {
	        statementParser.parseStatementNode();
	      } catch (IncompleteElementException e) {
	        configuration.addIncompleteStatement(statementParser);
	      }
	    }
	  }

　　接下来进入XMLStatementBuilder 处理具体sql内容：

	public void parseStatementNode() {
	    //.....
	    LanguageDriver langDriver = getLanguageDriver(lang);
	    //.....
	    //创建sql源
	    SqlSource sqlSource = langDriver.createSqlSource(configuration, context, parameterTypeClass);
	    //.......
	  }

　　由于是动态sql，所以下面进入XMLLanguageDriver实现：

	public SqlSource createSqlSource(Configuration configuration, XNode script, Class<?> parameterType) {
	    XMLScriptBuilder builder = new XMLScriptBuilder(configuration, script, parameterType);
	    return builder.parseScriptNode();
	  }

　　XMLLanguageDriver很懒，立即将任务转给了XMLScriptBuilder：

	public SqlSource parseScriptNode() {
	    //遍历sql中的所有节点，判断是动态sql还是静态sql
	    List<SqlNode> contents = parseDynamicTags(context);
	    //MixedSqlNode表示sql节点集合
	    MixedSqlNode rootSqlNode = new MixedSqlNode(contents);
	    SqlSource sqlSource = null;
	    if (isDynamic) {
	      //动态sql
	      sqlSource = new DynamicSqlSource(configuration, rootSqlNode);
	    } else {
	      //静态sql
	      sqlSource = new RawSqlSource(configuration, rootSqlNode, parameterType);
	    }
	    return sqlSource;
	  }

　　至此就完成了mapper配置文件的读取注册流程。

## 通过sql模板生成动态sql流程
![](/images/mybatis_28.png)

　　既然是动态sql，那自然是运行时才会根据参数生成执行sql了，下面以query为例，简析生成动态sql的流程：
　　首先在CachingExecutor中调用query

	public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
	    BoundSql boundSql = ms.getBoundSql(parameterObject);
	    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
	    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
	  }
　　可以看到此时调用了MappedStatement的getBoundSql方法，申请创建BoundSql实例

	  public BoundSql getBoundSql(Object parameterObject) {
		//根据参数动态生成sql对象
	    BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
	    //参数队列
	    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
	    if (parameterMappings == null || parameterMappings.isEmpty()) {
	      boundSql = new BoundSql(configuration, boundSql.getSql(), parameterMap.getParameterMappings(), parameterObject);
	    }

	    for (ParameterMapping pm : boundSql.getParameterMappings()) {
	      String rmId = pm.getResultMapId();
	      if (rmId != null) {
	        ResultMap rm = configuration.getResultMap(rmId);
	        if (rm != null) {
	          hasNestedResultMaps |= rm.hasNestedResultMaps();
	        }
	      }
	    }
	    return boundSql;
	  }

　　由于是动态sql，所以此时的sqlSource是DynamicSqlSource类型

	public BoundSql getBoundSql(Object parameterObject) {
	    DynamicContext context = new DynamicContext(configuration, parameterObject);
	    //遍历本sql下的所有子标签（如if，forEach等），与参数进行匹配，生成最终要执行的sql。
	    rootSqlNode.apply(context);
	    //SQLSource的创建者
	    SqlSourceBuilder sqlSourceParser = new SqlSourceBuilder(configuration);
	    //获取参数类型
	    Class<?> parameterType = parameterObject == null ? Object.class : parameterObject.getClass();
	    //创建SQLSource实例，此处其实也是使用的StaticSqlSource
	    SqlSource sqlSource = sqlSourceParser.parse(context.getSql(), parameterType, context.getBindings());
	    BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
	    //将parameterObject和DatabaseId放入boundSql中
	    for (Map.Entry<String, Object> entry : context.getBindings().entrySet()) {
	      boundSql.setAdditionalParameter(entry.getKey(), entry.getValue());
	    }
	    return boundSql;
	  }

　　值得关注的是rootSqlNode.apply(context)，这个方法就是组装动态节点的入口。在上面的创建流程中有提到动态sql的节点集合是包装在MixedSqlNode中的，此处的rootSqlNode自然也就是MixedSqlNode类型

	public boolean apply(DynamicContext context) {
	    for (SqlNode sqlNode : contents) {
	      sqlNode.apply(context);
	    }
	    return true;
	  }

　　在apply中将集合中的参数依次取出，依次拼接。以if标签对应的IfSqlNode为例：

	public boolean apply(DynamicContext context) {
	    if (evaluator.evaluateBoolean(test, context.getBindings())) {
	      contents.apply(context);
	      return true;
	    }
	    return false;
	  }

　　此时便完成了sql的动态生成。