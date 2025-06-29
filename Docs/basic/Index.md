# MySQL索引与数据结构演进

## 目录
1. [MySQL索引概述](#mysql索引概述)
2. [数据结构演进](#数据结构演进)
3. [InnoDB B+树实现](#innodb-b树实现)
4. [Elasticsearch倒排索引](#elasticsearch倒排索引)
5. [性能对比](#性能对比)

## MySQL索引概述

MySQL的索引是数据库性能优化的核心组件，它通过建立额外的数据结构来加速数据的检索。MySQL支持多种索引类型：

### 索引类型
- **主键索引（Primary Key）**：唯一标识每一行数据
- **唯一索引（Unique Index）**：确保列值的唯一性
- **普通索引（Normal Index）**：最基本的索引类型
- **复合索引（Composite Index）**：多列组合的索引
- **全文索引（Fulltext Index）**：用于文本搜索
- **空间索引（Spatial Index）**：用于地理数据

### 存储引擎索引支持
- **InnoDB**：支持B+树索引、全文索引、空间索引
- **MyISAM**：支持B+树索引、R树索引、全文索引
- **Memory**：支持哈希索引、B+树索引

## 数据结构演进

### 1. 二叉搜索树（Binary Search Tree）

二叉搜索树是最基础的树形数据结构：

```
       50
      /  \
     30   70
    /  \  /  \
   20  40 60  80
```

**特点：**
- 左子树的所有节点值 < 根节点值
- 右子树的所有节点值 > 根节点值
- 查找、插入、删除的时间复杂度：O(log n)

**问题：**
- 当数据有序插入时，会退化成链表，时间复杂度变为O(n)
- 树的高度不平衡，影响性能

### 2. 平衡二叉树（AVL树）

为了解决二叉搜索树的不平衡问题，引入了AVL树：

**特点：**
- 任意节点的左右子树高度差不超过1
- 通过旋转操作保持平衡
- 查找性能稳定，但插入删除需要频繁调整

**问题：**
- 节点存储的数据量小，树的高度较高
- 磁盘I/O次数多，不适合数据库存储

### 3. B树（B-Tree）

B树是为磁盘存储设计的平衡树：

**特点：**
- 每个节点可以包含多个键值
- 所有叶子节点在同一层
- 节点分裂和合并保持平衡
- 减少磁盘I/O次数

**结构：**
```
        [10, 20, 30]
       /     |      \
   [5,8]  [15,18]  [25,28,35]
```

### 4. B+树（B+ Tree）

B+树是B树的改进版本，也是InnoDB存储引擎使用的索引结构：

**特点：**
- 非叶子节点只存储键值，不存储数据
- 所有数据都存储在叶子节点
- 叶子节点通过链表连接，便于范围查询
- 树的高度更低，I/O次数更少

**InnoDB B+树结构：**
```
        [10, 20, 30]          (非叶子节点)
       /     |      \
   [5,8]  [15,18]  [25,28,35] (叶子节点)
    ↓       ↓        ↓
  数据页   数据页    数据页
```

## InnoDB B+树实现

### 1. 页面结构

InnoDB将B+树存储在页面中，每个页面大小为16KB：

```cpp
// 页面级别管理
struct Index_details {
  void add_page(const page_no_t page_no, const size_t level) {
    if (level < MAX_LEVEL) {
      m_pages[level].push_back(page_no);
    }
  }
  static const size_t MAX_LEVEL = 20;
  std::vector<page_no_t> m_pages[MAX_LEVEL];
};
```

### 2. 节点指针

非叶子节点存储指向子页面的指针：

```cpp
// 获取子页面号
static inline page_no_t btr_node_ptr_get_child_page_no(const rec_t *rec,
                                                       const ulint *offsets) {
  const byte *field;
  ulint len;
  page_no_t page_no;

  /* 子页面地址在最后一个字段 */
  field = rec_get_nth_field(nullptr, rec, offsets,
                            rec_offs_n_fields(offsets) - 1, &len);

  ut_ad(len == 4);
  page_no = mach_read_from_4(field);
  ut_ad(page_no > 1);

  return (page_no);
}
```

### 3. 叶子节点管理

叶子节点和非叶子节点使用不同的文件段管理：

```cpp
// 叶子页面和非叶子页面分别管理
if (flag == BTR_N_LEAF_PAGES) {
  seg_header = root + PAGE_HEADER + PAGE_BTR_SEG_LEAF;
  fseg_n_reserved_pages(seg_header, &n, mtr);
} else if (flag == BTR_TOTAL_SIZE) {
  seg_header = root + PAGE_HEADER + PAGE_BTR_SEG_TOP;
  n = fseg_n_reserved_pages(seg_header, &dummy, mtr);
  
  seg_header = root + PAGE_HEADER + PAGE_BTR_SEG_LEAF;
  n += fseg_n_reserved_pages(seg_header, &dummy, mtr);
}
```

### 4. 搜索算法

B+树搜索采用二分查找：

```cpp
// 页面内搜索
page_cur_search_with_match(block, index, tuple, page_mode, &up_match,
                           &low_match, page_cursor, nullptr);

// 递归搜索到叶子节点
while (!at_desired_level) {
  block = buf_page_get_gen(page_id, page_size, rw_latch, nullptr, fetch,
                           {file, line}, mtr, mark_dirty);
  
  if (level != height) {
    node_ptr = page_cur_get_rec(page_cursor);
    page_id.reset(space, btr_node_ptr_get_child_page_no(node_ptr, offsets));
  }
}
```

## Elasticsearch倒排索引

### 1. 倒排索引原理

倒排索引是搜索引擎的核心数据结构，与传统的正排索引相反：

**正排索引：** 文档ID → 文档内容
**倒排索引：** 词项 → 文档ID列表

### 2. 基本结构

```
词项词典 (Term Dictionary):
"mysql" → [1, 3, 5, 8]
"database" → [1, 2, 4, 6]
"index" → [1, 3, 7, 9]

倒排列表 (Posting List):
"mysql": [1, 3, 5, 8]
  - 文档1: 位置[10, 25, 50]
  - 文档3: 位置[15, 30]
  - 文档5: 位置[5, 20, 45]
  - 文档8: 位置[12, 28]
```

### 3. ES优化技术

#### a) 跳表（Skip List）
```
原始列表: [1, 3, 5, 8, 10, 15, 20, 25, 30]
跳表结构:
L3: [1, 15, 30]
L2: [1, 5, 15, 25, 30]  
L1: [1, 3, 5, 8, 10, 15, 20, 25, 30]
```

#### b) 位图（Bitmap）
对于高基数字段，使用位图压缩：
```
文档ID: 1  2  3  4  5  6  7  8  9  10
状态位: 1  0  1  0  1  0  1  0  1  0
```

#### c) FST（有限状态转换器）
用于词项词典的压缩存储，减少内存占用。

### 4. 段（Segment）结构

ES将索引分为多个段：
```
索引
├── 段1 (不可变)
│   ├── 倒排索引
│   ├── 正排索引  
│   └── 文档值
├── 段2 (不可变)
└── 段3 (活跃段，可写)
```

## 性能对比

| 特性 | 二叉树 | B树 | B+树 | 倒排索引 |
|------|--------|-----|------|----------|
| 查找复杂度 | O(log n) | O(log n) | O(log n) | O(1) |
| 范围查询 | 差 | 一般 | 优秀 | 优秀 |
| 磁盘I/O | 多 | 中等 | 少 | 中等 |
| 内存占用 | 低 | 中等 | 中等 | 高 |
| 适用场景 | 内存数据 | 数据库索引 | 数据库索引 | 全文搜索 |

## 总结

### 1. MySQL索引演进
- 从简单的二叉树发展到专门为磁盘存储优化的B+树
- 显著提升了数据库性能，特别是范围查询和磁盘I/O效率

### 2. InnoDB B+树优势
- 叶子节点存储完整数据，减少I/O
- 叶子节点链表连接，支持高效范围查询
- 树的高度低，查找路径短
- 支持并发控制（MVCC）

### 3. ES倒排索引优势
- 专为全文搜索设计
- 支持复杂的文本查询和聚合
- 通过多种压缩技术优化存储
- 支持实时搜索和分析

### 4. 选择原则
- **结构化数据查询**：B+树
- **全文搜索**：倒排索引
- **内存数据**：平衡二叉树
- **高并发写入**：LSM树（如RocksDB）

这些数据结构的发展体现了计算机科学中"为特定应用场景优化"的重要原则，每种结构都有其最适合的使用场景。在实际应用中，需要根据具体的业务需求、数据特征和性能要求来选择合适的索引结构。

