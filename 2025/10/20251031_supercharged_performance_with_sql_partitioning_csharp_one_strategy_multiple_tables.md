# [通过 SQL 分区和 C# 实现超高性能：一种策略，多张表](https://medium.com/@dotnetfullstackdev/supercharged-performance-with-sql-partitioning-c-one-strategy-multiple-tables-a84373916fa0)

## 概述

假设你正在开发一个遥测应用程序，该应用程序每天为不同的传感器存储数百万条记录。你希望按一个公共列（ ValueDate ）对所有与传感器相关的表（例如 SensorData 、 SensorLogs 和 SensorErrors ）进行分区。

![数据库表结构](https://miro.medium.com/v2/resize:fit:1400/format:webp/0*DJ8n8lXHcCxkSc5q)

## SQL 分区策略

### 第 1 步：创建分区函数

```sql
CREATE PARTITION FUNCTION pf_ByDate (DATETIME)
AS RANGE RIGHT FOR VALUES (
    '2025-01-01',
    '2025-04-01',
    '2025-07-01',
    '2025-10-01'
);
```

这将创建 5 个分区：

- 分区 1：小于 2025-01-01
- 分区 2：2025-01-01 到 2025-03-31
- 分区 3：2025-04-01 到 2025-06-30
- 分区 4：2025-07-01 到 2025-09-30
- 分区 5：大于或等于 2025-10-01

### 第 2 步：创建分区方案

```sql
CREATE PARTITION SCHEME ps_ByDate
AS PARTITION pf_ByDate
ALL TO ([PRIMARY]);
```

### 第 3 步：使用分区方案创建表

现在让我们创建 3 个表（ SensorData 、 SensorLogs 、 SensorErrors ），它们都在同一列上使用相同的分区方案。

```sql
CREATE TABLE SensorData (
    Id INT IDENTITY(1,1),
    SensorId INT,
    Value FLOAT,
    ValueDate DATETIME NOT NULL,
    CONSTRAINT PK_SensorData PRIMARY KEY (ValueDate, Id)
) ON ps_ByDate(ValueDate);
```

```sql
CREATE TABLE SensorLogs (
    LogId INT IDENTITY(1,1),
    SensorId INT,
    Message NVARCHAR(200),
    ValueDate DATETIME NOT NULL,
    CONSTRAINT PK_SensorLogs PRIMARY KEY (ValueDate, LogId)
) ON ps_ByDate(ValueDate);
```

```sql
CREATE TABLE SensorErrors (
    ErrorId INT IDENTITY(1,1),
    SensorId INT,
    ErrorCode INT,
    ValueDate DATETIME NOT NULL,
    CONSTRAINT PK_SensorErrors PRIMARY KEY (ValueDate, ErrorId)
) ON ps_ByDate(ValueDate);
```

## 用于插入、选择、删除的 C# 代码

### 插入示例

```csharp
using (SqlConnection conn = new SqlConnection(connStr))
{
    conn.Open();
    string insertQuery = @"INSERT INTO SensorData (SensorId, Value, ValueDate) VALUES (@sid, @val, @date)";
    
    using SqlCommand cmd = new SqlCommand(insertQuery, conn);
    cmd.Parameters.AddWithValue("@sid", 101);
    cmd.Parameters.AddWithValue("@val", 45.6);
    cmd.Parameters.AddWithValue("@date", DateTime.Today);
    cmd.ExecuteNonQuery();
}
```

如果需要，我们可以使用 .NET 中的 SQLBulkCopy 功能进行批量插入。

### 选择示例（单个分区）

```csharp
using (SqlConnection conn = new SqlConnection(connStr))
{
    conn.Open();
    string selectQuery = @"SELECT * FROM SensorData WHERE ValueDate = @date";

    using SqlCommand cmd = new SqlCommand(selectQuery, conn);
    cmd.Parameters.AddWithValue("@date", DateTime.Today);
    
    using var reader = cmd.ExecuteReader();
    while (reader.Read())
    {
        Console.WriteLine($"SensorId: {reader["SensorId"]}, Value: {reader["Value"]}");
    }
}
```

### 删除示例（分区感知删除）

```csharp
using (SqlConnection conn = new SqlConnection(connStr))
{
    conn.Open();
    string deleteQuery = @"DELETE FROM SensorData WHERE ValueDate = @date";

    using SqlCommand cmd = new SqlCommand(deleteQuery, conn);
    cmd.Parameters.AddWithValue("@date", new DateTime(2025, 01, 01));
    int rows = cmd.ExecuteNonQuery();
    Console.WriteLine($"{rows} rows deleted");
}
```

由于删除操作是基于分区键（ ValueDate ）的，此操作极其快速（如果切换分区，几乎是瞬时完成）。

## 使用分区切换表进行切换/删除

我们使用具有相同结构的暂存表来执行超快速删除。

### 创建用于切换的空暂存表

```sql
CREATE TABLE SensorData_Archive (
    Id INT NOT NULL,
    SensorId INT,
    Value FLOAT,
    ValueDate DATETIME NOT NULL,
    CONSTRAINT PK_SensorData_Archive PRIMARY KEY (ValueDate, Id)
) ON ps_ByDate(ValueDate);
```

必须与主表在架构和约束上完全相同。

### 切换分区到暂存表

```sql
ALTER TABLE SensorData
SWITCH PARTITION 2 TO SensorData_Archive PARTITION 2;
```

这种切换操作会物理移动分区——即使对数百万条记录也能瞬间完成。

现在你可以截断或归档 SensorData_Archive 。

### 如何获取分区号？

使用这个来找出一个日期属于哪个分区：

```sql
SELECT $PARTITION.pf_ByDate('2025-02-15') AS PartitionNumber
```

### C# 中的超快速删除

要从应用程序中执行快速删除：

```csharp
using var conn = new SqlConnection(connStr);
conn.Open();
var switchQuery = "ALTER TABLE SensorData SWITCH PARTITION 2 TO SensorData_Archive PARTITION 2";
using var cmd = new SqlCommand(switchQuery, conn);
cmd.ExecuteNonQuery();
```

切换后，我们可以完全截断归档/暂存表。

### 为什么归档/暂存表中的标识列存在问题

当你定义一个 IDENTITY 列时，SQL Server 会在插入过程中自动生成值。

但在使用分区切换时，SQL Server 要求：

- 架构必须完全相同，包括约束、数据类型、可空性
- 但不包括计算列或 IDENTITY 属性

因此，即使列和主键匹配， IDENTITY 的存在也会使架构不兼容，你将收到此错误：

```bash
Msg 4947: ALTER TABLE SWITCH statement failed. Identity mismatch.
```

## 监控分区使用情况

```sql
SELECT
    ps.name AS PartitionScheme,
    pf.name AS PartitionFunction,
    p.partition_number,
    COUNT(*) AS RowCount
FROM sys.partitions p
JOIN sys.indexes i ON p.object_id = i.object_id AND p.index_id = i.index_id
JOIN sys.partition_schemes ps ON i.data_space_id = ps.data_space_id
JOIN sys.partition_functions pf ON ps.function_id = pf.function_id
WHERE i.object_id = OBJECT_ID('SensorData') AND i.index_id <= 1
GROUP BY ps.name, pf.name, p.partition_number;
```

我们可以对剩余两个表应用类似的过程，通过创建归档表并在分区切换后截断归档表。

今天就到这里。希望这些内容对你有帮助。
