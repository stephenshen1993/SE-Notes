# ShardingSphere源码解析 -- 解析引擎

---

## 简介

ShardingSphere 的 3 个产品的数据分片主要流程完全一致，核心流程都是：SQL 解析 => 执行器优化 => SQL 路由 => SQL 改写 => SQL 执行 => 结果归并。

今天，我们走进SQL解析的核心：解析引擎



## 快速入门

首先，参阅 [官方介绍](https://shardingsphere.apache.org/document/current/cn/features/sharding/principle/parse/)，快速了解解析引擎的大致原理。

简单来说，解析引擎执行分为两步：

1. **解析抽象语法树**：解析过程分为词法解析和语法解析。 词法解析器用于将 SQL 拆解为不可再分的原子符号，称为 Token。并根据不同数据库方言所提供的字典，将其归类为关键字，表达式，字面量和操作符。 再使用语法解析器将词法解析器的输出转换为抽象语法树。
2. **提炼分片上下文**：通过 `visitor` 对抽象语法树遍历构造域模型，通过域模型（`SQLStatement`）去提炼分片所需的上下文，并标记有可能需要改写的位置。



## 源码解析

### SQL解析引擎

```java
public final class ShardingSphereSQLParserEngine {
    // SQLStatement 解析引擎
    private final SQLStatementParserEngine sqlStatementParserEngine;
    // distSQLStatement 解析引擎
    private final DistSQLStatementParserEngine distSQLStatementParserEngine;
    
    public ShardingSphereSQLParserEngine(final String databaseTypeName) {
        sqlStatementParserEngine = SQLStatementParserEngineFactory.getSQLStatementParserEngine(databaseTypeName);
        distSQLStatementParserEngine = new DistSQLStatementParserEngine();
    }
    
  	// 解析入口
    @SuppressWarnings("OverlyBroadCatchBlock")
    public SQLStatement parse(final String sql, final boolean useCache) {
        try {
            return parse0(sql, useCache);
            // CHECKSTYLE:OFF
            // TODO check whether throw SQLParsingException only
        } catch (final Exception ex) {
            // CHECKSTYLE:ON
            throw ex;
        }
    }
    
  	// 内部解析方法
    private SQLStatement parse0(final String sql, final boolean useCache) {
        try {
          	// 调用 SQLStatement 解析引擎 进行解析
            return sqlStatementParserEngine.parse(sql, useCache);
        } catch (final SQLParsingException | ParseCancellationException originalEx) {
            // 若解析失败，则使用 distSQLStatement 解析引擎 进行解析
          	try {
                return distSQLStatementParserEngine.parse(sql);
            } catch (final SQLParsingException ignored) {
                // 解析失败时，丢弃当前异常，抛出原异常
              	throw originalEx;
            }
        }
    }
}
```



### SQLStatement 解析引擎

```java
public final class SQLStatementParserEngine {
    // 解析执行器
    private final SQLStatementParserExecutor sqlStatementParserExecutor;
    // cache
    private final LoadingCache<String, SQLStatement> sqlStatementCache;
    
    public SQLStatementParserEngine(final String databaseType) {
        sqlStatementParserExecutor = new SQLStatementParserExecutor(databaseType);
        // 内部是个Guava Cache
        sqlStatementCache = SQLStatementCacheBuilder.build(new CacheOption(2000, 65535L, 4), databaseType);
    }
    
    public SQLStatement parse(final String sql, final boolean useCache) {
        // 走cache还是执行解析
      	return useCache ? sqlStatementCache.getUnchecked(sql) : sqlStatementParserExecutor.parse(sql);
    }
}
```

SQLStatement 解析引擎负责解析出 sqlStatement，内部持有一个解析执行器和一个cache， cache 存放允许缓存的 sqlStatement，用于加速解析过程。

sqlStatementCache 是个 Guava Cache，内部的 load 动作其实也是走 sqlStatementParserExecutor.parse(sql)



### SQLStatement 解析执行器

```java
public final class SQLStatementParserExecutor {
    // sql解析引擎
    private final SQLParserEngine parserEngine;
    // sql访问器引擎
    private final SQLVisitorEngine visitorEngine;
    
    public SQLStatementParserExecutor(final String databaseType) {
        parserEngine = new SQLParserEngine(databaseType);
      	// 初始化为STATEMENT类型的访问器引擎
        visitorEngine = new SQLVisitorEngine(databaseType, "STATEMENT", new Properties());
    }
    
    public SQLStatement parse(final String sql) {
        // step1：解析sql，生成抽象语法树
        // step2：访问抽象语法树，生成sqlStatment
      	return visitorEngine.visit(parserEngine.parse(sql, false));
    }
}
```

构造时，提前初始化为STATEMENT类型的访问器引擎；解析时，先解析成语法树再访问；最终返回 SQLStatement









## 总结