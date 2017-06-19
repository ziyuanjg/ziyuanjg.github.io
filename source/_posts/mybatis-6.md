---
title: MyBatis源码解析：ResultSetHandler
date: 2017-06-15 15:16:44
tags: MyBatis
---

　　现在要说第三个苦力ResultSetHandler（负责处理结果集映射）了，还记得在mapper配置中有一个标签叫resultMap，还有一个叫resultType，这两个标签就是这个时候用的了。ResultSetHandler接口只有一个默认的实现类，下面我们来看看关键的几个方法：
<!-- more -->

	/**
	 * 依照配置的返回类型映射查询结果集
	 */
	 @Override
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

	/**
	 * 按照规则映射结果集数据
	 */
	private void handleResultSet(ResultSetWrapper rsw, ResultMap resultMap, List<Object> multipleResults, ResultMapping parentMapping) throws SQLException {
	    try {
	      //parentMapping不为空是指嵌套结果集，在mapper配置中的resultMap标签控制
	      if (parentMapping != null) {
	        handleRowValues(rsw, resultMap, null, RowBounds.DEFAULT, parentMapping);
	      } else {
	        if (resultHandler == null) {//有多个结果集
	          //创建一个默认的结果集对象
	          DefaultResultHandler defaultResultHandler = new DefaultResultHandler(objectFactory);
	          //将查询结果按照字段与bean属性对应关系进行绑定
	          handleRowValues(rsw, resultMap, defaultResultHandler, rowBounds, null);
	          //将结果集添加到multipleResults末尾
	          multipleResults.add(defaultResultHandler.getResultList());
	        } else {//简单结果集
	          handleRowValues(rsw, resultMap, resultHandler, rowBounds, null);
	        }
	      }
	    } finally {
	      // issue #228 (close resultsets)
	      closeResultSet(rsw.getResultSet());
	    }
	  }
	/**
	 * 映射数据
	 */
	public void handleRowValues(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {
		if (resultMap.hasNestedResultMaps()) {
	      ensureNoRowBounds();
	      checkResultHandler();
	      //嵌套的结果集
	      handleRowValuesForNestedResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
	    } else {
	      //普通的结果集
	      handleRowValuesForSimpleResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
	    }
	  }
	/**
	 * 映射嵌套结果集
	 */
	private void handleRowValuesForNestedResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {
	    //创建结果集上下文对象
		final DefaultResultContext<Object> resultContext = new DefaultResultContext<Object>();
	    //跳过rowBounds指定offset行偏移量
	    skipRows(rsw.getResultSet(), rowBounds);
	    //前一行数据
	    Object rowValue = previousRowValue;
	    while (shouldProcessMoreRows(resultContext, rowBounds) && rsw.getResultSet().next()) {
	      //鉴别器通过制定字段获取另一个结果的映射
	      final ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(rsw.getResultSet(), resultMap, null);
	      //获取结果集的唯一key
	      final CacheKey rowKey = createRowKey(discriminatedResultMap, rsw, null);
	      //从缓存中获取key对应的结果对象
	      Object partialObject = nestedResultObjects.get(rowKey);
	      //获取配置项resultOrdered，是否为嵌套结果集。
	      if (mappedStatement.isResultOrdered()) {
	        if (partialObject == null && rowValue != null) {
	          nestedResultObjects.clear();
	          //存储结果对象
	          storeObject(resultHandler, resultContext, rowValue, parentMapping, rsw.getResultSet());
	        }
	        //获取行值
	        rowValue = getRowValue(rsw, discriminatedResultMap, rowKey, null, partialObject);
	      } else {
	        rowValue = getRowValue(rsw, discriminatedResultMap, rowKey, null, partialObject);
	        if (partialObject == null) {
	          storeObject(resultHandler, resultContext, rowValue, parentMapping, rsw.getResultSet());
	        }
	      }
	    }
	    if (rowValue != null && mappedStatement.isResultOrdered() && shouldProcessMoreRows(resultContext, rowBounds)) {
	      storeObject(resultHandler, resultContext, rowValue, parentMapping, rsw.getResultSet());
	      previousRowValue = null;
	    } else if (rowValue != null) {
	      previousRowValue = rowValue;
	    }
	  }
	/**
	 * 简单结果集的绑定行数据方法
	 */
	private void handleRowValuesForSimpleResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping)
	      throws SQLException {
	    DefaultResultContext<Object> resultContext = new DefaultResultContext<Object>();
	    //跳过rowBounds指定offset行偏移量
	    skipRows(rsw.getResultSet(), rowBounds);
	    while (shouldProcessMoreRows(resultContext, rowBounds) && rsw.getResultSet().next()) {
	      ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(rsw.getResultSet(), resultMap, null);
	      //获取一行数据
	      Object rowValue = getRowValue(rsw, discriminatedResultMap);
	      //存储数据对象
	      storeObject(resultHandler, resultContext, rowValue, parentMapping, rsw.getResultSet());
	    }
	  }