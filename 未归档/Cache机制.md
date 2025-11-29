## 1. 映射方式 (Mapping) - *考点*
将主存块（Block）放到 Cache 行（Line）的规则。

| 方式 | 规则 | 优点 | 缺点 | 地址结构 |
| :--- | :--- | :--- | :--- | :--- |
| **直接映射** (Direct) | $Line = Block \pmod N$ | 简单，查找快 | 冲突率高（抖动） | Tag + Index + Offset |
| **全相联** (Fully) | 任意放 | 冲突率最低 | 比较器电路复杂，慢 | Tag + Offset |
| **组相联** (Set Assoc) | $Set = Block \pmod G$ | 折中方案 | 工业界主流（如 8路组相联） | Tag + SetIndex + Offset |

- Offset 块内偏移：只与块相关，由于内存一次搬运一个块，所以需要用以区分块内元素。_公式_：$Offset = \log_2(\text{块大小})$。
- Index 组索引：由 Cache 的组数或行数决定
- Tag 标记：身份验证的关键，到指定层数以后，与其中包含的 Tag 比对
	- 命中，拿走数据
	- 缺失，去主存找
	- 计算：地址剩下的高位全都用来做 Tag

- 直接映射：指定了行号
- 全相联：没有行号限制，需要并行比较，速度慢
- 组相联：根据行号划定空间，兼顾性能与便利性

## 2. 替换算法 (Replacement)
当 Cache 满了，踢谁？
* **LRU (Least Recently Used)**：踢掉最久没用过的。利用了时间局部性，效果最好，硬件实现稍繁（计数器/栈）。
* **FIFO**：先进先出。效果差，可能出现 Belady 现象（容量大反而命中率低）。
* **Random**：随机踢。硬件最简单，效果在大容量 Cache 下不比 LRU 差多少。

## 3. 写策略 (Write Policy) - *一致性难题*
* **写命中 (Write Hit)**：
    * **写回 (Write Back)**：只写 Cache，设脏位 (Dirty Bit)。被替换时才写回主存。（性能高，适合多核一致性协议如 MESI）。
    * **直写 (Write Through)**：同时写 Cache 和主存。（简单，但慢，需 Write Buffer）。
* **写缺失 (Write Miss)**：
    * **写分配 (Write Allocate)**：先调入 Cache，再写。（通常配对 Write Back）。
    * **非写分配 (No-write Allocate)**：直接写主存，不调入。（通常配对 Write Through）。

## 4. 关联链接
* [[Redis]]：Redis 的 LRU 淘汰策略与 CPU Cache 的异同。
* [[一致性哈希]]：分布式缓存的映射算法。