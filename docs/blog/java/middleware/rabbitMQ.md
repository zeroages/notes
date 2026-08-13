Ubuntu安装MQ

# 添加RabbitMQ的GPG密钥
wget -O- https://www.rabbitmq.com/rabbitmq-release-signing-key.asc | sudo apt-key add -
# 添加Erlang的APT源
echo "deb https://dl.bintray.com/rabbitmq-erlang/debian $(lsb_release -cs) erlang" | sudo tee /etc/apt/sources.list.d/rabbitmq-erlang.list
# 更新APT缓存并安装Erlang
sudo apt update
sudo apt install -y erlang

安装RabbitMQ服务器
# 添加RabbitMQ的GPG密钥
wget -O- https://www.rabbitmq.com/rabbitmq-release-signing-key.asc | sudo apt-key add -
# 添加RabbitMQ的APT源
echo "deb https://dl.bintray.com/rabbitmq/debian $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/rabbitmq.list
# 更新APT缓存并安装RabbitMQ
sudo apt update
sudo apt install -y rabbitmq-server
启用管理插件
启用管理插件
sudo rabbitmq-plugins enable rabbitmq_management

配置远程访问权限
# 编辑配置文件
sudo nano /etc/rabbitmq/rabbitmq.conf
# 在配置文件中添加以下内容
loopback_users = none

创建管理用户
# 添加用户（替换<username>和<password>）
sudo rabbitmqctl add_user rabbit rabbit
# 设置为管理员角色
sudo rabbitmqctl set_user_tags rabbit administrator
# 设置权限
sudo rabbitmqctl set_permissions -p "/" rabbit ".*" ".*" ".*"

延时队列插件安装
1. 下载插件
wget https://github.com/rabbitmq/rabbitmq-delayed-message-exchange/releases/download/v3.12.0/rabbitmq_delayed_message_exchange-3.12.0.ez
2. 复制插件到RabbitMQ插件目录
sudo cp rabbitmq_delayed_message_exchange-3.12.0.ez /usr/lib/rabbitmq/lib/rabbitmq_server-3.12/plugins/
3. 启用延时队列插件
sudo rabbitmq-plugins enable rabbitmq_delayed_message_exchange
4. 重启服务
常用管理命令
● 停止服务：sudo rabbitmqctl stop
● 重启应用：sudo rabbitmqctl start_app
● 查看状态：sudo rabbitmqctl status
● 查看用户列表：sudo rabbitmqctl list_users
● 删除用户：sudo rabbitmqctl delete_user username


访问管理界面
http://服务器IP:15672/


延时队列
1.延时队列的应用场景：
场景：有一个订单，15 分钟内如果不支付，就把该订单设置为交易关闭，那么就不能再进行支付了，这类实现延迟任务的场景就可以采用延时队列的方式来实现，当然除了延时队列，还可以采用其它的方式——> 定时任务的方式。
1. 订单在十分钟之内未支付则自动取消。
2. 新创建的店铺，如果在十天内都没有上传过商品，则自动发送消息提醒。
3. 用户注册后，如果三天内没有登陆则进行短信提醒。
4. 用户发起退款，如果三天内没有得到处理则通知相关运营人员。
5. 预定会议后，需要在预定的时间点前 10 分钟，通知各个参会人员参加会议。
2.实现方式
● 队列TTL+死信
该延时队列存在的问题：
第一条消息在 10 S 后变成了死信消息，然后被消费者消费/读取掉了，第二条消息在 40S 之后变成了死信消息，然后被消费掉，这样一个延时队列就被打造完成了。
不过，该延时队列存在一个问题：
如果这样使用的话，岂不是每增加一个新的时间需求，就要新增一个队列。 这里只有 10S 和 40S 两个时间选项，如果需要一个小时后处理，那么就需要增加 TTL 为一个小时的队列，如果是预定会议室，然后提前通知这样的场景，岂不是要增加无数个队列才能满足需求。

● 消息TTL+死信
使用消息属性上设置 TTL 的方式，消息可能并不会按时 “死亡” ，因为RabbitMQ 只会检查第一个消息是否过期，如果过期则丢到死信队列，如果第一个消息的延时时长很长，而第二个消息的延时时长很短，第二个消息并不会优先得到执行 。

● rabbitmq-delayed-message-exchange 插件
延时的消息，可以不走队列，而是通过交换机本身进行一个处理，让消费者直接从交换机当中消费，读取消息。消息发送后不会直接投递到队列，而是存储到 Mnesia（嵌入式数据库），检查 x-delay 时间（消息头部）；延迟插件在 RabbitMQ 3.5.7 及以上的版本才支持，依赖 Erlang/OPT 18.0 及以上运行环境；Mnesia 是一个小型数据库，不适合于大量延迟消息的实现，解决了消息过期时间不一致(优先级)出现的问题。
● 更多实现
延时队列还有很多其它选择，比如利用 Java 的 DelayQueue，利用 Redis的 zset，利用 Quartz 或者利用 kafka 的时间轮，这些方式各有特点，看需要适用的场景。

# Java使用rabbitMQ实现延时队列
延迟队列配置类
```java
package com.example.rabbitmqdemo.config;

import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.CustomExchange;
import org.springframework.amqp.core.Queue;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.HashMap;
import java.util.Map;

@Configuration
public class DelayedQueueConfig {
    // 定义延迟队列的名称
    public static final String DELAYED_QUEUE_NAME = "delayed.queue";
    // 定义延迟交换机的名称
    public static final String DELAYED_EXCHANGE_NAME = "delayed.exchange";
    // 定义路由键
    public static final String DELAYED_ROUTING_KEY = "delayed.routing.key";

    /**
     * 创建延迟队列。
     * @return 创建的延迟队列实例。
     */
    @Bean
    public Queue delayedQueue() {
        return new Queue(DELAYED_QUEUE_NAME);
    }

    /**
     * 自定义延迟交换机。
     * 使用 CustomExchange 类型来创建，这是因为我们需要一个支持延迟消息的交换机。
     * @return 创建的自定义延迟交换机实例。
     */
    @Bean
    public CustomExchange delayedExchange() {
        Map<String, Object> args = new HashMap<>();
        // 设置延迟交换机的类型为 direct
        args.put("x-delayed-type", "direct");
        // 创建一个类型为 "x-delayed-message" 的交换机，这个类型是插件提供的，专门用于处理延迟消息
        CustomExchange delayedExchange =  new CustomExchange
                (DELAYED_EXCHANGE_NAME, "x-delayed-message", true, false, args);
        delayedExchange.setDelayed(true);
        return delayedExchange;
    }

    /**
     * 将延迟队列绑定到延迟交换机。
     * @param queue 延迟队列
     * @param delayedExchange 自定义的延迟交换机
     * @return 创建的绑定关系实例。
     */
    @Bean
    public Binding bindingDelayedQueue(@Qualifier("delayedQueue") Queue queue, @Qualifier("delayedExchange") CustomExchange delayedExchange) {
        // 绑定延迟队列与延迟交换机，并设置使用的路由键
        return BindingBuilder.bind(queue).to(delayedExchange).with(DELAYED_ROUTING_KEY).noargs();
    }
}
```

延迟队列消费者
```java
package com.example.rabbitmqdemo.consumer;

import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

import java.util.Date;

/**
 * 延迟队列消费者类
 */
@Slf4j
@Component
public class DelayQueueConsumer {

    // 延迟队列名称常量
    public static final String DELAYED_QUEUE_NAME = "delayed.queue";

    /**
     * 监听延迟队列。
     * @param message 接收到的消息体
     */
    @RabbitListener(queues = DELAYED_QUEUE_NAME)
    public void receiveDelayedQueue(Message message) {
        String msg = new String(message.getBody()); // 将消息体转换为字符串
        log.info("当前时间：{}, 收到延时队列的消息：{}", new Date(), msg); // 记录日志
    }
}
```

延迟队列生产者
```java
package com.feilala.boot.common.delayQueue.producer;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.stereotype.Component;

import java.util.concurrent.TimeUnit;

import static com.feilala.boot.config.DelayedQueueConfig.*;
import static com.feilala.boot.module.orders.enums.OrdersEnum.*;

/**
 * 延迟队列生产者类
 */
@Component
@RequiredArgsConstructor
@Slf4j
public class DelayQueueProducer {
    private final RabbitTemplate rabbitTemplate;

    /**
     * 发送延迟消息 - 支持各种时间单位
     */
    public void sendDelayedMessage(String message, String messageId,long delay, TimeUnit timeUnit) {
        long delayMillis = timeUnit.toMillis(delay);
        log.info("发送延迟消息: {}, 延迟: {}{} ({}ms)",
                message, delay, getTimeUnitName(timeUnit), delayMillis);

        rabbitTemplate.convertAndSend(
                DELAYED_EXCHANGE_NAME,
                DELAYED_ROUTING_KEY,
                message,
                msg -> {
                    msg.getMessageProperties().setDelayLong(delayMillis);
                    msg.getMessageProperties().setMessageId(messageId);
                    return msg;
                }
        );
    }


    /**
     * 发送订单过期消息 - 分钟级延迟
     */
    public void sendOrderExpireMessage(String orderNo, long delayMinutes) {
        String message = SHOP_ORDER_EXPIRE + ":" + orderNo;
        sendDelayedMessage(message, orderNo,delayMinutes, TimeUnit.MINUTES);
    }

    /**
     * 发送租赁订单到期消息 - 秒级延迟
     */
    public void sendRentalOrderExpireMessage(String orderNo, long delaySeconds) {
        String message = RENTAL_ORDER_EXPIRE + ":" + orderNo;
        sendDelayedMessage(message,orderNo, delaySeconds, TimeUnit.SECONDS);
    }


    private String getTimeUnitName(TimeUnit timeUnit) {
        return switch (timeUnit) {
            case SECONDS -> "秒";
            case MINUTES -> "分钟";
            case HOURS -> "小时";
            case DAYS -> "天";
            default -> "毫秒";
        };
    }
}
```

生产者接口
```java
package com.example.rabbitmqdemo.controller;

import com.rabbitmq.client.GetResponse;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.amqp.rabbit.core.RabbitTemplate;

import java.time.LocalDate;

import lombok.extern.slf4j.Slf4j;

import static com.example.rabbitmqdemo.config.DelayedQueueConfig.*;

/**
 * 演示如何通过Spring Boot应用的Controller层向RabbitMQ发送消息。
 */
@RestController
@RequestMapping("/ttl")
@Slf4j
public class SendMsgController {

    @Autowired // 自动注入RabbitTemplate
    private RabbitTemplate rabbitTemplate;

    /**
     * 发送带有延迟时间的消息到延迟队列。
     * @param message 消息内容
     * @param delayTime 延迟时间(毫秒)
     */
    @GetMapping("sendDelayMsg")
    public void sendMsg(String message, Long delayTime) {
        String messageId = "test";
        // 发送消息
        rabbitTemplate.convertAndSend(DELAYED_EXCHANGE_NAME, DELAYED_ROUTING_KEY, message,
                correlationData -> {
                    // 设置消息的延迟时间
                    correlationData.getMessageProperties().setDelayLong(delayTime);
                    correlationData.getMessageProperties().setMessageId(messageId);
                    return correlationData; // 返回配置后的消息
                });
        log.info(" 当 前 时 间 ： {}, 发 送 一 条 延 迟 {} 毫秒的信息给队列 delayed.queue:{},MessageId：{}", LocalDate.now(), delayTime, message,messageId);
    }
    /**
     * 删除消息。
     * @param targetMessage 消息
     */
    @GetMapping("remove/{targetMessage}")
    public void removeMsg(@PathVariable String targetMessage) {
        log.info("删除延迟消息: {}", targetMessage);

        rabbitTemplate.execute(channel -> {
            GetResponse response = channel.basicGet(DELAYED_QUEUE_NAME, false);
            if (response != null) {
                String message = new String(response.getBody());
                if (message.equals(targetMessage)) {
                    channel.basicAck(response.getEnvelope().getDeliveryTag(), false);
                    log.info("消息已删除: {}", targetMessage);
                } else {
                    channel.basicNack(response.getEnvelope().getDeliveryTag(), false, true);
                }
            }
            return null;
        });
    }
}
```