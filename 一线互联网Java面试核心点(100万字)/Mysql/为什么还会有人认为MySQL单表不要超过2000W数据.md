# 为什么还会有人认为MySQL单表不要超过2000W数据

<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">关于MySQL单表建议最大两千万行的说法，其核心依据主要源于InnoDB存储引擎的B+树索引结构对查询性能的影响，以及实际业务维护的经验总结。以下是详细分析：</font>

---

### **<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">1. B+树索引结构限制</font>**
![1747317753732-3fa5f873-fbf5-44b2-91b4-90c5152f3cc2.png](./img/JV32AXA3Gqr1LBGY/1747317753732-3fa5f873-fbf5-44b2-91b4-90c5152f3cc2-453103.png)

<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">InnoDB使用B+树组织数据，主键索引（聚簇索引）的叶子节点直接存储行数据。</font>

<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">B+树的高度直接影响查询效率，树越高，磁盘I/O次数越多。以下为具体计算：</font>

![1747317828754-6aef92cd-2107-4059-9122-83d54f463c2c.png](./img/JV32AXA3Gqr1LBGY/1747317828754-6aef92cd-2107-4059-9122-83d54f463c2c-788893.png)

#### **<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">关键参数</font>**
![1747317855029-436fe338-27ce-4996-afba-eed1100e1e8e.png](./img/JV32AXA3Gqr1LBGY/1747317855029-436fe338-27ce-4996-afba-eed1100e1e8e-118948.png)

+ **<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">页大小（Page Size）</font>**<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">：默认16KB（16384字节）。</font>
+ **<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">主键类型</font>**<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">：假设使用</font>`<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">BIGINT</font>`<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">（8字节）或</font>`<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">INT</font>`<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">（4字节）。</font>
+ **<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">指针大小</font>**<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">：InnoDB中每个页指针占6字节。</font>
+ **<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">单行数据大小</font>**<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">：假设行数据平均约1KB。</font>

#### **<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">计算过程</font>**
1. **<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">非叶子节点容量</font>**<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">  
</font><font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">每个非叶子节点存储主键值 + 指针。例如：</font>
    - `<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">BIGINT</font>`<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">主键：每个条目占</font><font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);"> </font>`<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">8+6=14字节</font>`<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">。</font>
    - <font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">单页可存条目数：</font>`<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">16384 / 14 ≈ 1170条</font>`<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">。</font>
2. **<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">B+树高度与最大行数</font>**
    - **<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">高度为3时</font>**<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">：  
</font><font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">根页 + 1170个二级页 + 1170×1170个叶子页。  
</font><font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">叶子页总数：</font>`<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">1170 × 1170 ≈ 1.37M页</font>`<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">。  
</font><font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">若每页存储约15行（1KB/行）：总行数</font><font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);"> </font>`<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">1.37M × 15 ≈ 20.5M</font>`<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">。</font>
    - **<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">高度为4时</font>**<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">：  
</font><font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">总行数飙升至</font><font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);"> </font>`<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">1170^3 × 15 ≈ 24亿</font>`<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">，但查询需4次I/O，性能显著下降。</font>

![1747547163604-7fba5c1b-01c6-4a19-a038-179b765c11b5.png](./img/JV32AXA3Gqr1LBGY/1747547163604-7fba5c1b-01c6-4a19-a038-179b765c11b5-423260.png)

#### **<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">为何建议两千万？</font>**
+ **<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">经验平衡点</font>**<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">：在高度为3时，若单页存储行数更少（如因大字段或行格式限制），总行数可能降至约2000万。</font>
+ **<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">性能拐点</font>**<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">：数据量接近B+树层级跃升临界值时（如从3层到4层），写入和范围查询效率会骤降。</font>

![1747547120502-f030243e-6b9b-47d1-8fd0-f2ae1783f53a.png](./img/JV32AXA3Gqr1LBGY/1747547120502-f030243e-6b9b-47d1-8fd0-f2ae1783f53a-470630.png)

---

### **<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">2. 实际业务维护考量</font>**
+ **<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">查询延迟</font>**<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">：单表过大时，即使走索引，扫描少量数据也可能因磁盘寻址变慢。</font>
+ **<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">DDL操作风险</font>**<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">：修改表结构（如加列）需锁表或重建表，耗时与数据量正相关。</font>
+ **<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">备份与恢复</font>**<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">：大表备份时间长，故障恢复风险高。</font>
+ **<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">分库分表阈值</font>**<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">：提前规划水平拆分，避免性能恶化后紧急处理。</font>

---

### **<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">3. 其他影响因素</font>**
+ **<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">行溢出（Row Overflow）</font>**<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">：若单行数据超过页大小（如含</font>`<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">TEXT/BLOB</font>`<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">），部分数据存于额外页，导致有效单页行数减少。</font>
+ **<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">碎片化</font>**<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">：频繁增删导致页空间利用率低，实际容量小于理论值。</font>
+ **<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">辅助索引</font>**<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">：二级索引叶子节点存储主键值，过多索引会增加存储压力，间接影响主树性能。</font>

---

### **<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">总结</font>**
+ **<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">理论极限</font>**<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">：InnoDB单表可存储数十亿行，但性能会随B+树高度增加而下降。</font>
+ **<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">经验建议</font>**<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">：2000万行是平衡查询效率、维护成本后的经验值，尤其在主键为</font>`<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">BIGINT</font>`<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">、行大小适中的场景下。</font>
+ **<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">灵活调整</font>**<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">：实际阈值需根据硬件性能、数据访问模式（如冷热数据分离）动态评估。</font>

---

### **<font style="color:rgba(0, 0, 0, 0.9);background-color:rgb(252, 252, 252);">优化建议</font>**
![1747547082813-c7a3c414-086a-4545-90ef-5b63ad58dc7f.png](./img/JV32AXA3Gqr1LBGY/1747547082813-c7a3c414-086a-4545-90ef-5b63ad58dc7f-509807.png)



> 更新: 2025-05-19 15:20:55  
> 原文: <https://www.yuque.com/tulingzhouyu/db22bv/pnba74buu7rzd3hw>