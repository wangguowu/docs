## maven
    <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-amqp</artifactId>
            <!-- <version>2.6.6</version> -->
    </dependency>

## yml
    rabbitmq:
        host: xxx
        port: 5672
        username: xxx
        password: xxx
        virtual-host: tiku_test
        listener:
            simple:
                acknowledge-mode: manual

## 声明Queue
      @Configuration
      public class RabbitConfig {
          Queue queue1(){
              return new Queue("simple.queue");
          }
      }

* exclusive: false, //队列是否专属
* durable：是否持久化，RabbitMQ关闭后，没有持久化的Exchange将被清除
* autoDelete：是否自动删除，如果没有与之绑定的Queue，直接删除
* internal：是否内置的，如果为true，只能通过Exchange到Exchange
* arguments：结构化参数

## send
    @ActiveProfiles("dev")
    @RunWith(SpringRunner.class)
    @SpringBootTest
    public class ProducerTest {

        @Autowired
        private RabbitTemplate rabbitTemplate;

        @Test
        public void testSendMessage2SimpleQueue() {
            String queueName = "simple.queue";
            for (int i = 5; i<8; i++) {
                Course course = new Course();
                course.setTitle("222活动22活动d"+i);
                Message build = MessageBuilder.withBody(JSON.toJSONString(course).getBytes(StandardCharsets.UTF_8))
                        .setMessageId(IdUtil.simpleUUID())
                        .setContentType(MessageProperties.CONTENT_TYPE_JSON)
                        .setContentEncoding(StandardCharsets.UTF_8.name())
                        .build();
                rabbitTemplate.convertAndSend(queueName, build);
            }
        }
    }
## receive
    @Component
    public class CertDataListener {

        @RabbitListener(queues = "simple.queue")
        public void listenData(Message msg, String data, String data2, Channel channel, @Header(AmqpHeaders.MESSAGE_ID) String msgId, @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws Exception {
    
            System.out.println("消费者msg = " + JSON.toJSONString(msg));
            System.out.println("消费者data = " + data);
            System.out.println("消费者data2 = " + data2);
            System.out.println("消费者msgId = " + msgId);
    
            // System.out.println("消费者接收到"+JSON.toJSONString(course));
            // System.out.println("消费者接收到direct.queue1的消息：【" + JSON.toJSONString(course) + "】");
            // System.out.println("tag = " +tag);
            channel.basicAck(tag, false);
            // channel.basicNack(tag,false,true);
        }
    
        /*@RabbitListener(bindings = @QueueBinding(
                value = @Queue(name = "exchange_direct_tiku_cert_push"),
                exchange = @Exchange(name = "tiku.cert.direct", type = ExchangeTypes.DIRECT),
                key = {"certTestPaper", "certQuestion"}
        ))
        public void listenDirectQueue1(String msg){
            System.out.println("消费者接收到direct.queue1的消息：【" + msg + "】");
        }*/
    }
## ack机制
+ @Header(AmqpHeaders.DELIVERY_TAG) tag 取出来当前消息在队列中的的索引
+ multiple:为true的话就是批量确认
+ channel.basicAck(tag, false);
  ### ack应答
1. 拒绝，有异常就绝收消息
   > requeue:true为将消息重返当前消息队列,还可以重新发送给消费者; false:将消息丢弃

   > basicNack(long deliveryTag, boolean multiple, boolean requeue)

   >> channel.basicNack(tag,false,true)
   >
   > > mq每60秒重试一次，总共16次
2. 响应成功
   > channel.basicAck(tag, false);
3. 不响应
   > mq每60秒重试一次，总共16次