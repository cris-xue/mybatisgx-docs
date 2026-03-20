---
sidebar_position: 6
---

# 抓取模式

> SIMPLE/BATCH/JOIN 三种抓取模式详解

## 概述

MyBatisGX 提供三种抓取模式解决关联查询的性能问题，默认使用 BATCH 模式。

## 三种抓取模式

```
┌─────────────────────────────────────────────────────────────────┐
│                      抓取模式对比                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SIMPLE: N+1 问题                                               │
│  ┌────────┐      ┌────────┐      ┌────────┐                    │
│  │ 查询主表 │ ───▶ │ 查询关联1│ ───▶ │ 查询关联2│  ...           │
│  └────────┘      └────────┘      └────────┘                    │
│  1 条 + N 条 = N+1 条 SQL                                       │
│                                                                 │
│  BATCH: 批量查询                                                │
│  ┌────────┐      ┌────────────────────────────────┐            │
│  │ 查询主表 │ ───▶ │ 批量查询所有关联（WHERE id IN）│            │
│  └────────┘      └────────────────────────────────┘            │
│  1 条 + M 条 = 1+M 条 SQL（M 为关联表数量）                      │
│                                                                 │
│  JOIN: 联表查询                                                 │
│  ┌────────────────────────────────────────────────┐            │
│  │ SELECT * FROM main LEFT JOIN related ON ...    │            │
│  └────────────────────────────────────────────────┘            │
│  1 + 1 = 2 条 SQL（主表 + 无限 JOIN）                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## SIMPLE 模式

### 特点

- 简单查询模式
- 每条关联数据单独查询
- 存在 N+1 问题
- 支持懒加载

### 使用场景

- 结果集很小
- 关联数据访问不频繁

### 示例

```java
@OneToMany(fetch = FetchType.EAGER)
@Fetch(FetchMode.SIMPLE)
@JoinColumn(name = "org_id")
private List<User> userList;
```

### SQL 行为

```sql
-- 查询 10 个部门
SELECT * FROM org;

-- 对每个部门单独查询用户（10 次）
SELECT * FROM user WHERE org_id = 1;
SELECT * FROM user WHERE org_id = 2;
...
SELECT * FROM user WHERE org_id = 10;

-- 总计：1 + 10 = 11 条 SQL
```

## BATCH 模式（推荐）

### 特点

- 批量查询模式
- 收集所有外键后批量查询
- 解决 N+1 问题
- 支持懒加载
- **默认推荐模式**

### 使用场景

- 结果集较大
- 需要访问所有关联数据

### 示例

```java
@OneToMany(fetch = FetchType.EAGER)
@Fetch(FetchMode.BATCH)
@JoinColumn(name = "org_id")
private List<User> userList;
```

### SQL 行为

```sql
-- 查询 10 个部门
SELECT * FROM org;

-- 批量查询所有用户
SELECT * FROM user WHERE org_id IN (1, 2, 3, ..., 10);

-- 总计：1 + 1 = 2 条 SQL
```

## JOIN 模式

### 特点

- 联表查询模式
- 第一级单表查询，获取主键
- 第二级无限 JOIN 查询
- 可能产生结果膨胀
- 不支持懒加载

### 使用场景

- 结果集很小
- 需要一次性获取所有关联数据

### 示例

```java
@OneToMany(fetch = FetchType.EAGER)
@Fetch(FetchMode.JOIN)
@JoinColumn(name = "org_id")
private List<User> userList;
```

### SQL 行为

```sql
-- 第一步：单表查询获取主键
SELECT id FROM org;

-- 第二步：无限 JOIN 查询
SELECT *
FROM org
LEFT JOIN user ON user.org_id = org.id
LEFT JOIN user_detail ON user_detail.user_id = user.id
...
WHERE org.id IN (1, 2, 3, ...);

-- 总计：1 + 1 = 2 条 SQL
```

### 结果膨胀问题

```sql
-- 一对多关联可能导致结果集膨胀
-- 1 个部门 + 100 个用户 = 100 行结果
-- 需要在应用层去重
```

## 模式选择建议

| 场景 | 推荐模式 | 原因 |
|------|----------|------|
| 结果集小 | JOIN | 一次查询，性能好 |
| 结果集大 | BATCH | 避免 N+1，结果集可控 |
| 懒加载需求 | SIMPLE / BATCH | JOIN 不支持懒加载 |
| 关联层级深 | BATCH | JOIN 可能过度膨胀 |

## FetchType 与 FetchMode 配合

### FetchType

| 值 | 说明 |
|------|------|
| `LAZY` | 懒加载，访问时才查询 |
| `EAGER` | 立即加载，主查询时一并查询 |

### 组合使用

```java
// 立即加载 + 批量查询（推荐）
@OneToMany(fetch = FetchType.EAGER)
@Fetch(FetchMode.BATCH)
private List<User> userList;

// 懒加载 + 简单查询
@OneToMany(fetch = FetchType.LAZY)
@Fetch(FetchMode.SIMPLE)
private List<User> userList;

// 立即加载 + 联表查询
@OneToMany(fetch = FetchType.EAGER)
@Fetch(FetchMode.JOIN)
private List<User> userList;
```

## 自动抓取失效条件

当 mapper.xml 中存在对应方法时，自动抓取**失效**：

```java
// 如果 UserMapper.xml 定义了 findById
// MyBatisGX 不会自动处理关联抓取
// 需要在 XML 中手动定义关联查询
```

## 完整示例

```java
@Entity
@Table(name = "org")
public class Org {

    @Id
    private Long id;

    private String name;

    // 推荐配置：立即加载 + 批量查询
    @OneToMany(mappedBy = "org", fetch = FetchType.EAGER)
    @Fetch(FetchMode.BATCH)
    private List<User> userList;

    // 懒加载配置
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "parent_id")
    private Org parent;

    // 小结果集：JOIN 模式
    @OneToOne(fetch = FetchType.EAGER)
    @Fetch(FetchMode.JOIN)
    @JoinColumn(name = "manager_id")
    private User manager;
}
```

## 性能对比

查询 100 个部门，每个部门平均 10 个用户：

| 模式 | SQL 条数 | 性能 |
|------|----------|------|
| SIMPLE | 1 + 100 = 101 | 差 |
| BATCH | 1 + 1 = 2 | 好 |
| JOIN | 1 + 1 = 2 | 好（但结果膨胀） |

## 注意事项

1. **默认模式**：框架默认使用 BATCH 模式

2. **JOIN 模式限制**：不支持懒加载，可能结果膨胀

3. **懒加载要求**：懒加载需要在事务内或 Session 上下文访问

4. **XML 优先**：mapper.xml 定义的方法不会触发自动抓取

## 下一步

- 学习 [逻辑删除](../advanced/logic-delete)
- 了解 [字段值生成](../advanced/value-generation)
