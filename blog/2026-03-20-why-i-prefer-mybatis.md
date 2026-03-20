---
title: 为什么我不喜欢用 MyBatis Plus、JPA 框架，更愿意使用 MyBatis
authors: [crisxue]
tags: [mybatis, mybatis-plus, jpa, orm]
---

作为一个长期使用 MyBatis 的开发者，我经常被问到："为什么不用 MyBatis-Plus 或 JPA？它们不是更方便吗？"

今天我想聊聊我的思考。

<!-- truncate -->

## 先说结论

我不喜欢 MyBatis-Plus 和 JPA，不是因为它们不好，而是因为它们解决问题的方向和我想要的不一样。

我需要的是：**保留 SQL 的可控性，同时减少样板代码**。

而 MyBatis-Plus 和 JPA 的做法，多多少少都在牺牲可控性来换取便利。

## JPA 的问题：黑盒运行时

JPA 是一个伟大的规范，它的对象建模思想很优雅。但问题在于它的运行时机制：

### 1. 隐式 SQL 执行

```java
User user = entityManager.find(User.class, 1L);
user.setName("新名字");
// 没有 SQL？但数据库已经更新了
```

这行代码背后发生了什么？什么时候执行的 SQL？这些问题让我很不安。

### 2. 脏检查和自动 Flush

```java
@Transactional
public void updateUser() {
    User user = userDao.findById(1L);
    user.setName("新名字");
    // 方法结束时自动 flush，你无法精确控制
}
```

### 3. N+1 问题难以控制

JPA 的懒加载很容易触发 N+1 查询，而且很难预测什么时候会发生。

### 4. 复杂查询的噩梦

```java
Specification<User> spec = (root, query, cb) -> {
    List<Predicate> predicates = new ArrayList<>();
    if (name != null) {
        predicates.add(cb.like(root.get("name"), "%" + name + "%"));
    }
    if (age != null) {
        predicates.add(cb.gt(root.get("age"), age));
    }
    return cb.and(predicates.toArray(new Predicate[0]));
};
```

这段代码真的比 SQL 更清晰吗？

## MyBatis-Plus 的问题：逻辑泄露

MyBatis-Plus 确实很方便，但它的设计哲学让持久层逻辑渗透到了业务层：

### 1. Wrapper 的陷阱

```java
public List<User> searchUsers(String name, Integer age, String dept) {
    LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
    if (StringUtils.isNotBlank(name)) {
        wrapper.like(User::getName, name);
    }
    if (age != null) {
        wrapper.gt(User::getAge, age);
    }
    if (StringUtils.isNotBlank(dept)) {
        wrapper.eq(User::getDept, dept);
    }
    return userMapper.selectList(wrapper);
}
```

这段代码的问题：

- 业务层充斥着数据库字段名
- 查询逻辑散落在各个 Service 方法中
- 难以复用和测试

### 2. Service 层变得臃肿

本来 Service 层应该只关心业务流程，现在却充满了查询条件拼装逻辑。

### 3. 失去了 SQL 的精确控制

虽然 MyBatis-Plus 基于 MyBatis，但 Wrapper 的抽象让你远离了真正的 SQL。

## MyBatis 的魅力：显式、可控

原生 MyBatis 的问题是需要写很多样板代码。但它的核心理念我很认同：

### 1. SQL 是显式的

```xml
<select id="findActiveUsers" resultType="User">
    SELECT * FROM user
    WHERE status = 1
    AND create_time > #{startTime}
    ORDER BY create_time DESC
</select>
```

每一行 SQL 都是你写的，你知道它在做什么。

### 2. 执行时机是确定的

没有隐式 SQL，没有自动 flush，一切都在你的控制之中。

### 3. 调试友好

SQL 写在 XML 里，出问题时一目了然。

## MyBatisGX：我的解决方案

既然 MyBatis 的可控性这么好，能不能在保留它的同时，减少样板代码？

这就是我开发 MyBatisGX 的初衷。

### 1. 方法名派生 SQL

```java
// 不需要写 XML
List<User> findByNameLikeAndAgeGt(String name, Integer age);
```

### 2. 查询实体解耦

```java
@QueryEntity(User.class)
public class UserQuery extends User {
    private String nameLike;
    private Integer ageGt;
}

// Service 层保持干净
public List<User> search(UserQuery query) {
    return userDao.findList(query);
}
```

### 3. XML 仍然有最高优先级

如果你想精确控制某个查询，写 XML 就行：

```xml
<select id="findComplexUsers" resultType="User">
    -- 你的复杂 SQL
</select>
```

### 4. 声明式关联查询

不用写 ResultMap，用注解声明关联关系：

```java
@OneToMany(mappedBy = "org", fetch = FetchType.EAGER)
@Fetch(FetchMode.BATCH)
private List<User> userList;
```

## AI 时代的新考量

还有一个重要因素：AI 编程助手。

大模型的上下文是有限的。我们应该让宝贵的上下文承载更多有意义的代码，而不是重复的样板代码或复杂的 Wrapper 拼装逻辑。

```java
// 传统方式：占用了多少 token？
LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
wrapper.like(User::getName, name).gt(User::getAge, age);

// MyBatisGX：简洁明了
List<User> findByNameLikeAndAgeGt(String name, Integer age);
```

## 结语

选择框架就是选择一种思维方式。

如果你像我一样：
- 厌恶黑盒运行时
- 需要 SQL 的精确控制
- 希望业务层保持干净
- 在 AI 时代追求代码简洁

那么 MyBatis + MyBatisGX 可能是一个值得尝试的组合。

---

欢迎在 [GitHub](https://github.com/cris-xue/mybatisgx) 上关注 MyBatisGX 的最新动态！
