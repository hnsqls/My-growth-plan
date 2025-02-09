## Day 5 

今日目标 全文分词搜索 模板开发  spring八股

## 全文模糊搜索 

为了实现全文模糊搜索功能，需要使用ElasticSearch 搜索引擎。

> mysql 也支持全文搜索为什么不使用？

>1. **数据库自带全文搜索**

适用于小型或中型项目，直接利用数据库的全文索引功能。

+ **MySQL**: `FULLTEXT` 索引，适用于 `MyISAM` 和 `InnoDB` 引擎。
+ **PostgreSQL**: 提供强大的 `tsvector` 和 `tsquery` 搜索功能。
+ **MongoDB**: `text` 索引支持全文搜索。

> **示例（MySQL）：**
>
> ## **1. 创建数据库和表**
>
> ```sql
> CREATE DATABASE fulltext_search_demo;
> USE fulltext_search_demo;
> 
> CREATE TABLE articles (
>     id INT AUTO_INCREMENT PRIMARY KEY,
>     title VARCHAR(255),
>     content TEXT,
>     FULLTEXT(title, content)  -- 创建全文索引
> );
> ```
>
> ------
>
> ## **2. 插入测试数据**
>
> ```sql
> INSERT INTO articles (title, content) VALUES
> ('Elasticsearch 教程', 'Elasticsearch 是一个开源的全文搜索引擎。'),
> ('MySQL 全文搜索', 'MySQL 提供 FULLTEXT 索引用于全文检索。'),
> ('Python 搜索库 Whoosh', 'Whoosh 是一个轻量级的全文搜索库，适用于 Python 应用。'),
> ('全文搜索介绍', '全文搜索是一种高效的信息检索技术。');
> ```
>
> ------
>
> ## **3. 执行全文搜索**
>
> ```sql
> 
> SELECT * FROM articles WHERE MATCH(title, content) AGAINST('全文搜索');
> ```
>
> + `MATCH(title, content) AGAINST('关键词')` 用于在 `title` 和 `content` 字段中执行全文搜索。
>
> + 例如，搜索 **"全文搜索"**，会返回与该关键词匹配的文章。
>
> + MySQL **默认不支持中文分词**
>
> + 如果你使用 **MySQL 5.7+ 或 MySQL 8.0**，可以启用 **Ngram 分词插件**，让 MySQL 支持中文分词。
>
> + ```sql
>   ALTER TABLE articles ADD FULLTEXT(title, content) WITH PARSER ngram;
>   ```
>
> + 然后你可以使用 `MATCH() AGAINST()` 进行搜索：
>
>   ```sql
>   
>   SELECT * FROM articles WHERE MATCH(title, content) AGAINST('全文搜索');
>   ```
>
>   **注意**: Ngram 分词默认会把每 2 个字符作为一个 "词"，对于中文搜索的效果比默认好，但不如专业搜索引擎（如 Elasticsearch）。
>
>   以上仅仅实现全文搜索的功能，并没有实现全文的模糊搜索功能。
>
>   如果只是想实现简单的**模糊匹配**，可以使用 `LIKE`：
>
>   ```sql
>   SELECT * FROM articles WHERE content LIKE '%全文搜索%';
>   
>   LIKE '%关键词%' 无法利用索引，查询速度较慢。
>   当数据量较大时，性能会明显下降。
>   不能提供相关性排序。
>   ```
>
>   可以先用 `FULLTEXT` 进行搜索，找出相关性较高的结果，再用 `LIKE` 进一步筛选。
>
>   ```sql
>   SELECT * FROM articles 
>   WHERE MATCH(title, content) AGAINST('全文搜索') 
>   AND content LIKE '%全文搜索%';
>   
>   # 这样可以提高查询的精确度，但性能仍然不如专业搜索引擎。
>   ```
>
>   设用场景：
>
>   小规模数据（几万到百万条数据）。
>
>   **适合博客、新闻网站等** 需要基本全文搜索的应用。
>
>   性能远远不如专业的搜索引擎。



## ELasticSearch

Elasticsearch 是一个分布式、开源的搜索引擎,广泛用于 **全文检索、日志分析、监控数据分析** 等场景。

[Elasticsearch 指南 [8.17\] | 弹性的](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)

Elastic Stack（也称为 ELK Stack）由以下几部分组成：

+ Elasticsearch：核心搜索引擎，负责存储、索引和搜索数据。
+ Kibana：可视化平台，用于查询、分析和展示 Elasticsearch 中的数据。
+ Logstash：数据处理管道，负责数据收集、过滤、增强和传输到 Elasticsearch。
+ Beats：轻量级的数据传输工具，收集和发送数据到 Logstash 或 Elasticsearch。

### 数据的存放形式

[What is Elasticsearch? | Elasticsearch Guide [7.17\] | Elastic](https://www.elastic.co/guide/en/elasticsearch/reference/7.17/elasticsearch-intro.html)

Elasticsearch 不是将信息存储为列式数据行，而是存储已序列化的复杂数据结构作为 JSON 文档。

Elasticsearch 使用一种称为倒排索引的数据结构，它支持非常快速的全文搜索。**倒排索引**列出任何文档中出现的每个唯一单词，并标识所有每个单词出现的文档。

索引可以看作是文档的优化集合，每个document 都是字段的集合，这些字段是包含数据的键值对。默认情况下，Elasticsearch 会为每个字段中的所有数据编制索引，并且每个索引的字段都有一个专用的、优化的数据结构。例如，文本字段是存储在倒排索引中，而数字和地理字段存储在 BKD 树中。使用每个字段的数据结构来组合和返回搜索结果的能力是 Elasticsearch 如此快速的原因。

与mysql进行类比

**索引（Index）**：类似于关系型数据库中的表

**文档（Document）**：索引中的每条记录，类似于数据库中的行。文档以 JSON 格式存储。

**字段（Field）**：文档中的每个键值对，类似于数据库中的列。

**映射（Mapping）**：用于定义 Elasticsearch 索引中文档字段的数据类型及其处理方式，类似于关系型数据库中的 Schema 表结构，帮助控制字段的存储、索引和查询行为。

集群（Cluster）：多个节点组成的群集，用于存储数据并提供搜索功能。集群中的每个节点都可以处理数据。

分片（Shard）：为了实现横向扩展，ES 将索引拆分成多个分片，每个分片可以分布在不同节点上。

副本（Replica）：分片的复制品，用于提高可用性和容错性。

### 快速使用

[快速入门 |Elasticsearch 指南 [7.17\] | 弹性的](https://www.elastic.co/guide/en/elasticsearch/reference/7.17/getting-started.html#send-requests-to-elasticsearch)

widow 下安装

1.下载解压即可：https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.17.27-windows-x86_64.zip4

2.启动 bin/elasticsearch.bat    查看是否成功[localhost:9200](http://localhost:9200/)

3.一般不需要配置，配置的话看[配置 Elasticsearch |Elasticsearch 指南 [7.17\] | 弹性的 ](https://www.elastic.co/guide/en/elasticsearch/reference/7.17/settings.html)



> 查询语法DSL

```json
{
  "query": {
    "match": {
      "message": "Elasticsearch 是强大的"
    }
  }
}
```

match方法是全文分词查询。这会对 `message` 字段进行分词，并查找包含 "Elasticsearch" 和 "强大" 词条的文档。



| **查询条件**   | **介绍**                                                     | **示例**                                                     | **用途**                                           |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | -------------------------------------------------- |
| `match`        | 用于全文检索，将查询字符串进行分词并匹配文档中对应的字段。   | `{ "match": { "content": "鱼皮是帅小伙" } }`                 | 适用于全文检索，分词后匹配文档内容。               |
| `term`         | 精确匹配查询，不进行分词。通常用于结构化数据的精确匹配，如数字、日期、关键词等。 | `{ "term": { "status": "active" } }`                         | 适用于字段的精确匹配，如状态、ID、布尔值等。       |
| `terms`        | 匹配多个值中的任意一个，相当于多个 `term` 查询的组合。       | `{ "terms": { "status": ["active", "pending"] } }`           | 适用于多值匹配的场景。                             |
| `range`        | 范围查询，常用于数字、日期字段，支持大于、小于、区间等查询。 | `{ "range": { "age": { "gte": 18, "lte": 30 } } }`           | 适用于数值或日期的范围查询。                       |
| `bool`         | 组合查询，通过 `must`、`should`、`must_not` 等组合多个查询条件。 | `{ "bool": { "must": [ { "term": { "status": "active" } }, { "range": { "age": { "gte": 18 } } } ] } }` | 适用于复杂的多条件查询，可以灵活组合。             |
| `wildcard`     | 通配符查询，支持 `*` 和 `?`，前者匹配任意字符，后者匹配单个字符。 | `{ "wildcard": { "name": "鱼*" } }`                          | 适用于部分匹配的查询，如模糊搜索。                 |
| `prefix`       | 前缀查询，匹配以指定前缀开头的字段内容。                     | `{ "prefix": { "name": "鱼" } }`                             | 适用于查找以指定字符串开头的内容。                 |
| `fuzzy`        | 模糊查询，允许指定程度的拼写错误或字符替换。                 | `{ "fuzzy": { "name": "yupi~2" } }`                          | 适用于处理拼写错误或不完全匹配的查询。             |
| `exists`       | 查询某字段是否存在。                                         | `{ "exists": { "field": "name" } }`                          | 适用于查找字段存在或缺失的文档。                   |
| `match_phrase` | 短语匹配查询，要求查询的词语按顺序完全匹配。                 | `{ "match_phrase": { "content": "鱼皮 帅小伙" } }`           | 适用于严格的短语匹配，词语顺序和距离都严格控制。   |
| `match_all`    | 匹配所有文档。                                               | `{ "match_all": {} }`                                        | 适用于查询所有文档，通常与分页配合使用。           |
| `ids`          | 基于文档 ID 查询，支持查询特定 ID 的文档。                   | `{ "ids": { "values": ["1", "2", "3"] } }`                   | 适用于根据文档 ID 查找特定文档。                   |
| `geo_distance` | 地理位置查询，基于地理坐标和指定距离查询。                   | `{ "geo_distance": { "distance": "12km", "location": { "lat": 40.73, "lon": -74.1 } } }` | 适用于根据距离计算查找地理位置附近的文档。         |
| `aggregations` | 聚合查询，用于统计、计算和分组查询，类似 SQL 中的 `GROUP BY`。 | `{ "aggs": { "age_stats": { "stats": { "field": "age" } } } }` | 适用于统计和分析数据，比如求和、平均值、最大值等。 |

> 中文分词器

```json
POST /_analyze
{
  "analyzer": "standard", 
  "text": "hnsqls非常喜欢编程"
}
```

在 Elasticsearch 中使用 `_analyze` API 时，并不需要在库中已有数据，它只是用于分析指定的文本，查看它在指定分析器下的分词结果。你可以直接调用该 API，不必担心是否有索引或数据。

standard 分词器只会分英文，我们需要下载中午分词器插件

开源地址：https://github.com/medcl/elasticsearch-analysis-ik

进入 ES目录/pluglin 安装

```shell
.\bin\elasticsearch-plugin.bat install https://release.infinilabs.com/analysis-ik/stable/elasticsearch-analysis-ik-7.17.23.zip
```

重启ES

IK 分词器插件提供了两个分词器：`ik_smart` 和 `ik_max_word`。

+ ik_smart 是智能分词，尽量选择最像一个词的拆分方式，比如“好学生”会被识别为一个词， **搜索分词**
+ ik_max_word 尽可能地分词，可以包括组合词，比如“好学生”会被识别为 3 个词：好学生、好学、学生，**底层索引分词**





> 设计索引

text（长文本，需要分词的），keyword（不需要分词），long（长整数）

**字段类型**：定义每个字段的数据类型。常见类型包括：

+ `text`：用于全文搜索的字符串类型，会进行分词。
+ `keyword`：用于精确匹配的字符串类型，不进行分词。
+ `date`：日期类型，支持各种日期格式。
+ `integer`, `long`, `float`, `double`：数值类型。
+ `boolean`：布尔值类型。
+ `object`：对象类型，适合嵌套复杂结构。

**分词器**（Analyzers）：控制字段在索引时如何被分词。如果字段需要全文搜索，通常会使用分词器来把文本切分为单词。

**索引设置**（Index Settings）：定义一些全局的索引设置，如分片数、副本数等。

参考如下

```json
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "ik_max_word",
        "search_analyzer": "ik_smart",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      },
      "content": {
        "type": "text",
        "analyzer": "ik_max_word",
        "search_analyzer": "ik_smart"
      },
      "tags": {
        "type": "keyword"
      },
      "answer": {
        "type": "text",
        "analyzer": "ik_max_word",
        "search_analyzer": "ik_smart"
      },
      "userId": {
        "type": "long"
      },
      "editTime": {
        "type": "date",
        "format": "yyyy-MM-dd HH:mm:ss"
      },
      "createTime": {
        "type": "date",
        "format": "yyyy-MM-dd HH:mm:ss"
      },
      "updateTime": {
        "type": "date",
        "format": "yyyy-MM-dd HH:mm:ss"
      },
      "isDelete": {
        "type": "keyword"
      }
    }
  }
}

```

创建索引

```json
PUT /question_v1
{
  "mappings": {
    "properties": {
      ...
    }
  }
}
```

不过一般添加别名创建

* 零停机切换索引：在更新索引或重新索引数据时，你可以创建一个新索引并使用 alias 切换到新索引，而不需要修改客户端查询代码，避免停机或中断服务。
* 简化查询：通过 alias，可以使用一个统一的名称进行查询，而不需要记住具体的索引名称（尤其当索引有版本号或时间戳时）。
* 索引分组：alias 可以指向多个索引，方便对多个索引进行联合查询，例如用于跨时间段的日志查询或数据归档。

> 如何实现零停机切换。

例如一个索引 `products`，创建别名 `current_products`，并将其指向 `products_v1`。

当需要对 `products_v1` 进行更新时，你可以创建一个新的索引 `products_v2`，然后将别名 `current_products` 指向 `products_v2`。此时，应用程序只需要访问 `current_products`，而不必修改代码或配置。

```json
PUT /products_v1
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1
  },
  "mappings": {
    "properties": {
      "name": { "type": "text" },
      "price": { "type": "float" }
    }
  }
}

# 创建别名
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "products_v1",  // 指定索引
        "alias": "current_products"  // 为索引添加别名
      }
    }
  ]
}

```

当你需要切换索引时，可以直接操作别名来实现无缝切换。

假设你创建了一个新的索引 `products_v2`，并希望将别名 `current_products` 从 `products_v1` 切换到 `products_v2`。

```json
POST /_aliases
{
  "actions": [
    {
      "remove": {
        "index": "products_v1",
        "alias": "current_products"
      }
    },
    {
      "add": {
        "index": "products_v2",
        "alias": "current_products"
      }
    }
  ]
}
```





> springboot 集成操作

https://spring.io/projects/spring-data-elasticsearch/

1. 引入依赖

```xml
<!-- elasticsearch-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
</dependency>

```

2. 配置文件

```yaml
spring:
  elasticsearch:
    uris: http://xxx:9200
    username: elastic
    password: elastic

```

3. 使用

可以想mybatis 那样使用

1. 实体和ES库映射

model.dto.question

```java
@Document(indexName = "question")
@Data
public class QuestionEsDTO implements Serializable {

    private static final String DATE_TIME_PATTERN = "yyyy-MM-dd HH:mm:ss";

    /**
     * id
     */
    @Id
    private Long id;

    /**
     * 标题
     */
    private String title;

    /**
     * 内容
     */
    private String content;

    /**
     * 答案
     */
    private String answer;

    /**
     * 标签列表
     */
    private List<String> tags;

    /**
     * 创建用户 id
     */
    private Long userId;

    /**
     * 创建时间
     */
    @Field(type = FieldType.Date, format = {}, pattern = DATE_TIME_PATTERN)
    private Date createTime;

    /**
     * 更新时间
     */
    @Field(type = FieldType.Date, format = {}, pattern = DATE_TIME_PATTERN)
    private Date updateTime;

    /**
     * 是否删除
     */
    private Integer isDelete;

    private static final long serialVersionUID = 1L;

    /**
     * 对象转包装类
     *
     * @param question
     * @return
     */
    public static QuestionEsDTO objToDto(Question question) {
        if (question == null) {
            return null;
        }
        QuestionEsDTO questionEsDTO = new QuestionEsDTO();
        BeanUtils.copyProperties(question, questionEsDTO);
        String tagsStr = question.getTags();
        if (StringUtils.isNotBlank(tagsStr)) {
            questionEsDTO.setTags(JSONUtil.toList(tagsStr, String.class));
        }
        return questionEsDTO;
    }

    /**
     * 包装类转对象
     *
     * @param questionEsDTO
     * @return
     */
    public static Question dtoToObj(QuestionEsDTO questionEsDTO) {
        if (questionEsDTO == null) {
            return null;
        }
        Question question = new Question();
        BeanUtils.copyProperties(questionEsDTO, question);
        List<String> tagList = questionEsDTO.getTags();
        if (CollUtil.isNotEmpty(tagList)) {
            question.setTags(JSONUtil.toJsonStr(tagList));
        }
        return question;
    }
}

```

2. 接口继承 ElasticsearchRepository

```java
public interface QuestionEsDao 
    extends ElasticsearchRepository<QuestionEsDTO, Long> {

}
```

里面有常用的方法。

>  数据写入到ES

CommandLineRunner 实现这个接口的类，在类启动的时候会自动执行一次。所以说执行一次这个方法就行了，不执行在去掉注释，也就是不要让Springg感知到

```java
// todo 取消注释开启任务
@Component
@Slf4j
public class FullSyncQuestionToEs implements CommandLineRunner {

    @Resource
    private QuestionService questionService;

    @Resource
    private QuestionEsDao questionEsDao;

    @Override
    public void run(String... args) {
        // 全量获取题目（数据量不大的情况下使用）
        List<Question> questionList = questionService.list();
        if (CollUtil.isEmpty(questionList)) {
            return;
        }
        // 转为 ES 实体类
        List<QuestionEsDTO> questionEsDTOList = questionList.stream()
                .map(QuestionEsDTO::objToDto)
                .collect(Collectors.toList());
        // 分页批量插入到 ES
        final int pageSize = 500;
        int total = questionEsDTOList.size();
        log.info("FullSyncQuestionToEs start, total {}", total);
        for (int i = 0; i < total; i += pageSize) {
            // 注意同步的数据下标不能超过总数据量
            int end = Math.min(i + pageSize, total);
            log.info("sync from {} to {}", i, end);
            questionEsDao.saveAll(questionEsDTOList.subList(i, end));
        }
        log.info("FullSyncQuestionToEs end, total {}", total);
    }
}

```



> ES 全文分词搜索

接口 QuestionService

```java
/**
 * 从 ES 查询题目
 *
 * @param questionQueryRequest
 * @return
 */
Page<Question> searchFromEs(QuestionQueryRequest questionQueryRequest);

```

实现类

```java
@Override
public Page<Question> searchFromEs(QuestionQueryRequest questionQueryRequest) {
    // 获取参数
    Long id = questionQueryRequest.getId();
    Long notId = questionQueryRequest.getNotId();
    String searchText = questionQueryRequest.getSearchText();
    List<String> tags = questionQueryRequest.getTags();
    Long questionBankId = questionQueryRequest.getQuestionBankId();
    Long userId = questionQueryRequest.getUserId();
    // 注意，ES 的起始页为 0
    int current = questionQueryRequest.getCurrent() - 1;
    int pageSize = questionQueryRequest.getPageSize();
    String sortField = questionQueryRequest.getSortField();
    String sortOrder = questionQueryRequest.getSortOrder();

    // 构造查询条件
    BoolQueryBuilder boolQueryBuilder = QueryBuilders.boolQuery();
    // 过滤
    boolQueryBuilder.filter(QueryBuilders.termQuery("isDelete", 0));
    if (id != null) {
        boolQueryBuilder.filter(QueryBuilders.termQuery("id", id));
    }
    if (notId != null) {
        boolQueryBuilder.mustNot(QueryBuilders.termQuery("id", notId));
    }
    if (userId != null) {
        boolQueryBuilder.filter(QueryBuilders.termQuery("userId", userId));
    }
    if (questionBankId != null) {
        boolQueryBuilder.filter(QueryBuilders.termQuery("questionBankId", questionBankId));
    }
    // 必须包含所有标签
    if (CollUtil.isNotEmpty(tags)) {
        for (String tag : tags) {
            boolQueryBuilder.filter(QueryBuilders.termQuery("tags", tag));
        }
    }
    // 按关键词检索
    if (StringUtils.isNotBlank(searchText)) {
        boolQueryBuilder.should(QueryBuilders.matchQuery("title", searchText));
        boolQueryBuilder.should(QueryBuilders.matchQuery("content", searchText));
        boolQueryBuilder.should(QueryBuilders.matchQuery("answer", searchText));
        boolQueryBuilder.minimumShouldMatch(1);
    }
    // 排序
    SortBuilder<?> sortBuilder = SortBuilders.scoreSort();
    if (StringUtils.isNotBlank(sortField)) {
        sortBuilder = SortBuilders.fieldSort(sortField);
        sortBuilder.order(CommonConstant.SORT_ORDER_ASC.equals(sortOrder) ? SortOrder.ASC : SortOrder.DESC);
    }
    // 分页
    PageRequest pageRequest = PageRequest.of(current, pageSize);
    // 构造查询
    NativeSearchQuery searchQuery = new NativeSearchQueryBuilder()
            .withQuery(boolQueryBuilder)
            .withPageable(pageRequest)
            .withSorts(sortBuilder)
            .build();
    SearchHits<QuestionEsDTO> searchHits = elasticsearchRestTemplate.search(searchQuery, QuestionEsDTO.class);
    // 复用 MySQL 的分页对象，封装返回结果
    Page<Question> page = new Page<>();
    page.setTotal(searchHits.getTotalHits());
    List<Question> resourceList = new ArrayList<>();
    if (searchHits.hasSearchHits()) {
        List<SearchHit<QuestionEsDTO>> searchHitList = searchHits.getSearchHits();
        for (SearchHit<QuestionEsDTO> questionEsDTOSearchHit : searchHitList) {
            resourceList.add(QuestionEsDTO.dtoToObj(questionEsDTOSearchHit.getContent()));
        }
    }
    page.setRecords(resourceList);
    return page;
}

```

> 数据一致性

在上述代码中我们只是一次将数据库加入到ES 库中。后续如果有新增的数据或者修改原有的数据，就出现数据库不一致的问题。

解决方案

​	方案1： 双写，写数据库的同时，写ES 库。代码耦合度太高了，而且影响主任务。

​	方案2： 定时任务，没分钟执行一次，搜索近5分钟修改和创建的题目。加入到ES库。

​	方案3： 使用Canal监听binlog。



使用方案2：

注意，如果使用 MyBatis Plus 提供的 mapper 方法，查询时会默认过滤掉 isDelete = 1（逻辑已删除）的数据，而我们的需求是让 ES 和 MySQL 完全同步，所以需要在 QuestionMapper 中编写一个能查询出 isDelete = 1 数据的方法。

```java
public interface QuestionMapper extends BaseMapper<Question> {

    /**
     * 查询题目列表（包括已被删除的数据）
     */
    @Select("select * from question where updateTime >= #{minUpdateTime}")
    List<Question> listQuestionWithDelete(Date minUpdateTime);
}
```



```java
// todo 取消注释开启任务
//@Component
@Slf4j
public class IncSyncQuestionToEs {

    @Resource
    private QuestionMapper questionMapper;

    @Resource
    private QuestionEsDao questionEsDao;

    /**
     * 每分钟执行一次
     */
    @Scheduled(fixedRate = 60 * 1000)
    public void run() {
        // 查询近 5 分钟内的数据
        long FIVE_MINUTES = 5 * 60 * 1000L;
        Date fiveMinutesAgoDate = new Date(new Date().getTime() - FIVE_MINUTES);
        List<Question> questionList = questionMapper.listQuestionWithDelete(fiveMinutesAgoDate);
        if (CollUtil.isEmpty(questionList)) {
            log.info("no inc question");
            return;
        }
        List<QuestionEsDTO> questionEsDTOList = questionList.stream()
                .map(QuestionEsDTO::objToDto)
                .collect(Collectors.toList());
        final int pageSize = 500;
        int total = questionEsDTOList.size();
        log.info("IncSyncQuestionToEs start, total {}", total);
        for (int i = 0; i < total; i += pageSize) {
            int end = Math.min(i + pageSize, total);
            log.info("sync from {} to {}", i, end);
            questionEsDao.saveAll(questionEsDTOList.subList(i, end));
        }
        log.info("IncSyncQuestionToEs end, total {}", total);
    }
}

```





## 其他

### 1. 什么是倒排索引

1. **倒排索引（Inverted Index）** 是搜索引擎中用于实现高效全文搜索的核心数据结构。它的主要作用是将文档中的内容（通常是文本）转换为一种易于快速查找的形式，从而支持快速检索包含特定关键词的文档。、

   倒排索引是一种将词条（Term）映射到文档的数据结构。与传统的“正排索引”（文档到词条的映射）不同，倒排索引是从词条到文档的映射。

倒排索引通常由两部分组成：

1. **词条词典（Term Dictionary）**：
+ 存储所有唯一的词条。
   + 通常按字典序排序，方便快速查找。
   
2. **倒排列表（Posting List）**：
+ 存储每个词条对应的文档列表。
  

2.**倒排索引的构建过程**

   倒排索引的构建通常包括以下步骤：

   1. **分词（Tokenization）**：
      + 将文档中的文本分割成单独的词条。
      + 例如：`"Elasticsearch is fast"` -> `["Elasticsearch", "is", "fast"]`。
   2. **去除停用词（Stop Words Removal）**：
      + 去除无意义的词条（如“的”、“是”等）。
   3. **词条规范化（Normalization）**：
      + 将词条转换为统一形式（如小写化、词干提取等）。
      + 例如：`"running"` -> `"run"`。
   4. **构建倒排索引**：
      + 将词条映射到文档列表。
      + 例如：`"fast"` -> `[文档1]`。

 2.**倒排索引的查询过程**

当用户输入查询时，搜索引擎会使用倒排索引快速找到包含查询词条的文档。查询过程通常包括以下步骤：

1. **分词**：
   + 将查询字符串分割成词条。
   + 例如：`"fast search"` -> `["fast", "search"]`。
2. **查找倒排列表**：
   + 在倒排索引中查找每个词条对应的文档列表。
   + 例如：`"fast"` -> `[文档1]`，`"search"` -> `[文档2]`。
3. **合并结果**：
   + 根据查询逻辑（如 AND、OR）合并文档列表。
   + 例如：`"fast AND search"` -> 取 `[文档1]` 和 `[文档2]` 的交集。

 4. **倒排索引的优势**

    1. **高效检索**：
       + 通过词条直接定位文档，避免了全表扫描。
    2. **支持复杂查询**：
       + 支持布尔查询（AND、OR、NOT）、短语查询、范围查询等。
    3. **可扩展性**：
       + 适合处理大规模文本数据。
    4. **支持相关性排序**：
       + 可以结合词频（TF）、逆文档频率（IDF）等算法计算文档的相关性评分。



> 反思

ES库设计，同步，全文模糊搜索。

计划的八股没看。每天把计划要看的八股挑选一题对我不完全熟悉的写出来。

看文档的时间比写代码的时间长。还是要多实操，coding。

学习时间太少了。中午直接睡过去。下午学俩小时就不想学了。晚上开始学习的时间又太晚。

希望明天上午学俩小时（学一点也行。起太晚了）。继续记录

> 明日计划

批处理。 开发模板编写。 JUC八股。 java基础八股 java集合，mysql、spring。

早带睡觉。







