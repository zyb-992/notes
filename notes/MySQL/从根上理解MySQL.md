# 从根上理解MySQL

## 一条记录的存储格式

1. 按列逆序存储，因为和记录的`next_record`字段有关
2. 变长列字段(`varchar`)或`NULL`字段在页记录头会有字段记录它的真实长度
3. 隐藏的记录列字段，如`row_id`、`trx_id`、`rollback_pointer`、`deleted_flag`
   - 被硬删除的字段`deleted_flag`会置为1
   - `row_id`是指隐式主键，如果表设计了主键，那么该字段不会使用到
   - `trx_id`表示最后修改该行记录的事务 ID，用于实现 **MVCC**
   - `next_record`是一个指针，指向数据页中下一条记录的位置，用于连接页中的每条数据（页中的每条数据都是按主键顺序进行连接的

## 数据页结构

一个数据页大概有16KB

1. Page Directory



## 索引

页分裂

页类型