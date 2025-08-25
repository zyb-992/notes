# Redis

## 数据类型

- 哈希表
  - redis中所有的key都存储在同一个哈希表中，只不过key对应的value是一个指针，指向对应值，每个值的数据结构不唯一
  - **渐进式rehash**：当哈希表需要扩容时，redis中所有的key都需要重新进行hash函数得到散列值，然后再放入相应的桶中，渐进式rehash可以将全部key的rehash分散到各个请求当中进行处理。

## 底层数据结构

- **跳跃表**
  - 类似于红黑树
  - 时间复杂度 O(logN)
- List
- Set
- Sorted Set
  - 由一个关联性的score进行排序，相同的score由键的字典顺序排列
  - 使用场景
    - 限流器
    - 排行榜
- Stream
  - 使用场景
  - 结构：仅追加数据
  - 命令
    - XADD : 追加数据
    - XRANGE ：
    - XRANGE - + COUNT count 
    - XREAD COUNT count STREAMS stream ：
- Bitmap

## 持久化

- **AOF**
  - 原理；每条写命令执行后都会将该命令写入AOF日志中
  - 写机制
    - Always：写命令执行后主线程同步写日志，保持数据一致性，牺牲了高性能
    - Every second：一秒后将内存中的AOF日志flush到磁盘，保持高性能，牺牲数据一致性
    - No ：由操作系统控制内存中的数据持久化缓冲区flush到磁盘
- **RDB**
  - 原理：根据redis当前状态生成快照，在生成快照期间不允许有其他写命令操作
  - 命令：
    - SAVE
      - 
    - BGSAVE



## 数据库

### 设置生存时间（TTL）

```bash
# 秒级 设置key 5s后过期
EXPIRE key 5

# 毫秒级 设置key 500ms后过期
PEXPIRE key 500

# 设置key(覆盖) 并设置5s后过期
SETEX key value 5

# 设置key(如果数据库中不存在，存在则返回错误) 并设置5s后过期
SETNX key value 5

# 获取key的生存时间
TTL key
```

### 设置过期时间（unix时间戳）

1. 命令可以设置key在哪一时刻过期

   ```bash
   # 
   EXPIREAT key timestamp
   
   # 
   PEXPIREAT key timestamp
   ```

2. 移除过期时间

   ```bash
   # 持久化key
   PERSIST key
   ```

### 过期策略
-  TTL/PTTL都是通过计算键的过期时间与当前时间的差值来实现

-  数据库结构中的expires字典存储了数据库中所有键的过期时间，映射关系为键对象->该键的过期时间(ms级别的时间戳)

-  判定键是否过期

   -  键是否存在于expires字典，存在则有过期时间，否则返回-1
   -  根据当前时间与该键的过期时间检查键是否已过期

-  键过期后的删除策略

   -  主动删除
      -  **定时删除**：设置键过期同时创建定时器timer，执行过期时立即删除
      -  **定期删除**：
   -  被动删除
      -  **惰性删除**：当客户端获取键时检查过期时间，若过期了再从数据库中删除该键

   

### 操作数据库

  Redis中有0-15号数据库可以选用，若是集群节点只允许使用0号数据库，单机可以随意使用

```bash
# 使用某个数据库
SELECT 0-15
```



## 集群

### 集群配置

### 槽

## 