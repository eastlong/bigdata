# Flink读取kafka数据写入ES

## 基础环境搭建

### kafka安装

> 简单安装：https://blog.csdn.net/hbtj_1216/article/details/104100389
>
> 集群安装：https://dandelioncloud.cn/article/details/1596443458573385729

### ES安装

利用docker部署ES

```bash
docker pull elasticsearch:7.17.5
docker pull kibana:7.17.5

# 将docker里的目录挂载到linux的/mydata目录中
# 修改/mydata就可以改掉docker里的
mkdir -p /mydata/elasticsearch/config
mkdir -p /mydata/elasticsearch/data

# es可以被远程任何机器访问
# echo "http.host: 0.0.0.0" >/mydata/elasticsearch/config/elasticsearch.yml
# echo "xpack.security.enabled: false" >/mydata/elasticsearch/config/elasticsearch.yml
# 上述写法会被覆盖 elasticsearch.yml配置如下内容
http.host: 0.0.0.0
xpack.security.enabled: false

# 递归更改权限，es需要访问
chmod -R 777 /mydata/elasticsearch/

# 【启动ES】
# 9200是用户交互端口 9300是集群心跳端口
# -e指定是单阶段运行
# -e指定占用的内存大小，生产时可以设置32G
docker run --name elasticsearch -p 9200:9200 -p 9300:9300 \
-e  "discovery.type=single-node" \
-e ES_JAVA_OPTS="-Xms64m -Xmx512m" \
-v /mydata/elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml \
-v /mydata/elasticsearch/data:/usr/share/elasticsearch/data \
-v  /mydata/elasticsearch/plugins:/usr/share/elasticsearch/plugins \
-d elasticsearch:7.17.5 

# 设置开机启动elasticsearch
docker update elasticsearch --restart=always

# 【启动kibana】
# kibana指定了了ES交互端口9200  # 5600位kibana主页端口
docker run --name kibana -e ELASTICSEARCH_HOSTS=http://192.168.56.10:9200 -p 5601:5601 -d kibana:7.4.2


# 设置开机启动kibana
docker update kibana  --restart=always
```



## Flink读取kafka数据并sink2ES

### 引入依赖

```xml
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-connector-elasticsearch7</artifactId>
            <version>3.0.1-1.17</version>
        </dependency>
```

### 参考程序

```java
import org.apache.flink.api.common.eventtime.WatermarkStrategy;
import org.apache.flink.api.common.serialization.SimpleStringSchema;
import org.apache.flink.client.program.StreamContextEnvironment;
import org.apache.flink.connector.elasticsearch.sink.Elasticsearch7SinkBuilder;
import org.apache.flink.connector.kafka.source.KafkaSource;
import org.apache.flink.connector.kafka.source.enumerator.initializer.OffsetsInitializer;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.http.HttpHost;
import org.elasticsearch.action.index.IndexRequest;
import org.elasticsearch.client.Requests;
import org.elasticsearch.common.xcontent.XContentType;

import java.util.HashMap;
import java.util.Map;

/**
 * @Description:
 * @Date 2023/09/03 14:55:00
 **/
public class KafkaSinkES {
    public static final String KAFKA_TOPIC = "test";

    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamContextEnvironment.getExecutionEnvironment();
        KafkaSource<String> kafkaSource = KafkaSource.<String>builder()
                .setBootstrapServers("192.168.56.101:9092")
                .setTopics(KAFKA_TOPIC)
                .setGroupId("consumer_group1")
                .setStartingOffsets(OffsetsInitializer.latest())
                .setValueOnlyDeserializer(new SimpleStringSchema())
                .build();
        DataStream<String> kafkaStream = env.fromSource(kafkaSource, WatermarkStrategy.noWatermarks(), "kafka-source")
                .name(KAFKA_TOPIC + "-source")
                .uid(KAFKA_TOPIC + "-source")
                .setParallelism(1);

        kafkaStream.sinkTo(new Elasticsearch7SinkBuilder<String>()
                .setBulkFlushMaxActions(1)
                // .setBulkFlushInterval(100)
                // .setBulkFlushMaxSizeMb(1)
                .setHosts(new HttpHost("192.168.56.101", 9200, "http"))
                .setEmitter((element, context, indexer) -> indexer.add(createIndexRequest(element)))
                .build()
        );

        env.execute("kafka sink to ES");
    }

    private static IndexRequest createIndexRequest(String element) {
        return Requests.indexRequest()
                .index("student-test")
                //.id(element)
                .source(element, XContentType.JSON);
    }
}
```

### 测试

1、ES创建索引

```bash
PUT /student-test
```

2、启动kafka并发送消息

`{"age":18,"name":"Jack"}`

3、启动程序

4、查询ES

```json
GET /student-test/_search
{
  "query": {
    "match_all": {}
  }
}

// ================== 结果 ========================
{
  "took" : 436,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "student-test",
        "_type" : "_doc",
        "_id" : "WCLtWYoBqtnlsjTCwqca",
        "_score" : 1.0,
        "_source" : {
          "age" : 18,
          "name" : "Jack"
        }
      }
    ]
  }
}
```



参考程序：

https://nightlies.apache.org/flink/flink-docs-release-1.17/zh/docs/connectors/datastream/elasticsearch/

