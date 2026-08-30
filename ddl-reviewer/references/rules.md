# DDL 审查规则细则

审查 CREATE TABLE 时逐类对照。每条规则给出:检查点、判定标准、修法示例。MySQL 语法为例,PostgreSQL 差异见 SKILL.md 方言表。

## 1. 命名规范

| 检查点 | 标准 | 修法 |
|---|---|---|
| 大小写 | 库名/表名/字段名/索引名全部小写 + 下划线 snake_case | `UserOrder` → `user_order`;`createTime` → `create_time` |
| 见名知意 | 禁止拼音、无意义缩写(`a1`、`flag2`、`user_biao`) | 用完整业务词:`is_verified` 而非 `sfyz` |
| 保留字 | 禁用作表名/字段名:`order`、`group`、`key`、`select`、`desc`、`asc`、`user`(PG)、`level`、`type`(易冲突)、`status` 尚可但慎用 | 加业务前缀:`t_order`、`order_status`、`group_name` |
| 布尔前缀 | is_ / has_ / can_ 开头:`is_deleted`、`has_paid` | |
| 时间后缀 | _time 或 _at 结尾,同一项目内统一一种风格 | `created_at`、`updated_at` |
| 外键后缀 | 关联字段以 _id 结尾:`user_id`、`dept_id` | |
| 索引命名 | 普通索引 `idx_表名_列`、唯一索引 `uk_表名_列` | `idx_order_user_id` |

常见保留字速查:`order, group, key, select, from, where, desc, asc, left, right, inner, outer, union, all, distinct, having, limit, offset, case, when, then, else, end, as, on, and, or, not, null, in, exists, between, like, is, by, to, into, values, default, check, primary, foreign, unique, index, table, create, drop, alter, view, trigger, procedure, function, grant, revoke, user, level, type, name(PG 易冲突), date, time, timestamp`。遇到即报 🔴(MySQL 中反引号包裹可绕过,但属于埋雷,仍须报)。

## 2. 必备基础字段(三件套)

```sql
id         BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY COMMENT '主键',
created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
```

- 🔴 无主键。🟡 主键为业务字段(手机号、身份证号、订单号)——业务值会变更、会复用、长度不定,必须用代理主键。
- 主键推荐 `BIGINT UNSIGNED AUTO_INCREMENT`;分布式 ID 用雪花算法时去掉 AUTO_INCREMENT,保留 BIGINT。
- 不强制 UUID:UUID 作主键(随机写入导致页分裂、36 字符过长)应在 🟢 提示,除非用户已用。
- 软删除表:加 `is_deleted TINYINT NOT NULL DEFAULT 0` 或 `deleted_at DATETIME NULL`。
- 审计需要:加 `created_by` / `updated_by`。
- PostgreSQL:`id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY`;时间字段推荐 `TIMESTAMPTZ`。

## 3. 数据类型选择

| 场景 | ❌ 错误 | ✅ 正确 |
|---|---|---|
| 金额 | FLOAT / DOUBLE | `DECIMAL(12,2)`(🔴 级,浮点误差不可接受) |
| IP 地址 | VARCHAR(15) | `INT UNSIGNED`(MySQL,配 INET_ATON)/ `INET`(PG) |
| 布尔 | VARCHAR(1) 'Y'/'N' | `TINYINT(1)`(MySQL)/ `BOOLEAN`(PG) |
| 枚举 | ENUM 类型 | `TINYINT` + COMMENT 列出全部取值(🟡 级,ENUM 加值需 DDL 且排序陷阱) |
| 主键/关联 | INT(未来溢出) | BIGINT |
| 定长编码 | VARCHAR | CHAR(如国家码 CHAR(2)) |
| 时间 | VARCHAR 存时间 / BIGINT 时间戳 | DATETIME(或 TIMESTAMP,注意 2038 与时区) |
| 长度 | 一律 VARCHAR(255) | 按业务上限设:手机号 20、邮箱 128、名称 64~128(🟡 级) |
| 可空性 | 字段全部 NULL 无默认 | 尽量 `NOT NULL DEFAULT <值>`(🟡 级;NULL 使索引统计复杂、比较需 IS NULL) |
| 大对象 | TEXT/BLOB 在主表 | 拆扩展表 `主表_id + detail`(🟡 级,影响主表缓存与查询) |

## 4. 索引设计

- 🔴/🟡 级判定在 SKILL.md。细则:
- **最左前缀**:联合索引 `(a,b,c)` 仅查询含 a 或 (a,b) 前缀时生效;区分度高的列放左侧。等值列在前,范围列在后。
- **冗余索引**:存在 `(a,b)` 时单列 `(a)` 冗余;主键即 `(id)`,不再单建 id 索引。
- **索引数量**:单表 ≤ 5~6 个;超过报 🟡,列出现有索引让用户取舍。
- **索引失效写法**(审查 WHERE 相关字段时提示):对索引列做函数/运算 `WHERE YEAR(created_at)=2024` → 改范围 `created_at >= '2024-01-01' AND < '2025-01-01'`;隐式类型转换(字符串列用数字比较);前导 `%` 的 LIKE。
- **覆盖索引**:查询列都能被索引覆盖时避免回表(🟢)。
- **外键约束**:互联网高并发场景通常不建物理 FOREIGN KEY(影响写入与扩展),用应用层保证 + 索引;传统单体系统可保留。两种都可接受,发现"高频写入表带物理外键"报 🟢 提示。
- **唯一业务约束**:手机号、订单号等业务唯一字段必须有 UNIQUE KEY,只建普通索引报 🟡。

## 5. 表结构原则与三范式

三范式判定要点见 SKILL.md「三范式判定要点」。补充:

- **默认 3NF**:发现冗余字段(`dept_name` 随 `dept_id` 存储)→ 🔴,但 COMMENT 已注明冗余来源与同步策略的视为刻意反范式 → 🟢 通过。
- **垂直拆分**:字段 > 30~40 列,或少数大字段低频访问 → 🟢 建议拆 主表 + 扩展表。
- **水平拆分预警**:用户说明或字段语义暗示超大数据量(日志、流水、消息)→ 🟢 提示预估量并规划归档/分表。
- **存储过程/触发器**:DDL 中含 → 🔴(业务逻辑进应用层,数据库保持纯粹存储,便于迁移扩展)。
- **跨表冗余**(多表 DDL 一起审查时):同一份字典/配置数据在多表重复存储 → 🟡。

## 6. 字符集与引擎(MySQL)

- 字符集必须 `utf8mb4`(utf8=utf8mb3 存不了 Emoji,等同不合规,🟡);排序规则 `utf8mb4_general_ci` 或 `utf8mb4_unicode_ci`,全库统一。
- 引擎必须 InnoDB(🟡);MyISAM 无事务/行锁/崩溃安全,🔴。
- PostgreSQL:库级 `ENCODING 'UTF8'`;单表 DDL 无字符集概念,勿误报。

## 7. 注释

- 表和每个字段必须有 COMMENT(🟡)。内容要求:业务含义、单位(金额是分还是元!)、枚举字段列出全部取值 `0:待支付 1:已支付 2:已取消`、冗余字段注明来源与同步策略。
- `COMMENT '备注'`、`COMMENT '字段1'` 这类无信息量注释视为无注释。

## 8. 提交前自检清单

出报告前对每张表跑一遍,全部通过才给 ✅:

1. 表名、字段名全小写 + 下划线?
2. 有 `id`、`created_at`、`updated_at` 三件套?
3. 主键为非业务意义的 BIGINT?
4. 字段尽量 NOT NULL + DEFAULT?
5. 金额用 DECIMAL?
6. 索引符合最左前缀、无冗余、数量 ≤ 6?
7. 表和字段都有有效 COMMENT(枚举值已列出)?
8. 字符集 utf8mb4、引擎 InnoDB(MySQL)?
9. 无 TEXT/BLOB 在主表?无 ENUM?无保留字命名?
10. 无存储过程/触发器?
