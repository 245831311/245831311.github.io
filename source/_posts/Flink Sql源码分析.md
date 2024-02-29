---
title: Flink Sql源码分析
date: 2023-07-10 11:16:39
categories: 
- 大数据
tags:
- Flink
---

# 1. &#x20;flink-connector-jdbc源码分析

入口：resources/META_INF.services/org.apache.flink.table.factories.Factory

```java
org.apache.flink.connector.jdbc.table.JdbcDynamicTableFactory
org.apache.flink.connector.jdbc.catalog.factory.JdbcCatalogFactory
```

## 1.1. JdbcDynamicTableFactory

管理DynamicTableSink、DynamicTableSource、配置信息Options，DynamicTableSink负责向表写数据，DynamicTableSource负责从表读数据

```java
public class JdbcDynamicTableFactory implements DynamicTableSourceFactory, DynamicTableSinkFactory {
  public DynamicTableSink createDynamicTableSink(Context context);
  public DynamicTableSource createDynamicTableSource(Context context);
}
```

## 1.2. JdbcDynamicTableSource

负责从表读数据。其中ScanRuntimeProvider是扫描表的方式，LookupRuntimeProvider是点查询的方式。

```java
public class JdbcDynamicTableSource
        implements ScanTableSource,
                LookupTableSource,
                SupportsProjectionPushDown,
                SupportsLimitPushDown,
                SupportsFilterPushDown {
  public LookupRuntimeProvider getLookupRuntimeProvider(LookupContext context);
  public ScanRuntimeProvider getScanRuntimeProvider(ScanContext runtimeProviderContext)
}
```

## 1.3. JdbcDynamicTableSink

负责向表写数据。

```java
public class JdbcDynamicTableSink implements DynamicTableSink {
  public SinkRuntimeProvider getSinkRuntimeProvider(Context context);
}
```

## 1.4. JdbcCatalogFactory

获取database、table等元数据。

# 2. flink sql解析过程

1.  抽象语法树：SQLNode



1.  逻辑计划：RelNode



1.  物理计划：RelNode



1.  Flink算子：Transformation

## 2.1. 抽象语法树：SQLNode

```java
package org.apache.flink.table.planner.delegation;

public class ParserImpl implements Parser {
  public List<Operation> parse(String statement) {
    CalciteParser parser = calciteParserSupplier.get();
    FlinkPlannerImpl planner = validatorSupplier.get();

    // parse the sql query
    // use parseSqlList here because we need to support statement end with ';' in sql client.
    SqlNodeList sqlNodeList = parser.parseSqlList(statement);
    List<SqlNode> parsed = sqlNodeList.getList();
    return Collections.singletonList(
            SqlToOperationConverter.convert(planner, catalogManager, parsed.get(0))
                    .orElseThrow(() -> new TableException("Unsupported query: " + statement)));
  }
}
```



## 2.2. **逻辑计划：RelNode**

```java
package org.apache.calcite.sql2rel;

public class SqlToRelConverter {
  public RelRoot convertQuery(SqlNode query, final boolean needsValidation
						, final boolean top) {}
}
```

## 2.3. **物理计划：RelNode**

```java
package org.apache.calcite.plan.hep;

// 基于规则的优化器
public class HepPlanner extends AbstractRelOptPlanner {
  private void applyRules(Collection<RelOptRule> rules, boolean forceConversions) {}
}
```

## 2.4. **Flink算子：Transformation**

```java
package org.apache.flink.table.planner.plan.nodes.exec.batch;

public class BatchExecSink extends CommonExecSink implements BatchExecNode<Object> {
  protected Transformation<Object> translateToPlanInternal(PlannerBase planner, ExecNodeConfig config) {}
}
```



# 3. flink sql如何关联flink-connector-jdbc包

## 3.1. flink sql task：

```java
SourceStreamTask.LegacySourceFunctionThread.run():
  StreamSource.run():
    InputFormatSourceFunction.run():
      JdbcRowDataInputFormat.openInputFormat():
        //创建jdbc连接
        Connection dbConn = connectionProvider.getOrEstablishConnection();
```

## 3.2. flink-connector-jdbc包创建JdbcRowDataInputFormat

```java
JdbcDynamicTableSource.getScanRuntimeProvider():
  new JdbcRowDataInputFormat()
```





# 4. inner join和lookup join测试

1.  通过tenv.explainSql(sql)输出抽象语法树、逻辑执行计划、物理执行计划



1.  转成DataStream API，通过env.getExecutionPlan()输出flink算子



1.  lookup join是一条条地lookup，效率很低



## 4.1. inner join:

1.  where条件会下推



1.  limit不会下推

```java
@Test
public void scan() {
    Configuration conf = new Configuration();
    StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment(conf);
    EnvironmentSettings settings = EnvironmentSettings.newInstance().inBatchMode().build();
    StreamTableEnvironment tenv = StreamTableEnvironment.create(env, settings);

    JdbcCatalog jdbcCatalog = new JdbcCatalog(Thread.currentThread().getContextClassLoader(),
            "mysql", "gacnio-community",
            "hycan-read", "g8AxryVH*r%G3f",
            "jdbc:mysql://192.168.1.192:3317");
    tenv.registerCatalog(jdbcCatalog.getName(), jdbcCatalog);
    tenv.useCatalog(jdbcCatalog.getName());

    String sql = //
            "select ua.id, ua.content_id, ua.action_type, ua.create_time, ua.create_id, uc.content \n" +
                    "from `gacnio-community`.user_action ua \n" +
                    "join `gacnio-community`.ugc_content uc \n" +
                    "on ua.content_id = uc.id \n" +
                    "where ua.type = 1 and ua.content_type = 1 limit 10";
    String plan = tenv.explainSql(sql, ExplainDetail.ESTIMATED_COST);
    System.out.println(plan);

    TableResult result = tenv.executeSql(sql);
    result.print();
}
```



## 4.2. lookup join：

1.  配置：表options：'lookup.cache'='PARTIAL'，sql：FOR SYSTEM\_TIME AS OF ua.proctime



1.  一条条lookup

```java
@Test
public void lookup() {
    Configuration conf = new Configuration();
    StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment(conf);
    EnvironmentSettings settings = EnvironmentSettings.newInstance().inBatchMode().build();
    StreamTableEnvironment tenv = StreamTableEnvironment.create(env, settings);

    String sql1 = //
            "CREATE TABLE user_action(\n" +
            "\tid int,\n" +
            "\tcontent_id int,\n" +
            "\taction_type int,\n" +
            "\ttype int,\n" +
            "\tcontent_type int,\n" +
            "\tcreate_time timestamp,\n" +
            "\tcreate_id int,\n" +
            "\tproctime AS PROCTIME()\n" +
            ")\n" +
            "WITH(\n" +
            "\t'connector'='jdbc',\n" +
            "\t'url'='jdbc:mysql://192.168.1.192:3317/gacnio-community',\n" +
            "\t'table-name'='user_action',\n" +
            "\t'username'='hycan-read',\n" +
            "\t'password'='g8AxryVH*r%G3f'\n" +
            ");";

    String sql2 = "CREATE TABLE ugc_content(\n" +
            "\tid int,\n" +
            "\tcontent string\n" +
            ")\n" +
            "WITH(\n" +
            "\t'connector'='jdbc',\n" +
            "\t'url'='jdbc:mysql://192.168.1.192:3317/gacnio-community',\n" +
            "\t'table-name'='ugc_content',\n" +
            "\t'username'='hycan-read',\n" +
            "\t'password'='g8AxryVH*r%G3f',\n" +
            "\t'lookup.cache'='PARTIAL',\n" +
            "\t'lookup.partial-cache.max-rows'='1000'\n" +
            ");";

    String sql3 = "select ua.id, ua.content_id, ua.action_type, ua.create_time, ua.create_id, uc.content\n" +
            "from user_action as ua\n" +
            "join ugc_content FOR SYSTEM_TIME AS OF ua.proctime as uc\n" +
            "on ua.content_id = uc.id\n" +
            "where ua.type = 1 and ua.content_type = 1 limit 10";

    String[] sqls = new String[] {sql1, sql2};
    for (String sql : sqls) {
        TableResult result = tenv.executeSql(sql);
        result.print();
    }

    String plan = tenv.explainSql(sql3, ExplainDetail.ESTIMATED_COST);
    System.out.println(plan);

    TableResult result = tenv.executeSql(sql3);
    result.print();

}
```

