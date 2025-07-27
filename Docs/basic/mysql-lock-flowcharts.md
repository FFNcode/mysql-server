# MySQL锁机制流程图详解

## 1. 行锁请求处理流程

```mermaid
flowchart TD
    A[事务请求行锁] --> B{检查索引类型}
    B -->|有主键索引| C[定位到具体记录]
    B -->|有二级索引| D[先锁二级索引,再锁主键]
    B -->|无索引| E[全表扫描,锁定大量记录]
    
    C --> F[确定锁定范围]
    D --> F
    E --> F
    
    F --> G{检查锁冲突}
    G -->|无冲突| H[立即获得锁 GRANTED]
    G -->|有冲突| I[进入等待队列 WAITING]
    
    H --> J[执行SQL操作]
    I --> K[线程睡眠等待]
    K --> L{冲突锁释放?}
    L -->|是| M[重新检查冲突]
    L -->|否| N[继续等待]
    M --> G
    N --> K
    
    J --> O[事务提交/回滚]
    O --> P[释放所有锁]
    P --> Q[唤醒等待线程]
```

## 2. 锁定对象识别流程

```mermaid
flowchart TD
    A[SQL语句] --> B[解析WHERE条件]
    B --> C{使用哪个索引?}
    
    C -->|主键索引| D[page_id + heap_no]
    C -->|唯一索引| E[索引page_id + heap_no<br/>+ 主键page_id + heap_no]
    C -->|普通索引| F[多个索引记录<br/>+ 对应主键记录]
    C -->|无索引| G[表级扫描<br/>大量page_id + heap_no]
    
    D --> H[精确锁定单条记录]
    E --> I[锁定索引记录和主键记录]
    F --> J[锁定范围内所有记录]
    G --> K[可能锁定整个表]
    
    H --> L[确定锁类型]
    I --> L
    J --> L
    K --> L
    
    L --> M{隔离级别}
    M -->|READ COMMITTED| N[记录锁 LOCK_REC_NOT_GAP]
    M -->|REPEATABLE READ| O[Next-Key锁<br/>记录锁+间隙锁]
    M -->|SERIALIZABLE| P[范围锁定]
```

## 3. 三种行锁类型示意图

```mermaid
flowchart LR
    subgraph "数据页中的记录"
        A[记录1] --- B[记录2] --- C[记录3] --- D[记录4]
    end
    
    subgraph "记录锁 Record Lock"
        E[只锁定记录2本身]
        E -.-> B
    end
    
    subgraph "间隙锁 Gap Lock"
        F[锁定记录2和记录3之间的间隙]
        F -.-> G[间隙]
        G --- B
        G --- C
    end
    
    subgraph "Next-Key锁"
        H[锁定记录2 + 记录1到记录2的间隙]
        H -.-> B
        H -.-> I[间隙]
        I --- A
        I --- B
    end
```

## 4. 锁冲突检测流程

```mermaid
flowchart TD
    A[新锁请求] --> B[扫描锁队列]
    B --> C{检查已有锁}
    
    C -->|GRANTED锁| D{锁模式冲突?}
    C -->|WAITING锁| E{锁模式冲突?}
    
    D -->|冲突| F[标记阻塞事务]
    D -->|不冲突| G[继续检查下一个锁]
    
    E -->|冲突| F
    E -->|不冲突| G
    
    F --> H[加入等待队列]
    G --> I{队列检查完毕?}
    
    I -->|否| C
    I -->|是| J[立即获得锁]
    
    H --> K[按CATS权重排序]
    K --> L[线程睡眠]
    
    J --> M[执行操作]
    L --> N[等待唤醒]
```

## 5. 锁释放和唤醒流程

```mermaid
flowchart TD
    A[事务提交/回滚] --> B[释放所有锁]
    B --> C[从GRANTED队列移除]
    C --> D[检查WAITING队列]
    
    D --> E{有等待的锁?}
    E -->|否| F[释放完成]
    E -->|是| G[按CATS权重排序]
    
    G --> H[选择最高权重事务]
    H --> I{检查锁冲突}
    
    I -->|无冲突| J[授予锁 GRANTED]
    I -->|有冲突| K[更新阻塞事务]
    
    J --> L[唤醒对应线程]
    K --> M[检查下一个等待锁]
    
    L --> N[继续检查其他等待锁]
    M --> I
    N --> E
```

## 6. 实际应用场景示例

```mermaid
flowchart TD
    subgraph "场景1: 主键查询"
        A1[SELECT * FROM users WHERE id=1 FOR UPDATE]
        A1 --> A2[锁定: page_id + heap_no of id=1]
        A2 --> A3[锁类型: 记录锁]
    end
    
    subgraph "场景2: 范围查询"
        B1[SELECT * FROM users WHERE age BETWEEN 20 AND 30 FOR UPDATE]
        B1 --> B2[锁定: age索引范围 + 对应主键记录]
        B2 --> B3[锁类型: Next-Key锁]
    end
    
    subgraph "场景3: 无索引查询"
        C1[SELECT * FROM users WHERE name='John' FOR UPDATE]
        C1 --> C2[全表扫描]
        C2 --> C3[锁定: 大量记录的page_id + heap_no]
        C3 --> C4[锁类型: 大范围Next-Key锁]
    end
```

## 7. 行锁核心概念总结

### 行锁锁定的对象
- **page_id**: 数据页标识符
- **heap_no**: 记录在页面内的位置编号
- **不是直接锁定行数据本身**，而是锁定记录的位置信息

### 三种锁类型
1. **记录锁 (Record Lock)**: 只锁定具体记录
2. **间隙锁 (Gap Lock)**: 锁定记录之间的间隙
3. **Next-Key锁**: 记录锁 + 间隙锁的组合

### 锁的生命周期
1. **请求阶段**: 事务发起锁请求
2. **等待阶段**: 遇到冲突时进入等待
3. **获得阶段**: 冲突解除后获得锁
4. **释放阶段**: 事务结束时释放锁

### 性能优化要点
- 使用合适的索引减少锁定范围
- 避免长事务减少锁持有时间
- 合理设置隔离级别平衡一致性和性能
- 理解不同SQL语句的锁定行为