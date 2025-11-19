---
layout: default
title: 多数据源管理：从自动到手动
date: 2025-10-13 20:24
tags:
  - Spring
  - DataSource
  - Mybatis
  - Shardingsphere
categories: [tech, tutorial]
---

## 起源
系统初始开发时为单数据源，后续需要集成公司内部的 UAC 组件。 UAC 组件引入了动态数据源用于分离自身的数据，它们使用 dynamic-datasource 风格的配置文件。  
初始接入后，系统运行正常，但由于客户厂商认为主从多实例数据库+磁带备份还是不够安全，提出读写分离+数据库加密的需求。  
我在做概设时，基于一般性的考虑，考虑引入新的组件实现相关需求，调研后选择了 shardingsphere。
```xml
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>shardingsphere-jdbc</artifactId>
    <version>5.5.2</version>
</dependency>
```
随之而来带来了一系列问题，其中有两点最强烈：
1. Shardingsphere 仅支持一些应用广泛的开源、商业 DBMS，并不支持目前使用的 DBMS，需要定制化接入；
2. Shardingsphere 与 dynamic-datasource 并不直接兼容。

针对问题 1，Shardingsphere 存在一些接口用于实现自定义的数据库支持，实现复杂度有高有低，但仍然可以在高的实现维度上兼容，解决方案并不复杂，后续可以详细描述。本文档用于记录问题 2 的解决方式。

## 挑战
由于 dynamic-datasource 的文档闭源，因此并不清楚其是否内置了对 shardingsphere 的支持（大概率无），又基于如下因素，需要对数据源进行手动管理

1. shardingsphere 需要接管连接并生成逻辑数据源，同时需要配置其管理的表来提供关键能力；
2. dynamic-datasource 原生配置方式无法实现组件接入；
3. 直接移除 dynamic-datasource 的依赖会导致 UAC 异常（大量使用了 `@DS` 注解）

因此需要针对系统内的 `DataSource` 实例进行手动管理，保证线程正确调用。

## 前置条件
在进行手动数据源管理的改造前，需要了解如下系统特性。

### Spring Boot AutoConfigure自动配置原理
了解 Spring Boot 自动配置原理，配置条件组合以及**约定优于配置**思想下，配置文件结合 `@ConfigurationProperties`、`@Component` 生成配置 Bean。
### Shardingsphere 原理与配置
Shardingsphere 原理是接管物理数据源并生成逻辑数据源，分析 SQL，并依据读写操作，将请求使用逻辑数据源来转发到实际的数据库实例，进而实现读写分离。其在新版不再提供 spring boot starter ，需要手动引入配置文件，并使用其自身提供的 `Yaml` 解析器解析，可以分离使用单独的配置文件，也可以使用内嵌 `Yaml` 的方式在一个配置文件中添加 shardingsphere 配置。shardingsphere 提供了多种导入配置的方式，包括手动指定物理数据源的方式。

## 禁止动态数据源自动注册
首先禁止 dynamic-datasource 的自动注册
```yaml
spring:
  autoconfigure:
    exclude: com.baomidou.dynamic.datasource.spring.boot.autoconfigure.DynamicDataSourceAutoConfiguration
```
## 手动管理数据源
### 使用了 Dynamic 的组件
禁止自动注册后，手段创建、管理dynamic datasource 对象。
#### 手动注册 UAC 数据源
需要提供描述数据源的实体类对象，并做如下配置
```java
@Data
@Component
@ConfigurationProperties("spring.datasource.dynamic.datasource.uac")
public class UacDataSourceProperties {
  private String driverClassName;
  private String url;
  private String username;
  private String password;
}
```
需要在配置文件中进行如下配置
```yaml
spring:
  datasource:
    dynamic:
      datasource:
        uac:
          url: xxx
          username: xxx
          password: xxx
          driver-class-name: xxx
```
虽然该配置看似仍然为 dynamic 的自动配置，但实际上仅仅是复用了旧的配置文件，用于获取配置的值而已，随后需要用该配置来真正注册 UAC 的 DataSource
```java
@Bean
public DataSource uacDataSource(UacDataSourceProperties uacDataSourceProperties){
  // 使用 Hikari 连接池
  HikariConfig config = new HikariConfig();
  // 从 uacDataSourceProperties 中获取连接信息
  config.setJdbcUrl(uacDataSourceProperties.getUrl());
  config.setUsername(uacDataSourceProperties.getUsername());
  config.setPassword(uacDataSourceProperties.getPassword());
  config.setDriverClassName(uacDataSourceProperties.getDriverClassName());
  // 其他参数，省略
  // 响应 DataSource
  return new HikariDataSource(config);
}
```
#### dynamic-datasource 原理
此时虽然已经生成了 UAC 的数据源，但由于 UAC 组件内大量使用了 `@DS` 注解，而dynamic-datasource 原理为基于 AOP 针对存在 `@DS` 的 Bean 使用 DynamicRoutingDataSource 中的数据源取出并使用指定的 DataSource。因此仍需将 UAC DataSource 注册到 DynamicRoutingDataSource  中
```java
@Bean
public DynamicRoutingDataSource dataSource(@AutoWired @Qualifier("uacDataSource") DataSource uacDataSource){
  DynamicRoutingDataSource ds = new DynamicRoutingDataSource(new ArrayList<>());
  ds.setPrimary("UAC");
  ds.setStrict(false);
  ds.addDataSource(uacDataSource);
  return ds;
}
```
此时完成了 UAC 的数据源需求。

### ShardingSphere 配置
由于数据源定义与分库分表的逻辑本质上是分离的两部分，因为关联的数据源直接绑定一个 DataSource 实例，而分库分表实际上是针对 SQL 的分析与处理。因此，在本系统中，针对数据源的配置与分库分表、数据加密、读写分离的配置是分离的两处，只有在生成逻辑数据源时才绑定到一起。
#### Shardingsphere 能力配置
由于使用独立的 Shardingsphere 配置文件会极大增加配置文件的管理成本，故使用 YAML 的内嵌文本来实现 Shardingsphere 配置文件的定义
```yaml
spring:
  sharding: |
    rules:
    - !SINGLE
      tables: 
# 仅作示例，后续省略
```
上文的配置文件中 `|` 后的部分会被认为是字符串，Spring 并不会再对内部的信息做任何处理，于是可以在此处配置，此部分的值可以使用，注意，此处的`spring.sharding` 为自定义的 item，用于后续配置文件定位
```java
@Value("${spring.sharding}")
private String yamlConfig;
```
#### Shardingsphere 数据源配置
首先需要定义 shardingsphere 的配置逻辑，由于数据库配置需要绑定在配置文件中，又由于我们需要手动管理该配置类，因此仍采用上文中，UAC 的配置模式，但此时有一个问题，上文是针对单例的配置 Bean，而读写分离至少会存在两个数据源配置，因此需要另外的方案，该方案必须支持创建多个数据源，同时新增、修改数据源不能重新打包。`@ConfigurationProperties` 支持`Map` 格式的配置
```yaml
mySys:
  dataSources:
    ds_0:
      url: xxx
      username: xxx
      password: xxx
      driver-class-name: xxx
    ds_0_slave:
      url: xxx
      username: xxx
      password: xxx
      driver-class-name: xxx
```
在系统中以如下方式创建配置 Bean
```java
@Data
@Component
@ConfigurationProperties("mySys")
public class DataSourceGroupProperties {
  private Map<String,DataSourceProperties> dataSources = new HashMap<>();
}
```
`DataSourceProperties` 类定义与上文中 `UacDataSourceProperties` 定义一致，此处不再重复。需要明确注意的是，`@ConfigurationProperties("mySys") ` 标注的类，其属性要与配置文件中对应，如配置文件中为 `dataSources` 类的属性名也必须为 `dataSources`，否则会无法实现属性注入。  
得到 `DataSourceGroupProperties` 实例后，可以获取内部的属性，使用上文中介绍的方法，生成一个 DataSource 的 Map 如
```java
Map<String,DataSource> allDataSource = generateDataSource(dataSourceGroupProperties);
```
#### Shardingsphere 逻辑数据源配置
到目前位置，我们获得了 shardingsphere 能力的配置，如读写分离、分库分表、数据加密等，我们也获得了对应的数据源信息，需要将两者绑定为一个逻辑数据源，该逻辑数据源提供了数据库连接和增量操作。逻辑数据源配置格外简单
```java
@Value("$spring.sharding")
private String yamlConfig;

@Resource
private DataSourceGroupProperties dataSourceGroupProperties;

@Bean("shardingsphere")
@Primary
public DataSource shardingsphereDataSource(){
  Map<String,DataSource> allDataSource = generateDataSource(dataSourceGroupProperties);
  return YamlShardingSphereDataSourceFactory.createDataSource(allDataSource,yamlConfig.getBytes(StandardCharsets.UTF_8));
}
```
逻辑数据源已创建，基于该逻辑数据源的数据库连接均会执行 SQL 分析与数据库连接选择。

### 注入逻辑数据源
虽然上文已经创建了支持增量操作的逻辑数据源，实际上该数据源目前仅仅是作为 Bean 存在于系统中，并不能发起数据库连接，这是因为系统使用的 Mybatis 需要将其生成的 SQL 绑定到某个具体的 DataSource 上，如果系统中仅有一个 DataSource，那么会自动选择该 DataSource，但当前系统内存在多个 DataSource，若不明确指定使用的数据源，那么系统将无法启动，解决方案如下。

#### 生成 SqlSessionTemplate
**MyBatis** 底层使用 `SqlSessionTemplate` 进行数据库访问操作，因此我们需要生成一个指定的 `SqlSessionTemplate`。
首先，创建 `SqlSessionFactory`
```java
@Bean
public SqlSessionFactory shardingsphereSqlSessionFactory(@Qualifier("shardingsphere") DataSource dataSource){
  // 使用上文中自定义的逻辑数据源
  MybatisSqlSessionFactoryBean factory = new MybatisSqlSessionFactoryBean();
  // 设置 Mapper.xml 文件路径
  factory.setMapperLocations(new PathMatchingResourcePatternResolver().getResources("your/mapper/xml/file/pattern"));
  // 添加插件，如果有
  // factory.addPlugins();
 return factory.getObject();
}
```
其次，使用 `SqlSessionFactory ` 生成 `SqlSessionTemplate`
```java
@Bean
public SqlSessionTemplate shardingsphereSqlSessionTemplate(@Qualifier("shardingsphereSqlSessionFactory") SqlSessionFactory sqlSessionFactory ){
  // 使用上文中自定义的 SqlSessionFactory 
  // 这一步非常简单
 return new SqlSessionTemplate(sqlSessionFactory );
}
```
最后，在`@MapperScan` 注解中，指定扫描的包路径，以及需要使用的逻辑数据源
```java
@MapperScan(basePackages = "com.your.own.domain.**.mapper",sqlSessionTemplateRef = "shardingsphereSqlSessionTemplate")
```
到此，完成了数据源手动控制的全流程。