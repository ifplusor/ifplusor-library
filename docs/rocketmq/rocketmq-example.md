---
tags: RocketMQ
---

# RocketMQ Example

## 并发消费

```java
public class Consumer {

    public static void main(String[] args) throws InterruptedException, MQClientException {

        // 实例化消费者
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("please_rename_unique_group_name");

        // 设置NameServer的地址
        consumer.setNamesrvAddr("localhost:9876");

        // 订阅一个或者多个Topic，以及Tag来过滤需要消费的消息
        consumer.subscribe("TopicTest", "*");

        // 注册回调实现类来处理从broker拉取回来的消息
        consumer.registerMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                System.out.printf("%s Receive New Messages: %s %n", Thread.currentThread().getName(), msgs);
                // 标记该消息已经被成功消费
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });

        // 启动消费者实例
        consumer.start();
        System.out.printf("Consumer Started.%n");
    }
}
```

### 拉模式订阅

```java
public class LitePullConsumerSubscribe {

    public static volatile boolean running = true;

    public static void main(String[] args) throws Exception {

        // 实例化消费者
        DefaultLitePullConsumer litePullConsumer = new DefaultLitePullConsumer("lite_pull_consumer_test");

        // 设置NameServer的地址
        litePullConsumer.setNamesrvAddr("localhost:9876");

        // 设置消费位点
        litePullConsumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);

        // 订阅一个或者多个Topic，以及Tag来过滤需要消费的消息
        litePullConsumer.subscribe("TopicTest", "*");

        // 启动消费者实例
        litePullConsumer.start();

        try {
            while (running) {
                // 拉取消息
                List<MessageExt> messageExts = litePullConsumer.poll();
                System.out.printf("%s%n", messageExts);
            }
        } finally {
            // 关闭消费者实例
            litePullConsumer.shutdown();
        }
    }
}
```

## 请求-应答

```java
public class RequestProducer {
    public static void main(String[] args) throws MQClientException, InterruptedException {
        String producerGroup = "please_rename_unique_group_name";
        String topic = "RequestTopic";
        long ttl = 3000;

        // 实例化生产者
        DefaultMQProducer producer = new DefaultMQProducer(producerGroup);
        producer.start();

        try {
            // 创建请求消息
            Message msg = new Message(topic,
                "",
                "Hello world".getBytes(RemotingHelper.DEFAULT_CHARSET));

            long begin = System.currentTimeMillis();
            // 发送请求并等待应答
            Message retMsg = producer.request(msg, ttl);
            long cost = System.currentTimeMillis() - begin;
            System.out.printf("request to <%s> cost: %d replyMessage: %s %n", topic, cost, retMsg);
        } catch (Exception e) {
            e.printStackTrace();
        }
        producer.shutdown();
    }
}
```

```java
public class ResponseConsumer {
    public static void main(String[] args) throws InterruptedException, MQClientException {
        // 为发送应该消息实例化生产者
        DefaultMQProducer replyProducer = new DefaultMQProducer("please_rename_unique_group_name");
        replyProducer.start();

        // 实例化消息者
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("please_rename_unique_group_name");
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_LAST_OFFSET);

        // recommend client configs
        consumer.setPullTimeDelayMillsWhenException(0L);

        consumer.registerMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                System.out.printf("%s Receive New Messages: %s %n", Thread.currentThread().getName(), msgs);
                for (MessageExt msg : msgs) {
                    try {
                        System.out.printf("handle message: %s", msg.toString());
                        String replyTo = MessageUtil.getReplyToClient(msg);
                        byte[] replyContent = "reply message contents.".getBytes();
                        // 使用消息工具创建应答消息
                        Message replyMessage = MessageUtil.createReplyMessage(msg, replyContent);
                        // 发送应答消息
                        SendResult replyResult = replyProducer.send(replyMessage, 3000);
                        System.out.printf("reply to %s , %s %n", replyTo, replyResult.toString());
                    } catch (MQClientException | RemotingException | MQBrokerException | InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });

        consumer.subscribe("RequestTopic", "*");
        consumer.start();
        System.out.printf("Consumer Started.%n");
    }
}
```
