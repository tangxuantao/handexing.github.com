---
layout: post
title: spring,quartz整合（一）
categories: spring
description: spring,quartz整合（一）
keywords: spring,quartz，任务调度
---

有时工作中可能会用到定时任务功能，所以提前来配置玩一下。quartz是比较常用的定时器工具，并且在spring框架中也已经做了很好的集成,接下来开始配置使用quartz来做任务定时器功能。

##说明
> 还是在WISH项目基础中进行配置，想重头开始配置请移驾[WISH](https://github.com/handexing/wish),里面有详细的配置。

## pom.xml

> 首先先在pom.xml中添加依赖的jar。

```
<!-- quartz -->
<dependency>
    <groupId>org.quartz-scheduler</groupId>
	<artifactId>quartz</artifactId>
	<version>2.2.3</version>
</dependency>
```

## 配置spring.xml

```
<!--要调度的对象-->
	<bean id="taskJob" class="com.wish.service.test.QuartzJobService"/>
	<bean id="jobDetail" class="org.springframework.scheduling.quartz.MethodInvokingJobDetailFactoryBean">
		<property name="group" value="job_work"/>
		<property name="name" value="job_work_name"/>
		<!--false表示等上一个任务执行完后再开启新的任务-->
		<property name="concurrent" value="false"/>
		<property name="targetObject" ref="taskJob"/>
		<!-- 调用的方法名称 -->
		<property name="targetMethod" value="run"/>
	</bean>
	 
	<!-- 调度触发器 -->
	<bean id="myTrigger" class="org.springframework.scheduling.quartz.CronTriggerFactoryBean">
		<property name="name" value="work_default_name"/>
		<property name="group" value="work_default"/>
		<property name="jobDetail" ref="jobDetail"/>
		<property name="cronExpression" value="0/5 * * * * ?"/>
	</bean>
	 
	<!-- 调度工厂 -->
	<bean id="scheduler" class="org.springframework.scheduling.quartz.SchedulerFactoryBean">
		<property name="triggers">
			<list>
				<ref bean="myTrigger"/>
			</list>
		</property>
	</bean>
```

## 调用的对象

```
public class QuartzJobService {
	Logger logger = LoggerFactory.getLogger(this.getClass());
	public void run() {
		logger.info("==========run===========");
	}
}
```

## 测试

```
public static void main(String[] args) {
		ApplicationContext context = new ClassPathXmlApplicationContext("spring.xml");
		context.getBean("scheduler");
	}
```

## 结果

运行结果：

![运行结果](/images/posts/testquartz.png)


## 结语
以上代码都在我的github上，其中有问题或者不对的地方欢迎交流。
项目地址：[WISH](https://github.com/handexing/wish)




