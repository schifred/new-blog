---
title: spring boot 入门：定时任务
category:
  - 后端
  - spring
tags:
  - 后端
  - spring
  - spring boot
keywords: spring boot
abbrlink: 2290be3f
date: 2019-03-17 05:00:00
updated: 2019-03-17 05:00:00
---

当应用指定实体的活动时间时，定时任务就派上用场了。Spring Boot 创建定时任务有三种方式：

* 基于 @EnableScheduling、@Scheduled 注解实现一般定时任务
* 实现 SchedulingConfigurer 接口，动态注册定时任务。
* 基于 @EnableAsync、@Async 启动多线程定时任务

### 一般定时任务

@EnableScheduling 注解用于开启定时任务。

@Scheduled 注解用于指定定时任务的执行时机。其参数可以是 corn 表达式，也可以是 fixedDelay 指定延时，也可以是 fixedRate 执行频率。

```java
@Component
@EnableScheduling
public class ScheduledTask {
    // 每分钟执行
    @Scheduled(cron = "0 0/1 * * * ?")
    // @Scheduled(fixedRate=5000) 间隔 5s 执行一次
    public void timerToNow(){
        System.out.println("now time:" + LocalDateTime.now());
    }
}
```

### 动态注册

动态注册一般指从数据库获取定时任务的执行时机，然后基于实现 SchedulingConfigurer 接口的类动态注册定时任务。以下示例使用 mybatis 的另一种用法：

```java
@Component
@EnableScheduling
public class DynamicScheduleTask implements SchedulingConfigurer {
    @Autowired
    @SuppressWarnings("all")
    CronMapper cronMapper;// 从数据库获取 corn 配置

    @Mapper
    public interface CronMapper {
        @Select("select cron from cron limit 1")
        public String getCron();
    }

    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        taskRegistrar.addTriggerTask(
            () -> System.out.println("执行动态定时任务: " + LocalDateTime.now().toLocalTime()),
            // 设置执行周期
            triggerContext -> {
                String cron = cronMapper.getCron();
                if (StringUtils.isEmpty(cron)) {
                    // Omitted Code ..
                }
                return new CronTrigger(cron).nextExecutionTime(triggerContext);
            }
        );
    }
}
```

### 多线程定时任务

我们使用 @EnableAsync、@Async 注解开启多线程定时任务。

```java
@Component
@EnableScheduling
@EnableAsync
public class MultithreadScheduleTask {
    @Async// spring 内部会创建线程池，不阻塞接下来的进程
    @Scheduled(fixedDelay = 1000)
    public void first() throws InterruptedException {
        System.out.println("first task: " + LocalDateTime.now().toLocalTime() + "\r\n thread: " + Thread.currentThread().getName());
        System.out.println();
        Thread.sleep(1000 * 10);
    }

    @Async
    @Scheduled(fixedDelay = 2000)
    public void second() {
        System.out.println("second task: " + LocalDateTime.now().toLocalTime() + "\r\n trhead: " + Thread.currentThread().getName());
        System.out.println();
    }
}
```

### corn 表达式

cron 表达式依次指定：

* 秒（0~59） 
* 分钟（0~59） 
* 小时（0~23） 
* 天（月）（0~31，但是你需要考虑你月的天数）
* 月（0~11） 
* 天（星期）（1~7 1=SUN 或 SUN，MON，TUE，WED，THU，FRI，SAT） 
* 年份（1970－2099） 

每个元素可以设定为一个值（如 6 指 6:00），一个连续区间（如 9-12 指 9:00 到 12：00），一个间隔时间（如 0/4 指从 0 点开始每隔4小时），一个列表（如 1,3,5 指 1:00、3:00、5:00），通配符 *（指每个小时）。

corn 表达式指定天的方式有两种，为了避免冲突，在设置某一种时，需要使用 ? 将另一种置空。在指定天时，L 表示一个月或一个礼拜的最后一天，6L 表示一个月或的一个礼拜的倒数第 6 天。

更多内容可戳 [cron表达式](https://www.cnblogs.com/tommyli/p/3760671.html)。
