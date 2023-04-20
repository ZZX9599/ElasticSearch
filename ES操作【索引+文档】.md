# ES操作【索引+文档】

# 概念对比

![image-20220928104447057](https://zzx-note.oss-cn-beijing.aliyuncs.com/es/image-20220928104447057.png)

# 1:索引相关

## 1.1:创建索引库

```json
PUT /student
{
  "mappings": {
    "properties": {
      "name":{
        "type": "keyword",
        "index": true
      },
      "age":{
        "type": "integer",
        "index": true
      },
      "email":{
        "type": "keyword",
        "index": true
      },
      "address":{
        "type": "text",
        "analyzer": "ik_smart"
      }
    }
  }
}
```



## 1.2:查看索引库

```http
GET /student
```



## 1.3:删除索引库

```http
DELETE /student
```



## 1.4:修改索引库

倒排索引结构虽然不复杂，但是一旦数据结构改变（比如改变了分词器），就需要重新创建倒排索引，这简

直是灾难。因此索引库**一旦创建，无法修改mapping**。

虽无法修改mapping中已有的字段，但却允许添加新的字段到mapping中，因为不会对倒排索引产生影响

很少使用，不演示了



# 2:文档相关

## 2.1:新增文档

```json
POST /student/_doc/1001
{
  "name":"周志雄",
  "age":20,
  "email":"536509593@qq.com",
  "address":"四川省广安市广安区"
}
```



## 2.2:查询文档

```http
GET /student/_doc/1001
```



## 2.3:删除文档

```http
DELETE /student/_doc/1001
```



## 2.4:修改文档

- 全量修改：直接覆盖原来的文档

- 全量修改是覆盖原来的文档，其本质是：

  - 根据指定的id删除文档
  - 新增一个相同id的文档
  - 如果根据id删除时，id不存在，第二步的新增也会执行，也就从修改变成了新增操作了
  - 所以和新增文档的语法一样，只是请求方式不一样
  - 如果存在，则先删除，再插入文档，不存在就直接新建文档

  ```json
  PUT /student/_doc/1001
  {
    "name":"周志雄",
    "age":20,
    "email":"536509593@qq.com",
    "address":"四川省广安市广安区"
  }
  ```

  

- 增量修改：修改文档中的部分字段

  - 增量修改是只修改指定id匹配的文档中的部分字段

  ```json
  POST /student/_update/1001
  {
    "doc": {
      "name":"蒋雪丽",
      "age":23
    }
  }
  ```



# 3:案例

## 3.1:导入数据库

```sql
CREATE TABLE `tb_hotel` (
  `id` bigint(20) NOT NULL COMMENT '酒店id',
  `name` varchar(255) NOT NULL COMMENT '酒店名称；例：7天酒店',
  `address` varchar(255) NOT NULL COMMENT '酒店地址；例：航头路',
  `price` int(10) NOT NULL COMMENT '酒店价格；例：329',
  `score` int(2) NOT NULL COMMENT '酒店评分；例：45，就是4.5分',
  `brand` varchar(32) NOT NULL COMMENT '酒店品牌；例：如家',
  `city` varchar(32) NOT NULL COMMENT '所在城市；例：上海',
  `star_name` varchar(16) DEFAULT NULL COMMENT '酒店星级，从低到高分别是：1星到5星，1钻到5钻',
  `business` varchar(255) DEFAULT NULL COMMENT '商圈；例：虹桥',
  `latitude` varchar(32) NOT NULL COMMENT '纬度；例：31.2497',
  `longitude` varchar(32) NOT NULL COMMENT '经度；例：120.3925',
  `pic` varchar(255) DEFAULT NULL COMMENT '酒店图片；例:/img/1.jpg',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```



## 3.2:创建酒店的索引结构

创建索引库，最关键的是mapping映射，而mapping映射要考虑的信息包括：

- 字段名
- 字段数据类型
- 是否参与搜索
- 是否需要分词
- 如果分词，分词器是什么？

其中：

- 字段名、字段数据类型，可以参考数据表结构的名称和类型
- 是否参与搜索要分析业务来判断，例如图片地址，就无需参与搜索
- 是否分词呢要看内容，内容如果是一个整体就无需分词，反之则要分词
- 分词器，我们可以统一使用ik_max_word【前提需要自己安装了分词器】
- 几个特殊字段说明：
  - location：地理坐标，里面包含精度、纬度
  - all：一个组合字段，其目的是将多字段的值 利用copy_to合并，提供给用户搜索

```json
PUT /hotel
{
  "mappings": {
    "properties": {
      "id": {
        "type": "keyword"
      },
      "name":{
        "type": "text",
        "analyzer": "ik_max_word",
        "copy_to": "all"
      },
      "address":{
        "type": "keyword",
        "index": false
      },
      "price":{
        "type": "integer"
      },
      "score":{
        "type": "integer"
      },
      "brand":{
        "type": "keyword",
        "copy_to": "all"
      },
      "city":{
        "type": "keyword",
        "copy_to": "all"
      },
      "starName":{
        "type": "keyword"
      },
      "business":{
        "type": "keyword"
      },
      "location":{
        "type": "geo_point"
      },
      "pic":{
        "type": "keyword",
        "index": false
      },
      "all":{
        "type": "text",
        "analyzer": "ik_max_word"
      }
    }
  }
}
```

地理坐标说明：

<img src="https://zzx-note.oss-cn-beijing.aliyuncs.com/es/image-20220928111559714.png" alt="image-20220928111559714" style="zoom: 67%;" />



# 4:RestClient操作索引

## 4.1:初始化RestClient

在elasticsearch提供的API中，与elasticsearch一切交互都封装在一个名为RestHighLevelClient的类中，必须先完成这个对象的初始化，建立与elasticsearch的连接。

分为三步：

1）引入es的RestHighLevelClient依赖：

```xml
<dependency>
    <groupId>org.elasticsearch.client</groupId>
    <artifactId>elasticsearch-rest-high-level-client</artifactId>
</dependency>
```



2）因为SpringBoot默认的可能和我们的ES版本不同，我使用的是7.12.1，所以我们需要覆盖默认的ES版本：

```xml
<properties>
    <java.version>1.8</java.version>
    <elasticsearch.version>7.12.1</elasticsearch.version>
</properties>
```



3）初始化RestHighLevelClient：

初始化的代码如下：

```java
RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(
        HttpHost.create("http://192.168.150.101:9200")
));
```



这里为了单元测试方便，我们创建一个测试类，然后将初始化的代码编写在@BeforeEach方法中：

```java
@SpringBootTest
class HotelDemoApplicationTests {

    @Resource
    private RestHighLevelClient client;

    @BeforeEach
    void init(){
        this.client=new RestHighLevelClient(RestClient.builder(
                HttpHost.create("http://192.168.61.133:9200")
        ));
    }

    @AfterEach
    void close() throws IOException {
        this.client.close();
    }
}
```



## 4.2:创建索引库

```java
public class HotelConstants {
    public static final String MAPPING_TEMPLATE = " 复制创建索引的语句到此处 "
}
```



```java
@Test
void createHotelIndex() throws IOException {
    // 创建request对象
    CreateIndexRequest request = new CreateIndexRequest("hotel");
    // 使用source来添加需要的参数
    request.source(HotelConstants.MAPPING_TEMPLATE, XContentType.JSON);
    // 发送请求
    client.indices().create(request, RequestOptions.DEFAULT);
    log.info("创建索引成功");
}
```



## 4.3:删除索引库

```java
@Test
void deleteHotelIndex() throws IOException {
    // 创建request对象
    DeleteIndexRequest request = new DeleteIndexRequest("hotel");
    // 发送请求
    client.indices().delete(request,RequestOptions.DEFAULT);
    log.info("删除索引成功");
}
```



## 4.4:判断索引库是否存在

```java
@Test
void existHotelIndex() throws IOException {
    // 创建request对象
    GetIndexRequest request = new GetIndexRequest("hotel");
    //发送请求
    boolean exists = client.indices().exists(request, RequestOptions.DEFAULT);
    log.info("hotel索引是否存在：{}",exists);
}
```



## 4.5:小总结

JavaRestClient操作elasticsearch的流程基本类似。核心是client.indices()方法来获取索引库的操作对象。

索引库操作的基本步骤：

- 初始化RestHighLevelClient
- 创建XxxIndexRequest。XXX是Create、Get、Delete
- 准备DSL（ Create时需要，其它是无参）
- 发送请求。调用RestHighLevelClient#indices().xxx()方法，xxx是create、exists、delete



# 5:RestClient操作文档

## 5.1:新增文档

我们要将数据库的酒店数据查询出来，写入elasticsearch中，查数据库就不展示了，直接使用查询后的数据

### 5.1.1.索引库实体类

数据库查询后的结果是一个Hotel类型的对象。实体类结构如下：

```java
@Data
@TableName("tb_hotel")
public class Hotel {
    @TableId(type = IdType.INPUT)
    private Long id;
    private String name;
    private String address;
    private Integer price;
    private Integer score;
    private String brand;
    private String city;
    private String starName;
    private String business;
    private String longitude;
    private String latitude;
    private String pic;
}
```

与我们的索引库结构存在差异：

- longitude和latitude需要合并为location
- 我们定义的ES的location的type是geo_point，格式是`精度,纬度`

因此，我们需要定义一个新的类型，与索引库结构吻合：

```java
@Data
@NoArgsConstructor
public class HotelDoc {
    private Long id;
    private String name;
    private String address;
    private Integer price;
    private Integer score;
    private String brand;
    private String city;
    private String starName;
    private String business;
    private String location;
    private String pic;

    public HotelDoc(Hotel hotel) {
        this.id = hotel.getId();
        this.name = hotel.getName();
        this.address = hotel.getAddress();
        this.price = hotel.getPrice();
        this.score = hotel.getScore();
        this.brand = hotel.getBrand();
        this.city = hotel.getCity();
        this.starName = hotel.getStarName();
        this.business = hotel.getBusiness();
        this.location = hotel.getLatitude() + ", " + hotel.getLongitude();
        this.pic = hotel.getPic();
    }
}

```



### 5.1.2.操作

我们导入酒店数据，基本流程一致，但是需要考虑几点变化：

- 酒店数据来自于数据库，我们需要先查询出来，得到hotel对象
- hotel对象需要转为HotelDoc对象
- HotelDoc需要序列化为json格式来发送给ES

因此，代码整体步骤如下：

- 1）根据id查询酒店数据Hotel
- 2）将Hotel封装为HotelDoc
- 3）将HotelDoc序列化为JSON
- 4）创建IndexRequest，指定索引库名和id
- 5）准备请求参数，也就是JSON文档
- 6）发送请求

在hotel-demo的HotelDocumentTest测试类中，编写单元测试：

```java
@Test
void addDoc() throws IOException {
    // 先查询数据库的数据
    Hotel hotel = hotelService.getById(61083L);
    // 转换为HotelDoc，这样才能和ES的location对应
    HotelDoc hotelDoc = new HotelDoc(hotel);
    // 转换为json
    String hotelDocJson = JSON.toJSONString(hotelDoc);
    log.info("数据准备完毕，即将发送请求给ES");
    // 创建Request对象
    IndexRequest request = new IndexRequest("hotel").id(hotelDoc.getId().toString());
    // 准备json文档
    request.source(hotelDocJson, XContentType.JSON);
    // 发送请求
    client.index(request, RequestOptions.DEFAULT);
    log.info("添加文档成功");
}
```



## 5.2:查询文档

查询很简单，发请求拿到响应即可的

目的是得到响应，解析为HotelDoc，因此难点是结果的解析

![image-20220928131012097](https://zzx-note.oss-cn-beijing.aliyuncs.com/es/image-20220928131012097.png)

发现我们的数据是在_source里面

发送请求拿到响应里面，存在一个source属性

通过RestClient查询文档，得到的响应存在这样一个方法

`response.getSourceAsString()`直接把数据转换成字符串，ES存储的就是JSON，拿到的就是JSON字符串

再通过工具类，就可以转换成Java对象了

```java
@Test
void getDoc() throws IOException {
    // 创建request对象
    GetRequest request = new GetRequest("hotel", "61083");
    // 发送请求
    GetResponse response = client.get(request, RequestOptions.DEFAULT);
    // 解析响应的结果为json数据
    String json = response.getSourceAsString();
    // 把json数据转化为java对象
    HotelDoc hotelDoc = JSON.parseObject(json, HotelDoc.class);
    log.info("查询的ES数据转化为java对象：{}",hotelDoc);
}
```



## 5.3:删除文档

```java
@Test
void deleteDoc() throws IOException {
    // 创建request对象
    DeleteRequest request = new DeleteRequest("hotel", "61083");
    // 发送请求
    client.delete(request, RequestOptions.DEFAULT);
    log.info("删除文档成功");
}
```



## 5.4:修改文档

修改我们说过两种方式：

- 全量修改：本质是先根据id删除，再新增
- 增量修改：修改文档中的指定字段值



在RestClient的API中，全量修改与新增的API完全一致，判断依据是ID：

- 如果新增时，ID已经存在，则修改
- 如果新增时，ID不存在，则新增

这里不再赘述，我们主要关注增量修改。

![image-20220928131731228](https://zzx-note.oss-cn-beijing.aliyuncs.com/es/image-20220928131731228.png)

```java
@Test
void updateDoc() throws IOException {
    // 创建request对象
    UpdateRequest request = new UpdateRequest("hotel", "61083");
    // 准备参数
    request.doc(
            "price","999",
            "starName","五钻"
    );
    // 发送请求
    client.update(request,RequestOptions.DEFAULT);
    log.info("修改成功");
}
```



## 5.5:批量导入文档

步骤如下：

- 查询出所有的酒店数据
- 将查询到的酒店数据（Hotel）转换为文档类型数据（HotelDoc）
- 利用JavaRestClient中的BulkRequest批处理，实现批量新增文档



### 5.5.1:BulkRequest介绍

批量处理BulkRequest，其本质就是将多个普通的CRUD请求组合在一起发送。

其中提供了一个add方法，用来添加其他请求：

![image-20220928132530273](https://zzx-note.oss-cn-beijing.aliyuncs.com/es/image-20220928132530273.png)

可以看到，能添加的请求包括：

- IndexRequest，也就是新增文档
- UpdateRequest，也就是修改文档
- DeleteRequest，也就是删除文档

因此我们可以在Bulk中添加多个IndexRequest，就能实现批量新增功能了

其实还是三步走：

- 1）创建Request对象。这里是BulkRequest
- 2）准备参数。批处理的参数，就是其它Request对象，这里就是多个IndexRequest
- 3）发起请求。这里是批处理，调用的方法为client.bulk()方法



### 5.5.2.完整代码

在hotel-demo的测试类中，编写单元测试：

```java
@Test
void bulkAddDoc() throws IOException {
    // 查询全部酒店数据
    List<Hotel> hotelList = hotelService.list();
    // 创建BulkRequest对象
    BulkRequest bulkRequest = new BulkRequest();
    hotelList.stream().forEach((hotel -> {
        // 把数据转化为HotelDoc
        HotelDoc hotelDoc = new HotelDoc(hotel);
        // 创建多个IndexRequest对象
        IndexRequest request = new IndexRequest("hotel").id(hotel.getId().toString());
        // 给IndexRequest对象添加数据
        request.source(JSON.toJSONString(hotelDoc),XContentType.JSON);
        //把请求添加到BulkRequest对象里面
        bulkRequest.add(request);
    }));
    // 发送批量的请求到ES
    client.bulk(bulkRequest,RequestOptions.DEFAULT);
    log.info("批量加入酒店数据到ES文档成功");
}
```

导入之后可以在浏览器查看一下数据

```json
GET hotel/_search
{
  "query": {
    "match_all": {}
  }
}
```



## 5.6.小结

文档操作的基本步骤：

- 初始化RestHighLevelClient
- 创建XxxRequest。XXX是Index、Get、Update、Delete、Bulk
- 准备参数（Index、Update、Bulk时需要）
- 发送请求。调用RestHighLevelClient#.xxx()方法，xxx是index、get、update、delete、bulk
- 解析结果（Get时需要） 
