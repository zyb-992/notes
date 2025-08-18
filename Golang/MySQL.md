# MySQL笔记



## 子句执行顺序

FROM->JOIN->ON->WHERE->GROUP BY->HAVING->SELECT->DISTINCT->ORDER BY->LIMIT

## JOIN

- inner join
- left join
- right join
- cross join

## HAVING

## GROUP BY

- 在5.7以后的版本，默认sql_mode有**ONLY_FULL_GROUP_BY**模式，这导致在SELECT子句检索字段时需要指定检索的字段必须是GROUP子句中的字段或者其他的聚合函数

## UNIQUE

- 使用**UNIQUE**可以用于指定单列或者多列唯一，指定后，插入数据时若存在重复(duplicate)会报错，跳过插入

- 使用CONSTRAINT指定约束名称
- NULL值在指定该约束的列中是允许被重复的

## UNION

- 用于联结多个SELECT语句集合，但指定的**SELECT**检索的字段的顺序、数量、以及类型必须是一致的 否则会报错
- **UNION**检索结果会去重 **UNION ALL**不会执行去重 会将所有结果都读取出来
- **UNION** vs **JOIN**  =  horizontally vs vertical

## 窗口函数

> https://www.mysqltutorial.org/mysql-window-functions/

#### 子句

- `PARTITION BY`: `PARTITION BY <expression>[{,<expression>...}]`
- `ORDER BY`: `ORDER BY <expression> [ASC|DESC], [{,<expression>...}]`
- `ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING`:
- `ROWS UNBOUNDED PRECEDING`:

#### 语法

```
window_function_name(expression) OVER ( 
   [partition_defintion]
   [order_definition]
   [frame_definition]
)
```



## 聚合函数

- ROW_NUMBER：为结果集中的每一行分配一个**唯一的连续序号**
- 

## 分区

#### 类型

1. `RANGE分区`

   ```sql
   CREATE TABLE user_data (
       user_id INT NOT NULL,
       data TEXT,
       PRIMARY KEY (user_id)
   )
   PARTITION BY RANGE (user_id) (
       PARTITION p0 VALUES LESS THAN (100000),
       PARTITION p1 VALUES LESS THAN (200000),
       PARTITION p2 VALUES LESS THAN (300000),
       PARTITION p3 VALUES LESS THAN MAXVALUE
   );
   ```

2. `LIST分区`

   ```sql
   CREATE TABLE sales (
       id INT NOT NULL,
       region VARCHAR(20) NOT NULL,
       sales_amount DECIMAL(10,2),
       sale_date DATE,
       PRIMARY KEY (id, region)
   )
   PARTITION BY LIST COLUMNS(region) (
       PARTITION p_north VALUES IN ('北京', '天津', '河北', '山西'),
       PARTITION p_east VALUES IN ('上海', '江苏', '浙江', '安徽'),
       PARTITION p_south VALUES IN ('广东', '广西', '海南', '深圳'),
       PARTITION p_west VALUES IN ('重庆', '四川', '贵州', '云南')
   );
   ```

3. `HASH分区`

   ​	 **LINEAR HASH**：只需要移动少量数据

   ```sql
   CREATE TABLE user_sessions (
       session_id VARCHAR(64) NOT NULL,
       user_id INT NOT NULL,
       login_time TIMESTAMP,
       data JSON,
       PRIMARY KEY (session_id, user_id)
   )
   PARTITION BY HASH (user_id)
   PARTITIONS 8;  -- 创建8个分区
   ```

4. `KEY分区` （MySQL自动选择复合函数）

   ```sql
   CREATE TABLE user_messages (
       user_id INT NOT NULL,
       message_id BIGINT NOT NULL,
       content TEXT,
       created_at TIMESTAMP,
       PRIMARY KEY (user_id, message_id)
   )
   PARTITION BY KEY (user_id, message_id)
   PARTITIONS 12;
   ```

   

## 类型

- CHAR
  - 定长
  - 不满足长度的使用空格弥补
  - 字段值后缀存在空格超过限定长度，会截断该空格
  - 超过限定值会插入出错
- VARCHAR
  - 变长
- BOOLEAN

## 索引相关

```mysql
# 
show index from table 'table_name'

# 
drop index 'index_name' on table

# 
alter table 'table_name' drop index 'index_name'

# 添加约束
alter table 'table_name' add constraint 'constraint_name' unique (column list)
```

## 问题

1. **深分页问题**
   1. **OFFSET机制缺陷**：必须扫描并丢弃前面的所有记录
   2. **线性性能退化**：页码越大，查询越慢
   3. **资源浪费**：大量CPU和内存用于处理不需要的数据
2. 
