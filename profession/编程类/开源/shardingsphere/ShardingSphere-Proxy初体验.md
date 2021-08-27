# ShardingSphere-Proxy 初体验

## 引言

前面已经搭建好本地环境，本篇来体验下 ShardingSphere-Proxy 的三个功能：分库分表、读写分离、数据加密。



## 官方文档

在体验功能之前，我们先了解下官方文档：

- [ShardingSphere概览](https://shardingsphere.apache.org/document/current/cn/overview/)
- 概念&功能
  - [数据分片](https://shardingsphere.apache.org/document/current/cn/features/sharding/)
  - [读写分离](https://shardingsphere.apache.org/document/current/cn/features/readwrite-splitting/)
  - [数据加密](https://shardingsphere.apache.org/document/current/cn/features/encrypt/)
- 配置参考说明
  - [数据源配置](https://shardingsphere.apache.org/document/current/cn/user-manual/shardingsphere-proxy/configuration/data-source/)
  - [权限配置](https://shardingsphere.apache.org/document/current/cn/user-manual/shardingsphere-proxy/configuration/authentication/)
  - [属性配置](https://shardingsphere.apache.org/document/current/cn/user-manual/shardingsphere-proxy/configuration/props/)

ShardingSphere-Proxy 定位为透明的数据库代理端，非侵入式，可直接当做 MySQL/PostgreSQL 使用，但同时也增加了系统复杂度。



## 数据分片

### 1、准备数据

连接信息（MySQL）：

```
- host：127.0.0.1
- port：13307
- userName：root
- password：root
```

执行如下 SQL 建立分片所需数据库：

```sql
CREATE SCHEMA IF NOT EXISTS demo_ds_0;
CREATE SCHEMA IF NOT EXISTS demo_ds_1;
```



### 2、配置 Proxy

1）打开 ShardingSphere 项目，定位到 shardingsphere/shardingsphere-proxy 子项目。

2）配置 server.yaml ，放开 rules 和 props 的注释，sql-show改为true。
文件路径：shardingsphere-proxy/shardingsphere-proxy-bootstrap/src/main/resources/conf/server.yaml

```yaml
rules:
  - !AUTHORITY
    users:
      - root@%:root
      - sharding@:sharding
    provider:
      type: NATIVE
props:
  max-connections-size-per-query: 1
  executor-size: 16  # Infinite by default.
  proxy-frontend-flush-threshold: 128  # The default value is 128.
    # LOCAL: Proxy will run with LOCAL transaction.
  # XA: Proxy will run with XA transaction.
  # BASE: Proxy will run with B.A.S.E transaction.
  proxy-transaction-type: LOCAL
  xa-transaction-manager-type: Atomikos
  proxy-opentracing-enabled: false
  proxy-hint-enabled: false
  sql-show: true
  check-table-metadata-enabled: false
  lock-wait-timeout-milliseconds: 50000 # The maximum time to wait for a lock
  show-process-list-enabled: false
```

rules 部分配置了 proxy 的授权数据，props 部分是 proxy 的技术参数。

3）配置 config-sharding.yaml，放开最下面 schemaName、dataSources、rules 的注释，并调整 url、username 和 password
文件路径：shardingsphere-proxy/shardingsphere-proxy-bootstrap/src/main/resources/conf/config-sharding.yaml

```yaml
schemaName: sharding_db

dataSources:
  ds_0:
    url: jdbc:mysql://127.0.0.1:13307/demo_ds_0?serverTimezone=UTC&useSSL=false
    username: root
    password: root
    connectionTimeoutMilliseconds: 30000
    idleTimeoutMilliseconds: 60000
    maxLifetimeMilliseconds: 1800000
    maxPoolSize: 50
    minPoolSize: 1
    maintenanceIntervalMilliseconds: 30000
  ds_1:
    url: jdbc:mysql://127.0.0.1:13307/demo_ds_1?serverTimezone=UTC&useSSL=false
    username: root
    password: root
    connectionTimeoutMilliseconds: 30000
    idleTimeoutMilliseconds: 60000
    maxLifetimeMilliseconds: 1800000
    maxPoolSize: 50
    minPoolSize: 1
    maintenanceIntervalMilliseconds: 30000

rules:
  - !SHARDING
    tables:
      t_order:
        actualDataNodes: ds_${0..1}.t_order_${0..1}
        tableStrategy:
          standard:
            shardingColumn: order_id
            shardingAlgorithmName: t_order_inline
        keyGenerateStrategy:
          column: order_id
          keyGeneratorName: snowflake
      t_order_item:
        actualDataNodes: ds_${0..1}.t_order_item_${0..1}
        tableStrategy:
          standard:
            shardingColumn: order_id
            shardingAlgorithmName: t_order_item_inline
        keyGenerateStrategy:
          column: order_item_id
          keyGeneratorName: snowflake
    bindingTables:
      - t_order,t_order_item
    defaultDatabaseStrategy:
      standard:
        shardingColumn: user_id
        shardingAlgorithmName: database_inline
    defaultTableStrategy:
      none:

    shardingAlgorithms:
      database_inline:
        type: INLINE
        props:
          algorithm-expression: ds_${user_id % 2}
      t_order_inline:
        type: INLINE
        props:
          algorithm-expression: t_order_${order_id % 2}
      t_order_item_inline:
        type: INLINE
        props:
          algorithm-expression: t_order_item_${order_id % 2}

    keyGenerators:
      snowflake:
        type: SNOWFLAKE
        props:
          worker-id: 123
```

4）启动 proxy
启动类位置：shardingsphere-proxy/shardingsphere-proxy-bootstrap/src/main/java/org/apache/shardingsphere/proxy/Bootstrap.java



启动成功日志：

```
Thanks for using Atomikos! Evaluate http://www.atomikos.com/Main/ExtremeTransactions for advanced features and professional support
or register at http://www.atomikos.com/Main/RegisterYourDownload to disable this message and receive FREE tips & advice
[INFO ] 2021-08-25 06:47:41.552 [main] o.a.s.p.i.i.AbstractBootstrapInitializer - Database name is `MySQL`, version is `5.7.32`
[INFO ] 2021-08-25 06:47:41.928 [main] o.a.s.p.frontend.ShardingSphereProxy - ShardingSphere-Proxy start success.
```



### 3、运行示例

1）打开 examples 项目，定位到 examples/shardingsphere-proxy-example/shardingsphere-proxy-boot-mybatis-example 子项目。

2）配置 application.yaml，使其与 proxy 数据源信息保持一致。
文件路径：shardingsphere-proxy-boot-mybatis-example/src/main/resources/application.properties

```properties
mybatis.config-location=classpath:META-INF/mybatis-config.xml
spring.datasource.type=com.zaxxer.hikari.HikariDataSource
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3307/sharding_db?useServerPrepStmts=true&cachePrepStmts=true
spring.datasource.username=root
spring.datasource.password=root
```

3）运行示例启动类，类路径：org.apache.shardingsphere.example.proxy.spring.boot.mybatis.SpringBootStarterExample

启动日志：

```
-------------- Process Success Begin ---------------
---------------------------- Insert Data ----------------------------
---------------------------- Print Order Data -----------------------
order_id: 637181105006948352, user_id: 1, address_id: 1, status: INSERT_TEST
order_id: 637181106542063616, user_id: 2, address_id: 2, status: INSERT_TEST
order_id: 637181106651115520, user_id: 3, address_id: 3, status: INSERT_TEST
order_id: 637181106739195904, user_id: 4, address_id: 4, status: INSERT_TEST
order_id: 637181106831470592, user_id: 5, address_id: 5, status: INSERT_TEST
order_id: 637181106906968064, user_id: 6, address_id: 6, status: INSERT_TEST
order_id: 637181107003437056, user_id: 7, address_id: 7, status: INSERT_TEST
order_id: 637181107095711744, user_id: 8, address_id: 8, status: INSERT_TEST
order_id: 637181107187986432, user_id: 9, address_id: 9, status: INSERT_TEST
order_id: 637181107301232640, user_id: 10, address_id: 10, status: INSERT_TEST
---------------------------- Print OrderItem Data -------------------
order_item_id:637181106420428801, order_id: 637181105006948352, user_id: 1, status: INSERT_TEST
order_item_id:637181106588200961, order_id: 637181106542063616, user_id: 2, status: INSERT_TEST
order_item_id:637181106688864257, order_id: 637181106651115520, user_id: 3, status: INSERT_TEST
order_item_id:637181106781138945, order_id: 637181106739195904, user_id: 4, status: INSERT_TEST
order_item_id:637181106869219329, order_id: 637181106831470592, user_id: 5, status: INSERT_TEST
order_item_id:637181106965688321, order_id: 637181106906968064, user_id: 6, status: INSERT_TEST
order_item_id:637181107049574401, order_id: 637181107003437056, user_id: 7, status: INSERT_TEST
order_item_id:637181107141849089, order_id: 637181107095711744, user_id: 8, status: INSERT_TEST
order_item_id:637181107229929473, order_id: 637181107187986432, user_id: 9, status: INSERT_TEST
order_item_id:637181107372535809, order_id: 637181107301232640, user_id: 10, status: INSERT_TEST
---------------------------- Delete Data ----------------------------
---------------------------- Print Order Data -----------------------
---------------------------- Print OrderItem Data -------------------
-------------- Process Success Finish --------------
-------------- Process Failure Begin ---------------
---------------------------- Insert Data ----------------------------
-------------- Process Failure Finish --------------
Exception occur for transaction test.
---------------------------- Print Order Data -----------------------
---------------------------- Print OrderItem Data -------------------
```

启动后，看到 proxy 的出现如下格式日志：

```
[INFO ] 2021-08-25 06:50:14.287 [ShardingSphere-Command-1] ShardingSphere-SQL - Logic SQL: SELECT * FROM t_order;
[INFO ] 2021-08-25 06:50:14.287 [ShardingSphere-Command-1] ShardingSphere-SQL - SQLStatement: MySQLSelectStatement(limit=Optional.empty, lock=Optional.empty, window=Optional.empty)
[INFO ] 2021-08-25 06:50:14.287 [ShardingSphere-Command-1] ShardingSphere-SQL - Actual SQL: ds_0 ::: SELECT * FROM t_order_0 ORDER BY order_id ASC ;
[INFO ] 2021-08-25 06:50:14.287 [ShardingSphere-Command-1] ShardingSphere-SQL - Actual SQL: ds_0 ::: SELECT * FROM t_order_1 ORDER BY order_id ASC ;
[INFO ] 2021-08-25 06:50:14.288 [ShardingSphere-Command-1] ShardingSphere-SQL - Actual SQL: ds_1 ::: SELECT * FROM t_order_0 ORDER BY order_id ASC ;
[INFO ] 2021-08-25 06:50:14.288 [ShardingSphere-Command-1] ShardingSphere-SQL - Actual SQL: ds_1 ::: SELECT * FROM t_order_1 ORDER BY order_id ASC ;
```

大致可以看出来，分片策略是生效的。



## 读写分离

### 1、准备数据

连接信息（MySQL）：

```
- host：127.0.0.1
- port：13307
- userName：root
- password：root
```

执行如下 SQL 建立分片所需数据库：

```sql
CREATE SCHEMA IF NOT EXISTS demo_write_ds;
CREATE SCHEMA IF NOT EXISTS demo_read_ds_0;
CREATE SCHEMA IF NOT EXISTS demo_read_ds_1;

CREATE TABLE IF NOT EXISTS demo_read_ds_0.t_order (order_id BIGINT NOT NULL AUTO_INCREMENT, user_id INT NOT NULL, status VARCHAR(50), PRIMARY KEY (order_id));
CREATE TABLE IF NOT EXISTS demo_read_ds_1.t_order (order_id BIGINT NOT NULL AUTO_INCREMENT, user_id INT NOT NULL, status VARCHAR(50), PRIMARY KEY (order_id));
CREATE TABLE IF NOT EXISTS demo_read_ds_0.t_order_item (order_item_id BIGINT NOT NULL AUTO_INCREMENT, order_id BIGINT NOT NULL, user_id INT NOT NULL, status VARCHAR(50), PRIMARY KEY (order_item_id));
CREATE TABLE IF NOT EXISTS demo_read_ds_1.t_order_item (order_item_id BIGINT NOT NULL AUTO_INCREMENT, order_id BIGINT NOT NULL, user_id INT NOT NULL, status VARCHAR(50), PRIMARY KEY (order_item_id));
```



### 2、配置 Proxy

1）打开 ShardingSphere 项目，定位到 shardingsphere/shardingsphere-proxy 子项目。

2）配置 server.yaml ，此处沿用上面数据分片的配置

3）为了单独演示读书分离，将上面数据分片的 config-sharding.yaml 配置内容注释掉。

4）配置 config-readwrite-splitting.yaml，放开最下面 schemaName、dataSources、rules 的注释，并调整 schemaName、url、username 和 password 到实际值。
文件路径：shardingsphere-proxy-bootstrap/src/main/resources/conf/config-readwrite-splitting.yaml

```yaml
schemaName: readwrite_splitting_db
dataSources:
  write_ds:
    url: jdbc:mysql://127.0.0.1:13307/demo_write_ds?serverTimezone=UTC&useSSL=false
    username: root
    password: root
    connectionTimeoutMilliseconds: 30000
    idleTimeoutMilliseconds: 60000
    maxLifetimeMilliseconds: 1800000
    maxPoolSize: 50
    minPoolSize: 1
    maintenanceIntervalMilliseconds: 30000
  read_ds_0:
    url: jdbc:mysql://127.0.0.1:13307/demo_read_ds_0?serverTimezone=UTC&useSSL=false
    username: root
    password: root
    connectionTimeoutMilliseconds: 30000
    idleTimeoutMilliseconds: 60000
    maxLifetimeMilliseconds: 1800000
    maxPoolSize: 50
    minPoolSize: 1
    maintenanceIntervalMilliseconds: 30000
  read_ds_1:
    url: jdbc:mysql://127.0.0.1:13307/demo_read_ds_1?serverTimezone=UTC&useSSL=false
    username: root
    password: root
    connectionTimeoutMilliseconds: 30000
    idleTimeoutMilliseconds: 60000
    maxLifetimeMilliseconds: 1800000
    maxPoolSize: 50
    minPoolSize: 1
    maintenanceIntervalMilliseconds: 30000
rules:
  - !READWRITE_SPLITTING
    dataSources:
      pr_ds:
        writeDataSourceName: write_ds
        readDataSourceNames:
          - read_ds_0
          - read_ds_1
```

4）启动 proxy
启动类位置：shardingsphere-proxy/shardingsphere-proxy-bootstrap/src/main/java/org/apache/shardingsphere/proxy/Bootstrap.java

启动日志：

```
Thanks for using Atomikos! Evaluate http://www.atomikos.com/Main/ExtremeTransactions for advanced features and professional support
or register at http://www.atomikos.com/Main/RegisterYourDownload to disable this message and receive FREE tips & advice
[INFO ] 2021-08-25 08:54:42.775 [main] o.a.s.p.i.i.AbstractBootstrapInitializer - Database name is `MySQL`, version is `5.7.32`
[INFO ] 2021-08-25 08:54:43.315 [main] o.a.s.p.frontend.ShardingSphereProxy - ShardingSphere-Proxy start success.
```



### 3、运行示例

1）打开 examples 项目，定位到 examples/shardingsphere-proxy-example/shardingsphere-proxy-boot-mybatis-example 子项目。

2）配置 application.yaml，使其与 proxy 数据源信息保持一致。
文件路径：shardingsphere-proxy-boot-mybatis-example/src/main/resources/application.properties

```properties
mybatis.config-location=classpath:META-INF/mybatis-config.xml
spring.datasource.type=com.zaxxer.hikari.HikariDataSource
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3307/readwrite_splitting_db?useServerPrepStmts=true&cachePrepStmts=true
spring.datasource.username=root
spring.datasource.password=root
```

3）运行示例启动类，类路径：org.apache.shardingsphere.example.proxy.spring.boot.mybatis.SpringBootStarterExample

启动日志：

```
-------------- Process Success Begin ---------------
---------------------------- Insert Data ----------------------------
---------------------------- Print Order Data -----------------------
order_id: 637181105006948352, user_id: 1, address_id: 1, status: INSERT_TEST
order_id: 637181106542063616, user_id: 2, address_id: 2, status: INSERT_TEST
order_id: 637181106651115520, user_id: 3, address_id: 3, status: INSERT_TEST
order_id: 637181106739195904, user_id: 4, address_id: 4, status: INSERT_TEST
order_id: 637181106831470592, user_id: 5, address_id: 5, status: INSERT_TEST
order_id: 637181106906968064, user_id: 6, address_id: 6, status: INSERT_TEST
order_id: 637181107003437056, user_id: 7, address_id: 7, status: INSERT_TEST
order_id: 637181107095711744, user_id: 8, address_id: 8, status: INSERT_TEST
order_id: 637181107187986432, user_id: 9, address_id: 9, status: INSERT_TEST
order_id: 637181107301232640, user_id: 10, address_id: 10, status: INSERT_TEST
---------------------------- Print OrderItem Data -------------------
order_item_id:637181106420428801, order_id: 637181105006948352, user_id: 1, status: INSERT_TEST
order_item_id:637181106588200961, order_id: 637181106542063616, user_id: 2, status: INSERT_TEST
order_item_id:637181106688864257, order_id: 637181106651115520, user_id: 3, status: INSERT_TEST
order_item_id:637181106781138945, order_id: 637181106739195904, user_id: 4, status: INSERT_TEST
order_item_id:637181106869219329, order_id: 637181106831470592, user_id: 5, status: INSERT_TEST
order_item_id:637181106965688321, order_id: 637181106906968064, user_id: 6, status: INSERT_TEST
order_item_id:637181107049574401, order_id: 637181107003437056, user_id: 7, status: INSERT_TEST
order_item_id:637181107141849089, order_id: 637181107095711744, user_id: 8, status: INSERT_TEST
order_item_id:637181107229929473, order_id: 637181107187986432, user_id: 9, status: INSERT_TEST
order_item_id:637181107372535809, order_id: 637181107301232640, user_id: 10, status: INSERT_TEST
---------------------------- Delete Data ----------------------------
---------------------------- Print Order Data -----------------------
---------------------------- Print OrderItem Data -------------------
-------------- Process Success Finish --------------
-------------- Process Failure Begin ---------------
---------------------------- Insert Data ----------------------------
-------------- Process Failure Finish --------------
Exception occur for transaction test.
---------------------------- Print Order Data -----------------------
---------------------------- Print OrderItem Data -------------------
```

启动后，看到 proxy 的出现如下格式日志：

```
[INFO ] 2021-08-25 09:02:40.911 [ShardingSphere-Command-1] ShardingSphere-SQL - Logic SQL: INSERT INTO t_order (user_id, address_id, status) VALUES (?, ?, ?);
[INFO ] 2021-08-25 09:02:40.911 [ShardingSphere-Command-1] ShardingSphere-SQL - SQLStatement: MySQLInsertStatement(setAssignment=Optional.empty, onDuplicateKeyColumns=Optional.empty)
[INFO ] 2021-08-25 09:02:40.912 [ShardingSphere-Command-1] ShardingSphere-SQL - Actual SQL: write_ds ::: INSERT INTO t_order (user_id, address_id, status) VALUES (?, ?, ?); ::: [5, 5, INSERT_TEST]
...
[INFO ] 2021-08-25 09:02:41.285 [ShardingSphere-Command-0] ShardingSphere-SQL - Logic SQL: SELECT * FROM t_order;
[INFO ] 2021-08-25 09:02:41.285 [ShardingSphere-Command-0] ShardingSphere-SQL - SQLStatement: MySQLSelectStatement(limit=Optional.empty, lock=Optional.empty, window=Optional.empty)
[INFO ] 2021-08-25 09:02:41.285 [ShardingSphere-Command-0] ShardingSphere-SQL - Actual SQL: read_ds_1 ::: SELECT * FROM t_order;
```

大致可以看出来，读写分离策略是正常的读走读库、写走写库。

大致可以看出来，读写分离策略是正常的读走读库、写走写库。



## 数据加密

### 1、准备数据

连接信息（MySQL）：

```
- host：127.0.0.1
- port：13307
- userName：root
- password：root
```

执行如下 SQL 建立分片所需数据库：

### 2、配置 Proxy

### 3、运行示例



## 总结