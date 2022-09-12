---
title: Spring--Bean的生命周期
date: 2022-09-12 22:39:13
tags:
- Java
- Spring
categories:
- Java
- Spring
comments: true
mathjax: true
---

​	Spring对于Java开发者来说应该是再熟悉不过的一样东西了，这两天我把Spring的源码从官网下载下来并且进行编译，方便后面自己写一些测试用例(尽管被网速gank了很久)，并且结合了网上的一些文章去理解Spring IOC方面的知识和运作过程，我也跟着一些文章敲了测试用例，通过控制台的输出去记忆bean的生命周期过程，毕竟之前一直靠背去记忆实在是太容易忘记了。当然这篇文章也不会讲的很详细，这篇文章是在我看了一天的文章和敲完用例趁着记忆还深刻来通过记录博客来加深记忆的，下面就开始吧。

<!--more-->

## Bean生命周期	

​	这里首先贴出bean的一张生命周期图(图源来自JavaGuide)

![image-20220912230549405](https://typora-oss-pic.oss-cn-guangzhou.aliyuncs.com/typoraImg/image-20220912230549405.png)

​	其大致的流程其实也可以理解成下面这样，我相信等后面自己敲用例后，读者会对这一过程更加的记忆深刻了：

​	在开始初始化容器后，容器就会去`配置文件`中读取关于`bean的定义`，如果在配置bean时有赋予属性值，是可以读到这个值的。读取到定义后就会生成关于这个bean的实例。有实例之后就会对这个实例的属性进行设置set()。设置完属性后，如果检查到这个bean有实现比如`BeanNameAware`、`BeanFactoryAware`等这些诸如`*Aware`的接口，那么就会调用它们对应的`setBeanName(String beanName)`、`setBeanFactory(BeanFactory factory)`等`setXXX()`方法。在做完前面这些bean的实例化和属性设置的操作后就要开始进行初始化，`BeanPostProcessor`这一处理器有`postProcessBeforeInitialization()`和`postProcessAfterInitialization()`两个方法，前者是在bean初始化前执行，后者就是在bean初始化完成后执行，也就是上图的前置和后置处理，可以自行去定义里面的行为，不过要注意因为这两个方法的返回值是作为一个新的bean实例，所以通常返回值都是传入进来的bean对象而不是null。在这初始化的过程中会检查这个bean有没有实现`InitializingBean`接口，如果有的话则会去调用该接口的`afterPropertiesSet()`方法，然后再检查配置文件的bean标签中有没有`init-method`这一属性，有的话就会去调用配置文件里`init-method`指定的初始化方法，初始化完成后就会执行上面所说得`postProcessAfterInitialization`方法了。这样一个简单的容器初始化就这样完成了，然后就可以在容器中获取它。当我们要销毁容器时，会检查这个bean有没有实现`DisposableBean`这个接口，如果有实现则会调用这个方法的`destroy()`方法，然后再检查配置文件的bean标签有没有`destroy-method`这一属性，有的话就会去调用配置文件里`destroy-method`指定的销毁后的回调方法。至此一个简单的bean声明周期就这么结束了。

## 测试实现

### 	实体类

```java
package com.demo.pojo;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.*;

public class Person implements BeanFactoryAware, BeanNameAware, InitializingBean, DisposableBean {

	private String name;
	private int age;

	private BeanFactory beanFactory;
	private String beanName;

	public Person(){
		System.out.println("[构造器]调用Person的构造器实例化一个Person对象");
	}

	public String getName() {
		return name;
	}

	public void setName(String name) {
		System.out.println("注入属性name,值为"+name);
		this.name = name;
	}

	public int getAge() {
		return age;
	}

	public void setAge(int age) {
		System.out.println("注入属性age,值为"+age);
		this.age = age;
	}

	@Override
	public String toString() {
		return "Person{" +
				"name='" + name + '\'' +
				", age=" + age +
				'}';
	}

	//BeanFactoryAware接口的方法
	@Override
	public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
		System.out.println("[BeanFactoryAware接口]调用BeanFactoryAware.setBeanFactory()方法");
		this.beanFactory = beanFactory;
	}

	//BeanNameAware接口的方法
	@Override
	public void setBeanName(String name) {
		System.out.println("[BeanNameAware接口]调用BeanNameAware.setBeanName()方法");
		this.beanName = name;
	}

	//InitializingBean接口方法
	@Override
	public void afterPropertiesSet() throws Exception {
		System.out.println("属性设置完后[InitializingBean接口]调用InitializingBean.afterPropertiesSet()方法");
	}

	//DisposableBean接口方法
	@Override
	public void destroy() throws Exception {
		System.out.println("要销毁bean时[DisposableBean接口]调用DisposableBean.destroy()");
	}

	// 通过<bean>的init-method属性指定的初始化方法
	public void myInit(){
		System.out.println("[init-method]调用<bean>的init-method属性指定的初始化方法");
	}

	//通过<bean>的destroy-method属性指定的销毁方法
	public void myDestroy(){
		System.out.println("[destroy-method]调用<bean>的destroy-method属性指定的初始化方法");
	}
}

```

### Processor

#### BeanFactoryPostProcessor

```java
package com.demo.processor;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanDefinition;
import org.springframework.beans.factory.config.BeanFactoryPostProcessor;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;

public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {

	public MyBeanFactoryPostProcessor() {
		super();
		System.out.println("BeanFactoryPostProcessor类构造方法");
		System.out.println("---------");
	}

	/**
	 * 在ApplicationContext内部的BeanFactory加载完bean的定义后，但是在对应的bean实例化之前进行回调。
	 * 可以通过实现该接口来对实例化之前的bean定义进行修改。
	 */
	@Override
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
		System.out.println("[BeanFactoryPostProcessor接口]postProcessBeanFactory方法在加载完bean的定义后，在bean实例化前进行修改");
		//获取bean的定义
		BeanDefinition bd = beanFactory.getBeanDefinition("person");
		StringBuilder sb = new StringBuilder();
		sb
				.append("beanName: person")
				.append(",beanClassName: ").append(bd.getBeanClassName())
				.append(",factoryBeanName: ").append(bd.getFactoryBeanName())
				.append(",factoryMethodName: ").append(bd.getFactoryMethodName())
				.append(",scope: ").append(bd.getScope())
				.append(",parent: ").append(bd.getParentName())
				.append(",nameValues: ").append(bd.getPropertyValues().get("name"))
				.append(",ageValues: ").append(bd.getPropertyValues().get("age"));
		System.out.println("该定义bean的信息为：");
		System.out.println(sb.toString());
		System.out.println("---------");
		bd.getPropertyValues().addPropertyValue("name","jason");
		bd.getPropertyValues().addPropertyValue("age","21");
	}
}
```

#### BeanPostProcessor

```java
package com.demo.processor;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanPostProcessor;

//后置处理器，在 bean 实例化完成、属性注入完成之后，会执行回调方法
public class MyBeanPostProcessor implements BeanPostProcessor {
	public MyBeanPostProcessor(){
		super();
		System.out.println("BeanPostProcessor类构造器");
		System.out.println("----------");
	}

	//实例化、依赖注入完毕，在调用显示的初始化之前完成一些定制的初始化任务
	//返回值会作为新的bean实例，所以不要随便返回null
	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		System.out.println("[BeanPostProcessor接口]postProcessBeforeInitialization执行初始化前任务");
		return bean;
	}

	//实例化、依赖注入、初始化完毕时执行
	//返回值会作为新的bean实例，所以不要随便返回null
	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		System.out.println("[BeanPostProcessor接口]postProcessAfterInitialization执行初始化完成后任务");
		return bean;
	}
}
```

#### InstantiationAwareBeanPostProcessor

```java
package com.demo.processor;

import org.springframework.beans.BeansException;
import org.springframework.beans.PropertyValues;
import org.springframework.beans.factory.config.InstantiationAwareBeanPostProcessor;


//InstantiationAwareBeanPostProcessor是感知Bean实例化的处理器
public class MyInstantiationAwareBeanPostProcessor implements InstantiationAwareBeanPostProcessor {

	public MyInstantiationAwareBeanPostProcessor() {
		super();
		System.out.println("InstantiationAwareBeanPostProcessor类构造器");
		System.out.println("-----------");
	}

	//bean实例化之前调用
	@Override
	public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
		System.out.println("[InstantiationAwareBeanPostProcessor接口]postProcessBeforeInstantiation在bean实例化前执行，此时没有bean对象");
		return null;
	}

	//bean实例化之后调用
	//其返回值决定要不要调用postProcessPropertyValues方法的其中一个因素
	//返回true则会调用postProcessProperties进行属性值修改
	@Override
	public boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
		System.out.println("[InstantiationAwareBeanPostProcessor接口]postProcessAfterInstantiation在bean实例化后执行，此时有bean对象");
		return true;
	}

	@Override
	public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) throws BeansException {
		System.out.println("假装postProcessProperties执行修改");
		System.out.println("---------");
		return pvs;
	}
}
```

### 配置文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:p="http://www.springframework.org/schema/p"
	   xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

	<bean id="beanPostProcessor" class="com.demo.processor.MyBeanPostProcessor"></bean>

	<bean id="instantiationAwareBeanPostProcessor" class="com.demo.processor.MyInstantiationAwareBeanPostProcessor"></bean>

	<bean id="beanFactoryPostProcessor" class="com.demo.processor.MyBeanFactoryPostProcessor"></bean>

	<bean id="person" class="com.demo.pojo.Person" init-method="myInit"
		destroy-method="myDestroy" scope="singleton"
		  p:name = "john" p:age = "22"
	/>
</beans>

```

### 测试

```java
package com.demo;

import com.demo.pojo.Person;
import com.demo.service.TestHelloService;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class Main {
	public static void main(String[] args) {
		System.out.println("----开始初始化容器----");
		ApplicationContext ctx = new ClassPathXmlApplicationContext("classpath:spring-config.xml");
		System.out.println("----初始化容器成功----");
		System.out.println("获取bean");
		Person person = ctx.getBean("person", Person.class);
		System.out.println(person);
		System.out.println("----开始关闭容器----");
		((ClassPathXmlApplicationContext)ctx).registerShutdownHook();
	}
}
```

### 结果

```shell
----开始初始化容器----
BeanFactoryPostProcessor类构造方法
---------
[BeanFactoryPostProcessor接口]postProcessBeanFactory方法在加载完bean的定义后，在bean实例化前进行修改
该定义bean的信息为：
beanName: person,beanClassName: com.demo.pojo.Person,factoryBeanName: null,factoryMethodName: null,scope: singleton,parent: null,nameValues: john,ageValues: 22
---------
BeanPostProcessor类构造器
----------
InstantiationAwareBeanPostProcessor类构造器
-----------
[InstantiationAwareBeanPostProcessor接口]postProcessBeforeInstantiation在bean实例化前执行，此时没有bean对象
[构造器]调用Person的构造器实例化一个Person对象
[InstantiationAwareBeanPostProcessor接口]postProcessAfterInstantiation在bean实例化后执行，此时有bean对象
假装postProcessProperties执行修改
---------
注入属性age,值为21
注入属性name,值为jason
[BeanNameAware接口]调用BeanNameAware.setBeanName()方法
[BeanFactoryAware接口]调用BeanFactoryAware.setBeanFactory()方法
[BeanPostProcessor接口]postProcessBeforeInitialization执行初始化前任务
属性设置完后[InitializingBean接口]调用InitializingBean.afterPropertiesSet()方法
[init-method]调用<bean>的init-method属性指定的初始化方法
[BeanPostProcessor接口]postProcessAfterInitialization执行初始化完成后任务
----初始化容器成功----
获取bean
Person{name='jason', age=21}
----开始关闭容器----
要销毁bean时[DisposableBean接口]调用DisposableBean.destroy()
[destroy-method]调用<bean>的destroy-method属性指定的初始化方法
```

​	通过打印语句就能很直观地看到这个过程：容器初始化后先构造**BeanFactoryPostProcessor**，然后调用其postProcessBeanFactory方法，获取bean的定义。接着构造**BeanPostProcessor**和**InstantiationAwareBeanPostProcessor**。到这里都还是没有bean对象的，在实例化bean之前就会调用InstantiationAwareBeanPostProcessor接口的postProcess**BeforeInstantiation**方法，接着实例化bean，实例化之后调用InstantiationAwareBeanPostProcessor接口的postProcess**AfterInstantiation**方法，这个方法返回值如果是true的话就会触发postProcessProperties的调用。bean实例化好后，就对其属性进行注入，对于实现的Aware接口调用对应的set方法。这样bean的实例化和属性设置就完成了，接下来就是要初始化，在初始化前，BeanPostProcessor会执行postProcess**BeforeInitialization**这一方法，然后就是初始化的操作，检查到实现了**InitializingBean**接口和有配置**init-method**，执行InitializingBean.**afterPropertiesSet**()方法和\<bean>的init-method属性指定的初始化方法，初始化完成后BeanPostProcessor就会执行postProcess**AfterInitialization**这一方法，容器的初始化也就完成了，接着的就是模拟一下获取bean的操作。关闭容器时，检查到实现了**DisposableBean**接口和配置了**destroy-method**，执行DisposableBean.**destroy**()方法和\<bean>的destroy-method属性指定的销毁后的回调方法。

​	这便是一个简单的bean生命周期过程了。

## 参考

> - https://juejin.cn/post/6844903694039793672
> - https://javaguide.cn/system-design/framework/spring/spring-knowledge-and-questions-summary.html#bean-%E7%9A%84%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F
> - https://www.cnblogs.com/zrtqsk/p/3735273.html
> - https://cloud.tencent.com/developer/article/1409315
> - https://developer.aliyun.com/article/459773
> - https://cloud.tencent.com/developer/article/1409273

