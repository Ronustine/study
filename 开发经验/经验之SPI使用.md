[TOC]

## 需求
公司钉钉迁移飞书，所有项目通知都要转移。在别人已经写好的飞书发送工具包上做修改，不再依赖特定的钉钉或者飞书sdk，而是依赖Alarm包，通过SPI的可插拔，实现Alarm接口后，可快速切换；

## 代码
SPI关键：在目录resource/META-INF/services/com.ronustine.common.alarm.Alarm 写入 Alarm的实现类`com.ronustine.common.alarm.LarkAlarm`
写了那么ServiceLoader.load(Alarm.class)的时候就可以默认装配了一个可用的告警。
```java

@Slf4j
@Configuration
@ConditionalOnProperty(prefix = "ronustine.common.alarm", name = "enabled", havingValue = "true")
// @ComponentScan(value = "com.ronustine.common.alarm")
public class AlarmConfiguration {

    @Value("${ronustine.common.alarm.type:}")
    private String alarmType;

    @Bean
    public Alarm ronustineAlarm() {
        ServiceLoader<Alarm> alarmList = ServiceLoader.load(Alarm.class);
        alarmList.forEach(alarm -> log.info("可用的告警[{}]", alarm.alarmInfo()));

        for (Alarm a : alarmList) {
            if (StringUtils.isEmpty(this.alarmType)) {
                log.info("默认使用第一个告警[{}]", a.alarmInfo());
                return a;
            }
            if (this.alarmType.equalsIgnoreCase(a.alarmType().name())) {
                log.info("指定使用告警[{}]", a.alarmInfo());
                return a;
            }

        }
        throw new RuntimeException("未找到可用的告警实现");
    }
}

// 接口
public interface Alarm {

    // 必实现
	AlarmEnum alarmType();
    // 必实现
	String alarmInfo();

	void sendMessage(MessageCmd messageCmd);

	void sendMarkdown(MdMessageCmd messageCmd);

	void sendCard();

}


// 枚举
@Getter
@AllArgsConstructor
public enum AlarmEnum {

    /**
     * 预警类型
     */
    DINGTALK(1, "钉钉 webhook"),
    LARK(2, "飞书 webhook"),
    ORTHERS(3, "其他 webhook"),
    ;

    private Integer code;
    private String desc;

}

// 提供一个默认实现飞书的告警
public class LarkAlarm implements Alarm{

    LarkRobotApi larkRobotApi = new LarkRobotApi();

    @Override
    public AlarmEnum alarmType() {
        return AlarmEnum.LARK;
    }

    @Override
    public String alarmInfo() {
        return alarmType().getDesc() + "告警实现";
    }

    @Override
    public void sendMessage(MessageCmd messageCmd) {
        LarkMessage textMessage = LarkMessage.textBuilder().text(messageCmd.getText())
            .atUserIds(messageCmd.getAtList()).isAtAll(messageCmd.getAtAll()).build();
        larkRobotApi.sendLarkMessage(messageCmd.getWebhook(), textMessage);
    }

    @Override
    public void sendMarkdown(MdMessageCmd messageCmd) {
        MarkdownElement markdown = MarkdownElement.builder()
            .content(messageCmd.getText())
            .atUserIds(messageCmd.getAtList())
            .isAtAll(messageCmd.getAtAll())
            .build();

        List<BaseCardElement> elements = new ArrayList<>();
        elements.add(markdown);

        LarkMessage cardMessage = LarkMessage.cardBuilder()
            .header(CardHeader.builder().title(messageCmd.getTitle()).template("green").builder())
            .elements(elements).build();
        larkRobotApi.sendLarkMessage(messageCmd.getWebhook(), cardMessage);
    }

    @Override
    public void sendCard() {
        throw new UnsupportedOperationException("暂不支持");
    }
}
```

## 说明

技术实现：SPI
包中自带实现是飞书（markdown语法支持参考（较少，不好用）
效果：

- Spring容器会默认装载LarkAlarm，即Alarm的实现

使用：

1. maven依赖
```xml
<dependency>
    <groupId>com.ronustine</groupId>
    <artifactId>ronustine-common-alarm</artifactId>
    <version>${revision}</version>
</dependency>
```

2. application.properties
- （必须配置）ronustine.common.alarm.enabled=true
- （可不指定，默认lark，参考AlarmEnum，大小写不敏感）ronustine.common.alarm.type=lark

3. 代码
```java
    // spring 构造器注入
    private final Alarm ronustineAlarm;
```

维护：
基于SPI，关键是这个路径/resources/META-INF/services，以及文件名com.ronustine.common.alarm.Alarm，内容写好实现类
