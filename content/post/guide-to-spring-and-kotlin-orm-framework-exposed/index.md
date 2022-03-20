---
title: Spring + Kotlin ORM 框架 Exposed 教程
slug: guide-to-spring-and-kotlin-orm-framework-exposed
date: 2022-03-19T16:34:00+08:00
image: exposed_logo.jpg
tags:
  - Kotlin
  - DB
  - Spring
categories:
  - tech
---

本教程包括 Kotlin ORM 框架 Exposed 的使用方法和一些进阶技巧，并介绍 Exposed 与 Spring 集成的方法以及博主踩过的一些坑

 <!--more-->

## Exposed 介绍

[Exposed](https://github.com/JetBrains/Exposed) 是 JetBrains 官方出品的 Kotlin ORM 框架，有如下优点：

1. 支持多种数据库：H2、MySQL、PostgreSQL、SQL Server、SQLite 等
2. 提供两套 API：SQL DSL 和 DAO API（不知道什么是 DSL 可以阅读我的之前的文章：[Kotlin DSL 简介]({{< relref "introduction-to-kotlin-dsl" >}})）
3. JetBrains 官方出品，文档完善，易于使用

### 基本概念

#### 连接数据库

首先需要引入依赖：

```xml
<dependency>
    <groupId>org.jetbrains.exposed</groupId>
    <artifactId>exposed-core</artifactId>
    <version>0.37.3</version>
</dependency>
<dependency>
    <groupId>org.jetbrains.exposed</groupId>
    <artifactId>exposed-dao</artifactId>
    <version>0.37.3</version>
</dependency>
<dependency>
    <groupId>org.jetbrains.exposed</groupId>
    <artifactId>exposed-jdbc</artifactId>
    <version>0.37.3</version>
</dependency>
```

然后建立数据库连接：
关于数据库和数据源的更多说明查看[官方 WIKI](https://github.com/JetBrains/Exposed/wiki/DataBase-and-DataSource)

```Kotlin
Database.connect("jdbc:h2:mem:test", driver = "org.h2.Driver")
```

最后开启事务操作数据库：
不论是 SQL DSL 还是 DAO API 都需要在 `transaction` 块中执行

```Kotlin
transaction {
    addLogger(StdOutSqlLogger)
    // Do something
    commit()
    // Do something
    rollback()
}
```

事务支持返回结果：
Blob、text 以及一对多/多对一的字段不能在事务外使用，更多说明查看[官方 WIKI](https://github.com/JetBrains/Exposed/wiki/Transactions)

```Kotlin
val result = transaction {
    QueryEntity.findById(1)
}
```

#### Table/DAO

建表：
如下的 Queries 类创建一个名为 query 的表，并添加了 4 个字段

```Kotlin
object Queries : IntIdTable("query") {
    val title = varchar("title", 1024)
    val userId = varchar("userId", 256)
    val type = varchar("type", 256)
    val createTime = timestamp("createTime")
}
```

Exposed 不会自动生成 migration，但是可以使用 `SchemaUtils#create` 来在数据库中运行建表语句
官方提供一个 gradle 插件来根据数据库结构生成 Table 代码：[exposed-intellij-plugin](https://github.com/JetBrains/exposed-intellij-plugin)

创建实体类：
如果要使用 Exposed 的 DAO API，需要创建表对应的实体（Queries -> QueryEntity）

```Kotlin
class QueryEntity(id: EntityID<Int>) : IntEntity(id) {
    companion object : IntEntityClass<QueryEntity>(Queries)

    var title by Queries.title
    var userId by Queries.userId
    var type by Queries.type
    var createTime by Queries.createTime
}
```

#### CRUD

这里给出两种 API 的简单示例：

SQL DSL：
[文档](https://github.com/JetBrains/Exposed/wiki/DSL)

```Kotlin
transaction {
    Queries.insert {
        it[title] = "title"
        it[userId] = "123"
        it[type] = "type"
        it[createTime] = Instant.now()
    }

    Queries.select { Queries.id eq 1 }

    Queries.update {
        it[title] = "titleUpdate"
        it[userId] = "456"
        it[type] = "typeUpdate"
        it[createTime] = Instant.now()
    }

    Queries.deleteWhere { Queries.id eq 1 }
}
```

DAO API：
[文档](https://github.com/JetBrains/Exposed/wiki/DAO)

```Kotlin
transaction {
    QueryEntity.new {
        title = "title"
        userId = "123"
        type = "type"
        createTime = Instant.now()
    }

    val result = QueryEntity.findById(1)

    result?.title = "titleUpdate"
    result?.userId = "456"
    result?.type = "titleUpdate"
    result?.createTime = Instant.now()

    result?.delete()
}
```

### 进阶使用

#### 索引

Exposed 支持创建单列/多列索引：

```Kotlin
object Queries : IntIdTable("query") {
    ...

    val userId = varchar("userId", 256).index()
    val userId = varchar("userId", 256).uniqueIndex()
    val userId = varchar("userId", 256).index("index_userId_unique", true)

    val index = index("index_title_userId_unique",true, title, userId)
    ...
}
```

#### 一对多/多对一

首先创建外键：

```Kotlin
object Histories : IntIdTable("history") {
    ...
    val queryId = reference("query_id", Queries, onDelete = ReferenceOption.CASCADE)
}
```

然后在实体类上添加相应的字段：

```Kotlin
class QueryEntity(id: EntityID<Int>) : IntEntity(id) {
    ...
    // 一对多
    val histories by HistoryEntity referrersOn Histories.queryId
}

class HistoryEntity(id: EntityID<Int>) : IntEntity(id) {
    ...
    // 多对一
    var query by QueryEntity referencedOn Histories.queryId
}
```

在查询中即可直接访问对应字段：

```Kotlin
transaction {
    // 使用 load 提前加载 histories 字段，避免 N+1 问题
    QueryEntity.findById(1)
        ?.load(QueryEntity::histories)
        ?.histories
        ?.forEach {
            // do something
        }
}
```

#### upsert

Exposed 并没有提供开箱即用的 upsert 功能（类似 MySQL 的 ON DUPLICATE KEY UPDATE），需要自己拓展（详见这个 [Issue](https://github.com/JetBrains/Exposed/issues/167)）

这里推荐一个库帮我们实现了 upsert：[exposed-upsert](https://github.com/reposilite-playground/exposed-upsert)

```Kotlin
transaction {
    // 也可以使用 conflictIndex 指定自行创建的唯一索引
    Queries.upsert(conflictColumn = Queries.userId,
        insertBody = {
            it[title] = queryData.title
            it[userId] = queryData.userId
            it[type] = queryData.type
            it[createTime] = Instant.now()
        }, updateBody = {
            it[title] = queryData.title
            it[type] = queryData.type
        })
}
```

## Exposed 与 Spring 集成

官方提供 Exposed Spring Boot Starter，用 Exposed 替换 Hibernate

### 配置

引入依赖：

```xml
<dependencies>
  <dependency>
    <groupId>org.jetbrains.exposed</groupId>
    <artifactId>exposed-spring-boot-starter</artifactId>
    <version>0.37.3</version>
  </dependency>
</dependencies>
```

配置数据库：

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    driverClassName: org.h2.Driver
  exposed:
    # 自动在数据库中建表
    generate-ddl: true
```

### transaction

Exposed 的两种 API 都需要在 `transaction` 块中执行：

```Kotlin
transaction {
    QueryEntity.all()
}
```

在 spring 中可以用 Transactional 注解代替 transaction 块：

```Kotlin
@Transactional
fun all(): List<QueryEntity> {
    return QueryEntity.all().toList()
}
```

如果需要调用 rollback、commit 等方法需要使用 `TransactionManager#current` 获取当前 Transaction 实例

```Kotlin
@Transactional
fun rollback() {
    // Do something
    TransactionManager.current().rollback()
}
```

### json 转换

这里以 spring 默认使用的 json 库 jackson 为例，gson 或 fastjson 原理上是一样的

首先回顾下我们之前创建的实体类 QueryEntity：

```Kotlin
class QueryEntity(id: EntityID<Int>) : IntEntity(id) {
    companion object : IntEntityClass<QueryEntity>(Queries)

    var title by Queries.title
    var userId by Queries.userId
    var type by Queries.type
    var createTime by Queries.createTime

    val histories by HistoryEntity referrersOn Histories.queryId
}
```

可以发现实体类 QueryEntity 继承自 IntEntity，序列化/反序列化时我们只需要非 IntEntity 类的字段，而 IntEntity 又继承自 Entity，我的做法是让 jackson 忽略所有属于 Entity 或 IntEntity 类的字段：

```Kotlin
@Configuration
class JacksonConfig : Jackson2ObjectMapperBuilderCustomizer {
    override fun customize(jacksonObjectMapperBuilder: Jackson2ObjectMapperBuilder) {
        jacksonObjectMapperBuilder.modulesToInstall(object :Module() {
            override fun version(): Version = Version(0, 0, 0,"")

            override fun getModuleName(): String = ""

            override fun setupModule(context: SetupContext) {
                context.insertAnnotationIntrospector(object : JacksonAnnotationIntrospector() {
                    override fun hasIgnoreMarker(m: AnnotatedMember): Boolean {
                        return m.declaringClass == IntEntity::class.java
                                || m.declaringClass == Entity::class.java
                                || super.hasIgnoreMarker(m)
                    }
                })
            }
        })
    }
}
```

{{< tip warning >}}
不能直接使用 `Jackson2ObjectMapperBuilder#annotationIntrospector` 注入 annotationIntrospector，否则会使 KotlinModule 的 annotationIntrospector 失效
{{</ tip >}}

如果需要序列化 `id` 字段，需要允许 id 和 getId 字段参与序列化。因为 id 字段不是 Int 或 String 等基础类型，所以我们还需要自定义序列化器：

```Kotlin
@Configuration
class JacksonConfig : Jackson2ObjectMapperBuilderCustomizer {
   override fun customize(jacksonObjectMapperBuilder: Jackson2ObjectMapperBuilder) {
       jacksonObjectMapperBuilder.serializerByType(EntityID::class.java, object: JsonSerializer<EntityID<*>>() {
           override fun serialize(value: EntityID<*>?, gen: JsonGenerator, serializers: SerializerProvider) {
               if (value == null) {
                   gen.writeNull();
               } else if (value.value is Int) {
                   gen.writeNumber(value.value as Int);
               } else if (value.value is Long) {
                   gen.writeNumber(value.value as Long);
               } else {
                   gen.writeString(value.value.toString())
               }
           }
       })

       jacksonObjectMapperBuilder.modulesToInstall(object :Module() {
           override fun version(): Version = Version(0, 0, 0,"")

           override fun getModuleName(): String = ""

           override fun setupModule(context: SetupContext) {
               context.insertAnnotationIntrospector(object : JacksonAnnotationIntrospector() {
                   override fun hasIgnoreMarker(m: AnnotatedMember): Boolean {
                       return (m.name != "id" && m.name != "getId" && m.declaringClass == IntEntity::class.java)
                               || (m.name != "id" && m.name != "getId" && m.declaringClass == Entity::class.java)
                               || super.hasIgnoreMarker(m)
                   }
               })
           }
       })
   }
}
```

因为委托属性不可以直接使用 JsonIgnore、JsonInclude   等注解，需要使用 `@get:JsonIgnore`

```Kotlin
class QueryEntity(id: EntityID<Int>) : IntEntity(id) {
    ...
    @get:JsonIgnore
    val histories by HistoryEntity referrersOn Histories.queryId
}
```
