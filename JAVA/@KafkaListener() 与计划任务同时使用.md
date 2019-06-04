### 前言
我们在开发过程中经常会用到计划任务，而计划任务中我们比较常用的是 @Scheduled() ，但是，当我们同时使用了计划任务和kafka监听消费后，我们的计划任务就无法成功生效了，因为我们不管是使用多少线程，kafka监听消费会一直占用，这样就会导致我们使用注解方式的计划任务无法生效，如果这个时候我们仍然需要使用计划任务，可以使用下面的方式去替换计划任务。

### 计划任务
#### 周期执行
```
 //设置一个当前计划任务使用一个线程
ScheduledExecutorService rollService = Executors.newScheduledThreadPool(
         1,
         new ThreadFactoryBuilder().setNameFormat(
                 "自定义线程名" +
                         Thread.currentThread().getId() + "-%d").build());
 rollService.scheduleAtFixedRate(new Runnable() {
     @Override
     public void run() {
         LOGGER.info("Marking time to execute task");
     }
 }, 60, 600, TimeUnit.SECONDS);
```
上面代码表示延迟60秒执行`run(){}`里面的方法，第一次执行后每隔600秒重复执行一次。
但是我们知道，我们使用计划任务通常是整点或者是在某个指定时间开始第一次执行任务，上面代码可以看出是不具备这个功能的，但是我们只需要修改很少的一部分代码就可以实现我们真正想要的周期执行的计划任务。
那就是设置延迟时间，也就是代码中的那个`60`。

#### 延迟时间
```
 /**
  * 当前时间与周期时间的时间差
  * 时间单位 秒
  * @param executorTime 执行周期
  * @return 周期执行的等待时间
  */
 public static int executorsDelayTime(int executorTime){
     //获取当前时间与周期时间的差
     long nowTime = System.currentTimeMillis();
     int timeDiscrepancy = Integer.parseInt(String.valueOf(nowTime%(executorTime*1000)/1000));
     int delayTime = executorTime-timeDiscrepancy;
     return delayTime;
 }
```
上面的代码可以计算出服务启动时间与设置的周期时间的一个时间差，例如当前时间是 `09:28:41`，执行周期是 `300`，也就是5分钟整点执行一次，那么我们期望的下次执行时间应该是  `09:30:00`，通过上面的方法就会计算出首次执行任务的等待时间是 `79`。

### 总结
所以，综上所述，如果只想要周期执行，用第一个周期方法即可，如果想要模拟注解方式的计划任务，只需要搭配上下面的获取延迟时间的方法即可。
需要注意的是两个方法的时间单位都是秒，周期方法里面的时间单位如果修改了，获取延迟时间的方法也需要进行修改。建议根据实际需求去进行改造。（PS：毕竟如果是按照小时计算等待时间和周期时间的，用秒单位就太大了）

### @KafkaListener() 两种方式加载多个topic
```
# application.properties
spring.kafkaListenerList = topic1,topic2
```
#### 第一种
```
@KafkaListener(topics = "#{'${spring.kafkaListenerList}'.split(',')}")
```
#### 第二种
`KafkaListenerReceiver`是自定义的消费对象
* KafkaListenerConfig.java
```
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang.StringUtils;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
import org.springframework.beans.factory.support.BeanDefinitionBuilder;
import org.springframework.beans.factory.support.BeanDefinitionRegistry;
import org.springframework.beans.factory.support.BeanDefinitionRegistryPostProcessor;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.Ordered;
import org.springframework.core.PriorityOrdered;

import java.io.File;
import java.io.FileInputStream;
import java.io.IOException;
import java.nio.file.Paths;
import java.util.Arrays;
import java.util.List;
import java.util.Properties;


@Configuration
@Slf4j
public class KafkaListenerConfig implements BeanDefinitionRegistryPostProcessor, PriorityOrdered {

    private List<String> listenerList;

    KafkaListenerConfig() throws IOException {
        Properties pro = new Properties();
        File f;
        f =new File( java.net.URLDecoder.decode(this.getClass().getResource("/application.properties").getPath(),"utf-8"));
        FileInputStream in = new FileInputStream(f);
        pro.load(in);
        in.close();
        String listString = pro.getProperty("spring.verify.kafkaListenerList");
        if (StringUtils.isEmpty(listString)){
            log.error("spring.verify.kafkaListenerList is empty or null");
            System.exit(0);
        }
        log.info("spring.kafkaListenerList: "+listString);
        listenerList = Arrays.asList(listString.split(","));
    }

    public List<String> getListenerList() {
        return listenerList;
    }

    public void setListenerList(List<String> listenerList) {
        this.listenerList = listenerList;
    }

    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry beanDefinitionRegistry) throws BeansException {
        for (String listener :  listenerList) {
            BeanDefinitionBuilder builder = BeanDefinitionBuilder.rootBeanDefinition(KafkaListenerReceiver.class);
            builder.addConstructorArgValue(listener);
            beanDefinitionRegistry.registerBeanDefinition(listener, builder.getBeanDefinition());
        }
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory configurableListableBeanFactory) throws BeansException {

    }

    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE;
    }
}
```
* KafkaListenerReceiver.java
```
@KafkaListener(topics = "#{__listener.topic}")
```
