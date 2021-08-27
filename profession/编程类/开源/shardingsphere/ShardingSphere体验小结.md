# ShardingSphere 体验小结

## 简介

在前面几天里，我们体验了 ShardingSphere 的 JDBC、Proxy 和 UI，结合官方文档，对 ShardingSphere 的产品和功能有了大体的认知，在这里简单做个小结。



## 产品列表

截至发文，ShardingSphere 的产品矩阵如下：

```
- ShardingSphere-JDBC（轻量级Java框架）
- ShardingSphere-Proxy（透明化数据库代理）
- ShardingSphere-Sidecar（云原生数据库代理）
- ShardingSphere-Scaling（数据接入迁移及弹性伸缩解决方案）
- ShardingSphere-UI（web管理控制台）
```

其中，JDBC、Proxy、Scaling 已发布 5.0.0-beta 版本，UI 还停留在 5.0.0-alpha 版本，Sidecar 仍在规划中。



### ShardingSphere-JDBC

定位为轻量级 Java 框架，在 Java 的 JDBC 层提供的额外服务。

它使用客户端直连数据库，以 jar 包形式提供服务，无需额外部署和依赖，可理解为增强版的 JDBC 驱动，兼容 JDBC 和各种 ORM 框架。



### ShardingSphere-Proxy

ShardingSphere-Proxy 定位为透明化的数据库代理端，提供封装了数据库二进制协议的服务端版本，用于完成对异构语言的支持。。

目前提供 MySQL 和 PostgreSQL 版本，它可以使用兼容 MySQL/PostgreSQL 协议的访问客户端操作数据。



### ShardingSphere-Sidecar

ShardingSphere-Sidecar 定位为 Kubernetes 或 Mesos 的云原生数据库代理，以 DaemonSet 的形式代理所有对数据库的访问。



### ShardingSphere-Scaling

ShardingSphere-Scaling 是一个提供给用户的通用的 ShardingSphere 数据接入迁移，及弹性伸缩的解决方案。



### ShardingSphere-UI

ShardingSphere-UI 是 ShardingSphere 的一个简单而有用的web管理控制台。

它用于帮助用户更简单地使用 ShardingSphere 相关功能，目前提供注册中心管理、动态配置管理、数据库编排等功能。



### 方案选型

|            | ShardingSphere-JDBC | ShardingSphere-Proxy | ShardingSphere-Sidecar |
| :--------- | :------------------ | :------------------- | ---------------------- |
| 数据库     | 任意                | MySQL/PostgreSQL     | MySQL/PostgreSQL       |
| 连接消耗数 | 高                  | 低                   | 高                     |
| 异构语言   | 仅 Java             | 任意                 | 任意                   |
| 性能       | 损耗低              | 损耗略高             | 损耗低                 |
| 无中心化   | 是                  | 否                   | 是                     |
| 静态入口   | 无                  | 有                   | 无                     |



## 功能列表

截至发文，ShardingSphere 的功能支持如下：

```
- 数据分片
	- 分库 & 分表
	- 读写分离
	- 分片策略定制化
	- 无中心化分布式主键
- 分布式事务
	- 标准化事务接口
  - XA 强一致事务
  - 柔性事务
  - 数据库治理
- 分布式治理
  - 弹性伸缩
  - 可视化链路追踪
  - 数据加密
```