# 假如有个业务信息表，我要删除表前三个月state为0的数据，如何设计索引？为什么不用state做索引

<font style="color:rgba(6, 8, 31, 0.88);">根据需求是要定期删除</font>**<font style="color:rgba(6, 8, 31, 0.88);">三个月前且 state=0</font>**<font style="color:rgba(6, 8, 31, 0.88);"> 的业务信息表数据，核心 SQL 类大概长这样：</font>

```plsql
DELETE FROM your_table
WHERE state = 0 AND create_time < '某个日期';
```

### <font style="color:rgba(6, 8, 31, 0.88);">一、如何设计索引？</font>
<font style="color:rgba(6, 8, 31, 0.88);">建议为</font><font style="color:rgba(6, 8, 31, 0.88);"> </font>`<font style="color:rgba(6, 8, 31, 0.88);background-color:rgba(175, 184, 193, 0.2);">state</font>`<font style="color:rgba(6, 8, 31, 0.88);"> </font><font style="color:rgba(6, 8, 31, 0.88);">和</font><font style="color:rgba(6, 8, 31, 0.88);"> </font>`<font style="color:rgba(6, 8, 31, 0.88);background-color:rgba(175, 184, 193, 0.2);">create_time</font>`<font style="color:rgba(6, 8, 31, 0.88);"> </font><font style="color:rgba(6, 8, 31, 0.88);">建</font>**<font style="color:rgba(6, 8, 31, 0.88);">复合索引</font>**<font style="color:rgba(6, 8, 31, 0.88);">，如下：</font>

```plsql
CREATE INDEX idx_state_createtime ON your_table (state, create_time);
```

**<font style="color:rgba(6, 8, 31, 0.88);">理由：</font>**

+ `<font style="color:rgba(6, 8, 31, 0.88);background-color:rgba(175, 184, 193, 0.2);">state</font>`<font style="color:rgba(6, 8, 31, 0.88);"> </font><font style="color:rgba(6, 8, 31, 0.88);">是等值匹配条件（</font>`<font style="color:rgba(6, 8, 31, 0.88);background-color:rgba(175, 184, 193, 0.2);">=</font>`<font style="color:rgba(6, 8, 31, 0.88);">），而</font><font style="color:rgba(6, 8, 31, 0.88);"> </font>`<font style="color:rgba(6, 8, 31, 0.88);background-color:rgba(175, 184, 193, 0.2);">create_time</font>`<font style="color:rgba(6, 8, 31, 0.88);"> </font><font style="color:rgba(6, 8, 31, 0.88);">是范围过滤条件（</font>`<font style="color:rgba(6, 8, 31, 0.88);background-color:rgba(175, 184, 193, 0.2);"><</font>`<font style="color:rgba(6, 8, 31, 0.88);">）。</font>
+ <font style="color:rgba(6, 8, 31, 0.88);">按最优实践，复合索引应把等值查询的列放前面，范围查询的列放后面。</font>
+ <font style="color:rgba(6, 8, 31, 0.88);">这样，数据库可以先快速定位所有 state=0 的数据，再在这些数据中按 create_time 判断是否早于三个月前，极大减少了扫描和对比的数据行数。</font>

---

### <font style="color:rgba(6, 8, 31, 0.88);">二、为什么不用单独对</font><font style="color:rgba(6, 8, 31, 0.88);"> </font>`<font style="color:rgba(6, 8, 31, 0.88);background-color:rgba(175, 184, 193, 0.2);">state</font>`<font style="color:rgba(6, 8, 31, 0.88);"> </font><font style="color:rgba(6, 8, 31, 0.88);">建索引？</font>
**<font style="color:rgba(6, 8, 31, 0.88);">单独给 state 建索引效果差的原因如下：</font>**

1. **<font style="color:rgba(6, 8, 31, 0.88);">选择性低</font>**<font style="color:rgba(6, 8, 31, 0.88);">（通常只有 0/1 等少量不同的值），用 state 索引只能帮你筛出所有 state=0 的数据，如果这些数据量很大，还是要全表扫描这部分，再看哪些是三个月前的，性能就不高。</font>
2. **<font style="color:rgba(6, 8, 31, 0.88);">不能用索引加速 create_time 条件</font>**<font style="color:rgba(6, 8, 31, 0.88);">。单列 state 索引无法支持“先按 create_time 再筛 state=0”的需求。</font>
3. **<font style="color:rgba(6, 8, 31, 0.88);">复合索引能同时支持这两种条件的组合筛选</font>**<font style="color:rgba(6, 8, 31, 0.88);">，效率远大于单列索引。</font>

---

### <font style="color:rgba(6, 8, 31, 0.88);">总结</font>
+ **<font style="color:rgba(6, 8, 31, 0.88);">正确做法</font>**<font style="color:rgba(6, 8, 31, 0.88);">：</font>`<font style="color:rgba(6, 8, 31, 0.88);background-color:rgba(175, 184, 193, 0.2);">(state, create_time)</font>`<font style="color:rgba(6, 8, 31, 0.88);"> </font><font style="color:rgba(6, 8, 31, 0.88);">复合索引。</font>
+ **<font style="color:rgba(6, 8, 31, 0.88);">不推荐只用 state 索引</font>**<font style="color:rgba(6, 8, 31, 0.88);">：选择性差，只能加速其中一部分，效率低。</font>



> 更新: 2025-05-26 21:22:28  
> 原文: <https://www.yuque.com/tulingzhouyu/db22bv/velgnom7tksibfn1>