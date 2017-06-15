---
title: MyBatis源码解析：SqlSessionFactoryBuilder
date: 2017-06-15 15:12:01
tags: MyBatis
---
　　SqlSessionFactoryBuilder是MyBatis的执行入口，使用MyBatis首先要构建一个SqlSessionFactory容器，SqlSessionFactoryBuilder就是构建SqlSessionFactory的创建者。
　　以下是SqlSessionFactoryBuilder的OutLine：
![](/images/mybatis_4.png)
<!-- more -->
实际执行的build只有三个:
* build(InputStream inputStream, String environment, Properties properties)
* build(Reader reader, String environment, Properties properties)
* build(Configuration config)

　　前两种的区别只有使用字符流还是字节流，本质上都是通过读取MyBatis的配置文件来进行初始化工作。其中environment可以指定使用的数据源，需要使用多个库的时候会用到这个参数，需要注意的是每个SqlSessionFactory只能对应一个数据源。properties参数可以定义额外的配置参数，与properties文件中配置的功能相同。而第三种是以编程的方式对MyBatis容器进行初始化。（注：前两种最后还是会调用第三种方法）

下面取第一个build的源码进行示例：

	public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
		try {
		  XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
		  return build(parser.parse());
		} catch (Exception e) {
		  throw ExceptionFactory.wrapException("Error building SqlSession.", e);
		} finally {
		  ErrorContext.instance().reset();
		  try {
		    inputStream.close();
		  } catch (IOException e) {
		    // Intentionally ignore. Prefer previous error.
		  }
		}
	}
由上面代码一看可知，初始化配置的关键在于XMLConfigBuilder这个解析类的parse()，parse()方法对MyBatis的一系列标签进行归类注册，创建容器实例，具体代码如下：

	private void parseConfiguration(XNode root) {
		try {
		  propertiesElement(root.evalNode("properties"));
		  Properties settings = settingsAsProperties(root.evalNode("settings"));
		  loadCustomVfs(settings);
		  typeAliasesElement(root.evalNode("typeAliases"));
		  pluginElement(root.evalNode("plugins"));
		  objectFactoryElement(root.evalNode("objectFactory"));
		  objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
		  reflectorFactoryElement(root.evalNode("reflectorFactory"));
		  settingsElement(settings);
		  environmentsElement(root.evalNode("environments"));
		  databaseIdProviderElement(root.evalNode("databaseIdProvider"));
		  typeHandlerElement(root.evalNode("typeHandlers"));
		  mapperElement(root.evalNode("mappers"));
		} catch (Exception e) {
		  throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
		}
	}
　　值得注意的是properties和environments，这两个在XMLConfigBuilder的构造方法中是可以当做参数传入的，其中properties会在初始化properties标签时将参数中的properties添加到对应集合中，而environments如果不为空，则会使用id为environments对应值的数据源，否则会使用默认的数据源。其余每个方法对应的都是MyBatis配置文件中的一种标签，具体内容可以在MyBatis的官方文档查询，此处不再赘述。