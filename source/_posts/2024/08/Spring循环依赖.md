---
title: Spring 如何解决循环依赖问题？
date: 2024-08-20 10:20:46
tags: 
	- 面试
	- 源码
	- Spring Boot
---

# Spring 如何解决循环依赖问题？

Spring Boot 版本号：3.3.2
Spring 版本号：6.1.11
JDK 版本号：17

```java
@Component
public class A {
    @Autowired
    private B b;
}
// ...
@Component
public class B {
    @Autowired
    private A a;
}
```

首先，我们应该知道，<b><u>Spring 在创建 Bean 的时候默认是按照自然排序来创建的</u></b>，所以第一步 Spring 会去创建 A。

与此同时，Spring 在创建 Bean 的过程分为三步：

1. 实例化。对应方法：`AbstractAutowireCapableBeanFactory` 中的 `createBeanInstance` 方法。

   简单来说就是 `new` 了一个新对象。

2. 属性注入。对应方法：`AbstractAutowireCapableBeanFactory` 中的 `populateBean` 方法。

   为实例化中 `new` 出来的对象填充属性。

3. 初始化。对应方法：`AbstractAutowireCapableBeanFactory` 中的 `initializeBean` 方法。

   执行 `aware` 接口中的方法，初始化方法，完成 `AOP` 代理。

![创建 A 的过程](https://raw.githubusercontent.com/zhangzhaolin/GraphBed/master/2024/08/202408201558614.jpg)

如图所示，创建 A 的过程实际上就是调用 `getBean` 方法，这个方法有两层含义：

1. 创建一个新的 Bean
2. 从缓存中获取到已经被创建的对象

## 调用 `getSingleton(String)` 方法

```java
@Override
@Nullable
public Object getSingleton(String beanName) {
	return getSingleton(beanName, true);
}
```

`getSingleton(String, boolean)` 这个方法实际上就是到缓存中尝试获取 bean，整个缓存分为三级。

1. `singletonObjects` —— 一级缓存，存储的是所有创建好的单例 Bean。
2. `earlySingletonObjects` —— 完成实例化，但还未进行属性注入及初始化对象。
3. `singletonFactories` —— 提前暴露的一个单例工厂，二级缓存中存储的就是从这个工厂中获取到的对象。

由于 A 是第一次创建，所以不管哪个缓存中必然都是没有的，因此会进入 `getSingleton(beanName, singletonFactory)` 方法。

```java
protected <T> T doGetBean(String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly){
	...
	Object sharedInstance = getSingleton(beanName);
	if (sharedInstance != null && args == null) {
		...
	}
	else {
		...
		try{
			RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
			...
			if (mbd.isSingleton()) {
					// 这里的 Lambda 表达式就是 ObjectFactory 类的实现
					sharedInstance = getSingleton(beanName, () -> {
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							destroySingleton(beanName);
							throw ex;
						}
					});
					beanInstance = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}
		}
		...
	}
}
```

## 调用 `getSingleton(string, ObjectFactory)`

```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
		// ... 省略
	synchronized (this.singletonObjects) {
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null) {
			// 省略日志 ...
			// 在单例对象创建之前先做一个标记
			// 将 beanName 字符串放在 Set 集合中，表示这个单例 bean 正在被创建
			// 如果同一个单例 Bean 多次被创建，则抛出异常
			beforeSingletonCreation(beanName);
			boolean newSingleton = false;
			boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
			if (recordSuppressedExceptions) {
				this.suppressedExceptions = new LinkedHashSet<>();
			}
			try {
				// 传入的 Lambda 表达式会在这里执行
				// 也就是说，createBean(beanName, mbd, args) 会被执行
				singletonObject = singletonFactory.getObject();
				newSingleton = true;
			}
			// .. .
			// 省略 catch 异常
			// ...
			finally {
				if (recordSuppressedExceptions) {
					this.suppressedExceptions = null;
				}
				// 创建完成后从 Set 集合中移除
				afterSingletonCreation(beanName);
			}
			if (newSingleton) {
				// 添加到一级缓存 singletonObjects 之中	
				addSingleton(beanName, singletonObject);
			}
		}
		return singletonObject;
	}
}

protected void beforeSingletonCreation(String beanName) {
	if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.add(beanName)) {
		throw new BeanCurrentlyInCreationException(beanName);
	}
}

protected void afterSingletonCreation(String beanName) {
	if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.remove(beanName)) {
		throw new IllegalStateException("Singleton '" + beanName + "' isn't currently in creation");
	}
}

protected void addSingleton(String beanName, Object singletonObject) {
	synchronized (this.singletonObjects) {
		this.singletonObjects.put(beanName, singletonObject);
		this.singletonFactories.remove(beanName);
		this.earlySingletonObjects.remove(beanName);
		this.registeredSingletons.add(beanName);
	}
}
```

在上面的代码中，通过 `createBean` 方法返回的 Bean 最终被放到了一级缓存，也就是单例池中。

那么我们可以说：<b><u>一级缓存中存储的是已经完全创建好的单例 Bean</u></b>

## 调用 `createBean` 方法

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {
	// ...
	try {
		Object beanInstance = doCreateBean(beanName, mbdToUse, args);
		// ...
		return beanInstance;
	}
	// ...
}
```

## 调用 `doCreateBean` 方法

```java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
	throws BeanCreationException {
	BeanWrapper instanceWrapper = null;
	// ...
	// 创建 Bean 实例
	if (instanceWrapper == null) {
		instanceWrapper = createBeanInstance(beanName, mbd, args);
	}
	// ...
	boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
		isSingletonCurrentlyInCreation(beanName));
	if (earlySingletonExposure) {
		// ...
		// 添加到三级缓存中
		addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
	}
	// ...
	Object exposedObject = bean;
	try {
		// bean 属性填充
		populateBean(beanName, mbd, instanceWrapper);
		exposedObject = initializeBean(beanName, exposedObject, mbd);
	}
	// ...
}
```

## 调用 `addSingletonFactory` 方法

在完成 Bean 的实例化之后，属性注入之前，Spring 将 Bean 包装成一个工厂，添加进了三级缓存之中，对应源码如下：

```java
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
	synchronized (this.singletonObjects) {
		if (!this.singletonObjects.containsKey(beanName)) {
			// 添加到三级缓存之中
			this.singletonFactories.put(beanName, singletonFactory);
			this.earlySingletonObjects.remove(beanName);
			this.registeredSingletons.add(beanName);
		}
	}
}
```

这里只是添加了一个工厂 `ObjectFactory` ，通过这个工厂的 `getObject` 函数可以得到一个对象，而这个对象实际上是 `getEarlyBeanReference` 去创建的。

那么，什么时候会调用 `ObjectFactory` 的 `getObject` 函数呢？这个时候就需要看创建 B 的流程了。

## 参考

- [面试必杀技，讲一讲 Spring 中的循环依赖](https://www.cnblogs.com/daimzh/p/13256413.html)
