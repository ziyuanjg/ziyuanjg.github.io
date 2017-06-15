---
title: MyBatis源码解析：ParameterHandler
date: 2017-06-15 15:16:47
tags: MyBatis
---

　　之前在Statementhandler中有一个成员变量叫ParameterHandler，这个就是第二个苦力（负责绑定参数）了，MyBatis中ParameterHandler只有一个默认实现，下面看一下绑定参数方法的源码：
<!-- more -->

	public void setParameters(PreparedStatement ps) {
	    ErrorContext.instance().activity("setting parameters").object(mappedStatement.getParameterMap().getId());
	    //获取sql对象中的参数集合
	    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
	    if (parameterMappings != null) {
	      for (int i = 0; i < parameterMappings.size(); i++) {
	    	//获取参数对象
	        ParameterMapping parameterMapping = parameterMappings.get(i);
	        //存储过程的输出参数在之前的CallableStatementHandler中绑定，此处不在赘述
	        if (parameterMapping.getMode() != ParameterMode.OUT) {
	          Object value;
	          String propertyName = parameterMapping.getProperty();
	          //获取需绑定参数的值
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
	          //获取此参数对应的类型处理器
	          TypeHandler typeHandler = parameterMapping.getTypeHandler();
	          JdbcType jdbcType = parameterMapping.getJdbcType();
	          //如果参数值和jdbc类型为null，则此参数使用配置中指定的null类型表示，默认为JdbcType.OTHER
	          if (value == null && jdbcType == null) {
	            jdbcType = configuration.getJdbcTypeForNull();
	          }
	          try {
	        	//将参数绑定到PreparedStatement
	            typeHandler.setParameter(ps, i + 1, value, jdbcType);
	          } catch (TypeException e) {
	            throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
	          } catch (SQLException e) {
	            throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
	          }
	        }
	      }
	    }
	  }
