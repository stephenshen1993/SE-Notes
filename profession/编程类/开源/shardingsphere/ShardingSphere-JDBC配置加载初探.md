# ShardingSphere-JDBC 配置加载初探

## 简介

前面体验了 ShardingSphere 的几个示例，对 ShardingSphere 的功能已经有了初步的了解，现在是时候探索源码了。

在体验示例的过程中，我注意到 ShardingSphere 是配置驱动的，但是未能在官网找到配置加载相应的叙述。那么，今天我们就从配置加载开始，一点点揭开 ShardingSphere 的神秘面纱。



## 探索思路

探索之前，需要先锚定我们使用的配置，再根据对应的配置项去反向定位配置加载位置及加载流程。

此处我们选择 shardingsphere-jdbc-example/sharding-example/sharding-spring-boot-mybatis-example/ 示例项目，其配置项相对简单，可以帮助快速理清流程。

因为是 spring-boot 项目，配置项大概率是自动装载的，从示例项目打断点没法断到。故考虑最笨的办法，使用配置项前缀去搜索该前缀出现位置，进而定位到配置类位置及其加载流程。



## 源码探索

### 配置分析

打开 examples 示例，定位到 shardingsphere-jdbc-example/sharding-example/sharding-spring-boot-mybatis-example/ 子项目

先查看启动文件 application.properties，定位到激活的 profile 为 readwrite-splitting

```properties
mybatis.config-location=classpath:META-INF/mybatis-config.xml
spring.profiles.active=readwrite-splitting
```

继续查看 application-readwrite-splitting.properties 的配置内容，根据前缀的不同，可以简单地将其分成 datasource 和 rules 两组

```properties
# 第一组：数据源配置
spring.shardingsphere.datasource.names=write_ds,read_ds_0,read_ds_1

spring.shardingsphere.datasource.write_ds.jdbc-url=jdbc:mysql://localhost:13307/demo_write_ds?serverTimezone=UTC&useSSL=false&useUnicode=true&characterEncoding=UTF-8
spring.shardingsphere.datasource.write_ds.type=com.zaxxer.hikari.HikariDataSource
spring.shardingsphere.datasource.write_ds.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.write_ds.username=root
spring.shardingsphere.datasource.write_ds.password=root

spring.shardingsphere.datasource.read_ds_0.jdbc-url=jdbc:mysql://localhost:13307/demo_read_ds_0?serverTimezone=UTC&useSSL=false&useUnicode=true&characterEncoding=UTF-8
spring.shardingsphere.datasource.read_ds_0.type=com.zaxxer.hikari.HikariDataSource
spring.shardingsphere.datasource.read_ds_0.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.read_ds_0.username=root
spring.shardingsphere.datasource.read_ds_0.password=root

spring.shardingsphere.datasource.read_ds_1.jdbc-url=jdbc:mysql://localhost:13307/demo_read_ds_1?serverTimezone=UTC&useSSL=false&useUnicode=true&characterEncoding=UTF-8
spring.shardingsphere.datasource.read_ds_1.type=com.zaxxer.hikari.HikariDataSource
spring.shardingsphere.datasource.read_ds_1.driver-class-name=com.mysql.jdbc.Driver
spring.shardingsphere.datasource.read_ds_1.username=root
spring.shardingsphere.datasource.read_ds_1.password=root

# 第二组：规则配置
spring.shardingsphere.rules.readwrite-splitting.load-balancers.round_robin.type=ROUND_ROBIN
spring.shardingsphere.rules.readwrite-splitting.data-sources.pr_ds.write-data-source-name=write_ds
spring.shardingsphere.rules.readwrite-splitting.data-sources.pr_ds.read-data-source-names=read_ds_0,read_ds_1
spring.shardingsphere.rules.readwrite-splitting.data-sources.pr_ds.load-balancer-name=round_robin
```

从而提取出两组关键前缀：

- spring.shardingsphere.datasource（数据源前缀）
- spring.shardingsphere.rules（规则前缀）

另外，数据源的 names 项描述了所有的数据源名，猜想应该是先按 names 拿到所有数据源名后，再进一步匹配数据源信息。

最后，规则下的读写分离规则处，有配置读写数据源，配置加载时可能会绑定相关映射关系。



### 源码追踪

#### 1、数据源加载

打开 sharding-sphere 项目，使用 Find in Path 进行查找，只看 java 文件，有且仅有 DataSourceMapSetter 一处是非测试代码。

进入到 DataSourceMapSetter，先看到以下字段：

```java
private static final String PREFIX = "spring.shardingsphere.datasource.";
private static final String DATA_SOURCE_NAME = "name";
private static final String DATA_SOURCE_NAMES = "names";
private static final String DATA_SOURCE_TYPE = "type";
private static final String JNDI_NAME = "jndi-name";
```

发现 PREFIX 有三处引用，重点关注其拼接了 DATA_SOURCE_NAMES 的地方

```java
private static List<String> getDataSourceNames(final Environment environment) {
    StandardEnvironment standardEnv = (StandardEnvironment) environment;
    standardEnv.setIgnoreUnresolvableNestedPlaceholders(true);
  	// 根据 name 获取数据源名称列表
  	String dataSourceNames = standardEnv.getProperty(PREFIX + DATA_SOURCE_NAME);
    if (StringUtils.isEmpty(dataSourceNames)) {
      	// 根据 names 获取数据源名称列表
        dataSourceNames = standardEnv.getProperty(PREFIX + DATA_SOURCE_NAMES);
    }
    return new InlineExpressionParser(dataSourceNames).splitAndEvaluate();
}
```

很明显，此处是个获取数据源名称列表的操作，先尝试按 name 获取，取不到才按 names 获取

查看引用该方法的位置

```java
public static Map<String, DataSource> getDataSourceMap(final Environment environment) {
    Map<String, DataSource> result = new LinkedHashMap<>();
    for (String each : getDataSourceNames(environment)) {
        try {
            result.put(each, getDataSource(environment, each));
        } catch (final ReflectiveOperationException ex) {
            throw new ShardingSphereException("Can't find data source type.", ex);
        } catch (final NamingException ex) {
            throw new ShardingSphereException("Can't find JNDI data source.", ex);
        }
    }
    return result;
}
```

这里遍历数据源名称列表，逐项获取数据源 put 到 result，最后返回

看下如何获取数据源的：

```java
private static DataSource getDataSource(final Environment environment, final String dataSourceName) throws ReflectiveOperationException, NamingException {
    // 拼接数据库前缀，获取数据库属性
  	Map<String, Object> dataSourceProps = PropertyUtil.handle(environment, String.join("", PREFIX, dataSourceName), Map.class);
    Preconditions.checkState(!dataSourceProps.isEmpty(), String.format("Wrong datasource [%s] properties.", dataSourceName));
  	// JNDI 数据源单独处理
    if (dataSourceProps.containsKey(JNDI_NAME)) {
        return getJNDIDataSource(dataSourceProps.get(JNDI_NAME).toString());
    }
  	// 内部通过反射创建数据源
    DataSource result = DataSourceUtil.getDataSource(dataSourceProps.get(DATA_SOURCE_TYPE).toString(), dataSourceProps);
		// 设置数据库自定义属性    
 	DataSourcePropertiesSetterHolder.getDataSourcePropertiesSetterByType(dataSourceProps.get(DATA_SOURCE_TYPE).toString()).ifPresent(
        propsSetter -> propsSetter.propertiesSet(environment, PREFIX, dataSourceName, result));
    return result;
}
```

这里印证了分析时的猜想，确实是先拿到 names，再按 name 前缀匹配属性再创建数据源。

再回到 DataSourceMapSetter#getDataSourceMap，查找到两处引用。

第二处 Governance 明显是治理相关的暂不关心，选择第一处进入 ShardingSphereAutoConfiguration

```java
@Configuration
@ComponentScan("org.apache.shardingsphere.spring.boot.converter")
@EnableConfigurationProperties(SpringBootPropertiesConfiguration.class)
@ConditionalOnProperty(prefix = "spring.shardingsphere", name = "enabled", havingValue = "true", matchIfMissing = true)
@AutoConfigureBefore(DataSourceAutoConfiguration.class)
@RequiredArgsConstructor
public class ShardingSphereAutoConfiguration implements EnvironmentAware {
    
    private final SpringBootPropertiesConfiguration props;
    
    private final Map<String, DataSource> dataSourceMap = new LinkedHashMap<>();
    
    @Bean
    @Autowired(required = false)
    public DataSource shardingSphereDataSource(final ObjectProvider<List<RuleConfiguration>> rules) throws SQLException {
        Collection<RuleConfiguration> ruleConfigurations = Optional.ofNullable(rules.getIfAvailable()).orElse(Collections.emptyList());
        // 根据数据源map、规则配置和额外的配置属性，创建 shardingSphereDataSource
      	return ShardingSphereDataSourceFactory.createDataSource(dataSourceMap, ruleConfigurations, props.getProps());
    }
  
    @Override
    public final void setEnvironment(final Environment environment) {
        dataSourceMap.putAll(DataSourceMapSetter.getDataSourceMap(environment));
    }
}
```

这个自动装配类实现了 EnvironmentAware 接口，通过  setEnvironment 方法，在项目启动时拿到 dataSourceMap。

类上通过 @EnableConfigurationProperties 启用 SpringBootPropertiesConfiguration 配置类，再配合 @RequiredArgsConstructor 构造注入到 props。

最后，根据数据源map、规则配置和额外的配置属性，创建 shardingSphereDataSource bean



#### 2、规则加载

上面未解决的问题是：rules 从哪儿来的？

咱们继续追踪，同样的配方使用 spring.shardingsphere.rules 反向查找，定位到 YamlReadwriteSplittingRuleSpringBootConfiguration 配置类

```java
@ConfigurationProperties(prefix = "spring.shardingsphere.rules")
@Getter
@Setter
public final class YamlReadwriteSplittingRuleSpringBootConfiguration {
    
    private YamlReadwriteSplittingRuleConfiguration readwriteSplitting;
}
```

再向上查找引用，定位到 ReadwriteSplittingRuleSpringbootConfiguration

```java
@Configuration
@EnableConfigurationProperties(YamlReadwriteSplittingRuleSpringBootConfiguration.class)
@ConditionalOnClass(YamlReadwriteSplittingRuleConfiguration.class)
@Conditional(ReadwriteSplittingSpringBootCondition.class)
@RequiredArgsConstructor
public class ReadwriteSplittingRuleSpringbootConfiguration {
    
    private final ReadwriteSplittingRuleAlgorithmProviderConfigurationYamlSwapper swapper = new ReadwriteSplittingRuleAlgorithmProviderConfigurationYamlSwapper();
    
    private final YamlReadwriteSplittingRuleSpringBootConfiguration yamlConfig;
    
    @Bean
    public RuleConfiguration readWriteSplittingRuleConfiguration(final ObjectProvider<Map<String, ReplicaLoadBalanceAlgorithm>> loadBalanceAlgorithms) {
        // 获取读写分离配置并转化为提供读写分离规则配置的算法对象
      	AlgorithmProvidedReadwriteSplittingRuleConfiguration result = swapper.swapToObject(yamlConfig.getReadwriteSplitting());
        Map<String, ReplicaLoadBalanceAlgorithm> balanceAlgorithmMap = Optional.ofNullable(loadBalanceAlgorithms.getIfAvailable()).orElse(Collections.emptyMap());
        // 绑定负载均衡算法
      	result.setLoadBalanceAlgorithms(balanceAlgorithmMap);
        return result;
    }
}
```

此处使用读写分离规则配置和负载均衡算法，创建提供读写分离规则的算法配置 bean。

这里创建的 RuleConfiguration bean，将通过 ObjectProvider<List<RuleConfiguration>> 自动注入到 ShardingSphereAutoConfiguration#shardingSphereDataSource，从而完成创建 shardingSphereDataSource 的要素闭环。

shardingSphereDataSource 是个聚合的数据源，内部持有所有的数据源信息和规则信息，对外包装成 dataSource bean 与应用进行交互。



## 总结

这次以配置为切入点，探索了 ShardingSphere-JDBC 的配置加载流程，包含数据源加载和规则加载两块儿内容。

简单来说，ShardingSphere 借助 springboot 强大的自动配置能力，将配置组转换为配置类，再封装成数据源和规则配置对象，最后聚合为 shardingSphereDataSource。

可以猜到，后续应该是由 shardingSphereDataSource 接管全部的数据源操作，内部再根据具体配置的规则进行分片、路由和归并等操作。