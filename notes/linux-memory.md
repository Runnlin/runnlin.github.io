# Linux 内存管理基础

Linux 内核的内存管理系统负责虚拟内存、物理内存分配、页面置换等核心功能，是系统编程的重要基础知识。

## 虚拟地址空间布局（x86-64）

```
0xFFFFFFFF FFFFFFFF  ← 内核空间（高 128 TiB）
      ...
0xFFFF8000 00000000  ← 内核空间起始
      ...
0x00007FFF FFFFFFFF  ← 用户空间上边界
      ...               栈（向下增长，默认 8 MB）
      ...               共享库 / mmap 区域
      ...               堆（向上增长，brk/mmap）
0x00000000 00400000  ← 代码段（text）
0x00000000 00000000
```

## 物理内存管理

### 伙伴系统（Buddy System）

内核使用伙伴系统管理物理页框，以 2 的幂次方为单位：

```c
// 分配 2^order 个连续物理页
struct page *page = alloc_pages(GFP_KERNEL, order);

// 分配单页
struct page *page = alloc_page(GFP_KERNEL);

// 释放
__free_pages(page, order);
```

### Slab 分配器

用于频繁分配/释放的小对象，减少碎片：

```c
// 创建缓存
struct kmem_cache *cache = kmem_cache_create(
    "myobj_cache",       // 名称
    sizeof(struct myobj), // 对象大小
    0,                   // 对齐
    SLAB_HWCACHE_ALIGN,  // 标志
    NULL                 // 构造函数
);

// 分配对象
struct myobj *obj = kmem_cache_alloc(cache, GFP_KERNEL);

// 释放对象
kmem_cache_free(cache, obj);

// 销毁缓存
kmem_cache_destroy(cache);
```

## 常用 GFP 标志

| 标志 | 含义 |
|------|------|
| `GFP_KERNEL` | 普通内核分配，可能睡眠 |
| `GFP_ATOMIC` | 原子上下文，不可睡眠 |
| `GFP_DMA` | 要求 DMA 区域内存 |
| `GFP_HIGHMEM` | 允许高端内存 |

## 诊断命令

```bash
free -h                   # 查看内存使用概览
cat /proc/meminfo         # 详细内存信息
cat /proc/buddyinfo       # 伙伴系统状态
cat /proc/slabinfo        # Slab 分配器状态
vmstat -s                 # 虚拟内存统计
valgrind --leak-check=full ./app   # 检查内存泄漏
```
