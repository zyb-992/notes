# MySQL笔记



## 子句执行顺序

---

FROM->JOIN->ON->WHERE->GROUP BY->HAVING->SELECT->DISTINCT->ORDER BY->LIMIT

## JOIN

---

- inner join
- left join
- right join
- cross join

## HAVING

---

- 

## GROUP BY

---

- 在5.7以后的版本，默认sql_mode有**ONLY_FULL_GROUP_BY**模式，这导致在SELECT子句检索字段时需要指定检索的字段必须是GROUP子句中的字段或者其他的聚合函数

## UNIQUE

---

- 使用**UNIQUE**可以用于指定单列或者多列唯一，指定后，插入数据时若存在重复(duplicate)会报错，跳过插入

- 使用CONSTRAINT指定约束名称
- NULL值在指定该约束的列中是允许被重复的

## UNION

---

- 用于联结多个SELECT语句集合，但指定的**SELECT**检索的字段的顺序、数量、以及类型必须是一致的 否则会报错
- **UNION**检索结果会去重 **UNION ALL**不会执行去重 会将所有结果都读取出来
- **UNION** vs **JOIN**  =  horizontally vs vertical

## 类型

---

- CHAR
  - 定长
  - 不满足长度的使用空格弥补
  - 字段值后缀存在空格超过限定长度，会截断该空格
  - 超过限定值会插入出错
- VARCHAR
  - 变长
- BOOLEAN

## 索引相关

---

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
