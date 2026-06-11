# 11-数据库压测(JDBC) 🗄️

> **难度：★★★☆☆** | JDBC Request 采样器 + JDBC Connection Configuration 配置元素

---

## 1. 场景概述

### 为什么需要数据库压测？

- **瓶颈定位**：高并发下数据库往往是性能瓶颈，需要独立评估数据库吞吐能力
- **SQL 优化验证**：对比索引优化前后的 QPS 差异
- **连接池验证**：验证数据库连接池大小配置是否合理
- **混合压测**：在 HTTP 压测的同时，验证数据库读写能力

### JMeter 数据库测试架构

```
JMeter (JDBC Sampler)
    │
    ├── JDBC Connection Configuration (连接池)
    │       ├── Database URL: jdbc:mysql://host:port/db
    │       ├── JDBC Driver Class
    │       ├── Username / Password
    │       └── Max Number of Connections
    │
    └── JDBC Request (SQL 执行)
            ├── Query Type: Select / Update / Callable / Prepared
            ├── SQL Query (可含变量 ${xxx})
            └── Result Variable Name (存储结果供后续使用)
```

---

## 2. 环境准备

### 2.1 下载 JDBC 驱动

将对应数据库的 JDBC 驱动 jar 放到 `JMETER_HOME/lib/` 目录下：

| 数据库 | 驱动类 | Jar 包 |
|--------|--------|--------|
| MySQL | `com.mysql.cj.jdbc.Driver` | `mysql-connector-java-8.x.x.jar` |
| PostgreSQL | `org.postgresql.Driver` | `postgresql-42.x.x.jar` |
| Oracle | `oracle.jdbc.OracleDriver` | `ojdbc8.jar` |
| SQL Server | `com.microsoft.sqlserver.jdbc.SQLServerDriver` | `mssql-jdbc-12.x.x.jar` |

> ⚠️ **重要**：放置 jar 后需要**重启 JMeter** 才能生效。

### 2.2 测试数据准备

```sql
-- 创建测试表
CREATE TABLE perf_test (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(200),
    age INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 插入基准数据（可选，用于查询测试）
INSERT INTO perf_test (name, email, age)
SELECT
    CONCAT('user_', seq),
    CONCAT('user_', seq, '@test.com'),
    18 + FLOOR(RAND() * 42)
FROM (
    SELECT @row := @row + 1 AS seq
    FROM information_schema.columns a,
         information_schema.columns b,
         (SELECT @row := 0) r
    LIMIT 100000
) t;
```

---

## 3. 测试计划构建

### 测试计划树结构

```
Test Plan
├── JDBC Connection Configuration     # 数据库连接配置
├── Thread Group (数据库压测)
│   ├── CSV Data Set Config          # 测试数据参数化
│   ├── JDBC Request - 查询          # SELECT 查询
│   ├── JDBC Request - 插入          # INSERT 操作
│   ├── JDBC Request - 更新          # UPDATE 操作
│   ├── JSR223 PostProcessor         # 处理查询结果
│   └── Response Assertion           # 断言
└── View Results Tree / Aggregate Report
```

### 3.1 JDBC Connection Configuration 配置

| 配置项 | 值 | 说明 |
|--------|-----|------|
| **Variable Name** | `mysqlPool` | 连接池变量名，JDBC Request 中引用 |
| **Database URL** | `jdbc:mysql://192.168.1.100:3306/testdb?useSSL=false&allowPublicKeyRetrieval=true` | 数据库连接串 |
| **JDBC Driver Class** | `com.mysql.cj.jdbc.Driver` | 驱动类 |
| **Username** | `root` | 数据库用户名 |
| **Password** | `password` | 数据库密码 |
| **Max Number of Connections** | `20` | 连接池最大连接数，建议 = 线程数 |
| **Pool timeout** | `1000` | 获取连接超时(ms) |
| **Connection Validation** | `SELECT 1` | 连接有效性验证 SQL |
| **Auto Commit** | `true` | 自动提交 |
| **Transaction Isolation** | `DEFAULT` | 事务隔离级别 |

> **最佳实践**：
> - 将 `Max Number of Connections` 设置为 `线程数 + 5` 的缓冲
> - `Pool timeout` 不要设置过大，避免线程长时间等待连接
> - 始终配置 `Connection Validation` 防止连接失效

### 3.2 JDBC Request - 查询示例

| 配置项 | 值 |
|--------|-----|
| **Variable Name** | `mysqlPool`（与 Connection Config 一致） |
| **Query Type** | `Select Statement` |
| **SQL Query** | `SELECT id, name, email, age FROM perf_test WHERE id = ?` |
| **Parameter values** | `${__Random(1,100000)}` |
| **Parameter types** | `INTEGER` |
| **Result variable name** | `queryResult` |
| **Handle ResultSet** | `Store as String` |

**Query Type 详解**：

| Query Type | 适用场景 | SQL 写法 |
|------------|----------|----------|
| **Select Statement** | 单条 SELECT | `SELECT * FROM t WHERE id = ?` |
| **Update Statement** | INSERT/UPDATE/DELETE | `INSERT INTO t VALUES (?,?)` |
| **Callable Statement** | 存储过程 | `{call sp_name(?,?)}` |
| **Prepared Select** | 预编译 SELECT | 同上 Select |
| **Prepared Update** | 预编译增删改 | 同上 Update |
| **Commit / Rollback** | 事务控制 | 无需 SQL |
| **AutoCommit(true/false)** | 切换自动提交 | 无需 SQL |

### 3.3 JDBC Request - 批量插入示例

```sql
INSERT INTO perf_test (name, email, age)
VALUES (?, ?, ?)
```

| Parameter values | `${randomName}`, `${randomEmail}`, `${__Random(18,60)}` |
| Parameter types | `VARCHAR`, `VARCHAR`, `INTEGER` |

### 3.4 JDBC Request - 调用存储过程

```sql
CALL get_user_by_age(?, ?)
```

| Parameter values | `25`, `OUT` |
| Parameter types | `INTEGER`, `INTEGER` |

---

## 4. 查询结果处理

### 4.1 使用 JSR223 PostProcessor 处理结果

```groovy
import java.sql.ResultSet;

// 从 JDBC Request 的 Result variable name 获取结果
Object resultObj = vars.getObject("queryResult");

if (resultObj instanceof java.util.Map) {
    def resultMap = (java.util.Map) resultObj;
    
    // 遍历结果列
    resultMap.each { columnName, value ->
        log.info("Column: ${columnName} = ${value}");
    }
    
    // 提取值到 JMeter 变量（供后续 sampler 使用）
    vars.put("userId", String.valueOf(resultMap.get("id")));
    vars.put("userName", String.valueOf(resultMap.get("name")));
    vars.put("userAge", String.valueOf(resultMap.get("age")));
}
```

### 4.2 ResultSet Handler 配置

在 JDBC Request 的 `Handle ResultSet` 中：

| 选项 | 说明 | 适用场景 |
|------|------|----------|
| **Store as String** | 将第一行第一列转为字符串 | 单值查询（如 COUNT） |
| **Store as Object** | 将整个 ResultSet 存为对象 | JSR223 后置处理 |
| **Count Records** | 只返回行数 | 验证数据量 |

---

## 5. 常见数据库压测场景

### 5.1 读多写少场景（典型 Web 应用）

```
Thread Group (100 并发)
├── Throughput Controller (80%)           # 80% 读
│   └── JDBC Request - 单条查询
├── Throughput Controller (15%)           # 15% 写
│   └── JDBC Request - 插入
└── Throughput Controller (5%)            # 5% 更新
    └── JDBC Request - 更新
```

### 5.2 连接池压力测试

```groovy
// JSR223 Sampler - 模拟连接泄漏检测
def pool = vars.get("mysqlPool");
// 记录当前活跃连接数
log.info("Active connections check at iteration: " + vars.get("__jm__Thread Group__idx"));
```

通过逐渐增大线程数，观察数据库连接数和响应时间的变化，找到连接池最优大小。

### 5.3 混合压测（HTTP + DB 联动）

```
setUp Thread Group
├── JDBC Connection Configuration
└── JDBC Request - 数据初始化

Thread Group (HTTP 业务)
├── HTTP Request (业务接口)
├── JSON Extractor (提取返回值中的 ID)
├── JDBC Request - 数据校验
│   └── SELECT * FROM orders WHERE id = ${orderId}
└── Response Assertion (DB 数据一致性校验)
```

---

## 6. 关键指标与断言

### 6.1 Duration Assertion

```xml
<DurationAssertion>
    <duration>5000</duration>  <!-- 数据库操作超 5 秒即失败 -->
</DurationAssertion>
```

### 6.2 Response Assertion - 验证返回数据

- 对 `Select Statement` 结果：验证 `queryResult` 变量不为空
- 对 `Update Statement`：验证 `更新行数 > 0`

### 6.3 关键监控指标

| 指标 | JMeter 获取方式 | 数据库侧监控 |
|------|----------------|-------------|
| **QPS** | Aggregate Report → Throughput | `SHOW GLOBAL STATUS LIKE 'Questions'` |
| **响应时间** | Aggregate Report → Average/p99 | `performance_schema` 慢查询 |
| **连接数** | JDBC Connection Config → Max Connections | `SHOW PROCESSLIST` |
| **锁等待** | 无直接获取 | `SHOW ENGINE INNODB STATUS` |
| **CPU/内存** | PerfMon 插件 | 系统监控 |

---

## 7. 常见问题排查

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| **No suitable driver found** | JDBC 驱动 jar 未放入 lib | 下载对应驱动 jar，放入 `$JMETER_HOME/lib/`，重启 JMeter |
| **Cannot create PoolableConnectionFactory** | 连接串/用户名/密码错误 | 检查 Database URL、防火墙、数据库权限 |
| **Pool exhausted** | 连接池耗尽 | 增大 Max Number of Connections 或减小线程数 |
| **Communications link failure** | 数据库连接超时 | 增大 wait_timeout，配置 autoReconnect |
| **ResultSet is closed** | 结果集已关闭 | 使用 `Store as Object` 模式，在 PostProcessor 中处理 |
| **Parameter index out of range** | SQL 参数数量不匹配 | 检查 `?` 占位符数量与 Parameter values 是否一致 |

---

## 8. 性能优化建议

1. **连接池大小**：`线程数 ≤ 连接池大小 ≤ 线程数 * 1.5`
2. **使用 Prepared Statement**：利用数据库预编译缓存
3. **合理设置 Fetch Size**：大批量查询时设置合理的 fetch size
4. **批量操作**：使用 batch insert 替代逐条 insert
5. **读写分离**：在 JDBC Connection Configuration 中配置只读和读写两个连接池
6. **关闭 AutoCommit**：批量操作时关闭 AutoCommit，手动 commit 提升性能

---

## 9. 完整示例：MySQL 压测脚本

```xml
<!-- JDBC Connection Configuration -->
<JDBCDataSource>
  <stringProp name="dataSource">mysqlPool</stringProp>
  <stringProp name="driver">com.mysql.cj.jdbc.Driver</stringProp>
  <stringProp name="dbUrl">jdbc:mysql://192.168.1.100:3306/testdb?useSSL=false</stringProp>
  <stringProp name="username">root</stringProp>
  <stringProp name="password">password</stringProp>
  <stringProp name="poolMax">20</stringProp>
  <stringProp name="timeout">10000</stringProp>
  <stringProp name="connectionAge">5000</stringProp>
  <stringProp name="checkQuery">SELECT 1</stringProp>
</JDBCDataSource>

<!-- JDBC Request - 查询 -->
<JDBCSampler>
  <stringProp name="dataSource">mysqlPool</stringProp>
  <stringProp name="queryType">Prepared Select Statement</stringProp>
  <stringProp name="query">SELECT * FROM perf_test WHERE id = ?</stringProp>
  <stringProp name="queryArguments">${__Random(1,100000)}</stringProp>
  <stringProp name="queryArgumentsTypes">INTEGER</stringProp>
  <stringProp name="resultVariable">queryResult</stringProp>
</JDBCSampler>
```
