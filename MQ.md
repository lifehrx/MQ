## 使用场景

消息队列的常见使用场景吧，其实场景有很多，但是比较核心的有3个：解耦、异步、削峰

## MQ 的技术选型

| ActiveMQ                                                     | RabbitMQ                                                     | RocketMQ                                                     | Kafka                                                        |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 万级                                                         | 万级                                                         | 十万级                                                       | 十万级                                                       |
| 不推荐<br />官方社区维护的越来越少了，偶尔丢消息（外围改源码）版本更新慢，数据量大别用。 | 中小型企业 <br />Elang 语言开发<br />界面化好。直接作为监控。社区活跃，一个月几个版本，快速的修复bug。 | 阿里开发<br />java 语言开发<br />活跃度比Rabbitg高<br />但是国内的技术，万一项目被抛弃了，team解散，东西没人维护，中小型企业这是灾难。 | 大数据，分布式<br />一般配合大数据类的系统来进行实时数据计算、日志采集等场。<br />用Kafka是业内标准的，绝对没问题，社区活跃度很高，绝对不会黄，何况几乎是全世界这个领域的事实性规范。 |



## SpringBoot  集成 RabbitMQ



#### step1 :  安装RabbitMQ 

#### step2:   引入依赖在pom.xml

```xml
  <!-- MQ -->
   <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-amqp</artifactId>
   </dependency>
```

#### step3:  在application.yml添加配置

```yml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
```



#### step4 ： 应用实例

```java
// 生产者 ： 将仓库更新内容消息推送到 MQ 中

public class HandleMessageServiceImpl {
    
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    public void sendMsgToMq(String msg) {
        rabbitTemplate.convertAndSend("directExchange","ppd.upload", msg);
    }
}
```



```java
// 消费者： 消费仓库推送的更新信息
@Component
public class Receiver {

    private static final Logger LOGGER = LoggerFactory.getLogger(Receiver.class);

    @Resource
    private PmDataSyncServiceImpl pmDataSyncService;

    @Resource
    private ESProxy esProxy;

    @RabbitListener(bindings = @QueueBinding(
            value = @Queue(value = "AIMSSENDMESSAGE", autoDelete = "true"),
            exchange = @Exchange(value = "AIMS", type = ExchangeTypes.DIRECT),
            key = "REPOSITORY"
    ))

    @RabbitHandler
    public void process(final String str) {

        try {
            final MqMessage mm = new ObjectMapper().readValue(str, new TypeReference<MqMessage>() { });
            if (mm.getRepositoryStatus().equals("UPDATED")) {
                mm.setRepositoryStatus("updateRepository");
                // 1.获取变更的仓库ID 和名称
                String updateRepositoryId = mm.getRepository().getId();
                mm.getRepository().getName();

                // 2. 根据所更新仓库的Id 在ES中查询回数据
                Map<String, List<ESPlantObject>> map = esProxy.selectAllObjectByRepositoryId(updateRepositoryId);
                // 3. 查询回的数据插入数据库
                pmDataSyncService.syncData(map);
            }

        } catch (IOException e) {
            e.printStackTrace();
        }

        LOGGER.info("接收 AIMS 仓库更新：" + str);
        System.out.println("接收 AIMS 仓库更新 : " + str);
    }


    @Getter
    @Setter
    @JsonIgnoreProperties(ignoreUnknown = true)
    public static class MqMessage {

        private String userId;

        private String time;

        private String repositoryStatus;

        private Repository repository;

    }

    @Getter @Setter
    @JsonIgnoreProperties(ignoreUnknown = true)
    public static class Repository {

        private String id;

        private String name;

        private List<String> groupIds;
    }
}

```

