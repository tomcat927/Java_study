# 哪些场景可以利用索引覆盖来优化SQL

<font style="color:rgb(36, 41, 47);">在SQL查询优化中，索引覆盖（Covering Index）是一种通过索引直接提供查询所需数据而无需回表的技术，可显著提升性能。以下是适用场景及具体说明：</font>

### <font style="color:rgb(36, 41, 47);">1.</font><font style="color:rgb(36, 41, 47);"> </font>**<font style="color:rgb(36, 41, 47);">全表COUNT查询优化</font>**
<font style="color:rgb(36, 41, 47);">当统计某非主键列的数量时，若为该列建立索引，数据库可直接扫描索引（而非数据行）完成统计。例如：</font>

```plsql
-- 原表user无索引时无法覆盖
SELECT COUNT(name) FROM user;  
-- 添加索引后，直接通过索引统计
ALTER TABLE user ADD KEY(name);
```

<font style="color:rgb(36, 41, 47);">此时索引</font>`<font style="color:rgb(36, 41, 47);">name</font>`<font style="color:rgb(36, 41, 47);">包含所有需要统计的数据，避免全表扫描</font><font style="color:rgb(36, 41, 47);">。</font>

---

### <font style="color:rgb(36, 41, 47);">2.</font><font style="color:rgb(36, 41, 47);"> </font>**<font style="color:rgb(36, 41, 47);">避免回表查询</font>**
<font style="color:rgb(36, 41, 47);">若查询列未被索引覆盖，需回表查询数据行。通过升级为联合索引可解决：</font>

```plsql
-- 原单列索引name需回表查sex
SELECT id, name, sex FROM user WHERE name = 'shenjian';  
-- 升级为联合索引(name, sex)后，直接通过索引返回结果
CREATE INDEX idx_name_sex ON user(name, sex);
```

<font style="color:rgb(36, 41, 47);">联合索引包含所有查询列，无需访问主键索引</font><font style="color:rgb(36, 41, 47);">。</font>

---

### <font style="color:rgb(36, 41, 47);">3.</font><font style="color:rgb(36, 41, 47);"> </font>**<font style="color:rgb(36, 41, 47);">分页查询优化</font>**
<font style="color:rgb(36, 41, 47);">分页查询（如</font>`<font style="color:rgb(36, 41, 47);">LIMIT</font>`<font style="color:rgb(36, 41, 47);">）若涉及排序和大量数据扫描，可通过覆盖索引减少回表。例如：</font>

```plsql
-- 原索引name需回表获取sex
SELECT id, name, sex FROM user ORDER BY name LIMIT 500, 100;  
-- 联合索引(name, sex)直接提供排序和返回列
CREATE INDEX idx_name_sex ON user(name, sex);
```

<font style="color:rgb(36, 41, 47);">索引覆盖避免全表扫描和额外排序操作</font><font style="color:rgb(36, 41, 47);">。</font>

---

### <font style="color:rgb(36, 41, 47);">4.</font><font style="color:rgb(36, 41, 47);"> </font>**<font style="color:rgb(36, 41, 47);">无WHERE条件的查询</font>**
<font style="color:rgb(36, 41, 47);">即使无过滤条件，若查询列被索引覆盖，仍可避免全表扫描：</font>

```plsql
-- 直接通过索引返回name和age
SELECT name, age FROM user;  
-- 创建联合索引(name, age)实现覆盖
CREATE INDEX idx_name_age ON user(name, age);
```

<font style="color:rgb(36, 41, 47);">适用于需要频繁读取特定列的报表类查询</font><font style="color:rgb(36, 41, 47);">。</font>

---

### <font style="color:rgb(36, 41, 47);">5.</font><font style="color:rgb(36, 41, 47);"> </font>**<font style="color:rgb(36, 41, 47);">WHERE条件区分度低的场景</font>**
<font style="color:rgb(36, 41, 47);">当WHERE条件区分度低（如性别字段）时，联合索引仍可通过覆盖减少数据访问：</font>

```plsql
-- 性别区分度低，但联合索引(sex, age)覆盖查询
SELECT age FROM user WHERE sex = 'male';
```

<font style="color:rgb(36, 41, 47);">即使需扫描较多索引条目，仍比回表访问数据行高效</font><font style="color:rgb(36, 41, 47);">。</font>

---

### <font style="color:rgb(36, 41, 47);">6.</font><font style="color:rgb(36, 41, 47);"> </font>**<font style="color:rgb(36, 41, 47);">仅查询主键的场景</font>**
<font style="color:rgb(36, 41, 47);">辅助索引的叶子节点存储主键值，因此仅查询主键时可天然覆盖：</font>

```plsql
-- 辅助索引name的叶子节点含主键id
SELECT id FROM user WHERE name = 'shenjian';
```

<font style="color:rgb(36, 41, 47);">无需回表，直接通过辅助索引返回结果</font><font style="color:rgb(36, 41, 47);">。</font>

---

### <font style="color:rgb(36, 41, 47);">7.</font><font style="color:rgb(36, 41, 47);"> </font>**<font style="color:rgb(36, 41, 47);">JOIN和ORDER BY优化</font>**
<font style="color:rgb(36, 41, 47);">若JOIN的关联字段或ORDER BY的排序列被索引覆盖，可优化连接和排序效率：</font>

```plsql
-- 联合索引(t.emp_no, t.title)覆盖JOIN和SELECT列
SELECT t.emp_no, t.title FROM employees e  
LEFT JOIN titles t ON e.emp_no = t.emp_no;
```

<font style="color:rgb(36, 41, 47);">覆盖索引减少表访问和排序开销</font><font style="color:rgb(36, 41, 47);">。</font>

---

### <font style="color:rgb(36, 41, 47);">注意事项</font>
+ **<font style="color:rgb(36, 41, 47);">索引类型限制</font>**<font style="color:rgb(36, 41, 47);">：仅B-Tree索引支持覆盖，Hash/全文索引不适用</font><font style="color:rgb(36, 41, 47);">。</font>
+ **<font style="color:rgb(36, 41, 47);">避免SELECT</font>**<font style="color:rgb(36, 41, 47);">*：明确列出所需列以增加覆盖可能性，减少索引体积</font><font style="color:rgb(36, 41, 47);">。</font>
+ **<font style="color:rgb(36, 41, 47);">维护成本</font>**<font style="color:rgb(36, 41, 47);">：覆盖索引可能增加写入开销，需权衡读写比例</font><font style="color:rgb(36, 41, 47);">。</font>

<font style="color:rgb(36, 41, 47);">通过合理设计索引覆盖上述场景，可显著减少I/O和CPU消耗，是SQL性能优化的核心手段之一。</font>



> 更新: 2025-05-19 22:09:37  
> 原文: <https://www.yuque.com/tulingzhouyu/db22bv/xztwyi8t0gufceba>