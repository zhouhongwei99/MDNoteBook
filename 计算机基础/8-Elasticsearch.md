# Elasticsearch

## 与 mysql

| **MySQL** | **Elasticsearch** | **说明**                                                                           |
| --------- | ----------------- | ---------------------------------------------------------------------------------- |
| Table     | Index             | 索引(index)，就是文档的集合，类似数据库的表(table)                                 |
| Row       | Document          | 文档（Document），就是一条条的数据，类似数据库中的行（Row），文档都是 JSON 格式    |
| Column    | Field             | 字段（Field），就是 JSON 文档中的字段，类似数据库中的列（Column）                  |
| Schema    | Mapping           | Mapping（映射）是索引中文档的约束，例如字段类型约束。类似数据库的表结构（Schema）  |
| SQL       | DSL               | DSL 是 elasticsearch 提供的 JSON 风格的请求语句，用来操作 elasticsearch，实现 CRUD |

- Mysql：擅长**事务类型操作**，可以确保**数据的安全和一致性**，适合对**安全性要求较高**的**写操作**

- Elasticsearch：擅长**海量数据**的**搜索、分析、计算**，适合对**查询性能要求较高**的**搜索需求**

因此在企业中，往往是两者结合使用：

- 对**安全性要求较高**的写操作，使用**mysql**实现
- 对**查询性能要求较高**的搜索需求，使用**elasticsearch**实现
- 两者再基于某种方式（**canal**），实现**数据同步保证一致性**

![image-20210720203534945](图集/image-20210720203534945.png)

## 文档和字段

elasticsearch 是面向**文档（Document）** 存储的，可以是数据库中的一条商品数据，一个订单信息。文档数据会被序列化为 json 格式后存储在 elasticsearch 中：

![](图集/image-20210720202707797.png)
而 Json 文档中往往包含很多的**字段（Field）**，类似于数据库中的列。

## 正向索引

那么什么是正向索引呢？例如给下表（tb_goods）中的 id 创建索引：

![image-20210720195531539](图集/image-20210720195531539.png)

如果是根据 id 查询，那么直接走索引，查询速度非常快。

但如果是基于 title 做模糊查询，只能是逐行扫描数据，流程如下：

1）用户搜索数据，条件是 title 符合`"%手机%"`

2）逐行获取数据，比如 id 为 1 的数据

3）判断数据中的 title 是否符合用户搜索条件

4）如果符合则放入结果集，不符合则丢弃。回到步骤 1

逐行扫描，也就是全表扫描，随着数据量增加，其查询效率也会越来越低。当数据量达到数百万时，就是一场灾难。

## 倒排索引

倒排索引中有两个非常重要的概念：

- 文档（`Document`）：用来搜索的数据，其中的每一条数据就是一个文档。例如一个网页、一个商品信息
- 词条（`Term`）：对文档数据或用户搜索数据，利用某种算法分词，得到的具备含义的词语就是词条。例如：我是中国人，就可以分为：我、是、中国人、中国、国人这样的几个词条

**创建倒排索引**是对正向索引的一种特殊处理，流程如下：

- 将每一个文档的数据利用算法分词，得到一个个词条
- 创建表，每行数据包括词条、词条所在文档 id、位置等信息
- 因为词条唯一性，可以给词条创建索引，例如 hash 表结构索引

![image-20210720200457207](图集/image-20210720200457207.png)

倒排索引的**搜索流程**如下（以搜索"华为手机"为例）：

1）用户输入条件`"华为手机"`进行搜索。

2）对用户输入内容**分词**，得到词条：`华为`、`手机`。

3）拿着词条在倒排索引中查找，可以得到包含词条的文档 id：1、2、3。

4）拿着文档 id 到正向索引中查找具体文档。

如图：

![image-20210720201115192](图集/image-20210720201115192.png)

## 正向和倒排

- **正向索引**是最传统的，根据 id 索引的方式。但根据词条查询时，必须先逐条获取每个文档，然后判断文档中是否包含所需要的词条，是**根据文档找词条的过程**。

- 而**倒排索引**则相反，是先找到用户要搜索的词条，根据词条得到保护词条的文档的 id，然后根据 id 获取文档。是**根据词条找文档的过程**。

**正向索引**：

- 优点：
  - 可以给多个字段创建索引
  - 根据索引字段搜索、排序速度非常快
- 缺点：
  - 根据非索引字段，或者索引字段中的部分词条查找时，只能全表扫描。

**倒排索引**：

- 优点：
  - 根据词条搜索、模糊搜索时，速度非常快
- 缺点：
  - 只能给词条创建索引，而不是字段
  - 无法根据字段做排序

## IK 分词器

### 作用

- 创建倒排索引时对文档分词
- 对用户搜索内容分词

### IK 有几种模式？

- ik_smart：智能切分，粗粒度
- ik_max_word：最细切分，细粒度

### 如何拓展、停用词条

- 利用 config 目录的 IkAnalyzer.cfg.xml 文件添加拓展词典和停用词典
- 在词典中添加拓展词条或者停用词条

## mapping 映射属性

mapping 是对索引库中文档的约束，常见的 mapping 属性包括：

- type：字段数据类型，常见的简单类型有：
  - 字符串：text（可分词的文本）、keyword（精确值，例如：品牌、国家、ip 地址）
  - 数值：long、integer、short、byte、double、float、
  - 布尔：boolean
  - 日期：date
  - 对象：object
- index：是否创建索引，默认为 true
- analyzer：使用哪种分词器
- properties：该字段的子字段

## RestAPI

ES 官方提供了各种不同语言的客户端，用来操作 ES。这些客户端的**本质就是组装 DSL 语句，通过 http 请求发送给 ES**。官方文档地址：https://www.elastic.co/guide/en/elasticsearch/client/index.html

其中的 Java Rest Client 又包括两种：

- Java Low Level Rest Client
- **Java High Level Rest Client**

![image-20210720214555863](图集/image-20210720214555863.png)

我们学习的是 Java HighLevel Rest Client 客户端 API

```java
@SpringBootTest
public class HotelDocumentTest {
    @Autowired
    private IHotelService hotelService;
    private RestHighLevelClient client;

    @BeforeEach
    void setUp() {
        this.client = new RestHighLevelClient(RestClient.builder(
     HttpHost.create("http://192.168.150.101:9200")
        ));
    }

    @AfterEach
    void tearDown() throws IOException {
        this.client.close();
    }
}
```

## 索引库的 CRUD

这里我们统一使用**Kibana 编写 DSL**的方式来演示。

- 创建索引库：PUT /索引库名
- 查询索引库：GET /索引库名
- 删除索引库：DELETE /索引库名
- 添加字段：PUT /索引库名/\_mapping

索引库操作的基本步骤：

- 初始化 RestHighLevelClient
- 创建 XxxIndexRequest。XXX 是 Create、Get、Delete
- 准备 DSL（ Create 时需要，其它是无参）
- 发送请求。调用 RestHighLevelClient#indices().xxx()方法，xxx 是 create、exists、delete

例如下面的 json 文档：

```json
{
    "age": 21,
    "weight": 52.1,
    "isMarried": false,
    "info": "黑马程序员Java讲师",
    "email": "zy@itcast.cn",
    "score": [99.1, 99.5, 98.9],
    "name": {
        "firstName": "云",
        "lastName": "赵"
    }
}
```

对应的每个字段映射（mapping）：

- age：类型为 integer；参与搜索，因此需要 index 为 true；无需分词器
- weight：类型为 float；参与搜索，因此需要 index 为 true；无需分词器
- isMarried：类型为 boolean；参与搜索，因此需要 index 为 true；无需分词器
- info：类型为字符串，需要分词，因此是 text；参与搜索，因此需要 index 为 true；分词器可以用 ik_smart
- email：类型为字符串，但是不需要分词，因此是 keyword；不参与搜索，因此需要 index 为 false；无需分词器
- score：虽然是数组，但是我们只看元素的类型，类型为 float；参与搜索，因此需要 index 为 true；无需分词器
- name：类型为 object，需要定义多个子属性
  - name.firstName；类型为字符串，但是不需要分词，因此是 keyword；参与搜索，因此需要 index 为 true；无需分词器
  - name.lastName；类型为字符串，但是不需要分词，因此是 keyword；参与搜索，因此需要 index 为 true；无需分词器

### 创建索引库和映射

```json
PUT /heima
{
  "mappings": {
    "properties": {
      "info":{
        "type": "text",
        "analyzer": "ik_smart"
      },
      "email":{
        "type": "keyword",
        "index": "falsae"
      },
      "name":{
        "properties": {
          "firstName": {
            "type": "keyword"
          }
        }
      },
      // ... 略
    }
  }
}
```

```java
@Test
void createHotelIndex() throws IOException {
    // 1.创建Request对象
    CreateIndexRequest request = new CreateIndexRequest("hotel");
    // 2.准备请求的参数：DSL语句
    request.source(MAPPING_TEMPLATE, XContentType.JSON);
    // 3.发送请求
    client.indices().create(request, RequestOptions.DEFAULT);
}
```

### 查询

```
GET /heima
```

```java
@Test
void testExistsHotelIndex() throws IOException {
    // 1.创建Request对象
    GetIndexRequest request = new GetIndexRequest("hotel");
    // 2.发送请求
    boolean exists = client.indices().exists(request, RequestOptions.DEFAULT);
    // 3.输出
    System.err.println(exists ? "索引库已经存在！" : "索引库不存在！");
}
```

### 修改

倒排索引结构虽然不复杂，但是一旦数据结构改变（比如改变了分词器），就需要重新创建倒排索引，这简直是灾难。因此索引库**一旦创建，无法修改 mapping**。

但是却允许添加新的字段到 mapping 中，因为不会对倒排索引产生影响。

```json
PUT /heima/_mapping
{
  "properties": {
    "新字段名age":{
      "type": "integer"
    }
  }
}
```

### 删除

```
DELETE /heima
```

在 kibana 中测试：

![image-20210720212123420](图集/image-20210720212123420.png)

```java
@Test
void testDeleteHotelIndex() throws IOException {
    // 1.创建Request对象
    DeleteIndexRequest request = new DeleteIndexRequest("hotel");
    // 2.发送请求
    client.indices().delete(request, RequestOptions.DEFAULT);
}
```

## 文档的 CRUD

- 创建文档：POST /{索引库名}/\_doc/文档 id { json 文档 }
- 查询文档：GET /{索引库名}/\_doc/文档 id
- 删除文档：DELETE /{索引库名}/\_doc/文档 id
- 修改文档：
  - 全量修改：PUT /{索引库名}/\_doc/文档 id { json 文档 }
  - 增量修改：POST /{索引库名}/\_update/文档 id { "doc": {字段}}

文档操作的基本步骤：

- 初始化 RestHighLevelClient
- 创建 XxxRequest。XXX 是 Index、Get、Update、Delete、Bulk
- 准备参数（Index、Update、Bulk 时需要）
- 发送请求。调用 RestHighLevelClient#.xxx()方法，xxx 是 index、get、update、delete、bulk
- 解析结果（Get 时需要）

### 新增文档

```json
POST /索引库名/_doc/文档id
{
    "字段1": "值1",
    "字段2": "值2",
    "字段3": {
        "子属性1": "值3",
        "子属性2": "值4"
    },
    // ...
}
```

```json
POST /heima/_doc/1
{
    "info": "黑马程序员Java讲师",
    "email": "zy@itcast.cn",
    "name": {
        "firstName": "云",
        "lastName": "赵"
    }
}
```

![image-20210720212933362](图集/image-20210720212933362.png)

![image-20210720230027240](图集/image-20210720230027240.png)

### 查询

根据 rest 风格，新增是 post，查询应该是 get，不过查询一般都需要条件，这里我们把文档 id 带上。

```json
GET /{索引库名称}/_doc/{id}
```

**通过 kibana 查看数据：**

```js
GET /heima/_doc/1
```

![image-20210720213345003](图集/image-20210720213345003.png)

![image-20210720230811674](图集/image-20210720230811674-1686819138587.png)

```java
@Test
void testGetDocumentById() throws IOException {
    // 1.准备Request
    GetRequest request = new GetRequest("hotel", "61082");
    // 2.发送请求，得到响应
    GetResponse response = client.get(request, RequestOptions.DEFAULT);
    // 3.解析响应结果
    String json = response.getSourceAsString();
    HotelDoc hotelDoc = JSON.parseObject(json, HotelDoc.class);
    System.out.println(hotelDoc);
}
```

### 删除

删除使用 DELETE 请求，同样，需要根据 id 进行删除：

```json
DELETE /{索引库名}/_doc/id值
```

```json
# 根据id删除数据
DELETE /heima/_doc/1
```

![image-20210720213634918](图集/image-20210720213634918.png)

```java
@Test
void testDeleteDocument() throws IOException {
    // 1.准备Request
    DeleteRequest request = new DeleteRequest("hotel", "61083");
    // 2.发送请求
    client.delete(request, RequestOptions.DEFAULT);
}
```

### 修改

修改有两种方式：

- 全量修改：直接覆盖原来的文档
- 增量修改：修改文档中的部分字段

#### 全量修改

全量修改是覆盖原来的文档，其本质是：**先根据 id 删除**文档，再**新增一个相同 id 的文档**，如果删除时 id 不存在，直接新增

```json
PUT /{索引库名}/_doc/文档id
{
    "字段1": "值1",
    "字段2": "值2",
    // ... 略
}
```

```json
PUT /heima/_doc/1
{
    "info": "黑马程序员高级Java讲师",
    "email": "zy@itcast.cn",
    "name": {
        "firstName": "云",
        "lastName": "赵"
    }
}
```

#### 增量修改

增量修改是只修改指定 id 匹配的文档中的部分字段。

```json
POST /{索引库名}/_update/文档id
{
    "doc": {
         "字段名": "新的值",
    }
}
```

```json
POST /heima/_update/1
{
  "doc": {
    "email": "ZhaoYun@itcast.cn"
  }
}
```

![image-20210720231040875](图集/image-20210720231040875.png)

## 批量导入文档

利用 BulkRequest 批量将数据库数据导入到索引库中。**本质就是将多个普通的 CRUD 请求组合到一个 add 方法中再一起发送。**

add 方法中能添加的请求包括：

- IndexRequest，也就是新增
- UpdateRequest，也就是修改
- DeleteRequest，也就是删除

![image-20210720232431383](图集/image-20210720232431383.png)

```java
@Test
void testBulkRequest() throws IOException {
    // 批量查询酒店数据
    List<Hotel> hotels = hotelService.list();
    // 1.创建Request
    BulkRequest request = new BulkRequest();
    // 2.准备参数，添加多个新增的Request
    for (Hotel hotel : hotels) {
        // 2.1.转换为文档类型HotelDoc
        HotelDoc hotelDoc = new HotelDoc(hotel);
        // 2.2.创建新增文档的Request对象
        request.add(new IndexRequest("hotel")
                    .id(hotelDoc.getId().toString())
                    .source(JSON.toJSONString(hotelDoc), XContentType.JSON));
    }
    // 3.发送请求
    client.bulk(request, RequestOptions.DEFAULT);
}
```

## DSL 查询分类

Elasticsearch 提供了基于 JSON 的 DSL（[Domain Specific Language](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html)）来定义查询。常见的查询类型包括：

- **查询所有**：查询出所有数据，一般测试用。例如：match_all

- **全文检索（full text）查询**：利用分词器对用户输入内容分词，然后去倒排索引库中匹配。例如：
  - match：根据一个字段查询
  - multi_match：根据多个字段查询，参与查询字段越多，查询性能越差
- **精确查询**：根据精确词条值查找数据，一般是查找 keyword、数值、日期、boolean 等类型字段。例如：
  - ids
  - range：根据值的范围查询
  - term：根据词条精确值查询
- **地理（geo）查询**：根据经纬度查询。例如：
  - geo_distance
  - geo_bounding_box
- **复合（compound）查询**：复合查询可以将上述各种查询条件组合起来，合并查询条件。例如：
  - bool
  - function_score

## 全文检索查询

全文检索查询的基本流程如下：

- 对用户搜索的内容做分词，得到词条
- 根据词条去倒排索引库中匹配，得到文档 id
- 根据文档 id 找到文档，返回给用户

比较常用的场景包括：

- 商城的输入框搜索
- 百度输入框搜索

**match**和**multi_match**查询示例：

![image-20210721170720691](图集/image-20210721170720691.png)

可以看到，两种查询结果是一样的，为什么？

因为我们将 brand、name、business 值都利用**copy_to 复制到了 all 字段**中。因此你根据三个字段搜索，和根据 all 字段搜索效果当然一样了。但是**搜索字段越多，对查询性能影响越大**，因此**建议采用 copy_to 单字段查询**的方式。

```java
@Test
void testMatch() throws IOException {
    // 1.准备Request
    SearchRequest request = new SearchRequest("hotel");
    // 2.准备DSL
    request.source()
        .query(QueryBuilders.matchQuery("all", "如家"));
    // 3.发送请求
    SearchResponse response = client.search(request, RequestOptions.DEFAULT);
    // 4.解析响应
    handleResponse(response);
}
```

## 精确查询

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

![image-20210721171655308](图集/image-20210721171655308.png)

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

![image-20210721220305140](图集/image-20210721220305140.png)

## 地理（geo）查询

### 矩形范围查询

![DKV9HZbVS6](图集/DKV9HZbVS6.gif)

查询时，需要指定矩形的**左上**、**右下**两个点的坐标，然后画出一个矩形，落在该矩形内的都是符合条件的点。

```json
// geo_bounding_box矩形范围查询
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

### 附近/距离查询

![vZrdKAh19C](图集/vZrdKAh19C.gif)

```json
// geo_distance 查询
GET /indexName/_search
{
  "query": {
    "geo_distance": {
      "distance": "15km", // 半径
      "FIELD": "31.21,121.5" // 圆心
    }
  }
}
```

![image-20210721175443234](图集/image-20210721175443234.png)

## 复合查询

复合（compound）查询：复合查询可以将其它简单查询组合起来，实现更复杂的搜索逻辑。常见的有两种：

- fuction score：算分函数查询，可以控制文档相关性算分，控制文档排名
- bool query：布尔查询，利用逻辑关系组合多个其它的查询，实现复杂搜索

### 算分函数查询

**TF-IDF 算法**：有一各缺陷，就是词条频率越高，文档得分也会越高，单个词条对文档影响较大。

**BM25 算法**：elasticsearch5.1 版本后采用的算法，而 BM25 则会让单个词条的算分有一个上限，曲线更加平滑。

![image-20210721190907320](图集/image-20210721190907320.png)

![image-20210721191544750](图集/image-20210721191544750.png)

function score 查询中包含四部分内容：

- **原始查询**条件：query 部分，基于这个条件搜索文档，并且基于**BM25 算法**给文档打分，**原始算分**（query score)
- **过滤条件**：filter 部分，符合该条件的文档才会重新算分
- **算分函数**：符合 filter 条件的文档要根据这个函数做运算，得到的**函数算分**（function score），有四种函数
  - weight：函数结果是常量
  - field_value_factor：以文档中的某个字段值作为函数结果
  - random_score：以随机数作为函数结果
  - script_score：自定义算分函数算法
- **运算模式**：算分函数的结果、原始查询的相关性算分，两者之间的运算方式，包括：
  - multiply：相乘
  - replace：用 function score 替换 query score
  - 其它，例如：sum、avg、max、min

测试，在未添加算分函数时，如家得分如下：

![image-20210721193152520](图集/image-20210721193152520.png)

添加了算分函数后，如家得分就提升了：

![image-20210721193458182](图集/image-20210721193458182.png)

### 布尔查询

布尔查询是一个或多个查询子句的组合，每一个子句就是一个**子查询**。子查询的组合方式有：

- must：必须匹配每个子查询，类似“与”
- should：选择性匹配子查询，类似“或”
- must_not：必须不匹配，**不参与算分**，类似“非”
- filter：必须匹配，**不参与算分**

比如在搜索酒店时，除了关键字搜索外，我们还可能根据品牌、价格、城市等字段做过滤：

![image-20210721193822848](图集/image-20210721193822848.png)

每一个不同的字段，其查询的条件、方式都不一样，必须是多个不同的查询，而要组合这些查询，就必须用 bool 查询了。

需要注意的是，搜索时，参与**打分的字段越多，查询的性能也越差**。因此这种多条件查询时，建议这样做：

- 搜索框的关键字搜索，是全文检索查询，使用 must 查询，参与算分
- 其它过滤条件，采用 filter 查询。不参与算分

需求：搜索名字包含“如家”，价格不高于 400，在坐标 31.21,121.5 周围 10km 范围内的酒店。

分析：

- 名称搜索，属于全文检索查询，应该参与算分。放到 must 中
- 价格不高于 400，用 range 查询，属于过滤条件，不参与算分。放到 must_not 中
- 周围 10km 范围内，用 geo_distance 查询，属于过滤条件，不参与算分。放到 filter 中

![image-20210721194744183](图集/image-20210721194744183.png)

## 排序

elasticsearch 默认是根据相关度算分（\_score）来排序，但是也支持自定义方式对搜索[结果排序](https://www.elastic.co/guide/en/elasticsearch/reference/current/sort-search-results.html)。可以排序字段类型有：keyword 类型、数值类型、地理坐标类型、日期类型等。

### 普通字段排序

keyword、数值、日期类型排序的语法基本一致。

需求描述：酒店数据按照用户评价（score)降序排序，评价相同的按照价格(price)升序排序

![image-20210721195728306](图集/image-20210721195728306.png)

### 地理坐标排序

需求描述：实现对酒店数据按照到你的位置坐标的距离升序排序

提示：获取你的位置的经纬度的方式：https://lbs.amap.com/demo/jsapi-v2/example/map/click-to-get-lnglat/

假设我的位置是：31.034661，121.612282，寻找我周围距离最近的酒店。

![image-20210721200214690](图集/image-20210721200214690.png)

这个**sort 值就是实际距离**，可以**回显到前端显示**。

![image-20210722095902314](图集/image-20210722095902314.png)

![image-20210722100838604](图集/image-20210722100838604.png)

## 分页

elasticsearch 默认情况下只返回 top10 的数据。

- from：从第几个文档开始
- size：总共查询几个文档

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

### 深度分页问题

现在，我要查询 990~1000 的数据，查询逻辑要这么写：

当查询**分页深度较大**时，**汇总数据过多**，**对内存和 CPU 会产生非常大的压力**，因此 elasticsearch 会**禁止 from+ size 超过 10000**的请求。

分页查询的常见实现方案以及优缺点：

- `from + size`：
  - 优点：**支持随机翻页**
  - 缺点：深度分页问题，默认查询上限（from + size）是**10000**
  - 场景：百度、京东、谷歌、淘宝这样的随机翻页搜索
- `after search`：分页时需要排序，原理是从上一次的排序值开始，查询下一页数据。官方推荐。

  - 优点：没有查询上限（单次查询的 size 不超过 10000）
  - 缺点：只能**向后逐页**查询，不支持随机翻页
  - 场景：**没有随机翻页需求的搜索**，例如**手机向下滚动翻页**

- `scroll`：将排序后的**文档 id 形成快照**，**保存在内存**。官方已经**不推荐**使用。
  - 优点：没有查询上限（单次查询的 size 不超过 10000）
  - 缺点：会有额外内存消耗，并且**搜索结果是非实时的**
  - 场景：**海量数据的获取和迁移**。从 ES7.1 开始**不推荐**，建议用 after search 方案。

## 高亮

- 1）给文档中的所有关键字都添加一个标签，例如`<em>`标签
- 2）页面给`<em>`标签编写 CSS 样式

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
- 如果要对**非搜索字段高亮**，则需要添加一个属性：required_field_match=false

![image-20210721203349633](图集/image-20210721203349633.png)

## 总结

查询的 DSL 是一个大的 JSON 对象，包含下列属性：

- query：查询条件
- from 和 size：分页条件
- sort：排序条件
- highlight：高亮条件

![image-20210721203657850](图集/image-20210721203657850.png)

## 数据同步

常见的数据同步方案有三种：

- 同步调用
- 异步通知
- 监听 binlog

### 同步调用

![image-20210723214931869](图集/image-20210723214931869.png)

基本步骤如下：

- hotel-demo 对外提供接口，用来修改 elasticsearch 中的数据
- 酒店管理服务在完成数据库操作后，直接调用 hotel-demo 提供的接口，

### 异步通知

![image-20210723215140735](图集/image-20210723215140735.png)

流程如下：

- hotel-admin 对 mysql 数据库数据完成增、删、改后，发送 MQ 消息
- hotel-demo 监听 MQ，接收到消息后完成 elasticsearch 数据修改

### 监听 binlog

![image-20210723215518541](图集/image-20210723215518541.png)

流程如下：

- 给 mysql 开启 binlog 功能
- mysql 完成增、删、改操作都会记录在 binlog 中
- hotel-demo 基于 canal 监听 binlog 变化，实时更新 elasticsearch 中的内容

### 优缺点

方式一：同步调用

- 优点：实现简单，粗暴
- 缺点：业务耦合度高

方式二：异步通知

- 优点：低耦合，实现难度一般
- 缺点：依赖 mq 的可靠性

方式三：监听 binlog

- 优点：完全解除服务间耦合
- 缺点：开启 binlog 增加数据库负担、实现复杂度高

## mq 异步实现数据同步

### 思路

利用课前资料提供的 hotel-admin 项目作为酒店管理的微服务。当酒店数据发生增、删、改时，要求对 elasticsearch 中数据也要完成相同操作。

步骤：

- 导入课前资料提供的 hotel-admin 项目，启动并测试酒店数据的 CRUD

- 声明 exchange、queue、RoutingKey

- 在 hotel-admin 中的增、删、改业务中完成消息发送

- 在 hotel-demo 中完成消息监听，并更新 elasticsearch 中数据

- 启动并测试数据同步功能

### 声明交换机、队列

![image-20210723215850307](图集/image-20210723215850307.png)

#### 1）引入依赖

在 hotel-admin、hotel-demo 中引入 rabbitmq 的依赖：

```xml
<!--amqp-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

#### 2）声明队列交换机名称

在 hotel-admin 和 hotel-demo 中的`cn.itcast.hotel.constatnts`包下新建一个类`MqConstants`：

```java
package cn.itcast.hotel.constatnts;

    public class MqConstants {
    /**
     * 交换机
     */
    public final static String HOTEL_EXCHANGE = "hotel.topic";
    /**
     * 监听新增和修改的队列
     */
    public final static String HOTEL_INSERT_QUEUE = "hotel.insert.queue";
    /**
     * 监听删除的队列
     */
    public final static String HOTEL_DELETE_QUEUE = "hotel.delete.queue";
    /**
     * 新增或修改的RoutingKey
     */
    public final static String HOTEL_INSERT_KEY = "hotel.insert";
    /**
     * 删除的RoutingKey
     */
    public final static String HOTEL_DELETE_KEY = "hotel.delete";
}
```

#### 3）声明绑定队列交换机

在 hotel-demo 中，定义配置类，声明队列、交换机：

```java
package cn.itcast.hotel.config;

import cn.itcast.hotel.constants.MqConstants;
import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.Queue;
import org.springframework.amqp.core.TopicExchange;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MqConfig {
    @Bean
    public TopicExchange topicExchange(){
        return new TopicExchange(MqConstants.HOTEL_EXCHANGE, true, false);
    }

    @Bean
    public Queue insertQueue(){
        return new Queue(MqConstants.HOTEL_INSERT_QUEUE, true);
    }

    @Bean
    public Queue deleteQueue(){
        return new Queue(MqConstants.HOTEL_DELETE_QUEUE, true);
    }

    @Bean
    public Binding insertQueueBinding(){
        return BindingBuilder.bind(insertQueue()).to(topicExchange()).with(MqConstants.HOTEL_INSERT_KEY);
    }

    @Bean
    public Binding deleteQueueBinding(){
        return BindingBuilder.bind(deleteQueue()).to(topicExchange()).with(MqConstants.HOTEL_DELETE_KEY);
    }
}
```

### 发送 MQ 消息

在 hotel-admin 中的增、删、改业务中分别发送 MQ 消息：

![image-20210723221843816](图集/image-20210723221843816.png)

### 接收 MQ 消息

hotel-demo 接收到 MQ 消息要做的事情包括：

- 新增消息：根据传递的 hotel 的 id 查询 hotel 信息，然后新增一条数据到索引库
- 删除消息：根据传递的 hotel 的 id 删除索引库中的一条数据

1）首先在 hotel-demo 的`cn.itcast.hotel.service`包下的`IHotelService`中新增新增、删除业务

```java
void deleteById(Long id);
void insertById(Long id);
```

2）给 hotel-demo 中的`cn.itcast.hotel.service.impl`包下的 HotelService 中实现业务：

```java
@Override
public void deleteById(Long id) {
    try {
        // 1.准备Request
        DeleteRequest request = new DeleteRequest("hotel", id.toString());
        // 2.发送请求
        client.delete(request, RequestOptions.DEFAULT);
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
}

@Override
public void insertById(Long id) {
    try {
        // 0.根据id查询酒店数据
        Hotel hotel = getById(id);
        // 转换为文档类型
        HotelDoc hotelDoc = new HotelDoc(hotel);

        // 1.准备Request对象
        IndexRequest request = new IndexRequest("hotel").id(hotel.getId().toString());
        // 2.准备Json文档
        request.source(JSON.toJSONString(hotelDoc), XContentType.JSON);
        // 3.发送请求
        client.index(request, RequestOptions.DEFAULT);
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
}
```

3）编写监听器

在 hotel-demo 中的`cn.itcast.hotel.mq`包新增一个类：

```java
package cn.itcast.hotel.mq;

import cn.itcast.hotel.constants.MqConstants;
import cn.itcast.hotel.service.IHotelService;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class HotelListener {

    @Autowired
    private IHotelService hotelService;

    /**
     * 监听酒店新增或修改的业务
     * @param id 酒店id
     */
    @RabbitListener(queues = MqConstants.HOTEL_INSERT_QUEUE)
    public void listenHotelInsertOrUpdate(Long id){
        hotelService.insertById(id);
    }

    /**
     * 监听酒店删除的业务
     * @param id 酒店id
     */
    @RabbitListener(queues = MqConstants.HOTEL_DELETE_QUEUE)
    public void listenHotelDelete(Long id){
        hotelService.deleteById(id);
    }
}
```

## ES 集群

单机的 elasticsearch 做数据存储，必然面临两个问题：

- **海量数据存储**问题：将索引库从**逻辑上**拆分为 N 个**分片**（shard），**存储到多个节点**
- **单点故障**问题：将分片数据**在不同节点备份**（replica ）

数据备份可以保证高可用，但是每个分片备份一份，所需要的节点数量就会翻一倍，成本实在是太高了！**为了在高可用和成本间寻求平衡**：

- 首先对数据分片，存储到不同节点
- 然后对每个分片进行备份，放到对方节点，完成互相备份

这样可以大大减少所需要的服务节点数量，如图，我们以 3 分片，每个分片备份一份为例：

![image-20200104124551912](图集/image-20200104124551912.png)

现在，每个分片都有 1 个备份，存储在 3 个节点：

- node0：保存了分片 0 和 1
- node1：保存了分片 0 和 2
- node2：保存了分片 1 和 2

## 集群脑裂问题

### 集群职责划分

![image-20210723223629142](图集/image-20210723223629142.png)

elasticsearch 中集群节点有不同的职责划分：

![image-20210723223008967](图集/image-20210723223008967.png)

默认情况下，集群中的任何一个节点都同时具备上述四种角色。

但是真实的集群一定要将集群职责分离：

- master 节点：对**CPU 要求高**，但是**内存要求低**
- data 节点：对 CPU 和内存要求**都高**
- coordinating 节点：对**网络带宽、CPU 要求高**

职责分离可以让我们根据不同节点的需求分配不同的硬件去部署。而且避免业务之间的互相干扰。

### 脑裂问题

脑裂是因为集群中的节点失联导致的。

例如一个集群中，主节点与其它节点失联：

![image-20210723223804995](图集/image-20210723223804995.png)

此时，node2 和 node3 认为 node1 宕机，就会重新选主：

![image-20210723223845754](图集/image-20210723223845754.png)

当 node3 当选后，集群继续对外提供服务，node2 和 node3 自成集群，node1 自成集群，两个集群数据不同步，出现数据差异。

当网络恢复后，因为集群中有两个 master 节点，集群状态的不一致，出现脑裂的情况：

![image-20210723224000555](图集/image-20210723224000555.png)

**解决脑裂的方案**是，要求选票**超过 ( eligible 节点数量 + 1 ）/ 2** 才能当选为主，因此**eligible 节点数量最好是奇数**。对应配置项是 discovery.zen.minimum_master_nodes，在 es7.0 以后，已经成为默认配置，因此一般不会发生脑裂问题

例如：3 个节点形成的集群，选票必须超过 **（3 + 1） / 2 ，也就是 2 票**。node3 得到 node2 和 node3 的选票，当选为主。node1 只有自己 1 票，没有当选。集群中依然只有 1 个主节点，没有出现脑裂。

## 集群分布式存储

当新增文档时，应该保存到不同分片，保证数据均衡，那么 coordinating node 如何确定数据该存储到哪个分片呢？

### 分片存储原理

elasticsearch 会通过 hash 算法来计算文档应该存储到哪个分片：

![image-20210723224354904](图集/image-20210723224354904.png)

- \_routing 默认是文档的 id
- 算法与分片数量有关，因此索引库一旦创建，分片数量不能修改！

新增文档的流程如下：

![image-20210723225436084](图集/image-20210723225436084.png)

- 1）新增一个 id=1 的文档
- 2）对 id 做 hash 运算，假如得到的是 2，则应该存储到 shard-2
- 3）shard-2 的主分片在 node3 节点，将数据路由到 node3
- 4）保存文档
- 5）同步给 shard-2 的副本 replica-2，在 node2 节点
- 6）返回结果给 coordinating-node 节点

## 集群故障转移

集群的 master 节点会监控集群中的节点状态，如果发现有节点宕机，会立即将宕机节点的分片数据迁移到其它节点，确保数据安全，这个叫做故障转移。

突然，node1 发生了故障：

![image-20210723230020574](图集/image-20210723230020574.png)

宕机后的第一件事，需要重新选主，例如选中了 node2：

![image-20210723230055974](图集/image-20210723230055974.png)

node2 成为主节点后，会检测集群监控状态，发现：shard-1、shard-0 没有副本节点。因此需要将 node1 上的数据迁移到 node2、node3：

![image-20210723230216642](图集/image-20210723230216642.png)
