---
title: Spring Boot多线程
date: 2020-02-03 20:07:28
categories:
- 实践
tags: Spring Boot
---

# 背景

实现一个**多线程**爬虫，**配置**目标地址，以**Command Line**形式运行（无图形界面），完成后退出。

# CommandLineRunner & ApplicationRunner

若Spring Boot不是一个web项目、或者在web项目启动后需要执行部分代码，则需要通过实现`CommandLineRunner`或者`ApplicationRunner`接口，将具体代码放在`run()`方法中。
``` Java
@Component
@Order(value = 1)
public class TaskRunner implements CommandLineRunner {
    @Override
    public void run(String... args) throws Exception {
        // your code here
    }
}
```

上例实现了`CommandLineRunner`接口，其中`@Order`定义了多个实现执行顺序，数字小的先执行。同理`ApplicationRunner`也可以实现，唯一不同的是`ApplicationRunner`的`run()`方法参数为`ApplicationArguments`，可以提供更详细的命令行参数。

# Spring Boot多线程

Java中推荐使用ThreadPoolExecutor的形式开发多线程项目，ThreadPoolExecutor的构造函数包括7个参数：
- int corePoolSize 
线程池核心线程数量，即初始线程数量，即使空闲也不会销毁。
- int maximumPoolSize
线程池最大线程数量，即当前线程池允许创建的最大线程数量，多于核心数量的线程在空闲**一段时间**后可能会被销毁。
- long keepAliveTime
 **一段时间**的长度。
- TimeUnit unit
 **一段时间**的单位。
- BlockingQueue<Runnable> workQueue
任务队列，被添加到线程池中，但尚未被执行的任务。
- ThreadFactory threadFactory
线程工厂，用于创建线程，一般可以默认，如果想自定义线程名称，可以自行定制。
- RejectedExecutionHandler handler
拒绝策略，当任务太多无法处理时如何拒绝。

在Spring Boot中需要一个配置类来定义线程池
``` Java
@Configuration
@EnableAsync
public class AsyncConfig {
    @Bean("taskExecutor")//线程池名
    public Executor asyncExecutor(){
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);//核心线程数
        executor.setMaxPoolSize(5);//最大线程数
        executor.setQueueCapacity(10);//队列容量
        executor.setKeepAliveSeconds(60);//
        executor.setThreadNamePrefix("HousePrice-");
        executor.initialize();
        return executor;
    }
}
```

Spring Boot在`org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor`中包装了一个`ThreadPoolExecutor`，该线程池的队列容量默认`queueCapacity`为2^31-1，`createQueue()`方法中提到当`queueCapacity`被设置为小于等于0的数时，会转变成一个直接提交队列(SynchronousQueue)，否则默认为一个无界任务队列(LinkedBlockingQueue)。

``` Java
protected ExecutorService initializeExecutor(ThreadFactory threadFactory, RejectedExecutionHandler rejectedExecutionHandler) {
    BlockingQueue<Runnable> queue = this.createQueue(this.queueCapacity);
    //省略
    return executor;
}

protected BlockingQueue<Runnable> createQueue(int queueCapacity) {
    return (BlockingQueue)(queueCapacity > 0 ? new LinkedBlockingQueue(queueCapacity) : new SynchronousQueue());
}
```

完成配置后在需要多线程运行的方法上加上`@Async`即可
``` Java
@Async("taskExecutor")
public void findHouse(Integer index, CountDownLatch countDownLatch) throws InterruptedException, IOException {
    String url = String.format(configList.getUrl(), index);
    CloseableHttpClient httpClient = HttpClients.createDefault();
    HttpGet httpGet = new HttpGet(url);
    CloseableHttpResponse response = null;
    try {
        response = httpClient.execute(httpGet);
        if(response.getStatusLine().getStatusCode() == 200){
            String content = EntityUtils.toString(response.getEntity(), "UTF-8");
            ArrayList<String> houseList = GetHouseList.getList(content, configList.getPattern());
            logger.info(String.format("Got %dth list", index));
        }
    } catch (Exception e) {
        logger.error(e.toString());
    } finally {
        if(response != null){
            response.close();
        }
        httpClient.close();
        countDownLatch.countDown();
    }
}
```

`CountDownLatch`作为一个计数器，通过`countDownLatch.await()`方法保证所有任务运行结束后才可以触发下一步操作，例如计时结束等事件。在任务执行过程中将`CountDownLatch.countDown()`置于`finally`块中，以保证所有任务最终会执行倒数操作。

# 配置

配置文件后缀为`properties`，自定义的配置文件放在`resources`文件夹下，推荐通过配置类方式读取配置文件。

``` properties
list.url=https://bj.lianjia.com/ershoufang/chaoyang/pg%d/
list.quantity=10
list.pattern=https://bj.lianjia.com/ershoufang/[\\d]*.html
```

``` Java
@Configuration
@ConfigurationProperties(prefix = "list", ignoreInvalidFields = true)
@PropertySource("classpath:config/configList.properties")
@Component
public class ConfigList {
    private String url;
    private Integer quantity;
    private String pattern;
    //getter & setter
}
```

`prefix`指前缀，`@PropertySource("classpath:config/configList.properties")`指定该配置类对应的具体配置文件，其余配置项一一对应。在使用到该配置项的时候，可以注入该配置类，通过`get`方法获取文件内容（注意字符串转义）。