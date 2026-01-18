# PostgreSQL 19 扩展统计信息导入/导出功能

## 引言

PostgreSQL 的扩展统计信息功能自版本 10 引入以来，使优化器能够理解简单单列统计信息无法捕获的列间相关性。这包括 n-distinct 系数（存在多少个不同的组合）、函数依赖（当一个列决定另一个列时）以及多列组合的最常见值（MCV）列表。

虽然 PostgreSQL 17 引入了用于导入和导出关系和属性统计信息的函数（`pg_restore_relation_stats`、`pg_restore_attribute_stats`），但扩展统计信息被排除在这一初始实现之外。最近在 pgsql-hackers 邮件列表上，由 Corey Huinker 发起的一个讨论线程解决了这一空白，提供了一个全面的补丁系列，添加了 `pg_restore_extended_stats()`、`pg_clear_extended_stats()` 及相关基础设施。

这项工作意义重大，原因如下：
- 实现跨 pg_dump/pg_restore 和 pg_upgrade 的完整统计信息保留
- 允许使用假设统计信息进行查询计划器实验
- 支持仅包含 schema 和统计信息的转储，用于在没有实际数据的情况下测试查询计划

## 技术分析

### 原始格式的问题

`pg_ndistinct` 和 `pg_dependencies` 类型的原始输出格式使用了一种 JSON 结构，其中键本身包含结构化数据：

```json
{"1, 2": 2323, "1, 3": 3232, "2, 3": 1500}
```

虽然这在技术上是有效的 JSON，但这种格式存在几个问题：
1. 包含逗号分隔属性号的键需要额外解析
2. 难以以编程方式操作
3. 不存在可用的输入函数——这些类型实际上只能输出

### 新的 JSON 格式

补丁系列引入了一种更清晰、更结构化的 JSON 格式。对于 `pg_ndistinct`：

```json
[
  {"attributes": [2, 3], "ndistinct": 4},
  {"attributes": [2, -1], "ndistinct": 4},
  {"attributes": [2, 3, -1], "ndistinct": 4}
]
```

对于 `pg_dependencies`：

```json
[
  {"attributes": [2], "dependency": 3, "degree": 1.000000},
  {"attributes": [2, 3], "dependency": -1, "degree": 0.850000}
]
```

主要改进：
- **规范的 JSON 数组**，每个元素都有命名的键
- **清晰分离**属性、值和元数据
- **机器可读**，无需自定义解析逻辑
- **负数属性号**表示统计对象中的表达式（例如，`-1` 是第一个表达式）

### 输入函数实现

新的输入函数使用 PostgreSQL 的 JSON 解析器基础设施，配合自定义语义动作处理器。以下是 `pg_ndistinct` 解析状态机的简化视图：

```c
typedef enum
{
    NDIST_EXPECT_START = 0,
    NDIST_EXPECT_ITEM,
    NDIST_EXPECT_KEY,
    NDIST_EXPECT_ATTNUM_LIST,
    NDIST_EXPECT_ATTNUM,
    NDIST_EXPECT_NDISTINCT,
    NDIST_EXPECT_COMPLETE
} ndistinctSemanticState;
```

解析器验证：
- 正确的 JSON 结构（对象数组）
- 必需的键（ndistinct 统计信息需要 `attributes` 和 `ndistinct`）
- 属性号在有效范围内（正数表示列，负数表示表达式，但不超过 `STATS_MAX_DIMENSIONS`）
- 单个项目内没有重复属性

### 扩展统计信息函数

补丁引入了三个主要 SQL 函数：

**pg_restore_extended_stats()** — 从先前导出的值导入扩展统计信息：

```sql
SELECT pg_restore_extended_stats(
    'public',                    -- 关系 schema
    'my_table',                  -- 关系名称
    'public',                    -- 统计信息 schema
    'my_stats',                  -- 统计信息名称
    false,                       -- inherited（是否继承）
    '{"version": ..., "ndistinct": [...], "dependencies": [...], "mcv": [...], "exprs": [...]}'::text
);
```

**pg_clear_extended_stats()** — 从 `pg_statistic_ext_data` 中删除扩展统计信息数据：

```sql
SELECT pg_clear_extended_stats(
    'public',        -- 统计信息 schema
    'my_stats',      -- 统计信息名称
    false            -- inherited（是否继承）
);
```

这些函数遵循为关系/属性统计信息建立的相同模式：
- 返回布尔值表示成功与否
- 遇到问题时发出 `WARNING`（而非 `ERROR`），以避免破坏 pg_restore 脚本
- 需要目标关系的 `MAINTAIN` 权限

### 验证与安全

实现包括仔细的验证：

1. **属性边界检查**：正数 attnum 必须存在于 `stxkeys` 中，负数 attnum 不得超过表达式数量
2. **组合完整性**：对于 `pg_ndistinct`，基于最长属性列表，必须存在所有 N 选 K 组合
3. **软错误处理**：使用 PostgreSQL 的 `ErrorSaveContext` 进行安全的错误报告而不会崩溃

属性号验证示例：

```c
if (attnum == 0 || attnum < (0 - STATS_MAX_DIMENSIONS))
{
    errsave(parse->escontext,
            errcode(ERRCODE_INVALID_TEXT_REPRESENTATION),
            errmsg("malformed pg_ndistinct: \"%s\"", parse->str),
            errdetail("Invalid \"%s\" element: %d.",
                      PG_NDISTINCT_KEY_ATTRIBUTES, attnum));
    return JSON_SEM_ACTION_FAILED;
}
```

## 社区洞察

### 关键讨论点

**格式更改时机**：Tomas Vondra 最初建议采用更结构化的 JSON 格式。社区认识到这是在可用输入函数锁定向后兼容性要求之前更改格式的最后机会。

**验证范围**：关于应执行多少验证存在重大讨论：
- 早期补丁对统计一致性进行了广泛检查（例如，MCV 频率总和为 1.0）
- 审查者提出反对意见，倾向于最小化验证以避免破坏合法但不寻常的导入
- 最终共识：验证结构和属性引用，但不验证统计值

**pg_dependencies 特殊情况**：与存储所有组合的 `pg_ndistinct` 不同，`pg_dependencies` 可能会省略统计上不显著的组合。这意味着输入函数无法对依赖项强制执行完整的组合覆盖。

### 审查反馈整合

Michael Paquier 提供了广泛的审查并贡献了重大改进：
- 重构补丁系列以获得更清晰的提交
- 将格式更改与输入函数添加分开
- 添加全面的回归测试，实现超过 90% 的代码覆盖率
- 修复旧版 GCC 上的编译器警告

Tom Lane 发现了风格问题：
- 错误详细消息转换为完整句子
- 用直接状态检查替换 `SOFT_ERROR_OCCURRED()` 宏以避免警告

## 当前状态

截至 2026 年 1 月，补丁系列已取得重大进展：

**已提交：**
- `pg_ndistinct` 的输出格式更改（新的 JSON 数组格式）
- `pg_dependencies` 的输出格式更改（新的 JSON 数组格式）
- 两种类型的输入函数及全面验证
- `pg_clear_extended_stats()` 函数

**审查中 (v27)：**
- `pg_restore_extended_stats()` 函数
- pg_dump 集成用于扩展统计信息导出/导入

pg_dump 集成支持向后兼容到 PostgreSQL 10，通过特定版本的 SQL 生成来处理格式差异。

## 技术细节

### 内部存储未更改

重要的是，内部二进制存储格式保持不变。新的输入/输出函数只影响文本表示。这意味着：
- 不需要目录更改
- 现有数据保持有效
- 二进制 COPY 操作不受影响

### 表达式统计支持

扩展统计信息可以包含表达式（例如，`CREATE STATISTICS s ON (a + b), c FROM t`）。实现通过负数属性号处理这些：
- `-1` = 第一个表达式
- `-2` = 第二个表达式
- 以此类推

恢复格式中的 `exprs` 元素包含类似于 `pg_statistic` 条目的每个表达式的统计信息，实现完整的往返保留。

### MCV 列表处理

扩展统计信息的 MCV（最常见值）列表特别复杂，包含：
- 跨多列的值组合
- 频率和基础频率数组
- 每个值的空值位图

实现重用了属性统计信息导入的基础设施，并针对多列值数组进行了扩展。

## 结论

这个补丁系列代表了 PostgreSQL 统计信息基础设施的重大增强。通过启用扩展统计信息的导入/导出，它：

1. **完善了统计信息功能**，延续了 PostgreSQL 17 中为关系和属性统计信息开始的工作
2. **实现了真实的测试**，在清理后的 schema 上使用类似生产环境的统计信息
3. **提高了升级可靠性**，通过 pg_upgrade 保留优化器信息

对于 DBA 和开发人员：
- 使用 `CREATE STATISTICS` 创建的扩展统计信息现在可以在 pg_dump/pg_restore 后保留
- 使用包含完整统计信息的 `--no-data` 转储，查询计划测试变得更加实用
- 新的 JSON 格式便于人类阅读，用于调试和假设场景测试

目标发布版本是 PostgreSQL 19，剩余的恢复函数和 pg_dump 集成预计很快会合并。

## 参考资料

- [pgsql-hackers 原始讨论线程](https://www.postgresql.org/message-id/CADkLM%3Ddpz3KFnqP-dgJ-zvRvtjsa8UZv8wDAQdqho%3DqN3kX0Zg%40mail.gmail.com)
- [PostgreSQL 扩展统计信息文档](https://www.postgresql.org/docs/current/planner-stats.html#PLANNER-STATS-EXTENDED)
- [Commitfest 条目](https://commitfest.postgresql.org/patch/5517/)
