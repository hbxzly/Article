# Springboot项目平滑关闭及自动化关闭脚本

## 核心代码
#### GracefulShutdown.java
```java
package cnkj.site.utils;

import org.apache.catalina.LifecycleException;
import org.apache.catalina.connector.Connector;
import org.apache.catalina.util.LifecycleBase;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.web.embedded.tomcat.TomcatConnectorCustomizer;
import org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory;
import org.springframework.boot.web.servlet.server.ConfigurableServletWebServerFactory;
import org.springframework.context.ApplicationListener;
import org.springframework.context.annotation.Bean;
import org.springframework.context.event.ContextClosedEvent;
import java.util.concurrent.Executor;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

/*
 * @version 1.0 created by Carol on 2019/4/25 16:22
 */
public class GracefulShutdown implements TomcatConnectorCustomizer, ApplicationListener<ContextClosedEvent> {
    private static final Logger LOGGER = LoggerFactory.getLogger(GracefulShutdown.class);

    private volatile Connector connector;

    @Override
    public void customize(Connector connector) {
        this.connector = connector;
    }
    @Override
    public void onApplicationEvent(ContextClosedEvent event) {
        try {
            // 指定执行的方法
            shutdown();
            //手动清理内存
            System.gc();
            LOGGER.warn("清理内存完毕，正在退出服务......");
            if (this.connector == null){
                return;
            }
            this.connector.pause();
            LOGGER.warn("关闭全部连接......");
            Executor executor = this.connector.getProtocolHandler().getExecutor();
            if (executor instanceof ThreadPoolExecutor) {
                try {
                    ThreadPoolExecutor threadPoolExecutor = (ThreadPoolExecutor) executor;
                    threadPoolExecutor.shutdown();
                    LOGGER.warn("当前服务线程池被关闭");
                    if (!threadPoolExecutor.awaitTermination(30, TimeUnit.SECONDS)) {
                        LOGGER.warn("Tomcat thread pool did not shut down gracefully within 30 seconds. Proceeding with forceful shutdown");
                    }
                } catch (InterruptedException ex) {
                    Thread.currentThread().interrupt();
                }
            }
            this.connector.stop();
        } catch (LifecycleException e) {
            e.printStackTrace();
        }
    }

    @Bean
    public GracefulShutdown gracefulShutdown() {
        return new GracefulShutdown();
    }
    @Bean
    public ConfigurableServletWebServerFactory webServerFactory(final GracefulShutdown gracefulShutdown) {
        TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();
        factory.addConnectorCustomizers(gracefulShutdown);
        return factory;
    }


    /**
     * 执行服务关闭前的一些定制化操作
     * 通常需要确认以下步骤
     * 1.关闭kafka等数据连接
     * 2.flush内存中全部的未处理数据
     * 3.清理服务中全部待处理的数据
     */
    public void shutdown(){}
}

```

#### Shutdown.java
```java
import cnkj.site.utils.GracefulShutdown;
import org.springframework.stereotype.Component;

/*
 * @version 1.0 created by Carol on 2019/4/25 16:39
 */
@Component
public class Shutdown extends GracefulShutdown {

    @Override
    public void shutdown() {
        // TODO 定制化关闭操作流程
        // 关闭 kafka 消费
        // flush全部读写流
        // 清空队列
        // 关闭全部文件流读写
    }
}
```

#### ApplicationStarterRunner.java
```java
package cn.migu.log.component;

import cnkj.site.utils.HttpCommonUtil;
import cnkj.site.CommonInfo;
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;

/*
 * @version 1.0 created by LXW on 2019/3/14 17:05
 */
@Component
public class ApplicationStarterRunner implements CommandLineRunner {

    @Override
    public void run(String... args) throws Exception {
        // 设置服务名
        commonInfo.setSERVICE_NAME("Service-Name");
        // 自动设置服务启动后的进程号
        commonInfo.setSERVICE_PID(HttpCommonUtil.getCurrentPid());
    }
}
```

#### CommonInfo.java
```java
package cnkj.site.utils;

import lombok.Builder;
import lombok.Data;
import org.springframework.boot.actuate.info.Info;
import org.springframework.boot.actuate.info.InfoContributor;
import org.springframework.stereotype.Component;

import java.util.HashMap;
import java.util.Map;


@Data
@Component
public class CommonInfo implements InfoContributor {

    //当前服务名
    private String SERVICE_NAME="SERVICE_NAME";
    //服务当前状态
    private int SERVICE_PID;


    @Override
    public void contribute(Info.Builder builder) {
        builder.withDetail("SERVICE_NAME",SERVICE_NAME);
        builder.withDetail("SERVICE_PID", SERVICE_PID);
    }

    public void clearAll(){
        this.SERVICE_NAME="";
        this.SERVICE_PID=-1;
    }

    public Map getAll(){
        Map map = new HashMap();
        map.put("SERVICE_NAME", getSERVICE_NAME());
        map.put("SERVICE_PID", getSERVICE_PID());
        return map;
    }


}

```

#### HttpCommonUtil.java
```java
package cnkj.site.utils;

import javax.servlet.http.HttpServletRequest;
import java.lang.management.ManagementFactory;
import java.net.InetAddress;
import java.net.UnknownHostException;

/*
 * @version 1.0 created by Carol on 2018/10/25 10:04
 */
public class HttpCommonUtil {

    /**
     * 获取当前服务的PID
     * @return PID
     */
    public static Integer getCurrentPid(){
        String name = ManagementFactory.getRuntimeMXBean().getName();
        String pid = name.split("@")[0];
        return Integer.valueOf(pid);
    }
}
```

#### application.properties
```java
#服务关闭
management.endpoint.shutdown.enabled=true
#监控相关
management.endpoint.prometheus.enabled=true
management.endpoints.web.exposure.include=info
```

## 操作步骤
项目使用步骤：

1.拷贝上面的 Shutdown.java 代码到自己的项目中

2.在 Shutdown.java 文件中的shutdown 方法中写定制化的关闭操作流程


脚本使用步骤：

1.从git获取最新的项目关闭脚本 https://github.com/carolcoral/CommonUtils/tree/master/shell/server_close

2.压缩server_close 为server_close.zip

3.上传 server_close.zip 到你服务所在服务器上的 /data/shell 路径下

4.配置环境变量 vim /etc/profile

5.在profile文件的最下面新增 export PATH=/data/shell/server_close:$PATH

6.保存并退出 :wq

7.如果提示  /bin/bash^M: bad interpreter: No such file or directory，请vim serviceControll.sh，然后 :set fileformat=unix ，然后 :wq 保存并退出即可

8.cd /data/shell/server_close & ./serviceControll.sh 运行即可使用服务关闭脚本
