> 原来写了一篇关于springboot实现计划任务的文章，但是有较多的人都在问为什么数据库变更后计划任务没刷新，怎么去动态获取，怎么实现多线程并发执行，所以现在新开一篇文章，重新实现计划任务的方法，抽象出刷新功能和并发功能。
### 动态获取并刷新的类：
```java
@Component
public class ScheduledTask implements SchedulingConfigurer {
    private static final Logger LOGGER = LoggerFactory.getLogger(ScheduledTask.class);

    private volatile ScheduledTaskRegistrar registrar;

    private final ConcurrentHashMap<Integer, ScheduledFuture<?>> scheduledFutures = new ConcurrentHashMap<Integer, ScheduledFuture<?>>();
    private final ConcurrentHashMap<Integer, CronTask> cronTasks = new ConcurrentHashMap<Integer, CronTask>();

    @Override
    public void configureTasks(ScheduledTaskRegistrar registrar) {
        //设置20个线程,默认单线程
        registrar.setScheduler(Executors.newScheduledThreadPool(20));
        this.registrar = registrar;
    }

    public void refresh(List<LogTask> tasks){
        //取消已经删除的策略任务
        Set<Integer> sids = scheduledFutures.keySet();
        for (Integer sid : sids) {
            if(!exists(tasks, sid)){
                scheduledFutures.get(sid).cancel(false);
            }
        }
        for (LogTask logTask : tasks) {
            //ScheduledTaskRunnable t = new ScheduledTaskRunnable(logTask.getTask_id(), logTask.getRule_db_id());
            String expression = logTask.getExpression();
            //计划任务表达式为空则跳过
            if(StringUtils.isEmpty(expression)){
                continue;
            }
            //计划任务已存在并且表达式未发生变化则跳过
            if(scheduledFutures.containsKey(logTask.getTask_id()) && cronTasks.get(logTask.getTask_id()).getExpression().equals(expression)){
                continue;
            }
            //如果策略执行时间发生了变化，则取消当前策略的任务
            if(scheduledFutures.containsKey(logTask.getTask_id())){
                scheduledFutures.get(logTask.getTask_id()).cancel(false);
                scheduledFutures.remove(logTask.getTask_id());
                cronTasks.remove(logTask.getTask_id());
            }
            CronTask task = new CronTask(new Runnable() {
                @Override
                public void run() {
                    //每个计划任务实际需要执行的具体业务逻辑
                }
            }, expression);
            ScheduledFuture<?> future = registrar.getScheduler().schedule(task.getRunnable(), task.getTrigger());
            cronTasks.put(logTask.getTask_id(), task);
            scheduledFutures.put(logTask.getTask_id(), future);
        }
    }

    private boolean exists(List<LogTask> tasks, Integer tid){
        for(LogTask logTask:tasks){
            if(logTask.getTask_id() == tid){
                return true;
            }
        }
        return false;
    }

    @PreDestroy
    public void destroy() {
        this.registrar.destroy();
    }

}
```

### 唯一用的对象类：
```java
@Data
public class LogTask {
    private int task_id;
    private String expression;
}
```

### 如何使用
1. 在你的项目中找个地方去实现从数据库获取这个对象集合的方法
2. 把获取到的数据通过调用 refresh 方法去刷新到计划任务列表中
3. 每次数据库中的计划任务发生变化后重新执行上面两步即可
