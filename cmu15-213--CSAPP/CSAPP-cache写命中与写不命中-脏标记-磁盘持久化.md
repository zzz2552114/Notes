你记得的那个“lazy”操作叫 **write-back（写回）**，靠 **dirty bit（脏位）** 实现。

先分清：CSAPP 缓存章节里，“下一层”通常是主存。CPU 写数据时先看 cache：

### 1. 写命中 cache
- **write-through（写直达/直写）**：同时写 cache 和内存。内存立刻更新。
- **write-back（写回）**：只写 cache，并把该 cache line 标为 dirty，**不立刻写内存**。等这个块被替换/驱逐（eviction）时，如果 dirty，才写回内存。  
  这就是你说的“lazy”：延迟写内存，多次写只最后写回一次。

### 2. 写不命中 cache
看是否 **write-allocate（写分配）**：
- **write-allocate**：先把目标块从内存读到 cache，可能先驱逐一个 victim；如果 victim 是 dirty，还要先写回内存。然后把新数据写进 cache。若是 write-back，就标 dirty。  
  所以是：**先拉到 cache，再写 cache，之后延迟写内存**。
- **no-write-allocate（非写分配）**：不把块拉进 cache，直接写到下一层内存。通常和 write-through 搭配。

典型组合：
- **write-back + write-allocate**：现代 CPU cache 常见。
- **write-through + no-write-allocate**：较简单。

所以顺序不是固定的，取决于策略。你记的 lazy 是：**写命中时只改 cache，标 dirty，等替换时再写内存**。

---

换成 SSD 硬盘时，要区分层次。  
文件系统/操作系统层面也有类似 write-back：你调用 `write()` 写文件，通常只是写到 **OS page cache**，标 dirty，延迟回写。突然断电会丢最近没回写的数据。

所以不是“SSD 不能 lazy”，而是：**不能把 lazy 当成持久化保证**。  
如果要保证断电前真的写进 SSD，需要：
- 应用层：`fsync()` / `fdatasync()`，或 `O_SYNC` / `O_DSYNC`；
- 文件系统/块层：发 flush / write barrier；
- 设备层：SSD 要把自己的 DRAM 缓存刷到 NAND。企业级 SSD 常有 **PLP（掉电保护）**，掉电时用电容把缓存刷入 NAND；消费级 SSD 不一定有。

所以结论：
- CSAPP 里的 **write-back** 是 CPU cache 的 lazy 写回。
- SSD/文件系统也可以 lazy，但如果你要“突然断电也不丢”，必须在关键点 `fsync`/flush，把数据从 page cache、设备缓存推到非易失介质。  
- 如果要求每次写返回前都持久，那就不能只靠 write-back，得用同步写/同步屏障，性能会差很多。