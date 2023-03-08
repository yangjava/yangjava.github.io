---
layout: post
categories: [SpringBoot,MongoDB]
description: none
keywords: SpringBoot
---
# SpringBoot整合MongoDB
Spring Data MongoDB 是Spring Data的下的一个模块，在SpringBoot中整合MongoDB就需要添加Spring Data MongoDB的依赖

## 添加依赖
在pom.xml文件中添加Spring Data MongoDB依赖
```text
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

## 配置文件
SpringBoot默认的配置文件格式是application.properties，而目前我们比较常用的SpringBoot配置文件格式为yml，所以这里我把配置文件的后缀改为.yml

在application.yml文件中添加MongoDB的连接参数
```yaml
spring:
  data:
    mongodb:
      host: 127.0.0.1 #指定MongoDB服务地址
      port: 27017 #指定端口，默认就为27017
      database: article#指定使用的数据库(集合)
      authentication-database: admin # 登录认证的逻辑库名
      username: admin #用户名
      password: admin #密码
```
mongodb数据库与mysql不一样 mysql 一个普通用户可以管理多个数据库，但是mongo每一个库都有一个独立的管理用户，连接时需要输入对应用户密码

## 代码实战
### 主要注解：
- @Document，文档是 MongoDB 中最基本的数据单元，由键值对组成，类似于 JSON 格式，可以存储不同字段，字段的值可以包括其他文档，数组和文档数组。
- @Id（主键）：用来将成员变量的值映射为文档的_id的值
- @Indexed（索引）： 索引是一种特殊的数据结构，存储在一个易于遍历读取的数据集合中，能够对数据库文档中的数据进行排序。索引能极大提高文档查询效率，如果没有设置索引，MongoDB 会遍历集合中的整个文档，选取符合查询条件的文档记录。这种查询效率是非常低的。
- @Field（字段）： 文档中的字段，类似于 MySql 中的列。
- @Aggregation（聚合）： 聚合主要用于数据处理，例如统计平均值、求和等。
```java
/**
 * 文章实体类
 **/

@Data
@Document(collection = "article") //指定要对应的文档名（表名）
@Accessors(chain = true)
public class Article {
    @Id
    private String id;//文章主键

    private String articleName; //文章名

    private String content; //文章内容
}

```

### 实现添加、删除、查询
SpringBoot操作MongoDB有两种方式，分别是继承MongoRepository类和service注入MongoTemplate

1. dao层继承MongoRepository
Repository是用于操作数据库的类
```java
/**
 * 继承 MongoRepository<实体类，主键类型>,以实现CRUD
 **/

public interface ArticleRepository extends MongoRepository<Article,String> {
	//根据id查询文章
    List<Article> findByid(String id);
}
```

2. service层注入MongoTemplate
```java
/**
 * ArticleService实现类
 **/
@Service
public class ArticleServiceImpl2 implements ArticleService2 {


    @Autowired
    private MongoTemplate mongoTemplate;

    /**
     * 添加文章
     *
     * @param article
     * @return
     */
    @Override
    public int create(Article article) {
        Article save = mongoTemplate.save(article);
        return 1;
    }

    /**
     * 删除文章
     *
     * @param id
     */
    @Override
    public int delete(String id) {
        List<Article> deleteList = new ArrayList<>();
        Query query = new Query();
        query.addCriteria(Criteria.where("id").is(id));
        mongoTemplate.remove(query,Article.class);
        return 1;
    }

    @Override
    public Article get(String id) {
        Article byId = mongoTemplate.findById(id, Article.class);
        return byId;
    }
}

```

## SpringBoot整合MongoDb去除_class
