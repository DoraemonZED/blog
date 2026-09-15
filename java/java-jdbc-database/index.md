## 1. JDBC 的职责边界

JDBC 统一了 Java 访问关系数据库的接口。典型流程是从 `DataSource` 获取连接、创建参数化语句、执行、读取结果并释放资源。应用应依赖 `DataSource`，而不是在每次请求中直接使用 `DriverManager` 创建物理连接。

```java
String sql = "select id, name from users where email = ?";
try (Connection connection = dataSource.getConnection();
     PreparedStatement statement = connection.prepareStatement(sql)) {
    statement.setString(1, email);
    try (ResultSet rs = statement.executeQuery()) {
        if (rs.next()) return new User(rs.getLong("id"), rs.getString("name"));
        return null;
    }
}
```

`PreparedStatement` 把参数和 SQL 结构分开，是防止 SQL 注入的基础。表名、排序字段等不能作为普通参数绑定，应使用白名单映射。

## 2. 连接池

数据库连接昂贵，因此使用连接池复用。池大小不是越大越好：连接数超过数据库能并行处理的能力，只会增加排队、锁竞争和上下文切换。

必须设置获取连接超时、连接有效性检测、最长生命周期和泄漏诊断。应用线程池、数据库连接池与数据库最大连接数应联合规划。请求获取不到连接时应快速失败或降级，不能无限等待。

## 3. 事务

关闭自动提交后，一组操作可以在提交或回滚时形成原子边界。事务范围应围绕业务不变量，避免在事务中调用慢速远程服务。所有异常路径都要回滚，并恢复连接状态后再归还连接池。

隔离级别解决脏读、不可重复读和幻读等并发现象，但更强的隔离可能带来更多等待或冲突。正确选择依赖业务语义，不能只把隔离级别调到最高。

## 4. 批处理与大数据量

批量写入使用 `addBatch/executeBatch`，控制单批尺寸并在失败时记录可以重放的业务标识。大结果集应分页、流式读取或使用游标，同时防止长事务持有快照和连接。

分页很深时，`offset` 需要跳过大量记录；稳定排序下可使用基于游标或最后键值的 keyset pagination。

## 5. 索引与执行计划

索引加速读取但增加写入成本和存储。复合索引设计要结合过滤、连接、排序和选择性。出现慢查询时先查看真实执行计划、扫描行数、返回行数和锁等待，不要凭 SQL 外观猜测。

避免典型问题：在索引列上做不必要的函数计算、隐式类型转换、返回无用大字段、循环执行 N+1 查询，以及没有限制条件的更新和删除。

## 6. 锁、死锁与重试

数据库通过锁或多版本并发控制维持隔离。死锁发生时数据库通常终止其中一个事务，应用应只对明确的瞬态错误进行有限重试。事务访问资源保持一致顺序、缩短事务时间、建立合适索引，都能降低死锁概率。

## 7. 复制、分片与迁移

读写分离存在复制延迟，刚写入的数据不一定能从副本立即读到。分库分表会引入跨分片查询、事务、扩容和全局 ID 问题，应在单库优化、归档、缓存等手段不足后再采用。

数据库变更使用迁移工具纳入版本控制。高风险变更采用向后兼容的展开—迁移—收缩流程：先加新结构，双写或回填，切换读取，最后删除旧结构。

## 上线检查表

- 所有输入是否参数化？
- 连接、语句和结果集是否关闭？
- 连接池是否有容量与超时？
- 事务是否包含远程调用？
- 查询是否经过真实数据量的执行计划验证？
- 是否考虑复制延迟和重试幂等？
- Schema 变更能否回滚或向前修复？

## 参考资料

- [JDBC API](https://docs.oracle.com/en/java/javase/21/docs/api/java.sql/java/sql/package-summary.html)
- [MySQL Reference Manual](https://dev.mysql.com/doc/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)

