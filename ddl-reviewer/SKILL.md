---
name: ddl-reviewer
description: 审查/甄别数据库建表 DDL(CREATE TABLE)语句,依据关系型数据库(MySQL/PostgreSQL)最佳实践:三大范式、必须有主键、命名规范(snake_case)、必备基础字段、字段类型、索引设计、字符集与引擎、注释规范。Use whenever the user provides a CREATE TABLE statement or SQL file, or asks to review / 检查 / 甄别 / 审查 建表语句、表结构、DDL、数据库 schema 设计,or mentions 三大范式、表设计规范 — even for a single table or a fragment of DDL.
---

# DDL 建表语句审查

对用户提供的 CREATE TABLE 语句(或整个 SQL 文件)做规范性甄别,输出结构化审查报告,用户要求时给出修正后的完整 DDL。

核心原则:**无主键、违背三大范式、金额用浮点、保留字命名 —— 一票否决,必须修复**。

## 工作流程

1. **解析 DDL**:逐表提取表名、字段(名/类型/NULL 约束/DEFAULT/COMMENT)、主键、唯一约束、索引、引擎与字符集。注意方言差异(见下)。
2. **逐类审查**:打开 `references/rules.md`,按 8 个类别(命名、必备字段、数据类型、索引设计、表结构原则与三范式、字符集与引擎、注释、自检清单)逐条对照。每个问题必须记录证据:表名.字段名 + DDL 原文片段。
3. **分级判定**(见严重级别)。
4. **输出报告**(格式见下)。
5. 仅当用户要求时,给出**修正后的完整 DDL**(整条可执行的 CREATE TABLE,不是 diff);修正版必须消除全部 🔴 与 🟡 项,并保持原有业务字段不丢失。

## 严重级别

- 🔴 **阻断(必须修复)**:无主键;业务字段作主键;违背 1NF/2NF/3NF 且无注释说明;金额/精确数值用 FLOAT 或 DOUBLE;表名或字段名使用保留字(order、group、key、select、desc 等);非 snake_case 命名;MySQL 显式使用 MyISAM。
- 🟡 **强烈建议**:缺 `id` / `created_at` / `updated_at` 三件套;字段可空且无默认值;使用 ENUM 类型;表或字段无 COMMENT;盲目 VARCHAR(255);索引超过 5~6 个或存在冗余索引;TEXT/BLOB 混在主表;MySQL 非 utf8mb4。
- 🟢 **可选优化**:覆盖索引机会;字段超过 30~40 列建议垂直拆分;单表预估行数超 500 万~1000 万提示分库分表/归档规划;存储过程/触发器提示移到应用层。

## 报告格式

```markdown
## 审查报告:user_order(共 1 张表)

**结论:❌ 需修改后提交** —— 🔴 2 项 / 🟡 3 项 / 🟢 1 项

### 🔴 阻断问题
1. **无主键** —— 必须添加 `id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY`
2. **违背 3NF**:`user_name` 冗余存储于 `user_id` 旁(传递依赖,用户名应只存 user 表)
   证据:`user_id BIGINT NOT NULL, user_name VARCHAR(50)`(若为刻意冗余,须在 COMMENT 注明来源与同步策略)

### 🟡 建议项
1. **缺三件套**:无 `created_at` / `updated_at` …
2. **ENUM 类型**:`status ENUM(...)` → 改 TINYINT + COMMENT 列出枚举值
3. **冗余索引**:已有 `KEY idx_a_b (a, b)` 又单独建 `KEY idx_a (a)`

### 🟢 可选优化
1. `SELECT a, b FROM t WHERE a = ?` 可用覆盖索引 (a, b) 避免回表
```

多张表时逐表出报告,最后给汇总表(表名 × 🔴/🟡/🟢 数量)。全部通过的表只输出一行 ✅。

## 三范式判定要点(如何从 DDL 识别)

- **1NF 违背**:一个字段存多个值 —— 逗号分隔字符串(`tags VARCHAR(200)` 存 "a,b,c")、tag1/tag2/tag3 平铺、JSON 存重复分组。
- **2NF 违背**:联合主键下,非主键字段只依赖主键的一部分。典型:PK 为 `(order_id, product_id)`,但 `product_name` 只依赖 `product_id`。
- **3NF 违背**:非主键字段依赖另一个非主键字段(传递依赖)。典型:`dept_id` 旁边存 `dept_name`;明细表里存可由明细汇总出的 `order_total`。
- **例外**:高并发读场景刻意反范式冗余是允许的,但该字段 COMMENT 必须注明「冗余字段,来源 xxx,同步策略 xxx」。没有注释说明的一律按 🔴 报告。

## 方言注意(MySQL vs PostgreSQL)

| 项 | MySQL | PostgreSQL |
|---|---|---|
| 表/字段注释 | 行内 `COMMENT '...'` | 独立的 `COMMENT ON TABLE/COLUMN` 语句 |
| 引擎 | 要求 InnoDB,禁 MyISAM | 无 ENGINE 概念,勿报此项 |
| 字符集 | 要求 utf8mb4 | 库级 UTF8 编码,勿对单表报此项 |
| 类型映射 | TINYINT / DATETIME / INT UNSIGNED 存 IP | SMALLINT / TIMESTAMPTZ(推荐)/ INET |
| 主键自增 | AUTO_INCREMENT | GENERATED ALWAYS AS IDENTITY / BIGSERIAL |

对 PostgreSQL 的 DDL,按右列标准审查,不要套用 MySQL 规则误报。

## 边界

- 只依据 DDL 本身能看出的问题做判定;业务语义导致的范式问题(如字段名暗示的依赖关系)用「疑似」标注并说明推断依据,不要断言。
- 一次只审查用户给的 DDL;用户给的是整个 schema 文件时,跨表冗余(同一份字典数据在多表重复)也要报。
- 不要主动改写用户语句,除非用户要求修正版。
