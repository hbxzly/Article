# 基于Springboot执行多个定时任务并且动态获取定时任务信息

@[toc]
## 简介
因为一些业务的需要所有需要使用多个不同的定时任务，并且每个定时任务中的定时信息是通过数据库动态获取的。下面是我写的使用了Springboot+Mybatis写的多任务定时器。
主要实现了以下功能：

		1.同时使用多个定时任务
		2.动态获取定时任务的定时信息

## 说明
因为我们需要从数据库动态的获取定时任务的信息，所以我们需要集成 SchedulingConfigurer 然后重写  configureTasks 方法即可，调用不同的定时任务只需要通过service方法调用不用的实现返回对应的定时任务信息。有多少个定时任务就重写多少个该方法（最好创建不同的类）。然后直接在application中启动即可。

## SpringApplication-启动类
```
package test;

import org.mybatis.spring.annotation.MapperScan;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.scheduling.annotation.EnableScheduling;
import org.springframework.transaction.annotation.EnableTransactionManagement;


@SpringBootApplication
@EnableTransactionManagement
@EnableScheduling
@ComponentScan(value = {"test.*"})
@MapperScan("test.mapper.*")
public class TomcatlogApplication {

	public static void main(String[] args) {
		SpringApplication.run(TomcatlogApplication.class, args);
	}

}
```

## 动态获取定时任务信息
### mapper
```
import org.apache.ibatis.annotations.Param;
import org.apache.ibatis.annotations.Select;

import java.util.List;

/*
 * @version 1.0 created by liuxuewen on 2018/8/21 14:39
 */
public interface TomcatlogMapper {
    @Select("SELECT * FROM scheduledtask s WHERE s.`enable` = 1")
    List<ScheduledtaskEntity> queryScheduledTask();
}

```
### service
```
package test.service;
import java.util.ArrayList;
import java.util.List;

/*
 * @version 1.0 created by liuxuewen on 2018/8/21 14:44
 */
public interface TomcatlogService {
    List<ScheduledtaskEntity> queryScheduledTask();
}

```
### service impl
```
import test.mapper.tomcatlog.TomcatlogMapper;
import test.service.TomcatlogService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import java.util.List;

/*
 * @version 1.0 created by liuxuewen on 2018/8/21 14:44
 */
@Service
public class TomcatlogServiceImpl implements TomcatlogService {
    private static final Logger LOGGER = LoggerFactory.getLogger(TomcatlogServiceImpl.class);

    @Autowired
    TomcatlogMapper tomcatlogMapper;

    @Override
    public List<ScheduledtaskEntity> queryScheduledTask() {
        try {
            List<ScheduledtaskEntity> res = tomcatlogMapper.queryScheduledTask();
            return res;
        } catch (Exception e) {
            LOGGER.info(e);
        }
        return null;
    }
```
## 定时任务
```
import test.service.TomcatlogService ;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.annotation.SchedulingConfigurer;
import org.springframework.scheduling.config.ScheduledTaskRegistrar;
import org.springframework.scheduling.support.CronTrigger;
import org.springframework.stereotype.Component;
import org.springframework.util.StringUtils;

/*
 * @version 1.0 created by liuxuewen on 2018/8/27 9:25
 */
@Component
public class ElasticsearchSchedultaskController implements SchedulingConfigurer {
    private static final Logger LOGGER = LoggerFactory.getLogger(ElasticsearchSchedultaskController.class);

    @Autowired
    private TomcatlogService controllerService;

    @Override
    public void configureTasks(ScheduledTaskRegistrar scheduledTaskRegistrar) {
        try {
            controllerService.queryScheduledTask().forEach((cron)->{
                scheduledTaskRegistrar.addTriggerTask(
                        //1.添加任务内容(Runnable)，可以为方法
                        () -> System.out.println("定时任务1"),
                        //2.设置执行周期(Trigger)
                        triggerContext -> {
                            //2.1 从数据库获取执行周期，在这里调用不同的方法返回不同的定时任务信息
                            System.out.println(cron);
                            //2.2 合法性校验.
                            if (StringUtils.isEmpty(cron)) {
                                // Omitted Code ..
                                LOGGER.error("计划任务为空");
                            }
                            //2.3 返回执行周期(Date)
                            return new CronTrigger(cron).nextExecutionTime(triggerContext);
                        }
                );
            });
            
        }catch (Exception e){
            LOGGER.info(e.toString());
        }
    }
}
```
