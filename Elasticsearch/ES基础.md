# 一、ES安装

## 1.1 Docker部署单机ES和Kibana

1、利用docker部署ES和kibana

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

2、测试安装是否成功

- 浏览器输入： http://192.168.56.10:9200 查看ES版本信息
- 显示elasticsearch 节点信息：http://192.168.56.10:9200/_cat/nodes
- 访问Kibana： http://192.168.56.10:5601/app/kibana

**kibana启动比较慢**

```Bash
# 查看集群健康状态
GET /_cat/health?v
# 查看节点健康状态
GET /_cat/nodes?v
# 查看索引信息
GET /_cat/indices?v
```



## Cenatos下单机ES安装

> https://blog.csdn.net/weixin_40816738/article/details/120426766



# 二、基本操作

## 2.1 操作索引

### 2.1.1 索引创建

1、创建

`PUT /索引名`

```json
PUT /products

{
  "acknowledged" : true,
  "shards_acknowledged" : true,
  "index" : "products"
}
```

2、创建索引 进行**索引分片配置**

```json
DELETE /products
PUT /products
{
  "settings": {
    "number_of_shards": 1, 
    "number_of_replicas": 0
  }
}

GET /_cat/indices
```



### 查询

- `GET /_cat/indices?v`

![img](image/微信截图_20220409094046.png)

### 删除

```
DELETE /shopping2

DELETE /*     `*代表通配符,代表所有索引`
```



## 2.2 映射 mapping

### 创建

字符串类型: keyword 关键字 关键词 、`text` 一段文本

数字类型：integer long

小数类型：float double

布尔类型：boolean

日期类型：`date` ,默认三种格式

- 2018-01-13” 或 “2018-01-13 12:10:30”
- long类型的毫秒数
- integer的秒数

复合类型:

- 数组类型 array
- 对象类型 object ：JSON格式对象数据
- 嵌套类型 `nested`
- 地理类型 地理坐标类型 geo_point
- 地理地图 geo_shape
- 特殊类型 IP类型 ip
- 范围类型 completion
- 令牌计数类型 token_count
- 附件类型 attachment
- 抽取类型 percolator

```json
PUT /products2
{ 
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  }, 
  "mappings": {
    "properties": {
      "title":{
        "type": "keyword"
      },
      "price":{
        "type": "double"
      },
      "cart_id":{
        "type": "date"
      },
      "description":{
        "type": "long"
      },
      "cart_info" : {
          "type" : "nested",
          "properties" : {
            "bargainId" : {
              "type" : "long"
            },
            "cartNum" : {
              "type" : "integer"
            }
          }
      },
      "store_name" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
        "createtime" : {
          "format" : "yyyy-MM-dd'T'HH:mm:ss.SSS'Z'||yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis",
          "index" : true,
          "type" : "date",
          "doc_values" : true
        }
    }
  }
}
```

> 说明
>
> 说明: ES中支持字段类型非常丰富，如：text、keyword、integer、long、ip 等。更多参见https://www.elastic.co/guide/en/elasticsearch/reference/7.15/mapping-types.html

### 查询

```
GET /products2/_mapping
```



### 索引数据复制

解决,源索引映射不符合目前需求,需要替换源索引,`改为新的映射关系`

```json
POST _reindex
{
  "source": {
    "index": "products2"
  }
  , "dest": {"index":  "products3"}
}
```



## 2.3 文档

### 添加文档

es默认会为es添加id,格式为uuid

```
PUT /products/_doc/3
{
  "title":"iphone14",
  "price":99.9,
  "create_time":"2022-04-04",
  "description":"iPhone 14屏幕采用6.1英寸OLED屏幕。"
}
```

如果不指定id，**必须用POST请求**

```json

POST /products/_doc/
{
  "title":"iphone14",
  "price":99.9,
  "create_time":"2022-04-04",
  "description":"iPhone 14屏幕采用6.1英寸OLED屏幕。"
}
```

### 查询文档

```
GET /products/_doc/3

{
  "_index" : "products",
  "_type" : "_doc",
  "_id" : "3",
  "_version" : 1,
  "_seq_no" : 0,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "title" : "iphone14",
    "price" : 99.9,
    "create_time" : "2022-04-04",
    "description" : "iPhone 14屏幕采用6.1英寸OLED屏幕。"
  }
}
```

### 删除文档

```json
DELETE /products/_doc/1
```



### 更新文档

1、彻底更新

- PUT请求

```json
PUT /products/_doc/3
{
  "title":"iphone15"
}
GET /products/_doc/3
```

> 说明: 这种更新方式是先删除原始文档,在将更新文档以新的内容插入。

2、保留更新 

 这种方式可以将数据原始内容保存,并在此基础上更新。相当于**局部更新**。

```json
POST /products/_update/3
{
  "doc" : {
    "title" : "iphon15"
  }
}
GET /products/_doc/3
```

## 2.4 高级查询

### 说明		

ES中提供了一种强大的检索数据方式,这种检索方式称之为`Query DSL<Domain Specified Language>`,Query DSL是利用Rest API传递JSON格式的请求体(Request Body)数据与ES进行交互，这种方式的丰富查询语法让ES检索变得更强大，更简洁。

### 语法

```json
 GET /索引名/_search {json格式请求体数据}
```



准备数据

```json
PUT /products/_doc/1
{"title":"iphone14","price":8999.99,"created_at":"2021-09-14","description":"iPhone 14屏幕采用6.8英寸OLED屏幕"}

PUT /products/_doc/2
{"title":"iphone13","price":89.99,"created_at":"2021-09-13","description":"iPhone 13屏幕采用6.8英寸OLED屏幕"}

PUT /products/_doc/3
{"title":"iphone15","price":899.99,"created_at":"2021-09-12","description":"iPhone 15屏幕采用6.8英寸OLED屏幕"}

PUT /products/_doc/4
{"title":"联想笔记本电脑","price":6888,"created_at":"2022-08-13","description":"24G运行内存，256G机械硬盘"}

PUT /products/_doc/5
{"title":"华为平板","price":4668,"created_at":"2023-01-21","description":"集成手机和电脑的功能"}
```







### 查询所有 match_all

```json
GET /products/_search
{
  "query": {
    "match_all": {}
  }
}
```



### 关键词不分词查询

> term
>
> term查询text字段,因为text字段会分词，而term不分词，所以term查询的条件必须是text字段分词后的某一个。 term查询keyword字段。 term不会分词。而keyword字段也不分词。需要完全匹配才可。

```json
GET /products/_search
{
 "query": {
   "term": {
     "price": {
       "value": 899.99
     }
   }
 }
}
```

### 关键字分词查询(match)

> match查询text字段,match分词，text也分词，只要match的分词结果和text的分词结果有相同的就匹配。 match查询keyword字段,match会被分词，而keyword不会被分词，match的需要跟keyword的完全匹配可以。

```json
GET /products/_search
{
  "query": {
    "match": {
      "description": "功能"
    }
  }
}
```



### in查询(terms)

> in
>
> 类似 select * from user where id in (1,2,4,5);

```json
PUT /student/_doc/1
{"id":1, "name":"张三", "age": 21, "group":1, "address":"上海市徐汇区"}
PUT /student/_doc/2
{"id":2, "name":"李四", "age": 21, "group":2, "address":"江西省上饶市"}
PUT /student/_doc/3
{"id":3, "name":"王五", "age": 21, "group":3, "address":"湖北省武汉市"}
PUT /student/_doc/4
{"id":4, "name":"马六", "age": 21, "group":4, "address":"上海市徐汇区"}

GET /student/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "terms": {
            "group": [1,2]
          }
        }
      ]
    }
  }
}
```



### 字段是否存在[exists,missing]



```json
GET /student/_search
{
  "query": {
    "exists": {
      "field": "name"
    }
  }
}

# missing 作用是缺失值替代，也就是加了这个关键字，并且带上参数以后，
# 在查询计算中如果查询的字段值缺失，那么就会默认使用到 missing 这个词。用在分组
```



### 对象类型查询(nested)



```json
PUT /test_yx_store_order_cart_info
 {
    "mappings" : {
      "properties" : {
        "cart_id" : {
          "type" : "long"
        },
        "cart_info" : {
          "type" : "nested",
          "properties" : {
            "bargainId" : {
              "type" : "long"
            },
            "cartNum" : {
              "type" : "integer"
            },
            "combinationId" : {
              "type" : "long"
            },
            "costPrice" : {
              "type" : "float"
            },
            "createTime" : {
              "type" : "date"
            },
            "originInfo" : {
              "type" : "nested",
              "properties" : {
                "itemImage" : {
                  "type" : "text",
                  "fields" : {
                    "keyword" : {
                      "type" : "keyword",
                      "ignore_above" : 256
                    }
                  }
                },
                "itemName" : {
                  "type" : "text",
                  "fields" : {
                    "keyword" : {
                      "type" : "keyword",
                      "ignore_above" : 256
                    }
                  }
                }
              }
            },
            "productAttrUnique" : {
              "type" : "text",
              "fields" : {
                "keyword" : {
                  "type" : "keyword",
                  "ignore_above" : 256
                }
              }
            },
            "productId" : {
              "type" : "integer"
            },
            "productInfo" : {
              "properties" : {
                "attrInfo" : {
                  "properties" : {
                    "barCode" : {
                      "type" : "text",
                      "fields" : {
                        "keyword" : {
                          "type" : "keyword",
                          "ignore_above" : 256
                        }
                      }
                    },
                    "image" : {
                      "type" : "text",
                      "fields" : {
                        "keyword" : {
                          "type" : "keyword",
                          "ignore_above" : 256
                        }
                      }
                    },
                    "sku" : {
                      "type" : "text",
                      "fields" : {
                        "keyword" : {
                          "type" : "keyword",
                          "ignore_above" : 256
                        }
                      }
                    },
                    "stock" : {
                      "type" : "long"
                    },
                    "unique" : {
                      "type" : "text",
                      "fields" : {
                        "keyword" : {
                          "type" : "keyword",
                          "ignore_above" : 256
                        }
                      }
                    },
                    "weight" : {
                      "type" : "float"
                    }
                  }
                },
                "browse" : {
                  "type" : "long"
                },
                "cateId" : {
                  "type" : "text",
                  "fields" : {
                    "keyword" : {
                      "type" : "keyword",
                      "ignore_above" : 256
                    }
                  }
                },
                "cost" : {
                  "type" : "float"
                },
                "description" : {
                  "type" : "text",
                  "fields" : {
                    "keyword" : {
                      "type" : "keyword",
                      "ignore_above" : 256
                    }
                  }
                },
                "image" : {
                  "type" : "text",
                  "fields" : {
                    "keyword" : {
                      "type" : "keyword",
                      "ignore_above" : 256
                    }
                  }
                },
                "image_base" : {
                  "type" : "text",
                  "fields" : {
                    "keyword" : {
                      "type" : "keyword",
                      "ignore_above" : 256
                    }
                  }
                },
                "keyword" : {
                  "type" : "text",
                  "fields" : {
                    "keyword" : {
                      "type" : "keyword",
                      "ignore_above" : 256
                    }
                  }
                },
                "serviceRule" : {
                  "type" : "text",
                  "fields" : {
                    "keyword" : {
                      "type" : "keyword",
                      "ignore_above" : 256
                    }
                  }
                },
                "shopService" : {
                  "type" : "long"
                },
                "sliderImage" : {
                  "type" : "text",
                  "fields" : {
                    "keyword" : {
                      "type" : "keyword",
                      "ignore_above" : 256
                    }
                  }
                },
                "sliderImageArr" : {
                  "type" : "text",
                  "fields" : {
                    "keyword" : {
                      "type" : "keyword",
                      "ignore_above" : 256
                    }
                  }
                },
                "taxRate" : {
                  "type" : "text",
                  "fields" : {
                    "keyword" : {
                      "type" : "keyword",
                      "ignore_above" : 256
                    }
                  }
                },
                "tempId" : {
                  "type" : "long"
                },
                "unitName" : {
                  "type" : "text",
                  "fields" : {
                    "keyword" : {
                      "type" : "keyword",
                      "ignore_above" : 256
                    }
                  }
                },
                "userCollect" : {
                  "type" : "boolean"
                },
                "userLike" : {
                  "type" : "boolean"
                },
                "vipPrice" : {
                  "type" : "float"
                }
              }
            },
            "seckillId" : {
              "type" : "long"
            },
            "truePrice" : {
              "type" : "float"
            },
            "trueStock" : {
              "type" : "long"
            },
            "type" : {
              "type" : "text",
              "fields" : {
                "keyword" : {
                  "type" : "keyword",
                  "ignore_above" : 256
                }
              }
            },
            "uid" : {
              "type" : "integer"
            },
            "updateTime" : {
              "type" : "long"
            },
            "vipTruePrice" : {
              "type" : "float"
            }
          }
        },
        "id" : {
          "type" : "long"
        },
        "oid" : {
          "type" : "long"
        },
        "product_id" : {
          "type" : "long"
        },
        "store_name" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          },
          "analyzer" : "standard"
        },
        "unique" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          },
          "analyzer" : "standard"
        }
      }
    }
  }
```

查询如下：

```json
GET /test_yx_store_order_cart_info/_search
{
  "query": {
    "nested": {
      "path": "cart_info.originInfo",
      "query": {
        "match": {
          "cart_info.originInfo.itemName": "洗漱"
        }
      }
    }
  }
}
```





### 关键字分词查询(match_phrase)

> match_phrase
>
> match_phrase匹配text字段,match_phrase的分词结果必须在text字段分词中都包含，而且顺序必须相同，而且必须都是连续的。 match_phrase匹配keyword字段,match_phrase会被分词，而keyword不会被分词，match_phrase的需要跟keyword的完全匹配才可以。



```json
GET /student/_search
{
  "query": {
    "match_phrase": {
      "store_name": "徐汇"
    }
  }
}
```



### 范围查询[range]

> rang
>
> 用来指定查询指定范围内的文档

```json
GET /products/_search
{
  "query": {
    "range": {
      "price": {
        "gte": 100,
        "lte": 1000
      }
    }
  }
}
```



###  前缀查询[prefix]

> prefix
>
> 用来检索含有指定前缀的关键词的相关文档

```json
GET /products/_search
{
  "query": {
    "prefix": {
      "title": {
        "value": "ipho"
      }
    }
  }
}
```

###  通配符查询[wildcard]

> wildcard
>
> 通配符查询 **? 用来匹配一个任意字符 \* 用来匹配多个任意字符**

```json
GET /products/_search
{
  "query": {
    "wildcard": {
      "description": {
        "value": "ipho*"
      }
    }
  }
}
```



### 多id查询

> ids
>
> 值为数组类型,用来根据一组id获取多个对应的文档;
>
> id指的是 _doc

```json
GET /products/_search
{
  "query": {
    "ids": {
      "values": ["1","2"]
    }
  }
}
```

###  模糊查询[fuzzy]

> fuzzy
>
> 用来模糊查询含有指定关键字的文档

```json
GET /products/_search
{
  "query": {
    "fuzzy": {
      "description": "iphooone"
    }
  }
}
```

> 注意
>
> fuzzy 模糊查询 最大模糊错误 必须在0-2之间
>
> - 搜索关键词长度为 2 不允许存在模糊
>
> - 搜索关键词长度为3-5 允许一次模糊
> - 搜索关键词长度大于5 允许最大2模糊



### 布尔查询[bool]

> bool
>
> 用来组合多个条件实现复杂查询
>
> **must: 相当于 && 同时成立**
>
> **should: 相当于 || 成立一个就行**
>
> **must_not: 相当于 ! 不能满足任何一个**

```json
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "term": {
            "price": {
              "value": 6888
            }
          }
        }
      ]
    }
  }
}
```



### 多字段查询[multi_match]

```json
GET /products/_search
{
  "query": {
    "multi_match": {
      "query": "iphone13 毫",
      "fields": [
        "title",
        "description"
      ]
    }
  }
}

# 注意: 字段类型分词,将查询条件分词之后进行查询改字段  如果该字段不分词就会将查询条件作为整体进行查询
```

### 默认字段分词查询[query_string]

```json
GET /products/_search
{
  "query": {
    "query_string": {
      "default_field": "description",
      "query": "屏幕真的非常不错"
    }
  }
}

# 注意: 查询字段分词就将查询条件分词查询  查询字段不分词将查询条件不分词查询
```

### 高亮查询[highlight]

> highlight
>
> 可以让符合条件的文档中的关键词高亮

```
GET /products/_search
{
  "query": {
    "term": {
      "description": {
        "value": "iphone"
      }
    }
  },
  "highlight": {
    "fields": {
      "*": {}
    }
  }
}
```

> 自定义高亮html标签
>
> 可以在highlight中使用`pre_tags`和`post_tags`

```json
GET /products/_search
{
  "query": {
    "term": {
      "description": {
        "value": "iphone"
      }
    }
  },
  "highlight": {
    "post_tags": [
      "</span>"
    ],
    "pre_tags": [
      "<span style='color:red'>"
    ],
    "fields": {
      "*": {}
    }
  }
}
```



###  返回指定条数[size]

> size
>
> 指定查询结果中返回指定条数。 **默认返回值10条**

> GET /products/_search
> {
>   "query": {
>     "match_all": {}
>   },
>   "size": 3
> }

### 分页查询[form]

> from
>
> 用来指定起始返回位置，和**size关键字连用可实现分页效果**

```json
GET /products/_search
{
  "query": {
    "match_all": {}
  },
  "size": 2,
  "from": 2
}
```

### 返回指定字段[_source]

> _source
>
> 是一个数组,在数组中用来指定展示那些字段



```json
GET /products/_search
{
  "query": {
    "match_all": {}
  },
  "_source": [
    "title",
    "description"
  ]
}
```



# 三、索引原理

## 倒排索引

`倒排索引（Inverted Index）`也叫反向索引，有反向索引必有正向索引。通俗地来讲，**正向索引是通过key找value，反向索引则是通过value找key。ES底层在检索时底层使用的就是倒排索引。**



# 四、分词器

## Analysis 和 Analyzer

`Analysis`： 文本分析是把全文本转换一系列单词(term/token)的过程，也叫分词(Analyzer)。**Analysis是通过Analyzer来实现的**。`分词就是将文档通过Analyzer分成一个一个的Term(关键词查询),每一个Term都指向包含这个Term的文档`。



## Analyzer 组成

- 注意: 在ES中默认使用标准分词器: StandardAnalyzer 特点: 中文单字分词 单词分词。

  我是中国人 this is good man----> analyzer----> 我 是 中 国 人 this is good man

> 分析器（analyzer）
>
> 都由三种构件组成的：`character filters` ， `tokenizers` ， `token filters`。

- `character filter` 字符过滤器

​		在一段文本进行分词之前，先进行预处理，比如说最常见的就是，过滤html标签

​	（`<span>hello<span>` --> hello），& --> and（I&you --> I and you）

- `tokenizers` 分词器
  - 英文分词可以根据空格将单词分开,中文分词比较复杂,可以采用机器学习算法来分词。
- `Token filters` Token过滤器
  - **将切分的单词进行加工**。大小写转换（例将“Quick”转为小写），去掉停用词（例如停用词像“a”、“and”、“the”等等），加入同义词（例如同义词像“jump”和“leap”）。

> 注意:
>
> - 三者顺序: Character Filters--->Tokenizer--->Token Filter
> - 三者个数：Character Filters（0个或多个） + Tokenizer + Token Filters(0个或多个)

## 内置分词器

- Standard Analyzer - 默认分词器，英文按单词词切分，并小写处理
- Simple Analyzer - 按照单词切分(符号被过滤), 小写处理
- Stop Analyzer - 小写处理，停用词过滤(the,a,is)
- Whitespace Analyzer - 按照空格切分，不转小写
- Keyword Analyzer - 不分词，直接将输入当作输出

## 内置分词器测试

- 标准分词器
  - 特点: 按照单词分词 英文统一转为小写 过滤标点符号 中文单字分词

```json
POST /_analyze
{
  "analyzer": "standard",
  "text": "this is a , good Man 中华人民共和国"
}
```

- Simple 分词器
  - 特点: 英文按照单词分词 英文统一转为小写 去掉符号 中文按照空格进行分词

```json
POST /_analyze
{
  "analyzer": "simple",
  "text": "this is a , good Man 中华人民共和国"
}
```

- Whitespace 分词器
  - 特点: 中文 英文 按照空格分词 英文不会转为小写 不去掉标点符号

```json
POST /_analyze
{
  "analyzer": "whitespace",
  "text": "this is a , good Man"
}
```

## 创建索引设置分词

```json
PUT /test_analysis
{
  "settings": {},
  "mappings": {
    "properties": {
      "title":{
        "type": "text",
        "analyzer": "standard" 
      }
    }
  }
}
```





## 中文分词器

在ES中支持中文分词器非常多 如 **smartCN**、**IK** 等，推荐的就是 `IK分词器`。



### 安装IK

开源分词器 Ik 的 https://github.com/medcl/elasticsearch-analysis-ik/releases

- `注意` IK分词器的版本要你安装ES的版本一致
- `注意` Docker 容器运行 ES 安装插件目录为 **/usr/share/elasticsearch/plugins**

ES安装的版本为7.17.5，因此分词器也应该是7.17.5： [elasticsearch-analysis-ik-7.17.5.zip](https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.17.5/elasticsearch-analysis-ik-7.17.5.zip)

```bash
# 解压压缩包
mkdir -p /opt/installs/elasticsearch/plugins/ik
# 解压安装包放到ik里面
unzip  elasticsearch-analysis-ik-7.17.5.zip
# 重启es容器生效
docker restart elasticsearch
```



### IK使用

IK有两种颗粒度的拆分：

- `ik_smart`: 会做最粗粒度的拆分
- `ik_max_word`: 会将文本做最细粒度的拆分

