---
layout:     post
title:      Kafka之Producer
subtitle:   Kafka之Producer介绍
date:       2018-10-19
author:     XINGXING
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - Kafka
---

>
>Kafka之Producer篇
>

本篇我们叙述Kafka是如何将数据发送到服务端的;  
首先我们来看下案例,如下代码所示，这是一个简单的将消息发送到服务端的例子,首先配置链接Kafka的参数,然后用生成的配置生成一个KafkaProducer,这个就是kafka的生产者,紧接着生成一个ProducerRecord用于封装要发送的消息,最终调用producer.send(recoder)将消息发送至服务端,这个Demo比较粗糙;    
消息发送有两种方式:    
1.同步发送消息,producer.send(recoder)会返回一个future对象，然后future.get()会一直等待Kafka服务器的响应;  
2.异步发送消息,producer.send(recoder,new DemoProducerCallback());传入一个DemoProducerCallback类，但是这类要实现kafka的Callback，而异步处理便在onCompletion()方法中完成;  

```java
  public class ProdectDemo {


    public static void main(String[] args) {

        Properties props = new Properties();
        //broker地址
        props.put("bootstrap.servers", "node1:9092,node2:9092,node3:9092");
        //请求时候需要验证
        props.put("acks", "all");
        //请求失败时候需要重试
        props.put("retries", 0);
        //指定消息key序列化方式
        props.put("key.serializer",
                "org.apache.kafka.common.serialization.StringSerializer");
        //指定消息本身的序列化方式
        props.put("value.serializer",
                "org.apache.kafka.common.serialization.StringSerializer");


        Producer<String, String> producer = new KafkaProducer<String, String>(props);
        ProducerRecord<String, String> recoder=new ProducerRecord<String, String>("test","test1","test value");
        //同步发送消息
        try {
            RecordMetadata recordMetadata=producer.send(recoder).get();
        } catch (Exception e) {
            e.printStackTrace();
        }

        //异步发送消息
        try {
            producer.send(recoder,new DemoProducerCallback());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

}

 class DemoProducerCallback implements Callback {
    public void onCompletion(RecordMetadata recordMetadata, Exception e) {
        if (e != null) {//如果Kafka返回一个错误，onCompletion方法抛出一个non null异常。
            e.printStackTrace();//对异常进行一些处理，这里只是简单打印出来
        }
    }
}
```

上面介绍了Kafka如何生产数据的,下面我们来看下Kafka的数据生产的工作流程，如下图所示，根据上面的Demo我们可以看的出:  
1.我们先构建了一个KafkaProducer，然后就构建出了ProducerRecord；  
2.然后就调用producer.send(recoder)进行发送；  
3.而KafkaProducer接收到消息后会先对其进行序列化，然后结合本地缓存的元数据信息一起发送给partitioner去确定目标分区；  
4.确认分区后追加写入到内存中的消息缓冲池(accumulator),此时的send方法完成；  
5.最后发送给服务端Broker,服务器对生产者做出相应,至此消息发送完毕；  

![](https://ws4.sinaimg.cn/large/006tNbRwly1fwi7kfqmgtj30i906umxc.jpg)




