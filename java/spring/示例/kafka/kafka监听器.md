```java
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.support.Acknowledgment;

@Component
public class DataAsyncKafkaListener {
    // 可以直接从配置文件读取，也可以写成常量
    @KafkaListener(topics = "${data.async.land-mass-topic}")
    public void listenLandMassDataAsync(ConsumerRecord<String, Data> record, Acknowledgment ack) {
        //...
        ack.acknowledge();
    }
}
```