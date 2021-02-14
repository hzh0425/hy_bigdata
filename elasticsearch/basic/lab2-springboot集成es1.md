本文讲述springboot中集成使用ElasticSearch的步骤，注意，需要安装启动好ElasticSearch环境；
如果还有安装好环境，可以按照此链接步骤进行： [《ElasticSearch系列（一）linux环境ElasticSearch+Kibana（6.8.2）下载安装启动步骤》](https://blog.csdn.net/csdn_20150804/article/details/98313143)



es本身没有客户端界面，为了数据查询方便，我们使用head插件[《ElasticSearch系列（三）linux环境中安装配置head插件以及使用方法》](https://blog.csdn.net/csdn_20150804/article/details/105457041)

## 一、创建springboot项目

### 1.创建springboot web项目

这个比较简单，不详细说了。
注意本文使用**springboot版本是2.2.0，此本版内部依赖的ES客户端版本是6.8.1**。

### 2.在pom文件中增加es依赖

```java
<!--es-->
 <dependency>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
 </dependency>
12345
```

版本号就是跟随springboot版本；

**注意：**
如果es服务安装的是es7及以上版本，需要springboot版本为2.2.0及以上，不然启动项目会提示failed load nodes…
本文服务端安装的是ES7.3.0，支持的客户端版本最低是6.8.0，所以需要springboot依赖的es版本超过6.8.0。

如果要自定义ES版本号，指定下就行：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200419100934772.png)
但是这里不建议自定义，除非你使用es原生api，而不是springboot封装的api，不然有些会报错。

### 3.配置appication.yml

```java
spring:
  data:
    elasticsearch:
      cluster-name: myes
      cluster-nodes: 192.168.32.129:9301,192.168.32.129:9302,192.168.32.129:9303
12345
```

- 注意端口号是9300，不是9200；
- cluster-name是集群名称，需要在ES的config配置文件中指定，不然启动项目访问接口时，会报如下错误：
  None of the configured nodes are available

注意，在ES7.0版本后，**上述配置已经废弃**，推荐使用基于http的REST通信（https://docs.spring.io/spring-data/elasticsearch/docs/3.2.6.RELEASE/reference/html/#elasticsearch.clients.transport），如下配置：

```java
@Configuration
public class EsConfig {
    @Bean
    RestHighLevelClient client() {
        ClientConfiguration clientConfiguration = ClientConfiguration.builder()
                .connectedTo("192.168.17.101:9201", "192.168.17.101:9202", "192.168.17.101:9203")
                .build();
        return RestClients.create(clientConfiguration).rest();
    }
}
12345678910
```

注意通信端口也使用9200，这个是http的。

## 二、代码

### 1.编写实体类对象

```java
@Document(indexName = "index_blog", type = "article")
@Data
public class ArticleEntity {
    @Id
    @Field(type = FieldType.Long, store = true)
    private Long id;

    @Field(type = FieldType.Text, store = true, analyzer = "ik_smart")
    private String title;

    @Field(type = FieldType.Text, store = true, analyzer = "ik_smart")
    private String content;
}
12345678910111213
```

其中，

- @Document注解，指定了es搜索引擎的索引和类型（7.0后不用指定类型）；
- @Field注解，指定字段类型，以及是否存储，还有**分词器类型等；这里注意，在查询时，查询参数分词后是and关系，不是or的关系。**

默认es只有标准分词器，这里使用的是IK分词器，需要单独配置，可参考文章[《ElasticSearch系列（四）ES集成IK分词器以及使用方式》](https://blog.csdn.net/csdn_20150804/article/details/105474229)

**另外，Elasticsearch和关系数据库概念对应关系：**

| 关系数据库 =>    | 数据库      | 表         | 行             | 列           |
| ---------------- | ----------- | ---------- | -------------- | ------------ |
| Elasticsearch => | 索引(Index) | 类型(type) | 文档(Docments) | 字段(Fields) |

**注意：**
ES7中，已经废弃了type的概念，默认使用_doc作为类型名；也就是一个索引中，只能存在一个类型（一个表），那就是 _toc。
创建mapping的时候，不用指定type这一层级，否则报错。

### 2.编写dao层接口

```java
@Repository
public interface ArticleRepository extends ElasticsearchRepository<ArticleEntity, Long> {
}

```

其中，
@Repository注解可以使此接口被spring扫描到，如果不加此注解，也可以手动配置dao层的包扫描路径：

```java
@SpringBootApplication
//添加dao层包扫描
@EnableElasticsearchRepositories(basePackages = "com.example.demo.es.dao")
public class DemoApplication extends SpringBootServletInitializer {
1234
```

ElasticsearchRepository接口，提供了一些增删改查等众多方法，集成它就可以直接使用这些方法了。

### 3.编写controller层接口

```java
@RestController
@RequestMapping("/article")
public class EsArticleController {
    @Autowired
    private ArticleRepository articleReposiory;

    @PostMapping("/addArticle")
    public ArticleEntity addArticle(@RequestBody ArticleEntity article) {
        return articleReposiory.save(article);
    }

    @GetMapping("/findArticle/{id}")
    public Optional<ArticleEntity> findArticle(@PathVariable Long id) {
        return articleReposiory.findById(String.valueOf(id));
    }

    @DeleteMapping("/delete/{id}")
    public void deleteArticle(@PathVariable Long id) {
        articleReposiory.deleteById(String.valueOf(id));
    }
    
}
12345678910111213141516171819202122
```

## 三. 测试

启动项目，spring就是自动帮助创建代码中指定过的索引，比如本文中写了个ArticleEntity，指定了index为"index_blog"，启动项目后，登陆head插件，发现已经创建好了索引：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200418160915137.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NzZG5fMjAxNTA4MDQ=,size_16,color_FFFFFF,t_70)
索引信息：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200418161008368.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NzZG5fMjAxNTA4MDQ=,size_16,color_FFFFFF,t_70)

### 1.添加数据

使用postman访问下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200418161118391.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NzZG5fMjAxNTA4MDQ=,size_16,color_FFFFFF,t_70)
使用head插件看下存进去的数据：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200418161224880.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NzZG5fMjAxNTA4MDQ=,size_16,color_FFFFFF,t_70)

### 2.删除文档

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020041816295944.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NzZG5fMjAxNTA4MDQ=,size_16,color_FFFFFF,t_70)

### 3.修改文档

ES的修改，过程是先删除后添加，因此修改方法就是添加方法，只要保证id相同就行。

### 4.根据id查询数据

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200418161651328.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NzZG5fMjAxNTA4MDQ=,size_16,color_FFFFFF,t_70)

### 5.查询所有

```
articleReposiory.findAll();
1
```

### 6.自定义查询

上述查询，使用的都是articleReposiory自带的方法，实现一些简单查询，但是如果想实现复杂一点查询，比如根据ArticleEntity中的title字段查询：

```java
public interface ArticleRepository extends ElasticsearchRepository<ArticleEntity, String> {
    List<ArticleEntity> findByTitle(String title);
    List<ArticleEntity> findByTitleOrContent(String title, String content);
    List<ArticleEntity> findByTitleOrContent(String title, String content, Pageable pageable);
}
12345
```

命名遵循规范就行，idea本身也有提提示，然后就可以使用这个方法进行查询了：

```java
articleReposiory.findByTitle(title);
1
```

### 7.分页查询

```java
List<ArticleEntity> findByTitleOrContent(String title, String content, Pageable pageable);

 @GetMapping("/findByTitleOrContentPage")
 public List<ArticleEntity> findByTitleOrContentPage(String title, String content, Integer pageNum, Integer pageSize) {
     Pageable pageable = PageRequest.of(pageNum, pageSize);
     return articleReposiory.findByTitleOrContent(title, content, pageable);
 }
1234567
```

使用**Pageable**参数，注意页码从0开始。

### 8.使用原生的NativeSearchQuery查询

使用NativeSearchQuery查询条件，可以做到更灵活更复杂的查询：

```java
@Autowired
private ElasticsearchRestTemplate elasticsearchRestTemplate;

@GetMapping("/findByTitleOrContentPageByTemplate")
public List<ArticleEntity> findByTitleOrContentPageByTemplate(String title, String content, Integer pageNum, Integer pageSize) {

    NativeSearchQuery nativeSearchQuery = new NativeSearchQueryBuilder()
            .withQuery(QueryBuilders.queryStringQuery(title).defaultField("title"))
            .withQuery(QueryBuilders.queryStringQuery(content).defaultField("content"))
            .withPageable(PageRequest.of(pageNum, pageSize))
            .build();
    return elasticsearchRestTemplate.queryForList(nativeSearchQuery, ArticleEntity.class);
}
```

本文就先介绍到这里，关于NativeSearchQuery更多的用法，放在后文[《ElasticSearch系列（六）springboot中使用QueryBuilders、NativeSearchQuery实现复杂查询》](https://blog.csdn.net/csdn_20150804/article/details/105618933)
中介绍。