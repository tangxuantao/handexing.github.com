---
layout: post
title: spring,quartz任务动态调用（二）
categories: spring
description: spring,quartz任务动态调用（二）
keywords: spring,quartz，任务调度，动态调动
---

前面已经对spring、quartz进行了整合，如果业务需求简单可以满足，但是很多时候常会遇到需要动态的添加修改任务，而spring提供的的只能通过配置文件才能控制任务，这样失去了灵活性。接我们就开始撸码。

## 说明
> 还是在WISH项目基础中进行配置，想重头开始配置请移驾[WISH](https://github.com/handexing/wish),里面有详细的配置。
也有单独抽离出来的单独项目[quartz_demo](https://github.com/handexing/frameworkAggregate)

## pom.xml

> 首先先在pom.xml中添加依赖的jar。

```
<!-- quartz -->
<dependency>
    <groupId>org.quartz-scheduler</groupId>
    <artifactId>quartz</artifactId>
    <version>${quartz.version}</version>
</dependency>
<dependency>
    <groupId>org.quartz-scheduler</groupId>
    <artifactId>quartz-jobs</artifactId>
    <version>${quartz.version}</version>
</dependency> 
```

## 配置spring.xml

```
<bean id="schedulerFactoryBean" class="org.springframework.scheduling.quartz.SchedulerFactoryBean" />
<bean id="springUtils" class="com.wish.util.SpringUtils" />
```

## 代码

### 数据库

```
CREATE TABLE `schedule_job` (
  `JOB_ID` bigint(20) NOT NULL AUTO_INCREMENT,
  `JOB_NAME` varchar(255) DEFAULT NULL COMMENT '任务名称',
  `JOB_GROUP` varchar(255) DEFAULT NULL COMMENT '任务分组',
  `JOB_STATUS` varchar(255) DEFAULT NULL COMMENT '任务状态 1开启：0停止',
  `CRON_EXPRESSION` varchar(255) DEFAULT NULL COMMENT 'cron表达式',
  `DESCRIPTION` varchar(255) DEFAULT NULL COMMENT '描述',
  `BEAN_CLASS` varchar(255) DEFAULT NULL COMMENT '调用类（包名加类名）',
  `SPRING_ID` varchar(255) DEFAULT NULL COMMENT 'spring bean 名称',
  `METHOD_NAME` varchar(255) DEFAULT NULL COMMENT '调用方法名称',
  `CREATE_TIME` datetime DEFAULT NULL,
  `UPDATE_TIME` datetime DEFAULT NULL,
  PRIMARY KEY (`JOB_ID`)
) ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT CHARSET=utf8;
```

### 实体类

```
package com.wish.entity;

import com.fasterxml.jackson.annotation.JsonFormat;

import org.springframework.format.annotation.DateTimeFormat;

import java.util.Date;

import javax.persistence.Column;
import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;
import javax.persistence.Table;

/**
 * @author handx 908716835@qq.com
 * @date 2017年5月7日 下午5:29:45
 */

@Entity
@Table(name = "SCHEDULE_JOB")
public class ScheduleJob {

	public static final String STATUS_RUNNING = "1";
	public static final String STOP_RUNNING = "0";

	@Id
	@GeneratedValue(strategy = GenerationType.AUTO)
	@Column(name = "JOB_ID")
	private Long jobId;
	@Column(name = "JOB_NAME")
	private String jobName;// 任务名称
	@Column(name = "JOB_GROUP")
	private String jobGroup;// 任务分组
	@Column(name = "JOB_STATUS")
	private String jobStatus;// 任务状态 是否启动任务
	@Column(name = "CRON_EXPRESSION")
	private String cronExpression;// cron表达式
	@Column(name = "DESCRIPTION")
	private String description;// 描述
	@Column(name = "BEAN_CLASS")
	private String beanClass;// 任务执行时调用哪个类的方法 包名+类名
	@Column(name = "SPRING_ID")
	private String springId;// spring bean
	@Column(name = "METHOD_NAME")
	private String methodName;// 任务调用的方法名
	@DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss")
	@JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss", timezone = "GMT+8")
	@Column(name = "CREATE_TIME")
	private Date createTime;
	@DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss")
	@JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss", timezone = "GMT+8")
	@Column(name = "UPDATE_TIME")
	private Date updateTime;
	
    省略get，set方法.....
    
```

### 接口

```
package com.wish.dao;

import com.wish.entity.ScheduleJob;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

/**
 * @author handx 908716835@qq.com
 * @date 2017年5月8日 上午10:15:44
 */

@Repository
public interface ScheduleJobDao extends JpaRepository<ScheduleJob, Long> {

}

```

### SERVICE

```
package com.wish.service;

import com.wish.dao.ScheduleJobDao;
import com.wish.entity.ScheduleJob;
import com.wish.quartz.QuartzJobFactory;

import org.quartz.CronScheduleBuilder;
import org.quartz.CronTrigger;
import org.quartz.JobBuilder;
import org.quartz.JobDetail;
import org.quartz.JobKey;
import org.quartz.Scheduler;
import org.quartz.SchedulerException;
import org.quartz.TriggerBuilder;
import org.quartz.TriggerKey;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.quartz.SchedulerFactoryBean;
import org.springframework.stereotype.Service;

import java.util.Date;

import javax.transaction.Transactional;

@Service
public class ScheduleJobService {

	@Autowired
	ScheduleJobDao scheduleJobDao;
	@Autowired
	SchedulerFactoryBean schedulerFactoryBean;

	Logger logger = LoggerFactory.getLogger(this.getClass());

	/**
	 * 添加任务
	 * 
	 * @param scheduleJob
	 * @throws SchedulerException
	 */
	public void addJob(ScheduleJob job) throws SchedulerException {
		if (job == null || !ScheduleJob.STATUS_RUNNING.equals(job.getJobStatus())) {
			return;
		}

		Scheduler scheduler = schedulerFactoryBean.getScheduler();
		//在这里把它设计成一个Job对应一个trigger，两者的分组及名称相同，方便管理，条理也比较清晰，在创建任务时如果不存在新建一个，如果已经存在则更新任务
		TriggerKey triggerKey = TriggerKey.triggerKey(job.getJobName(), job.getJobGroup());
		CronTrigger trigger = (CronTrigger) scheduler.getTrigger(triggerKey);

		// 不存在，创建一个
		if (null == trigger) {
			JobDetail jobDetail = JobBuilder.newJob(QuartzJobFactory.class)
					.withIdentity(job.getJobName(), job.getJobGroup()).build();
			jobDetail.getJobDataMap().put("scheduleJob", job);
			CronScheduleBuilder scheduleBuilder = CronScheduleBuilder.cronSchedule(job.getCronExpression());
			trigger = TriggerBuilder.newTrigger().withIdentity(job.getJobName(), job.getJobGroup()).withSchedule(scheduleBuilder).build();
			scheduler.scheduleJob(jobDetail, trigger);
		} else {
			// Trigger已存在，那么更新相应的定时设置
			CronScheduleBuilder scheduleBuilder = CronScheduleBuilder.cronSchedule(job.getCronExpression());
			// 按新的cronExpression表达式重新构建trigger
			trigger = trigger.getTriggerBuilder().withIdentity(triggerKey).withSchedule(scheduleBuilder).build();
			// 按新的trigger重新设置job执行
			scheduler.rescheduleJob(triggerKey, trigger);
		}
	}

	@Transactional
	public void addTask(ScheduleJob job) {
		job.setCreateTime(new Date());
		scheduleJobDao.save(job);
	}

	/**
	 * 更改任务状态
	 * 
	 * @throws SchedulerException
	 */
	@Transactional
	public void changeStatus(Long jobId, String cmd) throws SchedulerException {

		ScheduleJob job = scheduleJobDao.findOne(jobId);
		if (job == null) {
			return;
		}
		if ("stop".equals(cmd)) {
			deleteJob(job);
			job.setJobStatus(ScheduleJob.STOP_RUNNING);
		} else if ("start".equals(cmd)) {
			job.setJobStatus(ScheduleJob.STATUS_RUNNING);
			addJob(job);
		}
		job.setUpdateTime(new Date());
		scheduleJobDao.save(job);
	}

	/**
	 * 删除一个job
	 * 
	 * @param scheduleJob
	 * @throws SchedulerException
	 */
	public void deleteJob(ScheduleJob scheduleJob) throws SchedulerException {
		Scheduler scheduler = schedulerFactoryBean.getScheduler();
		JobKey jobKey = JobKey.jobKey(scheduleJob.getJobName(), scheduleJob.getJobGroup());
		scheduler.deleteJob(jobKey);
	}

	/**
	 * 更改任务 cron表达式
	 * 
	 * @throws SchedulerException
	 */
	public void updateCron(Long jobId, String cron) throws SchedulerException {
		ScheduleJob job = scheduleJobDao.findOne(jobId);
		if (job == null) {
			return;
		}
		job.setCronExpression(cron);
		if (ScheduleJob.STATUS_RUNNING.equals(job.getJobStatus())) {
			updateJobCron(job);
		}
		job.setUpdateTime(new Date());
		scheduleJobDao.save(job);
	}

	/**
	 * 更新job时间表达式
	 * 
	 * @param scheduleJob
	 * @throws SchedulerException
	 */
	public void updateJobCron(ScheduleJob scheduleJob) throws SchedulerException {
		Scheduler scheduler = schedulerFactoryBean.getScheduler();
		TriggerKey triggerKey = TriggerKey.triggerKey(scheduleJob.getJobName(), scheduleJob.getJobGroup());
		CronTrigger trigger = (CronTrigger) scheduler.getTrigger(triggerKey);
		CronScheduleBuilder scheduleBuilder = CronScheduleBuilder.cronSchedule(scheduleJob.getCronExpression());
		trigger = trigger.getTriggerBuilder().withIdentity(triggerKey).withSchedule(scheduleBuilder).build();
		scheduler.rescheduleJob(triggerKey, trigger);
	}

}
```

### job实例

```
package com.wish.quartz;

import com.wish.entity.ScheduleJob;
import com.wish.util.TaskUtils;

import org.apache.log4j.Logger;
import org.quartz.Job;
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;

/**
 * @Description: 计划任务执行处
 * @author handx 908716835@qq.com
 * @date 2017年5月7日 下午10:04:55
 *
 */
public class QuartzJobFactory implements Job {

	public final Logger log = Logger.getLogger(this.getClass());

	public void execute(JobExecutionContext context) throws JobExecutionException {
		ScheduleJob scheduleJob = (ScheduleJob) context.getMergedJobDataMap().get("scheduleJob");
		TaskUtils.invokMethod(scheduleJob);
	}
}
```

### TaskUtils类，java反射

```
package com.wish.util;

import com.wish.entity.ScheduleJob;

import org.apache.commons.lang.StringUtils;
import org.apache.log4j.Logger;

import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;

public class TaskUtils {

	public final static Logger log = Logger.getLogger(TaskUtils.class);
	
	/**
	 * 通过反射调用scheduleJob中定义的方法
	 * 
	 * @param scheduleJob
	 */
	@SuppressWarnings({ "rawtypes", "unchecked" })
	public static void invokMethod(ScheduleJob scheduleJob) {
		Object object = null;
		Class clazz = null;
		//判断是springbean还是Java类
		if (StringUtils.isNotBlank(scheduleJob.getSpringId())) {
			object = SpringUtils.getBean(scheduleJob.getSpringId());
		} else if (StringUtils.isNotBlank(scheduleJob.getBeanClass())) {
			try {
				clazz = Class.forName(scheduleJob.getBeanClass());
				object = clazz.newInstance();
			} catch (Exception e) {
				e.printStackTrace();
			}

		}
		if (object == null) {
			log.error("未启动成功，请检查是否配置正确！任务名称 = [" + scheduleJob.getJobName() + "]");
			return;
		}
		clazz = object.getClass();
		Method method = null;
		try {
			method = clazz.getDeclaredMethod(scheduleJob.getMethodName());
		} catch (NoSuchMethodException e) {
			log.error("未启动成功，方法名设置错误！任务名称 = [" + scheduleJob.getJobName() + "]");
		} catch (SecurityException e) {
			e.printStackTrace();
		}
		if (method != null) {
			try {
				method.invoke(object);
			} catch (IllegalAccessException e) {
				e.printStackTrace();
			} catch (IllegalArgumentException e) {
				e.printStackTrace();
			} catch (InvocationTargetException e) {
				e.printStackTrace();
			}
		}
		System.out.println("启动成功，任务名称 = [" + scheduleJob.getJobName() + "]");
	}

}
```

### controller

```
package com.wish.controller;

import com.wish.dao.ScheduleJobDao;
import com.wish.entity.ScheduleJob;
import com.wish.model.ExecuteResult;
import com.wish.model.RetJson;
import com.wish.service.ScheduleJobService;
import com.wish.util.PageUtil;
import com.wish.util.SpringUtils;

import org.apache.commons.lang.StringUtils;
import org.quartz.CronScheduleBuilder;
import org.quartz.SchedulerException;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.data.domain.Sort;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.servlet.ModelAndView;

import java.lang.reflect.Method;
import java.util.List;

import javax.servlet.http.HttpServletRequest;

/**
 * @Description: 任务调度
 * @author handx 908716835@qq.com
 * @date 2017年5月6日 下午1:23:32
 */

@RestController
@RequestMapping("job")
public class SchedulerJobController {

	Logger logger = LoggerFactory.getLogger(this.getClass());

	@Autowired
	ScheduleJobDao scheduleJobDao;
	@Autowired
	ScheduleJobService scheduleJobService;

	@SuppressWarnings({ "unchecked", "rawtypes", "unused" })
	@RequestMapping("add")
	public ExecuteResult<Boolean> add(@RequestBody ScheduleJob scheduleJob) {
		ExecuteResult<Boolean> result = new ExecuteResult<Boolean>();
		result.setSuccess(false);
		try {
			CronScheduleBuilder scheduleBuilder = CronScheduleBuilder.cronSchedule(scheduleJob.getCronExpression());
		} catch (Exception e) {
			result.setErrorMsg("cron表达式有误，不能被解析！");
			return result;
		}

		Object obj = null;
		try {
			if (StringUtils.isNotBlank(scheduleJob.getSpringId())) {
				obj = SpringUtils.getBean(scheduleJob.getSpringId());
			} else {
				Class clazz = Class.forName(scheduleJob.getBeanClass());
				obj = clazz.newInstance();
			}
		} catch (Exception e) {
			e.printStackTrace();
		}

		if (obj == null) {
			result.setErrorMsg("未找到目标类！");
			return result;
		} else {
			Class clazz = obj.getClass();
			Method method = null;
			try {
				method = clazz.getMethod(scheduleJob.getMethodName(), null);
			} catch (Exception e) {
				e.printStackTrace();
			}
			if (method == null) {
				result.setErrorMsg("未找到目标方法！");
				return result;
			}
		}

		try {
			scheduleJobService.addTask(scheduleJob);
			result.setSuccess(true);
		} catch (Exception e) {
			e.printStackTrace();
			result.setSuccess(false);
			result.setErrorMsg("保存失败，检查 name group 组合是否有重复！");
		}
		return result;
	}

	@RequestMapping("changeJobStatus")
	@ResponseBody
	public ExecuteResult<Boolean> changeJobStatus(Long jobId, String cmd) {
		ExecuteResult<Boolean> result = new ExecuteResult<Boolean>();
		try {
			scheduleJobService.changeStatus(jobId, cmd);
			result.setSuccess(true);
		} catch (SchedulerException e) {
			result.setSuccess(false);
			logger.error("", e);
			result.setErrorMsg("任务状态改变失败！");
		}
		return result;
	}

	@RequestMapping("jobList")
	public RetJson jobList(Integer draw, Integer length, Integer start) {
		RetJson retJson = new RetJson();
		final Sort sort = new Sort(Sort.Direction.DESC, "jobId");
		final Pageable pageRequest = new PageRequest(PageUtil.calcPage(start), length, sort);
		Page<ScheduleJob> pageData = scheduleJobDao.findAll(pageRequest);
		List<ScheduleJob> jobList = pageData.getContent();
		try {
			for (ScheduleJob job : jobList) {
				if ("1".equals(job.getJobStatus())) {
					scheduleJobService.addJob(job);
				}
			}
		} catch (SchedulerException e) {
			e.printStackTrace();
		}
		retJson.setData(pageData.getContent());
		retJson.setRecordsTotal(pageData.getTotalElements());
		retJson.setRecordsFiltered(pageData.getTotalElements());
		retJson.setDraw(draw == null ? 0 : draw);
		return retJson;
	}

	@RequestMapping("jobPage")
	public ModelAndView showArticlePage() {
		return new ModelAndView("/job/jobList");
	}

	@RequestMapping("updateCron")
	@ResponseBody
	public ExecuteResult<Boolean> updateCron(HttpServletRequest request, Long jobId, String cron) {
		ExecuteResult<Boolean> result = new ExecuteResult<Boolean>();
		try {
			CronScheduleBuilder scheduleBuilder = CronScheduleBuilder.cronSchedule(cron);
		} catch (Exception e) {
			result.setSuccess(false);
			logger.error("", e);
			result.setErrorMsg("cron表达式有误，不能被解析！");
			return result;
		}
		try {
			scheduleJobService.updateCron(jobId, cron);
			result.setSuccess(true);
		} catch (SchedulerException e) {
			result.setSuccess(false);
			result.setErrorMsg("cron更新失败！");
			logger.error("", e);
		}
		return result;
	}

}
```

### java反射说明

#### 反射的基石（如何得到字节码？）

- User.class
- Class.forName("java.lang.string")
- new Date().getClass();

#### 得到够得着的函数

```
//得到String带StringBuffer的构造函数
Constructor stro = String.class.getConstructor(StringBuffer.class);
//实例Stirng带有Stringbuffer的构造函数
String str = (Stirng)stro.newInstance(new StringBuffer("abc"));
//输出第2个字母
System.out.println(str.chartAt(2));
```

#### 得到成员变量

```
User user = new User("张三",23);
Field f = user.getClass().getField("name");
System.out.println(f.get(user));//f.get(user)得到user的name值
//private修饰的用下面这种方法获得
Field f1 = user.getClass().getDeclaredField("age");
f1.setAccessible(true);//设置可以访问（暴力反射）
System.out.println(f.get(user));
```

#### 将一个类中所有的String成员变量的值中带有  'b’的全部换成  'a'

```
private static void changeStringValue(Object obj){
    Field[] fields = obj.getClass().getField();//得到类中所有的成员变量
    for(Field field : fields){
        if(field.getType() == Stirng.class){
           String oldValue = (String)field.get(obj);//获取到值
           Stirng newValue = oldValue.replace('b','a');//替换值
           field.set(obj,newValue);//设置值
        }
    }
}
```

#### 得到某个类中的某个method（方法）

```
String str = "abc";
Method method = String.class.getMethod("charAt",int.class);//得到charAt方法
method.invoke(str,1);//调用charAt方法
```

#### 用类加载器加载配置文件

```
Person.class.getClassLoader().getResouseAsStream("com/test/file.properties");
Person.class.getResouseAsStream("com/test/file.properties");
```

> 以上就是反射的复习。希望对你有点用处。

## 运行结果

运行结果：

![运行结果](https://handexing.github.io/images/posts/springquartz.png)


## 结语
以上代码都在我的github上，其中有问题或者不对的地方欢迎交流。
项目地址：[WISH](https://github.com/handexing/wish)
框架集合地址：[frameworkAggregate](https://github.com/handexing/frameworkAggregate)里面有一个单独的quartz_demo案例，欢迎star，fork，里面会不定时更新一些框架的使用案例。欢迎交流。




