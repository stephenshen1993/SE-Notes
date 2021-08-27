# ShardingSphere环境准备

## 前言

在开始 ShardingSphere 源码解析前，应先翻阅相关介绍文档，了解其解决的问题及其使用场景。然后 run 官方 demo，简单串一下使用流程，为后续深入源码做准备。



## 一、官方介绍

[GitHub](https://github.com/apache/shardingsphere) 和 [官网](https://shardingsphere.apache.org/) 都是很好的了解途径，从 GitHub 上 README.md 的 Document 一栏亦可跳转到官网。

以下摘自官网：

> Apache ShardingSphere is an open-source ecosystem consisted of a set of distributed database solutions, including 3 independent products, JDBC, Proxy & Sidecar (Planning). They all provide functions of data scale out, distributed transaction and distributed governance, applicable in a variety of situations such as Java isomorphism, heterogeneous language and cloud native.
>
> Apache ShardingSphere 是一套开源的分布式数据库解决方案组成的生态圈，它由 JDBC、Proxy 和 Sidecar（规划中）这 3 款既能够独立部署，又支持混合部署配合使用的产品组成。 它们均提供标准化的数据水平扩展、分布式事务和分布式治理等功能，可适用于如 Java 同构、异构语言、云原生等各种多样化的应用场景。
>
> Apache ShardingSphere aiming at reasonably making full use of the computation and storage capacity of existed database in distributed system, rather than a totally new database. As the cornerstone of enterprises, relational database still takes a huge market share. Therefore, we prefer to focus on its increment instead of a total overturn.
>
> Apache ShardingSphere 旨在充分合理地在分布式的场景下利用关系型数据库的计算和存储能力，而并非实现一个全新的关系型数据库。 关系型数据库当今依然占有巨大市场份额，是企业核心系统的基石，未来也难于撼动，我们更加注重在原有基础上提供增量，而非颠覆。



简单来说，ShardingSphere 是一个生态圈，专注于提供分布式数据库解决方案。其由JDBC、Proxy 和 Sidecar 组成，底层依赖关系型数据库的计算和存储能力实现。



## 二、环境准备

### 1、克隆项目

众所周知，国内从 GiHhub 拉项目贼慢，加上 ShardingSphere 又大，不建议直接从 GitHub clone。

这里仅用于学习，通过 [Gitee（码云）](https://gitee.com/Sharding-Sphere/sharding-sphere)进行 clone 即可。

```shell
# ShardingSphere 文件名过长，必须配置此项
git config --global core.longpaths true
# 从码云 clone 到本地
git clone https://gitee.com/Sharding-Sphere/sharding-sphere.git
```

### 2、定位版本

这里用 git 标签定位最新的 5.0.0-beta 版本：

![image-20210824050128100](https://gitee.com/stephenshen/pic-bed/raw/master/img/20210824050128.png)

使用 Intelij IDEA 打开项目，点击右下角 git:master，选择 Checkout Tag，输入 5.0.0-beta，OK确认，等待加载完成。

### 3、本地安装

![image-20210824084147343](https://gitee.com/stephenshen/pic-bed/raw/master/img/20210824084147.png)

定位到目标版本后，需要将 ShardingSphere install 到本地仓库。

![image-20210824045735870](https://gitee.com/stephenshen/pic-bed/raw/master/img/20210824045741.png)

打开 Maven 窗口，执行 clean、install，注意跳过测试。

成功日志：

```
...
[INFO] shardingsphere-proxy-distribution .................. SUCCESS [  1.018 s]
[INFO] shardingsphere-scaling-distribution ................ SUCCESS [  1.019 s]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 08:52 min
[INFO] Finished at: 2021-08-24T03:03:54+08:00
[INFO] Final Memory: 260M/1141M
[INFO] ------------------------------------------------------------------------
```

### 4、运行 Demo

这里以 sharding-spring-boot-mybatis-example 为例：

1）本地安装：使用 IDEA 打开 sharding-sphere/examples 项目，执行 clean、install；

2）初始化数据库：使用 examples/src/resources/manual_schema.sql 脚本进行初始化

3）修改配置文件：先查看 application.properties，确定激活的 profiles

```properties
mybatis.config-location=classpath:META-INF/mybatis-config.xml

#spring.profiles.active=sharding-databases
#spring.profiles.active=sharding-tables
#spring.profiles.active=sharding-databases-tables
#spring.profiles.active=readwrite-splitting
spring.profiles.active=sharding-readwrite-splitting
```

激活的 profiles 是 sharding-readwrite-splitting，故打开 application-sharding-readwrite-splitting.properties，修改为对应的的数据库端口、数据库驱动以及用户名密码

```properties
...
spring.shardingsphere.datasource.write_ds_0.jdbc-url=jdbc:mysql://localhost:13307/demo_write_ds_0?serverTimezone=UTC&useSSL=false&useUnicode=true&characterEncoding=UTF-8
spring.shardingsphere.datasource.write_ds_0.type=com.zaxxer.hikari.HikariDataSource
spring.shardingsphere.datasource.write_ds_0.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.write_ds_0.username=root
spring.shardingsphere.datasource.write_ds_0.password=root

# 同上
...
```

4）运行：找到启动类 org.apache.shardingsphere.example.sharding.spring.boot.mybatis.ExampleMain.java 执行

输出如下：

```
-------------- Process Success Begin ---------------
---------------------------- Insert Data ----------------------------
---------------------------- Print Order Data -----------------------
order_id: 636771446429298688, user_id: 1, address_id: 1, status: INSERT_TEST
order_id: 636771446429298688, user_id: 1, address_id: 1, status: INSERT_TEST
order_id: 636771446429298688, user_id: 1, address_id: 1, status: INSERT_TEST
order_id: 636771446429298688, user_id: 1, address_id: 1, status: INSERT_TEST
order_id: 636771446777425920, user_id: 2, address_id: 2, status: INSERT_TEST
order_id: 636771446777425920, user_id: 2, address_id: 2, status: INSERT_TEST
order_id: 636771446777425920, user_id: 2, address_id: 2, status: INSERT_TEST
order_id: 636771446777425920, user_id: 2, address_id: 2, status: INSERT_TEST
order_id: 636771446911643648, user_id: 3, address_id: 3, status: INSERT_TEST
order_id: 636771446911643648, user_id: 3, address_id: 3, status: INSERT_TEST
order_id: 636771446911643648, user_id: 3, address_id: 3, status: INSERT_TEST
order_id: 636771446911643648, user_id: 3, address_id: 3, status: INSERT_TEST
order_id: 636771447045861376, user_id: 4, address_id: 4, status: INSERT_TEST
order_id: 636771447045861376, user_id: 4, address_id: 4, status: INSERT_TEST
order_id: 636771447045861376, user_id: 4, address_id: 4, status: INSERT_TEST
order_id: 636771447045861376, user_id: 4, address_id: 4, status: INSERT_TEST
order_id: 636771447297519616, user_id: 5, address_id: 5, status: INSERT_TEST
order_id: 636771447297519616, user_id: 5, address_id: 5, status: INSERT_TEST
order_id: 636771447297519616, user_id: 5, address_id: 5, status: INSERT_TEST
order_id: 636771447297519616, user_id: 5, address_id: 5, status: INSERT_TEST
order_id: 636771447456903168, user_id: 6, address_id: 6, status: INSERT_TEST
order_id: 636771447456903168, user_id: 6, address_id: 6, status: INSERT_TEST
order_id: 636771447456903168, user_id: 6, address_id: 6, status: INSERT_TEST
order_id: 636771447456903168, user_id: 6, address_id: 6, status: INSERT_TEST
order_id: 636771447746310144, user_id: 7, address_id: 7, status: INSERT_TEST
order_id: 636771447746310144, user_id: 7, address_id: 7, status: INSERT_TEST
order_id: 636771447746310144, user_id: 7, address_id: 7, status: INSERT_TEST
order_id: 636771447746310144, user_id: 7, address_id: 7, status: INSERT_TEST
order_id: 636771447888916480, user_id: 8, address_id: 8, status: INSERT_TEST
order_id: 636771447888916480, user_id: 8, address_id: 8, status: INSERT_TEST
order_id: 636771447888916480, user_id: 8, address_id: 8, status: INSERT_TEST
order_id: 636771447888916480, user_id: 8, address_id: 8, status: INSERT_TEST
order_id: 636771448069271552, user_id: 9, address_id: 9, status: INSERT_TEST
order_id: 636771448069271552, user_id: 9, address_id: 9, status: INSERT_TEST
order_id: 636771448069271552, user_id: 9, address_id: 9, status: INSERT_TEST
order_id: 636771448069271552, user_id: 9, address_id: 9, status: INSERT_TEST
order_id: 636771448253820928, user_id: 10, address_id: 10, status: INSERT_TEST
order_id: 636771448253820928, user_id: 10, address_id: 10, status: INSERT_TEST
order_id: 636771448253820928, user_id: 10, address_id: 10, status: INSERT_TEST
order_id: 636771448253820928, user_id: 10, address_id: 10, status: INSERT_TEST
---------------------------- Print OrderItem Data -------------------
order_item_id:636771446659985409, order_id: 636771446429298688, user_id: 1, status: INSERT_TEST
order_item_id:636771446659985409, order_id: 636771446429298688, user_id: 1, status: INSERT_TEST
order_item_id:636771446659985409, order_id: 636771446429298688, user_id: 1, status: INSERT_TEST
order_item_id:636771446659985409, order_id: 636771446429298688, user_id: 1, status: INSERT_TEST
order_item_id:636771446840340481, order_id: 636771446777425920, user_id: 2, status: INSERT_TEST
order_item_id:636771446840340481, order_id: 636771446777425920, user_id: 2, status: INSERT_TEST
order_item_id:636771446840340481, order_id: 636771446777425920, user_id: 2, status: INSERT_TEST
order_item_id:636771446840340481, order_id: 636771446777425920, user_id: 2, status: INSERT_TEST
order_item_id:636771446982946817, order_id: 636771446911643648, user_id: 3, status: INSERT_TEST
order_item_id:636771446982946817, order_id: 636771446911643648, user_id: 3, status: INSERT_TEST
order_item_id:636771446982946817, order_id: 636771446911643648, user_id: 3, status: INSERT_TEST
order_item_id:636771446982946817, order_id: 636771446911643648, user_id: 3, status: INSERT_TEST
order_item_id:636771447159107585, order_id: 636771447045861376, user_id: 4, status: INSERT_TEST
order_item_id:636771447159107585, order_id: 636771447045861376, user_id: 4, status: INSERT_TEST
order_item_id:636771447159107585, order_id: 636771447045861376, user_id: 4, status: INSERT_TEST
order_item_id:636771447159107585, order_id: 636771447045861376, user_id: 4, status: INSERT_TEST
order_item_id:636771447373017089, order_id: 636771447297519616, user_id: 5, status: INSERT_TEST
order_item_id:636771447373017089, order_id: 636771447297519616, user_id: 5, status: INSERT_TEST
order_item_id:636771447373017089, order_id: 636771447297519616, user_id: 5, status: INSERT_TEST
order_item_id:636771447373017089, order_id: 636771447297519616, user_id: 5, status: INSERT_TEST
order_item_id:636771447574343681, order_id: 636771447456903168, user_id: 6, status: INSERT_TEST
order_item_id:636771447574343681, order_id: 636771447456903168, user_id: 6, status: INSERT_TEST
order_item_id:636771447574343681, order_id: 636771447456903168, user_id: 6, status: INSERT_TEST
order_item_id:636771447574343681, order_id: 636771447456903168, user_id: 6, status: INSERT_TEST
order_item_id:636771447821807617, order_id: 636771447746310144, user_id: 7, status: INSERT_TEST
order_item_id:636771447821807617, order_id: 636771447746310144, user_id: 7, status: INSERT_TEST
order_item_id:636771447821807617, order_id: 636771447746310144, user_id: 7, status: INSERT_TEST
order_item_id:636771447821807617, order_id: 636771447746310144, user_id: 7, status: INSERT_TEST
order_item_id:636771447956025345, order_id: 636771447888916480, user_id: 8, status: INSERT_TEST
order_item_id:636771447956025345, order_id: 636771447888916480, user_id: 8, status: INSERT_TEST
order_item_id:636771447956025345, order_id: 636771447888916480, user_id: 8, status: INSERT_TEST
order_item_id:636771447956025345, order_id: 636771447888916480, user_id: 8, status: INSERT_TEST
order_item_id:636771448182517761, order_id: 636771448069271552, user_id: 9, status: INSERT_TEST
order_item_id:636771448182517761, order_id: 636771448069271552, user_id: 9, status: INSERT_TEST
order_item_id:636771448182517761, order_id: 636771448069271552, user_id: 9, status: INSERT_TEST
order_item_id:636771448182517761, order_id: 636771448069271552, user_id: 9, status: INSERT_TEST
order_item_id:636771448325124097, order_id: 636771448253820928, user_id: 10, status: INSERT_TEST
order_item_id:636771448325124097, order_id: 636771448253820928, user_id: 10, status: INSERT_TEST
order_item_id:636771448325124097, order_id: 636771448253820928, user_id: 10, status: INSERT_TEST
order_item_id:636771448325124097, order_id: 636771448253820928, user_id: 10, status: INSERT_TEST
---------------------------- Delete Data ----------------------------
---------------------------- Print Order Data -----------------------
---------------------------- Print OrderItem Data -------------------
-------------- Process Success Finish --------------
```



## 三、总结

本篇记录的是源码解析前的关键两步：看介绍和跑 demo，有助于快速形成初步印象。

为了照顾 ShardingSphere 的新朋友，在环境准备上略嫌啰嗦。



### 参考链接

- [ShardingSphere 官网](https://shardingsphere.apache.org/document/current/en/overview/)
- [idea中git标签（tag）的建立与使用](https://www.shangmayuan.com/a/321d4bef6a5e49149921c379.html)