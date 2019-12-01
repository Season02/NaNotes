
# 一次线上项目内存溢出问题处理实战

背景

公司是做运维监控产品的，产品是部署在客户环境上运行，然后由当地的运维同事看管。
某天突然收到一个现场的同事发来消息，说产品瘫痪了，界面进不去，然后就让运维查
日志，立马就发现有 oom 异常。
通过 jstat -gcutil <pid> 2000 命令查看，FGC已经九百多次了，已经完蛋了。
因为在启动脚本中堆溢出配置 -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/heap/dump
于是让运维把dump文件传回来开始分析问题。
另外，就算没有上面的配置，也可以通过命令手动生成dump文件
jcmd <PID> GC.heap_dump <PATH>

<!--more-->

# 分析

简单介绍下出问题的这个应用，它是通过`kafka`从上游获取数据，处理后再放入下游`kafka`。

是个标准的消费生产模式的应用。线上项目的最大堆内存设置为10G，这都能溢出还是有些意外。


## 内存分析

拿到`dump`文件后使用`VisualVM`分析，很快就发现其中有大量的大数组对象占空间，继续定位
发现他们都来源于一个 LinkedBlockingQueue 对象，这个队列已经基本把堆内存用光了。

![](https://i.loli.net/2019/11/30/R3PB26fsuzh9QwH.jpg)

通过名称在代码搜索发现在项目中使用这个队列来缓冲从`kafka`收到的数据

```java
LinkedBlockingQueue<String> queue = new LinkedBlockingQueue<>(500000);
```
而且这个队列的大小被设置为了50W，很恐怖。

>  缓冲队列设置的过大，结合目前的情况来看是后续处理过慢导致队列的任务越堆越多。



# 定位

除了队列很大外，队列内的每条数据体积也很大，一条能有7，8百K了，这也是一个大问题，分析上
游业务得知，这些数据会把所有相关数据的ID一次全部关联起来，如果有1W条有关数据，就会全部
加载并传输，设计非常不合理。

通过上面对堆dump文件的分析觉得已经定位到问题的大致原因，就开始了代码分析，就没有进一步
分析线程dump，实际上如果能再进一步分析线程情况，能够快更有效的定位问题，这也是一个疏忽，
线程`dump`可以使用命令jcmd <PID> Thread.print (JDK version > 8)来生成。

为了具体定位问题 review 了代码。


整理之后的伪代码如下：

```java
private static LinkedBlockingQueue<String> receiveFromCepQueue = new LinkedBlockingQueue<>(500000);
//从`kafka`获取数据
private class kafkaConsumerThread extends Thread {
    public void run() {
        while (true){
	    ConsumerRecords<String, String> records = consumer.poll(1);
	    for (ConsumerRecord<String,String> record:records){
	    	receiveFromCepQueue.offer(value);
	    }
        }
    }
}
new kafkaConsumerThread().start();

//处理数据线程
private class ActionHandleThread extends Thread{
public void run() {
    while (true) {
	String data = receiveFromCepQueue.take();
	handleData(data);//业务处理，写数据库
    }
}
new ActionHandleThread().start();

```

大致的流程如下：

- 不断的从 队列 中获取数据。
- 将数据丢到业务线程中。
- 一系列数据库操作。

在这里能发现两个问题，首先缓冲队列过大，然后同步消费过慢，在数据量陡增的情况下，消费不完的数据会
堆积在队列中无法回收，导致堆内存溢出。这是依据代码推出的结论，如果需要更确切的证据，还是查看线程
情况的。


其中最慢的部分：也就是有一个**数据库的操作**。

这里有一些列工作，查询，匹配，在满足特定条件后删除或更新，插入数据。
而且这里涉及到的不止一个库，牵连到两个额外的业务系统查询，使用同步方式处理确实会有不小压力。


# 优化

至此整个排查结束，而我们后续的调整措施大概如下：

- 将后面处理的逻辑优化，简化代码，并且把耗时操作异步化
- 数据库优化，优化索引，数据库缓存，并且查询的操作引入缓存，提升效率
- 限制一条数据能关联到事件的数量最大到 1000，超过的按最早删除，这是业务上设计失误
- 删除缓冲LinkedBlockingQueue，kafka自身就有优秀的抗压能力，不需要再额外处理

优化过的处理伪代码如下
```java
//异步处理耗时操作
ExecutorService actionPool = Executors.newFixedThreadPool(64);

//具体消费kafka数据的线程
@Component
@Scope("prototype")
public class ActionScheduler extends Thread{
private KafkaConsumer<String, String> kafkaConsumer = null;
public void run() {
    while (true) {
        ConsumerRecords<String, String> records = kafkaConsumer.poll(DATA_POLL_TIMEOUT);
        for (ConsumerRecord<String,String> record:records){
            actionPool.execute(() -> handleData(record.value()));
        }
    }
}
//创建线程池，按照topic，点对点消费
ExecutorService service = Executors.newFixedThreadPool(partitionList.size());
for (PartitionInfo partitionInfo : partitionList) {
    KafkaConsumer<String,String> consumer = createConsumer(URL,Topic,partitionInfo.partition(),i++);
    ActionScheduler as = getBean(ActionScheduler.class);//获取消费线程实例，要避免单例模式
    as.setKafkaConsumer(consumer);
    service.submit(as);
}

//设置consumer点对点消费，其他配置省略
private KafkaConsumer<String,String> createConsumer(String ip,String topic,int partitionNum,int partitionIndex){
    Properties props = new Properties();
    //一个消费者对应一个消费组避免争用，提升消费效率
    props.put("group.id", "alert"+partitionIndex);
    KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
    TopicPartition partition = new TopicPartition(topic,partitionNum);
    consumer.assign(Arrays.asList(partition));
    return consumer;
}


```

修改后重新发布执行，让运维连上`VisualVM`后，实时查看堆栈信息，已经能稳定维持了，问题处理到此结束。
![](https://i.loli.net/2019/12/01/5whPUNjfvIMJADs.jpg)


**你的点赞与分享是对我最大的支持**
