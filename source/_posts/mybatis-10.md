---
title: MyBatis源码解析：MetaObject
date: 2017-06-15 15:16:55
tags: MyBatis
---

　　MyBatis作为一款ORM框架，反射是必不可少的一环，reflection就是MyBatis的反射模块。从代码角度来看reflection包相当的独立，除了jdk之外没有其他的包外引用，完全可以单独提取出来作为工具使用。
<!-- more -->
![](/images/mybatis_18.png)


## MetaObject创建流程
![](/images/mybatis_19.png)

## MetaObject
![](/images/mybatis_20.png)

　　MetaObject作为reflection暴露在最外层的调用类，不仅承载了提供反射的各种实现调用的入口的责任，还担任了下层调用对象（ObjectWrapper包装者）的分发任务，MetaObject只有一个私有的构造方法，所以想创建MetaObject的实例只能通过MetaObject.forObject方法。

	public static MetaObject forObject(Object object, ObjectFactory objectFactory, ObjectWrapperFactory objectWrapperFactory, ReflectorFactory reflectorFactory) {
	    if (object == null) {
	      return SystemMetaObject.NULL_META_OBJECT;
	    } else {
	      return new MetaObject(object, objectFactory, objectWrapperFactory, reflectorFactory);
	    }
	  }
	private MetaObject(Object object, ObjectFactory objectFactory, ObjectWrapperFactory objectWrapperFactory, ReflectorFactory reflectorFactory) {
	    this.originalObject = object;
	    this.objectFactory = objectFactory;
	    this.objectWrapperFactory = objectWrapperFactory;
	    this.reflectorFactory = reflectorFactory;

	    //根据传入参数判断实例化哪种包装类
	    if (object instanceof ObjectWrapper) {
	      this.objectWrapper = (ObjectWrapper) object;
	    } else if (objectWrapperFactory.hasWrapperFor(object)) {
	      this.objectWrapper = objectWrapperFactory.getWrapperFor(this, object);
	    } else if (object instanceof Map) {
	      this.objectWrapper = new MapWrapper(this, (Map) object);
	    } else if (object instanceof Collection) {
	      this.objectWrapper = new CollectionWrapper(this, (Collection) object);
	    } else {
	      this.objectWrapper = new BeanWrapper(this, object);
	    }
	  }
　　既然MetaObject是以反射为基础，那么总重要的自然是getter和setter的实现，下面看一下MetaObject是如何操作属性的：

	 public Object getValue(String name) {
	    PropertyTokenizer prop = new PropertyTokenizer(name);
	    if (prop.hasNext()) {
	      //实例化指定次级属性的反射实例
	      MetaObject metaValue = metaObjectForProperty(prop.getIndexedName());
	      if (metaValue == SystemMetaObject.NULL_META_OBJECT) {
	    	//如果下级对象是null，则返回null。
	        return null;
	      } else {
	    	//如果传入的是多级参数名，则会依次向下直到获取最原始的被包装对象
	        return metaValue.getValue(prop.getChildren());
	      }
	    } else {
	      //获取指定属性
	      return objectWrapper.get(prop);
	    }
	  }
	public void setValue(String name, Object value) {
	    PropertyTokenizer prop = new PropertyTokenizer(name);
	    if (prop.hasNext()) {
	      //如果传入的是多级参数名，则会依次向下直到获取最原始的被包装对象
	      MetaObject metaValue = metaObjectForProperty(prop.getIndexedName());
	      if (metaValue == SystemMetaObject.NULL_META_OBJECT) {
	    	//此包装类的上一层为NULL
	        if (value == null && prop.getChildren() != null) {
	          //如果值为空且仍有下级对象，则不进行set
	          return;
	        } else {
	          //实例化子级对象的元对象
	          metaValue = objectWrapper.instantiatePropertyValue(name, prop, objectFactory);
	        }
	      }
	      //依次向下调用
	      metaValue.setValue(prop.getChildren(), value);
	    } else {
	      //将属性放入指定对象
	      objectWrapper.set(prop, value);
	    }
	  }
　　由上面代码可以看出，MetaObject作为一个反射入口，关键方法的抽象程度是很高的，在这一层也仅仅处理了对象这一层级的逻辑，凡事涉及具体属性的步骤都下放到ObjectWrapper中执行。

## ObjectWrapper
![](/images/mybatis_21.png)
　　上面的MetaObject将指定的属性定位到最后一级，然后就轮到ObjectWrapper出场了，ObjectWrapper主要的职责就是处理这个最后一级的类型，不仅支持简单属性的操作，还兼容集合类型的属性。

	public Object get(PropertyTokenizer prop) {
	    if (prop.getIndex() != null) {
	      //如果是子级对象是集合类型，则先取出集合对象
	      Object collection = resolveCollection(prop, object);
	      //从集合对象中取出对应属性的值
	      return getCollectionValue(prop, collection);
	    } else {
	      //从bean对象中取出对应属性的值
	      return getBeanProperty(prop, object);
	    }
	  }
	public void set(PropertyTokenizer prop, Object value) {
	    if (prop.getIndex() != null) {
	      //如果是子级对象是集合类型，则先取出集合对象
	      Object collection = resolveCollection(prop, object);
	      //将值放入对象的指定key中
	      setCollectionValue(prop, collection, value);
	    } else {
	      //为指定属性赋值
	      setBeanProperty(prop, object, value);
	    }
	  }
	/**
	 * 实例化子级对象的元对象
	 */
	public MetaObject instantiatePropertyValue(String name, PropertyTokenizer prop, ObjectFactory objectFactory) {
	    MetaObject metaValue;
	    Class<?> type = getSetterType(prop.getName());
	    try {
	      //创建子级对象实例
	      Object newObject = objectFactory.create(type);
	      //创建实例的元对象实例
	      metaValue = MetaObject.forObject(newObject, metaObject.getObjectFactory(), metaObject.getObjectWrapperFactory(), metaObject.getReflectorFactory());
	      //将子级对象放入父级对象中，类似user.setName()
	      set(prop, newObject);
	    } catch (Exception e) {
	      throw new ReflectionException("Cannot set value of property '" + name + "' because '" + name + "' is null and cannot be instantiated on instance of " + type.getName() + ". Cause:" + e.toString(), e);
	    }
	    return metaValue;
	  }
	private Object getBeanProperty(PropertyTokenizer prop, Object object) {
	    try {
	      //通过反射获取该属性的getter方法
	      Invoker method = metaClass.getGetInvoker(prop.getName());
	      try {
	        return method.invoke(object, NO_ARGUMENTS);
	      } catch (Throwable t) {
	        throw ExceptionUtil.unwrapThrowable(t);
	      }
	    } catch (RuntimeException e) {
	      throw e;
	    } catch (Throwable t) {
	      throw new ReflectionException("Could not get property '" + prop.getName() + "' from " + object.getClass() + ".  Cause: " + t.toString(), t);
	    }
	  }
　　从这个方法可以看出，实际的操作是下放到MetaClass中的，而Invoker才是真正的反射调用器。

## MetaClass
![](/images/mybatis_22.png)

　　MetaClass在我的理解里有些类似模板，MetaClass与MetaObject的关系类似于类与实例，MetaClass提供了类的各种属性方法的反射调用器，MetaObject提供了具体对象的引用，通过ObjectWrapper这个黑中介就联系到了一起。

	//获取name属性的getter调用器
	public Invoker getGetInvoker(String name) {
	    return reflector.getGetInvoker(name);
	  }

	//获取name属性的setter调用器
	public Invoker getSetInvoker(String name) {
	    return reflector.getSetInvoker(name);
	  }
	/**
	 * 获取prop中一级属性的类型
	 * @param prop
	 * @return
	 */
	private Class<?> getGetterType(PropertyTokenizer prop) {
	    Class<?> type = reflector.getGetterType(prop.getName());
	    if (prop.getIndex() != null && Collection.class.isAssignableFrom(type)) {
	      //prop指定属性为实现了Collection接口的集合类型
	      Type returnType = getGenericGetterType(prop.getName());
	      if (returnType instanceof ParameterizedType) {
	        Type[] actualTypeArguments = ((ParameterizedType) returnType).getActualTypeArguments();
	        if (actualTypeArguments != null && actualTypeArguments.length == 1) {
	          returnType = actualTypeArguments[0];
	          if (returnType instanceof Class) {
	            type = (Class<?>) returnType;
	          } else if (returnType instanceof ParameterizedType) {
	            type = (Class<?>) ((ParameterizedType) returnType).getRawType();
	          }
	        }
	      }
	    }
	    return type;
	  }
	/**
	 * 查询是否存在name属性的setter方法
	 * @param name
	 * @return
	 */
	public boolean hasSetter(String name) {
	    PropertyTokenizer prop = new PropertyTokenizer(name);
	    if (prop.hasNext()) {
	      //如果是多级属性，会依次向下递归，查询指定的最底层属性的setter方法
	      if (reflector.hasSetter(prop.getName())) {
	        MetaClass metaProp = metaClassForProperty(prop.getName());
	        return metaProp.hasSetter(prop.getChildren());
	      } else {
	        return false;
	      }
	    } else {
	      return reflector.hasSetter(prop.getName());
	    }
	  }

## Reflector
　　Reflector是整个反射模块的核心，是维护Invoker的容器，所有的反射操作都会放到Reflector来进行调度。下面看下Reflector的初始化流程：

	//反射容器对应的类
	private Class<?> type;
	//setter方法调用器缓存
	private Map<String, Invoker> setMethods = new HashMap<String, Invoker>();
	//getter方法调用器缓存
	private Map<String, Invoker> getMethods = new HashMap<String, Invoker>();
	//setter方法对应属性类型缓存
	private Map<String, Class<?>> setTypes = new HashMap<String, Class<?>>();
	//getter方法对应属性类型缓存
	private Map<String, Class<?>> getTypes = new HashMap<String, Class<?>>();
	//默认构造器
	private Constructor<?> defaultConstructor;
	//全大写属性名集合，为了兼容传入参数的大小写
	private Map<String, String> caseInsensitivePropertyMap = new HashMap<String, String>();

	public Reflector(Class<?> clazz) {
	    type = clazz;
	    //注册默认构造器
	    addDefaultConstructor(clazz);
	    //注册getter方法
	    addGetMethods(clazz);
	    //注册setter方法
	    addSetMethods(clazz);
	    //注册属性
	    addFields(clazz);
	    //可读属性名集合
	    readablePropertyNames = getMethods.keySet().toArray(new String[getMethods.keySet().size()]);
	    //可写属性名集合
	    writeablePropertyNames = setMethods.keySet().toArray(new String[setMethods.keySet().size()]);
	    //将属性名改为大写，为做大小写兼容做准备
	    for (String propName : readablePropertyNames) {
	      caseInsensitivePropertyMap.put(propName.toUpperCase(Locale.ENGLISH), propName);
	    }
	    for (String propName : writeablePropertyNames) {
	      caseInsensitivePropertyMap.put(propName.toUpperCase(Locale.ENGLISH), propName);
	    }
	  }
	/**
	 * 注册无参构造方法
	 */
	private void addDefaultConstructor(Class<?> clazz) {
		//获取clazz的所有构造器
	    Constructor<?>[] consts = clazz.getDeclaredConstructors();
	    for (Constructor<?> constructor : consts) {
	      if (constructor.getParameterTypes().length == 0) {
	    	//无参构造方法
	        if (canAccessPrivateMethods()) {
	          try {
	        	//关闭java安全检查，提高反射性能
	            constructor.setAccessible(true);
	          } catch (Exception e) {
	          }
	        }
	        //上边取消了无参构造方法的安全检查，此处将其赋值给defaultConstructor
	        if (constructor.isAccessible()) {
	          this.defaultConstructor = constructor;
	        }
	      }
	    }
	  }
	/**
	 * 注册getter方法
	 * @param cls
	 */
	 private void addGetMethods(Class<?> cls) {
	    Map<String, List<Method>> conflictingGetters = new HashMap<String, List<Method>>();
	    //获取cls中的所有方法，包括父类和接口中的方法
	    Method[] methods = getClassMethods(cls);
	    for (Method method : methods) {
	      //getter方法不应包含参数
	      if (method.getParameterTypes().length > 0) {
	        continue;
	      }
	      String name = method.getName();
	      //判断方法名是否符合get或者布尔值get命名
	      if ((name.startsWith("get") && name.length() > 3)
	          || (name.startsWith("is") && name.length() > 2)) {
	        name = PropertyNamer.methodToProperty(name);
	        //将get方法实例添加至conflictingGetters
	        addMethodConflict(conflictingGetters, name, method);
	      }
	    }
	    resolveGetterConflicts(conflictingGetters);
	  }
	/**
	 * 过滤掉同一名称的getter方法，返回值为同一继承树的类型以子类型为主
	 * @param conflictingGetters
	 */
	private void resolveGetterConflicts(Map<String, List<Method>> conflictingGetters) {
	    for (Entry<String, List<Method>> entry : conflictingGetters.entrySet()) {
	      Method winner = null;
	      String propName = entry.getKey();
	      for (Method candidate : entry.getValue()) {
	        if (winner == null) {
	          winner = candidate;
	          continue;
	        }
	        Class<?> winnerType = winner.getReturnType();
	        Class<?> candidateType = candidate.getReturnType();
	        //因为方法对应属性名相同（不论前缀是is或get），所以如果两者返回值类型相同时必为布尔类型（非布尔类型默认不使用is前缀）。
	        if (candidateType.equals(winnerType)) {
	          if (!boolean.class.equals(candidateType)) {
	            throw new ReflectionException(
	                "Illegal overloaded getter method with ambiguous type for property "
	                    + propName + " in class " + winner.getDeclaringClass()
	                    + ". This breaks the JavaBeans specification and can cause unpredictable results.");
	          } else if (candidate.getName().startsWith("is")) {
	            winner = candidate;
	          }
	        } else if (candidateType.isAssignableFrom(winnerType)) {
	          // OK getter type is descendant
	        } else if (winnerType.isAssignableFrom(candidateType)) {
	          //winnerType是candidateType的父类或接口或两者相同，此时以子类类型为主。
	          winner = candidate;
	        } else {
	          throw new ReflectionException(
	              "Illegal overloaded getter method with ambiguous type for property "
	                  + propName + " in class " + winner.getDeclaringClass()
	                  + ". This breaks the JavaBeans specification and can cause unpredictable results.");
	        }
	      }
	      //将method及返回类型添加至缓存区
	      addGetMethod(propName, winner);
	    }
	  }
	/**
	 * 注册setter方法
	 * @param cls
	 */
	private void addSetMethods(Class<?> cls) {
	    Map<String, List<Method>> conflictingSetters = new HashMap<String, List<Method>>();
	    Method[] methods = getClassMethods(cls);
	    for (Method method : methods) {
	      String name = method.getName();
	      if (name.startsWith("set") && name.length() > 3) {
	        if (method.getParameterTypes().length == 1) {
	          //以set开头且只有一个参数的判定为setter方法
	          name = PropertyNamer.methodToProperty(name);
	          //将set方法实例添加至conflictingSetters
	          addMethodConflict(conflictingSetters, name, method);
	        }
	      }
	    }
	    resolveSetterConflicts(conflictingSetters);
	  }
	/**
	 * 将method添加值conflictingMethods集合,可以支持同一属性多个方法
	 * @param conflictingMethods
	 * @param name
	 * @param method
	 */
	private void addMethodConflict(Map<String, List<Method>> conflictingMethods, String name, Method method) {
	    List<Method> list = conflictingMethods.get(name);
	    if (list == null) {
	      list = new ArrayList<Method>();
	      conflictingMethods.put(name, list);
	    }
	    list.add(method);
	  }
	/**
	 * 过滤掉同一名称的setter方法，返回值为同一继承树的类型以子类型为主
	 * @param conflictingGetters
	 */
	private void resolveSetterConflicts(Map<String, List<Method>> conflictingSetters) {
	    for (String propName : conflictingSetters.keySet()) {
	      List<Method> setters = conflictingSetters.get(propName);
	      //从缓存集合中获取propName属性对应的get类型
	      Class<?> getterType = getTypes.get(propName);
	      Method match = null;
	      ReflectionException exception = null;
	      for (Method setter : setters) {
	        Class<?> paramType = setter.getParameterTypes()[0];
	        if (paramType.equals(getterType)) {
	          // should be the best match
	          match = setter;
	          break;
	        }
	        if (exception == null) {
	          try {
	        	//取match与setter中返回类型为子类的方法
	            match = pickBetterSetter(match, setter, propName);
	          } catch (ReflectionException e) {
	            match = null;
	            exception = e;
	          }
	        }
	      }
	      if (match == null) {
	        throw exception;
	      } else {
	    	//将propName的set方法和set类型放入缓存
	        addSetMethod(propName, match);
	      }
	    }
	  }
	/**
	 * 注册属性
	 * @param clazz
	 */
	private void addFields(Class<?> clazz) {
	    Field[] fields = clazz.getDeclaredFields();
	    for (Field field : fields) {
	      //检查是否能够访问类中的字段和调用方法。注意，这不仅包括 public. 而且还包括 protected 和 private 字段和方法。
	      if (canAccessPrivateMethods()) {
	        try {
	          //关闭属性的安全检查
	          field.setAccessible(true);
	        } catch (Exception e) {
	        }
	      }
	      if (field.isAccessible()) {
	    	//检查缓存中是否包含field的get. set方法，如果没有则创建新的get. set方法放入缓存区
	        if (!setMethods.containsKey(field.getName())) {
	          int modifiers = field.getModifiers();
	          //同时使用static和final修饰的变量不能使用反射获取
	          if (!(Modifier.isFinal(modifiers) && Modifier.isStatic(modifiers))) {
	            addSetField(field);
	          }
	        }
	        if (!getMethods.containsKey(field.getName())) {
	          addGetField(field);
	        }
	      }
	    }
	    if (clazz.getSuperclass() != null) {
	      //注册父类的属性
	      addFields(clazz.getSuperclass());
	    }
	  }

## Invoker
作为具体的反射操作调用器，Invoker提供了三种默认实现。
* GetFieldInvoker：属性get方法调用器


	public Object invoke(Object target, Object[] args) throws IllegalAccessException, InvocationTargetException {
	    return field.get(target);
	  }
* SetFieldInvoker：属性set方法调用器


	public Object invoke(Object target, Object[] args) throws IllegalAccessException, InvocationTargetException {
	    field.set(target, args[0]);
	    return null;
	  }
* MethodInvoker：方法调用器


	public MethodInvoker(Method method) {
	    this.method = method;
	    //此处设置类型是为了方便get. set方法
	    if (method.getParameterTypes().length == 1) {
	      type = method.getParameterTypes()[0];
	    } else {
	      type = method.getReturnType();
	    }
	  }
	public Object invoke(Object target, Object[] args) throws IllegalAccessException, InvocationTargetException {
	    return method.invoke(target, args);
	  }

## PropertyTokenizer
　　PropertyTokenizer负责处理传入的属性名字符串，这里不仅支持单级，还支持多级和集合类型的属性名。

	public PropertyTokenizer(String fullname) {
	    int delim = fullname.indexOf('.');
	    if (delim > -1) {
	      //如果传入的是多级属性名，则name为一级属性，children为次级属性
	      name = fullname.substring(0, delim);
	      children = fullname.substring(delim + 1);
	    } else {
	      //如果传入的是单级属性，name为属性名
	      name = fullname;
	      children = null;
	    }
	    //此处先备份name为了处理属性类型为集合的情况
	    indexedName = name;
	    delim = name.indexOf('[');
	    if (delim > -1) {
	      //如果name代表的属性为集合类型，MyBatis默认用[]表示，如user.books[math]
	      //index表示集合中的指定属性名
	      index = name.substring(delim + 1, name.length() - 1);
	      //name表示集合名
	      name = name.substring(0, delim);
	    }
	  }

## PropertyNamer
　　PropertyNamer通过方法名获取对应属性名，仅支持get. set. is开头的方法

	/**
	 * 通过方法名获取对应的属性名，只支持get. set和is开头的方法
	 * @param name
	 * @return
	 */
	public static String methodToProperty(String name) {
	    if (name.startsWith("is")) {
	      name = name.substring(2);
	    } else if (name.startsWith("get") || name.startsWith("set")) {
	      name = name.substring(3);
	    } else {
	      throw new ReflectionException("Error parsing property name '" + name + "'.  Didn't start with 'is', 'get' or 'set'.");
	    }
	    if (name.length() == 1 || (name.length() > 1 && !Character.isUpperCase(name.charAt(1)))) {
	      name = name.substring(0, 1).toLowerCase(Locale.ENGLISH) + name.substring(1);
	    }
	    return name;
	  }

## 为什么使用MetaObject
　　MetaObject提供了方便而优雅的方式来操作对象的属性，不仅不需要手动处理各种reflect异常，还可以无视变量的作用域。

　　MetaObject还支持Map和Connection形式的对象，不过相对javaBean来讲简单很多，此处不在赘述。

