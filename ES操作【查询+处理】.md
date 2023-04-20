# ES操作【查询+处理】

# 1.DSL查询文档

## 1.1.DSL查询分类

Elasticsearch提供了基于JSON的DSL（[Domain Specific Language](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html)）来定义查询。常见的查询类型包括：

- **查询所有**：查询出所有数据，一般测试用。例如：match_all

- **全文检索（full text）查询**：利用分词器对用户输入内容分词，然后去倒排索引库中匹配。例如：
  - match_query
  - multi_match_query
- **精确查询**：根据精确词条值查找数据，一般是查找keyword、数值、日期、boolean等类型字段。例如：
  - ids
  - range
  - term
- **地理（geo）查询**：根据经纬度查询。例如：
  - geo_distance
  - geo_bounding_box
- **复合（compound）查询**：复合查询可以将上述各种查询条件组合起来，合并查询条件。例如：
  - bool
  - function_score

语法：

```json
GET /indexName/_search
{
  "query": {
    "查询类型": {
      "查询条件": "条件值"
    }
  }
}
```

我们以查询所有为例，其中：

- 查询类型为match_all
- 没有查询条件

```json
// 查询所有
GET /indexName/_search
{
  "query": {
    "match_all": {
    }
  }
}
```

其它查询无非就是**查询类型**、**查询条件**的变化。



## 1.2:全文检索查询

### 1.2.1:使用场景

全文检索查询的基本流程如下：

- 对用户搜索的内容做分词，得到词条
- 根据词条去倒排索引库中匹配，得到文档id
- 根据文档id找到文档，返回给用户

比较常用的场景包括：

- 商城的输入框搜索
- 百度输入框搜索

![image-20220928161208022](https://zzx-note.oss-cn-beijing.aliyuncs.com/es/image-20220928161208022.png)

因为是拿着词条去匹配，因此参与搜索的字段也必须是可分词的text类型的字段



### 1.2.2.基本语法

常见的全文检索查询包括：

- match查询：单字段查询
- multi_match查询：多字段查询，任意一个字段符合条件就算符合查询条件

match查询语法如下：

```json
GET /indexName/_search
{
  "query": {
    "match": {
      "FIELD": "TEXT"
    }
  }
}
```

mulit_match语法如下：

```json
GET /indexName/_search
{
  "query": {
    "multi_match": {
      "query": "TEXT",
      "fields": ["FIELD1", " FIELD12"]
    }
  }
}
```





### 1.2.3:使用演示

match的示例：

```json
# 查看酒店品牌为如家的酒店信息
GET /hotel/_search
{
  "query": {
    "match": {
      "brand": "如家"
    }
  }
}
```

```json
# 查看all属性涉及到如家的
GET /hotel/_search
{
  "query": {
    "match": {
      "all": "如家"
    }
  }
}
```

multi_match的示例：

```json
# 使用multi_match查看以下属性涉及到如家的
GET /hotel/_search
{
  "query": {
    "multi_match": {
      "query": "如家",
      "fields": ["brand","name","business"]
    }
  }
}
```

可以看到，使用multi_match和使用all字段，两种查询结果是一样的，为什么？

因为我们将brand、name、business值都利用copy_to复制到了all字段中

因此你根据三个字段搜索，和根据all字段搜索效果当然一样了。

但是，搜索字段越多，对查询性能影响越大，因此建议采用copy_to，然后单字段查询的方式



### 1.2.4:总结

match和multi_match的区别是什么？

- match：根据一个字段查询
- multi_match：根据多个字段查询，参与查询字段越多，查询性能越差



## 1.3:精准查询

精确查询一般是查找keyword、数值、日期、boolean等类型字段。所以**不会**对搜索条件分词。常见的有：

- term：根据词条精确值查询
- range：根据值的范围查询

### 1.3.1:term查询

因为精确查询的字段搜是不分词的字段，因此查询的条件也必须是**不分词**的词条。查询时，用户输入的内容跟自动值完全匹配时才认为符合条件。如果用户输入的内容过多，反而搜索不到数据。

语法说明：

```json
// term查询
GET /indexName/_search
{
  "query": {
    "term": {
      "FIELD": {
        "value": "VALUE"
      }
    }
  }
}
```

term演示：

```json
# 查看北京的酒店
GET /hotel/_search
{
  "query": {
    "term": {
      "city": {
        "value": "北京"
      }
    }
  }
}
```



### 1.3.2:range查询

范围查询，一般应用在对数值类型做范围过滤的时候。比如做价格范围过滤。

基本语法：

```json
// range查询
GET /indexName/_search
{
  "query": {
    "range": {
      "FIELD": {
        "gte": 10, // 这里的gte代表大于等于，gt则代表大于
        "lte": 20 // lte代表小于等于，lt则代表小于
      }
    }
  }
}
```

range示例：

```json
# 查看价格在300到800之间的酒店
GET /hotel/_search
{
  "query": {
    "range": {
      "price": {
        "gte": 300,
        "lte": 800
      }
    }
  }
}
```

### 1.3.3:总结

精确查询常见的有哪些？

- term查询：根据词条精确匹配，一般搜索keyword类型、数值类型、布尔类型、日期类型字段
- range查询：根据数值范围查询，可以是数值、日期的范围



## 1.4:地理坐标查询

所谓的地理坐标查询，其实就是根据经纬度查询

常见的使用场景包括：

- 携程：搜索我附近的酒店
- 滴滴：搜索我附近的出租车
- 微信：搜索我附近的人

### 1.4.1:矩形范围查询

矩形范围查询，也就是geo_bounding_box查询，查询坐标落在某个矩形范围的所有文档：

<img src="https://zzx-note.oss-cn-beijing.aliyuncs.com/es/image-20220928163002420.png" alt="image-20220928163002420" style="zoom:50%;" />

查询时，需指定矩形的**左上**、**右下**两点的坐标，然后画出一个矩形，落在该矩形内的都是符合条件的点

语法：

```json
GET /indexName/_search
{
  "query": {
    "geo_bounding_box": {
      "FIELD": {
        "top_left": { // 左上点
          "lat": 31.1,
          "lon": 121.5
        },
        "bottom_right": { // 右下点
          "lat": 30.9,
          "lon": 121.7
        }
      }
    }
  }
}
```

示例：

```json
# 查看某个经纬度矩形框内的酒店
GET /hotel/_search
{
  "query": {
    "geo_bounding_box":{
      "location":{
        "top_left":{ 
          "lat":31.1,
          "lon":121.5
        },
        "bottom_right":{
          "lat":30.9,
          "lon":121.7
        }
      }
    }
  }
}
```

这种并不符合“附近的人”这样的需求



### 1.4.2:附近查询

附近查询，也叫做距离查询（geo_distance）：查询到指定中心点小于某个距离值的所有文档。

换句话来说，在地图上找一个点作为圆心，以指定距离为半径，画一个圆，落在圆内的坐标都算符合条件：

![image-20220928163523741](https://zzx-note.oss-cn-beijing.aliyuncs.com/es/image-20220928163523741.png)

语法说明：

```json
GET /indexName/_search
{
  "query": {
    "geo_distance": {
      "distance": "半径", // 半径
      "FIELD": "经纬度" // 圆心
    }
  }
}
```

示例：

我们先搜索陆家嘴【lication=31.21,121.5】附近3km的酒店：

```json
# 查看陆家嘴3km内的酒店
GET /hotel/_search
{
  "query": {
    "geo_distance":{
      "distance":"3km",
      "location":"31.21,121.5"
    }
  }
}
```

## 1.5:复合查询

### 1.5.1:相关性算分

复合查询：复合查询可以将其它简单查询组合起来，实现更复杂的搜索逻辑。常见的有两种：

- fuction score：算分函数查询，可以控制文档相关性算分，控制文档排名
- bool query：布尔查询，利用逻辑关系组合多个其它的查询，实现复杂搜索

例如，我们搜索 "虹桥如家"，结果如下：

```json
[
  {
    "_score" : 17.850193,
    "_source" : {
      "name" : "虹桥如家酒店真不错",
    }
  },
  {
    "_score" : 12.259849,
    "_source" : {
      "name" : "外滩如家酒店真不错",
    }
  },
  {
    "_score" : 11.91091,
    "_source" : {
      "name" : "迪士尼如家酒店真不错",
    }
  }
]
```

不需要具体关心算分的算法，只需要知道，ES会对结果做算分，匹配度越高在越前面



### 1.5.2:算分函数查询

根据相关度打分是比较合理的需求，但**合理的不一定是产品经理需要**的。

以百度为例，你搜索的结果中，并不是相关度越高排名越靠前，可能是谁掏的钱多排名就越靠前

要想认为控制相关性算分，就需要利用elasticsearch中的function score 查询了



#### 1）语法说明

![image-20220928164310215](https://zzx-note.oss-cn-beijing.aliyuncs.com/es/image-20220928164310215.png)

function score 查询中包含四部分内容：

- **原始查询**条件：query部分，基于这个条件搜索文档，并且基于BM25算法给文档打分，**原始算分**（query score)

- **过滤条件**：filter部分，符合该条件的文档才会重新算分

- **算分函数**：符合filter条件的文档要根据这个函数做运算，得到的**函数算分**（function score），有四种函数

  - weight：函数结果是常量
  - field_value_factor：以文档中的某个字段值作为函数结果
  - random_score：以随机数作为函数结果
  - script_score：自定义算分函数算法

- **运算模式**：算分函数的结果、原始查询的相关性算分，两者之间的运算方式，包括：

  - multiply：相乘
  - replace：用function score替换query score
  - 其它，例如：sum、avg、max、min

  

function score的运行流程如下：

- 1）根据**原始条件**查询搜索文档，并且计算相关性算分，称为**原始算分**（query score）
- 2）根据**过滤条件**，过滤文档
- 3）符合**过滤条件**的文档，基于**算分函数**运算，得到**函数算分**（function score）
- 4）将**原始算分**（query score）和**函数算分**（function score）基于**运算模式**做运算，得到最终结果，作为相关性算分。



因此，其中的关键点是：

- 过滤条件：决定哪些文档的算分被修改
- 算分函数：决定函数算分的算法
- 运算模式：决定最终算分结果



#### 2）示例

需求：查询上海的所有酒店，如家给了钱，让“如家”这个品牌的酒店排名靠前一些，得分乘以10倍

翻译一下这个需求，转换为之前说的四个要点：

- 原始条件：不确定，可以任意变化
- 过滤条件：brand="上海"
- 算分函数：可以简单粗暴，直接给固定的算分结果，weight
- 运算模式：比如相乘

```json
GET /hotel/_search
{
  "query": {
    "function_score": {
      "query": {
        "match": {
          "city": "上海"
        }
      },
      
      "functions": [
        {
          "filter": {
            "term": {
              "brand": "如家"
            }
          },
          "weight": 10
        }
      ],
      
      "boost_mode": "sum"
    }
  }
}
```

#### 3）小结

function score query定义的三要素是什么？

- 过滤条件：哪些文档要加分
- 算分函数：如何计算function score
- 加权方式：function score 与 query score如何运算



### 1.5.3:布尔查询

布尔查询是一个或多个查询子句的组合，每一个子句就是一个**子查询**。子查询的组合方式有：

- must：必须匹配每个子查询，类似“与”
- should：选择性匹配子查询，类似“或”
- must_not：必须不匹配，**不参与算分**，类似“非”
- filter：必须匹配，**不参与算分**



比如在搜索酒店时，除了关键字搜索外，我们还可能根据品牌、价格、城市等字段做过滤：



每一个不同的字段，其查询的条件、方式都不一样，又必须是多个不同的查询

而要组合这些查询，就必须用bool查询了。



注意：搜索时，参与**打分的字段越多，查询的性能也越差**。因此这种多条件查询时，建议这样做：

- 搜索框的关键字搜索，是全文检索查询，使用must查询，参与算分
- 其它过滤条件，采用filter查询。不参与算分

#### 1）语法示例：

```json
GET /hotel/_search
{
  "query": {
    "bool": {
      "must": [
        {"term": {"city": "上海" }}
      ],
      "should": [
        {"term": {"brand": "皇冠假日" }},
        {"term": {"brand": "华美达" }}
      ],
      "must_not": [
        { "range": { "price": { "lte": 500 } }}
      ],
      "filter": [
        { "range": {"score": { "gte": 45 } }}
      ]
    }
  }
}
```

#### 2）示例

需求：搜索名字包含“如家”，价格不高于400，在坐标31.21,121.5周围10km范围内的酒店。

分析：

- 名称搜索，属于全文检索查询，应该参与算分。放到must中
- 价格不高于400，用range查询，属于过滤条件，不参与算分。放到must_not中
- 周围10km范围内，用geo_distance查询，属于过滤条件，不参与算分。放到filter中

```json
GET /hotel/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "name": "如家"
          }
        }
      ],
      
      "must_not": [
        {
          "range": {
            "price": {
              "gt": 400
            }
          }
        }
      ],
      
      "filter": [
        {
          "geo_distance": {
            "distance": "10km",
            "location": {
              "lat": 31.21,
              "lon": 121.5
            }
          }
        }
      ]
    }
  }
}
```

#### 3）小结

bool查询有几种逻辑关系？

- must：必须匹配的条件，可以理解为“与”
- should：选择性匹配的条件，可以理解为“或”
- must_not：必须不匹配的条件，不参与打分
- filter：必须匹配的条件，不参与打分



# 2:搜索结果处理

## 2.1:排序

elasticsearch默认是根据相关度算分（_score）来排序，但是也支持自定义方式对搜索[结果排序](https://www.elastic.co/guide/en/elasticsearch/reference/current/sort-search-results.html)。可以排序字段类型有：keyword类型、数值类型、地理坐标类型、日期类型等。

### 2.1.1:普通字段排序

keyword、数值、日期类型排序的语法基本一致

**语法**：

```json
GET /indexName/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "FIELD": "desc"  // 排序字段、排序方式ASC、DESC
    }
  ]
}
```

排序条件是一个数组，也就是可以写多个排序条件。按照声明的顺序，当第一个条件相等时，再按照第二个条件排序，以此类推



**示例**：

需求描述：酒店数据按照用户评价（score)降序排序，评价相同的按照价格(price)升序排序

```json
GET /hotel/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "score": {
        "order": "desc"
      },
      "price": {
        "order": "asc"
      }
    }
  ]
}
```

### 2.1.2:地理坐标排序

地理坐标排序略有不同。

**语法说明**：

```json
GET /indexName/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "_geo_distance" : {
          "FIELD" : "纬度，经度", // 文档中geo_point类型的字段名、目标坐标点
          "order" : "asc", // 排序方式
          "unit" : "km" // 排序的距离单位
      }
    }
  ]
}
```

这个查询的含义是：

- 指定一个坐标，作为目标点
- 计算每一个文档中，指定字段（必须是geo_point类型）的坐标 到目标点的距离是多少
- 根据距离排序



**示例：**

需求描述：实现对酒店数据按照到你的位置坐标的距离升序排序

假设我的位置是：31.034661，121.612282，寻找我周围距离最近的酒店

```json
# 我的位置是：31.034661，121.612282，寻找我周围距离最近的酒店
GET /hotel/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "_geo_distance": {
        "location": {
          "lat": 31.034661,
          "lon": 121.612282
        },
        "order": "asc",
        "unit": "km"
      }
    }
  ]
}
```

注意到和上面的语法有点不同，实际上都可以



## 2.2:分页

elasticsearch 默认情况下只返回top10的数据。而如果要查询更多数据就需要修改分页参数了。elasticsearch中通过修改from、size参数来控制要返回的分页结果：

- from：从第几个文档开始
- size：总共查询几个文档

类似于mysql中的`limit ?, ?`

### 2.2.1:基本的分页

分页的基本语法如下：

```json
GET /hotel/_search
{
  "query": {
    "match_all": {}
  },
  "from": 0, // 分页开始的位置，默认为0
  "size": 10, // 期望获取的文档总数
  "sort": [
    {"price": "asc"}
  ]
}
```

### 2.2.2:深度分页问题

现在，我要查询990~1000的数据，查询逻辑要这么写：

```json
GET /hotel/_search
{
  "query": {
    "match_all": {}
  },
  "from": 990, // 分页开始的位置，默认为0
  "size": 10, // 期望获取的文档总数
  "sort": [
    {"price": "asc"}
  ]
}
```

这里是查询990开始的数据，也就是 第990~第1000条 数据。

不过，elasticsearch内部分页时，必须先查询 0~1000条，然后截取其中的990 ~ 1000的这10条：

![image-20220928172321574](https://zzx-note.oss-cn-beijing.aliyuncs.com/es/image-20220928172321574.png)

查询TOP1000，如果es是单点模式，这并无太大影响。

但是elasticsearch将来一定是集群，例如我集群有5个节点，我要查询TOP1000的数据，并不是每个节点查询200条就可以了。

因为节点A的TOP200，在另一个节点可能排到10000名以外了。

因此要想获取整个集群的TOP1000，必须先查询出每个节点的TOP1000，汇总结果后，重新排名，重新截取TOP1000。

![image-20220928172411950](https://zzx-note.oss-cn-beijing.aliyuncs.com/es/image-20220928172411950.png)

那如果我要查询9900~10000的数据呢？是不是要先查询TOP10000呢？那每个节点都要查询10000条？汇总到内存中？

当查询分页深度较大时，汇总数据过多，对内存和CPU会产生非常大的压力，因此elasticsearch会禁止from+ size 超过10000的请求。

针对深度分页，ES提供了两种解决方案，[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/paginate-search-results.html)：

- after search：分页需要排序，原理是从上一次的排序值开始，查询下一页数据。官方推荐使用的方式。
- scroll：原理将排序后的文档id形成快照，保存在内存。官方已经不推荐使用。

### 2.2.3:小结

分页查询的常见实现方案以及优缺点：

- `from + size`：
  - 优点：支持随机翻页
  - 缺点：深度分页问题，默认查询上限（from + size）是10000
  - 场景：百度、京东、谷歌、淘宝这样的随机翻页搜索
- `after search`：
  - 优点：没有查询上限（单次查询的size不超过10000）
  - 缺点：只能向后逐页查询，不支持随机翻页
  - 场景：没有随机翻页需求的搜索，例如手机向下滚动翻页

- `scroll`：
  - 优点：没有查询上限（单次查询的size不超过10000）
  - 缺点：会有额外内存消耗，并且搜索结果是非实时的
  - 场景：海量数据的获取和迁移。从ES7.1开始不推荐，建议用 after search方案。



## 2.3:高亮

### 2.3.1:高亮原理

什么是高亮显示呢？

我们在百度，京东搜索时，关键字会变成红色，比较醒目，这叫高亮显示：

![image-20220928172715397](https://zzx-note.oss-cn-beijing.aliyuncs.com/es/image-20220928172715397.png)

这种红色的带有我们搜索的变成了红色，就是高亮

高亮显示的实现分为两步：

- 1）给文档中的所有关键字都添加一个标签，例如`<em>`标签【ES会对高亮的默认添加`<em>`】
- 2）页面给`<em>`标签编写CSS样式【前端的工作】



### 2.3.2:实现高亮

**高亮的语法**：

```json
GET /hotel/_search
{
  "query": {
    "match": {
      "FIELD": "TEXT" // 查询条件，高亮一定要使用全文检索查询
    }
  },
  "highlight": {
    "fields": { // 指定要高亮的字段
      "FIELD": {
        "pre_tags": "<em>",  // 用来标记高亮字段的前置标签
        "post_tags": "</em>" // 用来标记高亮字段的后置标签
      }
    }
  }
}
```



**注意：**

- 高亮是对关键字高亮，因此**搜索条件必须带有关键字**，而不能是范围这样的查询。
- 默认情况下，**高亮的字段，必须与搜索指定的字段一致**，否则无法高亮
- 如果要对非搜索字段高亮，则需要添加一个属性：required_field_match=false
- 说明：比如我们在创建索引库的时候，搜索的字段一般会加在`all`字段里面，也就是按照all搜索的时候
- 只会对all高亮，但是并没有all，all里面的是其他的字段，这个时候默认其他的就不会高亮，才添加属性



示例：这样不需要添加属性，因为搜索的字段是brand，高亮的也是brand

```json
GET /hotel/_search
{
  "query": {
    "match": {
      "brand": "如家"
    }
  },
  "highlight": {
    "fields": {
      "brand": {}
    }
  }
}
```

示例：搜索的是all，默认是all高亮，但是没有all高亮，只是一个方便我们搜索的字段，这样才需要添加属性

```json
GET /hotel/_search
{
  "query": {
    "match": {
      "all": "如家"
    }
  },
  "highlight": {
    "fields": {
      "name":{
        "require_field_match": "false"
      }
    }
  }
}
```

## 2.4:总结

查询的DSL是一个大的JSON对象，包含下列属性：

- query：查询条件
- from和size：分页条件
- sort：排序条件
- highlight：高亮条件



整体查询结果处理的示例：

![image-20220928173842352](https://zzx-note.oss-cn-beijing.aliyuncs.com/es/image-20220928173842352.png)



# 3:RestClient查询文档

文档的查询同样适用昨天学习的 RestHighLevelClient对象，基本步骤包括：

- 1）准备Request对象
- 2）准备请求参数
- 3）发起请求
- 4）解析响应



## 3.1:快速入门

我们以match_all查询为例

### 3.1.1:发起查询请求

- 第一步，创建`SearchRequest`对象，指定索引库名

- 第二步，利用`request.source()`构建DSL，DSL中可以包含查询、分页、排序、高亮等
  - `query()`：代表查询条件，利用`QueryBuilders.matchAllQuery()`构建一个match_all查询的DSL
- 第三步，利用client.search()发送请求，得到响应

这里关键的API有两个，一个是`request.source()`，其中包含了查询、排序、分页、高亮等所有功能：

<img src="https://zzx-note.oss-cn-beijing.aliyuncs.com/es/image-20220928214008197.png" alt="image-20220928214008197" style="zoom:50%;" />

另一个是`QueryBuilders`，其中包含match、term、function_score、bool等各种查询：

![image-20220928214046370](https://zzx-note.oss-cn-beijing.aliyuncs.com/es/image-20220928214046370.png)

### 3.1.2:解析响应

响应结果的解析：

![image-20220928214135604](https://zzx-note.oss-cn-beijing.aliyuncs.com/es/image-20220928214135604.png)

elasticsearch返回的结果是一个JSON字符串，结构包含：

- `hits`：命中的结果
  - `total`：总条数，其中的value是具体的总条数值
  - `max_score`：所有结果中得分最高的文档的相关性算分
  - `hits`：搜索结果的文档数组，其中的每个文档都是一个json对象
    - `_source`：文档中的原始数据，也是json对象

因此，我们解析响应结果，就是逐层解析JSON字符串，流程如下：

- `SearchHits`：通过response.getHits()获取，就是JSON中的最外层的hits，代表命中的结果
  - `SearchHits#getTotalHits().value`：获取总条数信息
  - `SearchHits#getHits()`：获取SearchHit数组，也就是文档数组
    - `SearchHit#getSourceAsString()`：获取文档结果中的_source，也就是原始的json文档数据

### 3.1.3:完整代码

完整代码如下：

```java
@Test
void matchAll() throws IOException {
    // 准备Request对象
    SearchRequest request = new SearchRequest("hotel");
    //  准备查询的DSL语句
    request.source().query(QueryBuilders.matchAllQuery());
    // 发送请求
    SearchResponse response = client.search(request, RequestOptions.DEFAULT);
    log.info("查询所有数据成功，即将解析数据");
    // 解析响应，拿到第一层hits
    SearchHits searchHits = response.getHits();
    // 解析第一层hits，里面有记录总条数
    long total = searchHits.getTotalHits().value;
    log.info("数据总条数:{}",total);
    // 解析第二层hits【文档的数组】
    SearchHit[] searchHitsHits = searchHits.getHits();
    // 转换为stream流
    Stream<SearchHit> stream = Arrays.stream(searchHitsHits);
    stream.forEach((doc)->{
        // 拿到ES文档的json的数据
        String hotelDocJson = doc.getSourceAsString();
        // 反序列化为HotelDoc对象
        HotelDoc hotelDoc = JSON.parseObject(hotelDocJson, HotelDoc.class);
        log.info("解析的酒店数据是:{}",hotelDoc);
    });
}
```

### 3.1.4:小结

查询的基本步骤是：

1. 创建SearchRequest对象

2. 准备Request.source()，也就是DSL。

   ① QueryBuilders来构建查询条件

   ② 传入Request.source() 的 query() 方法

3. 发送请求，得到结果

4. 解析结果（参考JSON结果，从外到内，逐层解析）



## 3.2:match查询

全文检索的match和multi_match查询与match_all的API基本一致。差别是查询条件，也就是query的部分

![image-20220928220121184](https://zzx-note.oss-cn-beijing.aliyuncs.com/es/image-20220928220121184.png)

因此，Java代码上的差异主要是request.source().query()中的参数了。同样是利用QueryBuilders提供的方法

而结果解析代码则完全一致，可以抽取并共享

我们先把解析响应的代码封装一下

```java
/**
 * 响应结果解析【普通解析】
 * @param response
 */
private void analysisResp(SearchResponse response) {
    // 解析响应，拿到第一层hits
    SearchHits searchHits = response.getHits();
    // 解析第一层hits，里面有记录总条数
    long total = searchHits.getTotalHits().value;
    log.info("数据总条数:{}",total);
    // 解析第二层hits【文档的数组】
    SearchHit[] searchHitsHits = searchHits.getHits();
    // 转换为stream流
    Stream<SearchHit> stream = Arrays.stream(searchHitsHits);
    stream.forEach((doc)->{
        // 拿到ES文档的json的数据
        String hotelDocJson = doc.getSourceAsString();
        // 反序列化为HotelDoc对象
        HotelDoc hotelDoc = JSON.parseObject(hotelDocJson, HotelDoc.class);
        log.info("解析的酒店数据是:{}",hotelDoc);
    });
}
```



使用match示例：查询name包含如家二字的酒店信息

```java
@Test
void testMatch() throws IOException {
    // 准备request对象
    SearchRequest request = new SearchRequest("hotel");
    // 构建DSL语句
    request.source().query(QueryBuilders.matchQuery("name","如家"));
    //发送请求
    SearchResponse response = client.search(request, RequestOptions.DEFAULT);
    //解析响应
    analysisResp(response);
}
```

使用matchall示例：在name和business中，查询包含如家二字的酒店信息

```java
@Test
void testMultiMatch() throws IOException {
    // 准备request对象
    SearchRequest request = new SearchRequest("hotel");
    // 构建DSL语句
    request.source().query(QueryBuilders.multiMatchQuery("北京","name","business"));
    //发送请求
    SearchResponse response = client.search(request, RequestOptions.DEFAULT);
    //解析响应
    analysisResp(response);
}
```

## 3.3:精确查询

精确查询主要是两者：

- term：词条精确匹配
- range：范围查询

与之前的查询相比，差异同样在查询条件，其它都一样。



term案例：查询在北京的酒店信息

```java
@Test
void testTerm() throws IOException {
    // 构建request对象
    SearchRequest request = new SearchRequest("hotel");
    // 构建DSL语句
    request.source().query(QueryBuilders.termQuery("city","北京"));
    // 获取响应
    SearchResponse response = client.search(request, RequestOptions.DEFAULT);
    // 解析响应
    analysisResp(response);
}
```

range案例：查询酒店价格在300-800之间的酒店

```java
@Test
void testRange() throws IOException {
    // 构建request对象
    SearchRequest request = new SearchRequest("hotel");
    // 构建DSL语句
    request.source().query(QueryBuilders.rangeQuery("price").gte(300).lte(800));
    // 获取响应
    SearchResponse response = client.search(request, RequestOptions.DEFAULT);
    // 解析响应
    analysisResp(response);
}
```

## 3.4:布尔查询

布尔查询是用must、must_not、filter等方式组合其它查询，代码示例如下：

![image-20220928222453714](https://zzx-note.oss-cn-beijing.aliyuncs.com/es/image-20220928222453714.png)

API与其它查询的差别同样是在查询条件的构建，QueryBuilders，结果解析等其他代码完全不变

布尔查询示例：

```java
@Test
void testBoolean() throws IOException {
    // 构建request对象
    SearchRequest request = new SearchRequest("hotel");
    // 构建DSL语句
    BoolQueryBuilder boolQuery = QueryBuilders.boolQuery();
    // 城市必须是北京
    boolQuery.must(QueryBuilders.termQuery("city","北京"))
            // 价格必须不大于600
            .mustNot(QueryBuilders.rangeQuery("price").gte(600))
            // 品牌应该为如家或者华美达
            .should(QueryBuilders.termQuery("brand","如家"))
            .should(QueryBuilders.termQuery("brand","华美达"));
    // 执行查询
    request.source().query(boolQuery);
    // 获取响应
    SearchResponse response = client.search(request, RequestOptions.DEFAULT);
    // 解析响应
    analysisResp(response);
}
```



算分函数查询：

接下来我们就要修改查询条件了。之前是用的boolean 查询，现在要改成function_socre查询。

function_score查询结构如下：

![image-20220930163530473](https://zzx-note.oss-cn-beijing.aliyuncs.com/es/image-20220930163530473.png)

对应的JavaAPI如下：

![image-20220930163549583](https://zzx-note.oss-cn-beijing.aliyuncs.com/es/image-20220930163549583.png)

我们可以将之前写的boolean查询作为**原始查询**条件放到query中，接下来就是添加**过滤条件**、**算分函数**、**加权模式**了。所以原来的代码依然可以沿用

```java
// 自定义算分
FunctionScoreQueryBuilder functionScoreQueryBuilder=QueryBuilders.functionScoreQuery(
        // 原始封装的DSL查询
        boolQuery,
    
    	// 自定义算分函数，不写默认是相乘
        new FunctionScoreQueryBuilder.FilterFunctionBuilder[]{
                // 其中的一个function score 元素
                new FunctionScoreQueryBuilder.FilterFunctionBuilder(
                        // 过滤条件
                        QueryBuilders.termQuery("isAD", true),
                        // 算分函数
                        ScoreFunctionBuilders.weightFactorFunction(10000)
                )
        }
);
```



## 3.5:排序、分页

搜索结果的排序和分页是与query同级的参数，因此同样是使用request.source()来设置。

对应的API如下：

![image-20220928223313348](https://zzx-note.oss-cn-beijing.aliyuncs.com/es/image-20220928223313348.png)

示例：查询前面30条数据

```java
@Test
void testLimit() throws IOException {
    // 构建request对象
    SearchRequest request = new SearchRequest("hotel");
    // 构建DSL语句
    request.source().query(QueryBuilders.matchAllQuery()).from(0).size(30);
    // 获取响应
    SearchResponse response = client.search(request, RequestOptions.DEFAULT);
    
    // 解析响应
    analysisResp(response);
}
```



示例：按照价格降序排序

```java
@Test
void testOrder() throws IOException {
    // 构建request对象
    SearchRequest request = new SearchRequest("hotel");
    // 构建DSL语句
    request.source().query(QueryBuilders.matchAllQuery()).sort("price", SortOrder.DESC);
    // 获取响应
    SearchResponse response = client.search(request, RequestOptions.DEFAULT);
    // 解析响应
    analysisResp(response);
}
```

示例：根据地址坐标排序【查找周围最近的酒店】

```java
@Test
void testOrder() throws IOException {
    // 构建request对象
    SearchRequest request = new SearchRequest("hotel");
    // 构建DSL语句
    // 按照地理坐标排序
    String location = params.getLocation();
    // 这里的location是经纬度字符串，例如  "321.42,874.24"
    if(location!=null && !"".equals(location.trim())){
    	request.source().sort(SortBuilders
                .geoDistanceSort("location",new GeoPoint(location))
                .order(SortOrder.ASC)
                .unit(DistanceUnit.KILOMETERS));
    }
    request.source().query(QueryBuilders.matchAllQuery());
    // 获取响应
    SearchResponse response = client.search(request, RequestOptions.DEFAULT);
    // 解析响应
    analysisResp(response);
}
```

查找出最近的酒店，一般要显示距离，距离和sources在同一级别

![image-20220930160808905](https://zzx-note.oss-cn-beijing.aliyuncs.com/es/image-20220930160808905.png)

sort就是距离信息，单位是什么，查出来的就是什么

```java
// 获取排序后显示的距离信息
Object[] sortValues = doc.getSortValues();
if(sortValues.length>0){
      // 假设只是添加了距离排序，则直接取第0个即可，存在多个，则遍历取出
    // sortValue就算上面的 sort 信息
      Object sortValue = sortValues[0];
}
```



## 3.6:高亮

高亮的代码与之前代码差异较大，有两点：

- 查询的DSL：其中除了查询条件，还需要添加高亮条件，同样是与query同级。
- 结果解析：结果除了要解析_source文档数据，还要解析高亮结果

### 3.6.1:高亮请求构建

高亮请求的构建API如下：

![image-20220928224609216](https://zzx-note.oss-cn-beijing.aliyuncs.com/es/image-20220928224609216.png)

上述代码省略了查询条件部分，但是大家不要忘了：高亮查询必须使用全文检索查询

也就是【match，multi_match】并且要有搜索关键字，将来才可以对关键字高亮

默认是对match 和 multi_match对应的字段高亮

我们查询 all 字段的时候，是不会对all 涉及的字段

如 brand,business,name等高亮的

如果要高亮，需要指定 require_filed_match 指定为false才可以



### 3.6.2.高亮结果解析

高亮的结果与查询的文档结果默认是分离的，并不在一起，但是都属于第二层 hits 里面

因此解析高亮的代码需要额外处理：

![image-20220928225802717](https://zzx-note.oss-cn-beijing.aliyuncs.com/es/image-20220928225802717.png)

所以解析高亮结果就只是比之前多一点点而已

- 第一步：从结果中获取source。hit.getSourceAsString()，这部分是非高亮结果，json字符串。还需要反序列为HotelDoc对象
- 第二步：获取高亮结果。hit.getHighlightFields()，返回值是一个Map，key是高亮字段名称，值是HighlightField对象，代表高亮值
- 第三步：从map中根据高亮字段名称，获取高亮字段值对象HighlightField
- 第四步：从HighlightField中获取Fragments，并且转为字符串。这部分就是真正的高亮字符串了
- 第五步：用高亮的结果替换HotelDoc中的非高亮结果



案例：搜索品牌为如家的酒店，对如家高亮

注意：这个时候高亮字段和全文索引查询字段一样，不需要设置require_filed_match

```java
@Test
void testHighLight() throws IOException {
    // 准备request对象
    SearchRequest request = new SearchRequest("hotel");
    // 准备DSL
    request.source().query(QueryBuilders.matchQuery("brand","如家"));
    // 高亮
    request.source().highlighter(new HighlightBuilder().field("brand"));
    // 发送请求
    SearchResponse response = client.search(request, RequestOptions.DEFAULT);
    analysisRespWithHighLight(response);
}
```

```java
/**
 * 响应结果解析【高亮解析】
 * @param response
 */
private void analysisRespWithHighLight(SearchResponse response) {
    // 解析响应，拿到第一层hits
    SearchHits searchHits = response.getHits();
    // 解析第一层hits，里面有记录总条数
    long total = searchHits.getTotalHits().value;
    log.info("数据总条数:{}",total);
    // 解析第二层hits【文档的数组】
    SearchHit[] searchHitsHits = searchHits.getHits();
    // 转换为stream流
    Stream<SearchHit> stream = Arrays.stream(searchHitsHits);
    stream.forEach((doc)->{
        // 拿到ES文档的json的数据
        String hotelDocJson = doc.getSourceAsString();
        // 反序列化为HotelDoc对象
        HotelDoc hotelDoc = JSON.parseObject(hotelDocJson, HotelDoc.class);
        // 拿到所有的高亮结果
        Map<String, HighlightField> highlightFields = doc.getHighlightFields();
        if(highlightFields!=null && highlightFields.size()!=0){
            // 根据字段名来获取高亮结果
            HighlightField highlightField = highlightFields.get("brand");
            // 解析高亮结果
            if(highlightField!=null){
                // 真正的ES里面的json字符串
                String highBrand = highlightField.getFragments()[0].string();
                // 覆盖字段
                hotelDoc.setBrand(highBrand);
            }
        }
        log.info("解析的酒店数据是:{}",hotelDoc);
    });
}
```



案例：搜索all，字段是如家，对含有如家的name，brand高亮

```java
@Test
void testHighLight1() throws IOException {
    // 准备request对象
    SearchRequest request = new SearchRequest("hotel");
    // 准备DSL
    request.source().query(QueryBuilders.matchQuery("all","如家"));
    // 高亮name和brand
    request.source().highlighter(new HighlightBuilder()
                    .field("name")
                    .field("brand")
                    .requireFieldMatch(false));
    // 发送请求
    SearchResponse response = client.search(request, RequestOptions.DEFAULT);
    analysisRespWithHighLight1(response);
}
```

多个字段高亮响应处理

```java
private void analysisRespWithHighLight1(SearchResponse response) {
    // 解析响应，拿到第一层hits
    SearchHits searchHits = response.getHits();
    // 解析第一层hits，里面有记录总条数
    long total = searchHits.getTotalHits().value;
    log.info("数据总条数:{}",total);
    // 解析第二层hits【文档的数组】
    SearchHit[] searchHitsHits = searchHits.getHits();
    // 转换为stream流
    Stream<SearchHit> stream = Arrays.stream(searchHitsHits);
    stream.forEach((doc)->{
        // 拿到ES文档的json的数据
        String hotelDocJson = doc.getSourceAsString();
        // 反序列化为HotelDoc对象
        HotelDoc hotelDoc = JSON.parseObject(hotelDocJson, HotelDoc.class);
        // 拿到所有的高亮结果
        Map<String, HighlightField> highlightFields = doc.getHighlightFields();
        // 判断是否存在高亮结果
        if(highlightFields!=null && highlightFields.size()!=0){
            // 根据字段名来获取高亮结果
            HighlightField highlightFieldName = highlightFields.get("name");
            HighlightField highlightFieldBrand = highlightFields.get("brand");
            // 解析高亮结果
            if(highlightFieldBrand!=null){
                // 真正的ES里面的json字符串
                String highBrand = highlightFieldBrand.getFragments()[0].string();
                // 覆盖字段
                hotelDoc.setBrand(highBrand);
            }
            // 解析高亮结果
            if(highlightFieldName!=null){
                // 真正的ES里面的json字符串
                String highName = highlightFieldName.getFragments()[0].string();
                // 覆盖字段
                hotelDoc.setName(highName);
            }
        }
        log.info("解析的酒店数据是:{}",hotelDoc);
    });
}
```

