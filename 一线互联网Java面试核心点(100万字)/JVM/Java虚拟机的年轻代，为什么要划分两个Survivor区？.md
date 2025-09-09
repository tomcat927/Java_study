# Java 虚拟机的年轻代，为什么要划分两个 Survivor 区？

<font style="color:rgb(64, 64, 64);">在Java虚拟机（JVM）的年轻代（Young Generation）中，划分两个Survivor区（通常称为S0和S1）是为了</font>**<font style="color:rgb(64, 64, 64);">优化垃圾回收（Garbage Collection, GC）的效率</font>**<font style="color:rgb(64, 64, 64);">，尤其是通过</font>**<font style="color:rgb(64, 64, 64);">复制算法（Copying Algorithm）</font>**<font style="color:rgb(64, 64, 64);">来减少内存碎片、降低GC停顿时间，并高效管理对象的存活周期。以下是具体原因和实现逻辑：</font>

---

### **<font style="color:rgb(64, 64, 64);">1. 复制算法</font>**
![1748351625795-53244465-cb50-4d34-8582-979d37fb67ad.png](./img/3diJ7AXJwqzrcoHA/1748351625795-53244465-cb50-4d34-8582-979d37fb67ad-211993.png)

<font style="color:rgb(64, 64, 64);">年轻代的设计目标是</font>**<font style="color:rgb(64, 64, 64);">快速回收短生命周期对象</font>**<font style="color:rgb(64, 64, 64);">。复制算法的核心思想是：</font>

+ **<font style="color:rgb(64, 64, 64);">内存分为两块（如S0和S1）</font>**<font style="color:rgb(64, 64, 64);">，每次GC只使用其中一块（例如S0）。</font>
+ <font style="color:rgb(64, 64, 64);">当触发Minor GC时，存活的对象会从Eden区和当前使用的Survivor区（如S0）</font>**<font style="color:rgb(64, 64, 64);">复制到另一个空闲的Survivor区（如S1）</font>**<font style="color:rgb(64, 64, 64);">。</font>
+ <font style="color:rgb(64, 64, 64);">复制完成后，直接清空原Survivor区（S0）和Eden区，避免内存碎片。</font>

**<font style="color:rgb(64, 64, 64);">优势：</font>**

+ **<font style="color:rgb(64, 64, 64);">内存无碎片</font>**<font style="color:rgb(64, 64, 64);">：每次复制后，存活对象在目标Survivor区中连续存放，无需处理碎片。</font>
+ **<font style="color:rgb(64, 64, 64);">快速回收</font>**<font style="color:rgb(64, 64, 64);">：仅需遍历存活对象并复制，时间复杂度为O(存活对象数量)，而非O(堆大小)。</font>
+ **<font style="color:rgb(64, 64, 64);">空间局部性</font>**<font style="color:rgb(64, 64, 64);">：存活对象连续存放，提高缓存命中率。</font>

---

### **<font style="color:rgb(64, 64, 64);">2. 对象年龄的渐进式晋升</font>**
![1748355041167-274dcf03-c5fa-4a2b-bb9a-041601fb47f7.png](./img/3diJ7AXJwqzrcoHA/1748355041167-274dcf03-c5fa-4a2b-bb9a-041601fb47f7-956216.png)

<font style="color:rgb(64, 64, 64);">JVM通过</font>**<font style="color:rgb(64, 64, 64);">分代年龄（Age）</font>**<font style="color:rgb(64, 64, 64);">机制决定对象何时晋升到老年代：</font>

+ <font style="color:rgb(64, 64, 64);">对象每在年轻代存活一次Minor GC，年龄加1。</font>
+ <font style="color:rgb(64, 64, 64);">默认当年龄达到阈值（如15）时，对象晋升到老年代。</font>

**<font style="color:rgb(64, 64, 64);">两个Survivor区的作用：</font>**

+ **<font style="color:rgb(64, 64, 64);">交替复制</font>**<font style="color:rgb(64, 64, 64);">：每次Minor GC时，存活对象在两个Survivor区之间复制，年龄逐步增加。</font>
+ **<font style="color:rgb(64, 64, 64);">过滤短期对象</font>**<font style="color:rgb(64, 64, 64);">：大部分对象在几次Minor GC后死亡，只有长期存活的对象才会晋升到老年代，减少老年代GC压力。</font>

---

### **<font style="color:rgb(64, 64, 64);">3. 内存利用率优化</font>**
<font style="color:rgb(64, 64, 64);">若仅使用一个Survivor区：</font>

+ **<font style="color:rgb(64, 64, 64);">内存浪费</font>**<font style="color:rgb(64, 64, 64);">：每次GC后，存活对象需占用整个Survivor区，可能导致内存浪费。</font>
+ **<font style="color:rgb(64, 64, 64);">碎片问题</font>**<font style="color:rgb(64, 64, 64);">：多次GC后，Survivor区可能产生碎片，降低内存使用效率。</font>

<font style="color:rgb(64, 64, 64);">使用两个Survivor区：</font>

+ **<font style="color:rgb(64, 64, 64);">动态平衡</font>**<font style="color:rgb(64, 64, 64);">：每次GC后，存活对象被压缩到另一个Survivor区，保证内存紧凑。</font>
+ **<font style="color:rgb(64, 64, 64);">内存利用率最大化</font>**<font style="color:rgb(64, 64, 64);">：两个Survivor区的总大小固定（如年轻代的1/10），交替使用确保空间高效分配。</font>

---

### **<font style="color:rgb(64, 64, 64);">4. 具体流程示例</font>**
![1748353256393-b8f8814b-30fb-4127-a3a0-ed8c42dec4d3.jpeg](./img/3diJ7AXJwqzrcoHA/1748353256393-b8f8814b-30fb-4127-a3a0-ed8c42dec4d3-765480.jpeg)

1. **<font style="color:rgb(64, 64, 64);">对象分配</font>**<font style="color:rgb(64, 64, 64);">：</font>
    - <font style="color:rgb(64, 64, 64);">新对象分配在Eden区。</font>
    - <font style="color:rgb(64, 64, 64);">当Eden区满时，触发Minor GC。</font>
2. **<font style="color:rgb(64, 64, 64);">Minor GC过程</font>**<font style="color:rgb(64, 64, 64);">：</font>
    - <font style="color:rgb(64, 64, 64);">存活对象从Eden区和当前Survivor区（如S0）复制到另一个Survivor区（S1）。</font>
    - <font style="color:rgb(64, 64, 64);">对象年龄加1，年龄达标（如15）的对象晋升到老年代。</font>
    - <font style="color:rgb(64, 64, 64);">清空Eden区和S0。</font>
3. **<font style="color:rgb(64, 64, 64);">Survivor区切换</font>**<font style="color:rgb(64, 64, 64);">：</font>
    - <font style="color:rgb(64, 64, 64);">下次GC时，存活对象从Eden和S1复制到S0，循环往复。</font>

---

### **<font style="color:rgb(64, 64, 64);">5. 为什么不能只有一个Survivor区？</font>**
+ **<font style="color:rgb(64, 64, 64);">内存碎片</font>**<font style="color:rgb(64, 64, 64);">：多次GC后，存活对象分散在Survivor区，产生碎片。</font>
+ **<font style="color:rgb(64, 64, 64);">复制效率低</font>**<font style="color:rgb(64, 64, 64);">：若仅有一个Survivor区，需额外空间存放存活对象，导致内存浪费。</font>
+ **<font style="color:rgb(64, 64, 64);">晋升机制失效</font>**<font style="color:rgb(64, 64, 64);">：无法通过交替复制逐步增加对象年龄，可能提前晋升大量对象到老年代。</font>

---

### **<font style="color:rgb(64, 64, 64);">总结</font>**
<font style="color:rgb(64, 64, 64);">划分两个Survivor区的核心目的是：</font>

1. **<font style="color:rgb(64, 64, 64);">利用复制算法高效回收内存</font>**<font style="color:rgb(64, 64, 64);">，避免碎片。</font>
2. **<font style="color:rgb(64, 64, 64);">通过年龄分阶段筛选对象</font>**<font style="color:rgb(64, 64, 64);">，减少老年代GC压力。</font>
3. **<font style="color:rgb(64, 64, 64);">优化内存利用率</font>**<font style="color:rgb(64, 64, 64);">，平衡性能与资源消耗。</font>

<font style="color:rgb(64, 64, 64);">这种设计是JVM对“</font>**<font style="color:rgb(64, 64, 64);">弱分代假说（Weak Generational Hypothesis）</font>**<font style="color:rgb(64, 64, 64);">”的实践，即大部分对象生命周期短暂，通过年轻代的高效回收机制显著提升GC效率。</font>



> 更新: 2025-05-28 21:28:00  
> 原文: <https://www.yuque.com/tulingzhouyu/db22bv/vuzh472dhqz50xyh>