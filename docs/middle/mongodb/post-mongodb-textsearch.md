# 前言
针对mongodb全文索引进行研究。处理生产环境日志全文搜索相关需求，结论部分均写在文章前面部分,后半部分主要对技术选型和问题的分析进行详细的阐述。
事实上以实际的搜索速度来看，mongodb真的不太适合做大规模的全文索引，速度实在是有些慢。如果不想看后面的解决步骤直接看文章的前半段内容就可以了。

# 需求背景
需求：基于mongodb创建的日志表，搜索其内容。要求按时间范围、日志等级筛选。按日志内容模糊搜索。

数据规模（测试）：1千万
数据样例：见文章尾部

# 结论
- mongodb全文索引的限制是 $text之前只允许出现eq的判断 否则就报错
例如：
```
{{field1: 123},{$text:{$search:{"测试"}}}}【正确的】
{{createTime: {$gte: now}},{$text:{$search:{"测试"}}}}【错误的】
```
- mongodb的全文索引本质上还是一个B树，并不适合用作日志搜索这种字段相似度和重复率极高的场景。由于一个关键字可能直接匹配出大量数据，这会导致性能的明显下降，如果有类似的需求还是用es吧。
- 这里最后还是放弃了mongo的全文索引，问题太多。直接用taskId+时间范围+regex查询效率还是比较稳定的，后期会迁移到es中。

# 功能实现的设计思路
- 针对日志内容构建全文索引，数据字段见数据样例部分。
- 添加一个额外字段来存放分词后的数据，以弥补mongodb不支持中文分词的短板
```java
class LogEntity{
 public void setMessage(String message) {
        this.message = message;
        textSearch =  ToAnalysis.parse(content).toStringWithOutNature(" ");
    }
}

```
- 使用springdata mongo 来实现动态的查询(像用mysql动态参数一样，当入参为空时不添加对应字段的查询条件。)，目前没有找到MongoRepository支持全文搜索的证据，这里暂时认为它不支持全文搜索，以后找到了再更新用法。
```java
class LogSearch{

    @Autowired
    MongoTemplate mongoTemplate;

    /**
      * searchParam 前端传过来的请求参数
     **/
    public Page<LogEntity> fullTextSearchBulletinsByPage(QueryRequest searchParam) {
            if(StringUtils.isBlank(searchParam.getTaskId())) return Page.empty();
            PageRequest pageable = PageRequest.of(searchParam.getPageNo(), searchParam.getPageSize(), Sort.Direction.DESC, "createTime");
            Query query = new Query();
            Criteria c = new Criteria();
            if(searchParam.getLogStartTime() != null) {
                if(searchParam.getLogEndTime() != null){
                    c.and("createTime").gte(searchParam.getLogStartTime()).lte(searchParam.getLogEndTime());
                } else {
                    c.and("createTime").gte(searchParam.getLogStartTime());
                }
            } else if(searchParam.getLogEndTime() != null) {
                c.and("createTime").lte(searchParam.getLogEndTime());
            }
            c.and("taskId").is(searchParam.getTaskId());
            if(StringUtils.isNotBlank(searchParam.getLogLevel())) {
                c.and("level").is(searchParam.getLogLevel());
            }
            query.addCriteria(c);
            if(StringUtils.isNotBlank(searchParam.getSearchKey())) {
                TextCriteria tc = new TextCriteria().matching(searchParam.getSearchKey());
                query.addCriteria(tc);
            }
            long start = System.currentTimeMillis();
            long count = mongoTemplate.count(query, LogEntity.class);
            System.out.println("count结果："+count+"count用时："+(System.currentTimeMillis() - start));
            query.with(pageable.getSort()).skip(pageable.getOffset()).limit(pageable.getPageSize());
            start = System.currentTimeMillis();
            List<LogEntity> content = mongoTemplate.find(query,LogEntity.class);
            System.out.println("find用时："+(System.currentTimeMillis() - start));
            return new PageImpl<>(content, pageable, count);
        }
}

```

# 遇到的问题
- mongodb4.2全文索引不支持中文分词
- 全文索引查询速度慢
- 通过explain命令确定了使用$text全文检索后普通索引无效
- 日志表随着时间推移会导致集合过大，会产生撑爆磁盘或大幅降低全文检索速度的风险

# 解决方案
- 中文分词采用ansj_seg(支持NLP分词，使用简单性能较高，社区关注度高),添加一个额外的字段用来保存分词后的数据。如果是中英混合的文本则使用mongo默认的english语言类型的即可，如果是纯中文可以将语言设置为none.
```xml
<dependency>
    <groupId>org.ansj</groupId>
    <artifactId>ansj_seg</artifactId>
    <version>5.1.6</version>
</dependency>
```
- 创建联合全文索引，包含taskId和createTime字段
需要注意的是，mongodb全文联合索引有限制，官方解释为 $text之前的查询条件必须是eq，不能是范围查询，否则会报错。 这里的taskId是必传字段，主要用来缩减数据规模。
```mongojs
db.getCollection("bulletins").createIndex({
    taskId:1,
    textSearch: "text",
	createTime: -1
}, {
    name: "IDX_TEXT_SEARCH"
}, {background: false})

```
- 以createTime作为TTL索引，超过半个月的数据由mongodb自动删除以节约表空间
```mongojs
db.getCollection("bulletins").createIndex({
    createTime: -1
}, {
    name: "IDX_TTL_CREATE_TIME",
	expireAfterSeconds: 129600
}, {background: false})
```

# 数据样例
## 表结构
```mongojs
db.createCollection("bulletins");

db.getCollection("bulletins").createIndex({
    "$**": "text"
}, {
    name: "IDX_TEXT_TIMESTAMP",
    weights: {
        createTime: NumberInt("1"),
        textSearch: NumberInt("1")
    },
    "default_language": "english",
    "language_override": "language",
    textIndexVersion: NumberInt("3")
});

db.getCollection("bulletins").createIndex({
    createTime: NumberInt("1")
}, {
    name: "IDX_TEXT_CREATEITME"
});

```
## 数据结构
```mongojs
db.getCollection("mylog").insert( {
    _id: "c655cc11-1c7c-41d4-8fea-4d9b5620aebd",
    taskId: "a6a939c8-0171-1000-c4b6-03a5a5ce6a53",
    componentId: "",
    timestamp: "2020-05-01T06:31:01.611Z",
    level: "INFO",
    message: "mobile 小明food apple engilish 性能engilish food document exception overflow 开始库error 库小明overflow error exception error mobile apple error 搞测试搞测试库搞测试engilish 数据mobile ",
    createTime: ISODate("2020-04-30T22:31:01.647Z"),
    textSearch: "mobile   小明 food   apple   engilish   性能 engilish   food   document   exception   overflow   开始 库 error   库小明 overflow   error   exception   error   mobile   apple   error   搞 测试 搞 测试 库 搞 测试 engilish   数据 mobile  ",
} );

```


