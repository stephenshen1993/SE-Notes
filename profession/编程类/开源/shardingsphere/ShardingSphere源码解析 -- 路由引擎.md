# ShardingSphere源码解析 -- 路由引擎

---

## 简介

ShardingSphere 数据分片的核心流程为：SQL 解析 => 执行器优化 => SQL 路由 => SQL 改写 => SQL 执行 => 结果归并。

今天，我们走进 SQL 路由的核心：路由引擎



## 快速入门

首先，参阅 [官方介绍](https://shardingsphere.apache.org/document/current/cn/features/sharding/principle/route/)，快速了解路由引擎。



简单来说，路由引擎的职责就是根据解析上下文匹配数据库和表的分片策略，并生成路由路径。

根据 SQL 是否携带分片键，路由可以分为：

- 分片路由

  用于根据分片键进行路由的场景，又细分为直接路由、标准路由和笛卡尔积路由这 3 种类型。

- 广播路由

  对于不携带分片键的 SQL，则采取广播路由的方式。根据 SQL 类型又可以划分为全库表路由、全库路由、全实例路由、单播路由和阻断路由这 5 种类型。



路由引擎的整体结构划分如下图：

![img](https://raw.githubusercontent.com/stephenshen1993/PicBed/main/img/20210901074150.png)



## 源码解析

本次解析的起点是路由引擎，位于 /shardingsphere-infra/shardingsphere-infra-route/ 子项目，项目包结构如下图：

![image-20210901074904526](https://raw.githubusercontent.com/stephenshen1993/PicBed/main/img/20210901074904.png)



### SQLRouteEngine

打开路由引擎 SQLRouteEngine，包路径：org.apache.shardingsphere.infra.route.engine.SQLRouteEngine

```java
@RequiredArgsConstructor
public final class SQLRouteEngine {
    // 规则集
    private final Collection<ShardingSphereRule> rules;
    // 配置属性
    private final ConfigurationProperties props;
    
    // 路由入口
    public RouteContext route(final LogicSQL logicSQL, final ShardingSphereMetaData metaData) {
        // 根据是否需要走所有的schema，选择 SQL 路由执行器
        SQLRouteExecutor executor = isNeedAllSchemas(logicSQL.getSqlStatementContext().getSqlStatement()) ? new AllSQLRouteExecutor() : new PartialSQLRouteExecutor(rules, props);
        // 路由执行
        return executor.route(logicSQL, metaData);
    }
    
    // TODO use dynamic config to judge UnconfiguredSchema
    // 是否需要走所有的schema
    private boolean isNeedAllSchemas(final SQLStatement sqlStatement) {
        return sqlStatement instanceof MySQLShowTablesStatement;
    }
}
```

SQLRouteEngine 是路由的入口，主要职责：根据是否需要走 all schemas 选择路由执行器，然后执行。



### AllSQLRouteExecutor

AllSQLRouteExecutor 内容如下：

```java
public final class AllSQLRouteExecutor implements SQLRouteExecutor {
    
    @Override
    public RouteContext route(final LogicSQL logicSQL, final ShardingSphereMetaData metaData) {
        RouteContext result = new RouteContext();
        // 遍历数据源名称
        for (String each : metaData.getResource().getDataSources().keySet()) {
            // 添加路由单元
            result.getRouteUnits().add(new RouteUnit(new RouteMapper(each, each), Collections.emptyList()));
        }
        return result;
    }
}
```

此处用到 RouteUnit 和 RouteMapper，先进去看下

RouteUnit 内容如下：

```java
@RequiredArgsConstructor
@Getter
@EqualsAndHashCode
@ToString
public final class RouteUnit {
    // 数据源映射
    private final RouteMapper dataSourceMapper;
    // 表映射
    private final Collection<RouteMapper> tableMappers;
    
    // 获取全部逻辑表名
    public Set<String> getLogicTableNames() {
        return tableMappers.stream().map(RouteMapper::getLogicName).collect(Collectors.toCollection(() -> new HashSet<>(tableMappers.size(), 1)));
    }
    
    // 获取全部真实表名
    public Set<String> getActualTableNames(final String logicTableName) {
        return tableMappers.stream().filter(each -> logicTableName.equalsIgnoreCase(each.getLogicName())).map(RouteMapper::getActualName).collect(Collectors.toSet());
    }
    
    // 查找表映射
    public Optional<RouteMapper> findTableMapper(final String logicDataSourceName, final String actualTableName) {
        for (RouteMapper each : tableMappers) {
            if (logicDataSourceName.equalsIgnoreCase(dataSourceMapper.getLogicName()) && actualTableName.equalsIgnoreCase(each.getActualName())) {
                return Optional.of(each);
            }
        }
        return Optional.empty();
    }
}
```

其内部持有数据源映射和表映射，提供了获取逻辑表/真实表名和查找表映射的方法。



### PartialSQLRouteExecutor

PartialSQLRouteExecutor 内容如下：

```java
public final class PartialSQLRouteExecutor implements SQLRouteExecutor {
    
    static {
        // 注册路由服务
        ShardingSphereServiceLoader.register(SQLRouter.class);
    }
    // 配置属性
    private final ConfigurationProperties props;
    
    @SuppressWarnings("rawtypes")
    // 路由表
    private final Map<ShardingSphereRule, SQLRouter> routers;
    
    public PartialSQLRouteExecutor(final Collection<ShardingSphereRule> rules, final ConfigurationProperties props) {
        this.props = props;
        // 获取注册的路由服务
        routers = OrderedSPIRegistry.getRegisteredServices(rules, SQLRouter.class);
    }
    
    @Override
    @SuppressWarnings({"unchecked", "rawtypes"})
    public RouteContext route(final LogicSQL logicSQL, final ShardingSphereMetaData metaData) {
        RouteContext result = new RouteContext();
        // 遍历路由表
        for (Entry<ShardingSphereRule, SQLRouter> entry : routers.entrySet()) {
            if (result.getRouteUnits().isEmpty()) {
                // 创建路由上下文
                result = entry.getValue().createRouteContext(logicSQL, metaData, entry.getKey(), props);
            } else {
                // 装饰路由上下文
                entry.getValue().decorateRouteContext(result, logicSQL, metaData, entry.getKey(), props);
            }
        }
        // 单数据源处理
        if (result.getRouteUnits().isEmpty() && 1 == metaData.getResource().getDataSources().size()) {
            String singleDataSourceName = metaData.getResource().getDataSources().keySet().iterator().next();
            result.getRouteUnits().add(new RouteUnit(new RouteMapper(singleDataSourceName, singleDataSourceName), Collections.emptyList()));
        }
        return result;
    }
}
```





## 总结