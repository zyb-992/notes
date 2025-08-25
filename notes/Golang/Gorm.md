# Gorm

> go version: v1.21.6

## MySQL驱动

### 驱动配置

| 参数                | 含义                                 | 备注                                                         |
| ------------------- | ------------------------------------ | ------------------------------------------------------------ |
| `InterpolateParams` | 选择客户端插值，而不是服务器端预处理 | 源码(`github.com\go-sql-driver\mysql@v1.7.0 connection.go 358-401`) |
|                     |                                      |                                                              |
|                     |                                      |                                                              |



### 如何防止SQL注入

> `在MySQL中，驱动程序会使用预处理语句（prepared statements），将参数化查询分为两步：首先发送SQL模板，然后发送参数值。数据库服务器会先解析SQL模板，确定结构，然后再将参数值填入相应的位置。由于SQL结构已经预先确定，参数值无法改变其结构，因此避免了注入攻击。`
>
> `MySQL驱动支持两种参数插值方式：使用预处理语句（服务器端预处理）或客户端插值。在客户端插值的情况下，驱动会在本地将参数替换到查询中，然后将完整的SQL语句发送到服务器`

#### `database/sql`与`go-sql-driver`的实现方式

`database/sql sql.go 1733` 的`queryDC`方法中，先判断是否实现`driver.QueryerContext`

- 如果实现，先通过**客户端插值**的方式预处理参数值并结合SQL子句生成SQL(前提：`go-sql-driver/Config.InterpolateParams = true`)
  - if `InterpolateParams  = false`  -> return `driver.ErrSkip` -> 走(2)的逻辑
  - 数据库服务端处理SQL并返回结果
- (2) 未实现，直接通过服务端预处理的方式分两次命令发送
  - `ctxDriverPrepare`:先发送SQL子句，并返回`*driver.Stmt`类型，其中包含`ID`、`ParamCount`字段
  - `rowsiFromStatement`:再将参数列表发送到服务端，由服务端处理

## 源码

> version: **gorm.io/gorm@v1.25.5**

### Query

---

#### 执行流程

BuildSQL -> ParseTable -> ParseModel -> Generate Schema -> Generate Fields -> Field Relation Mappings -> Call Driver Query -> gorm.Scan(rows)

##### notes

1. 传入的结果集类型为指针，但可以不用初始化(`scan.go 137-155`)，如传入一个map可以写成

   ```
   var set map[string]interface{}
   db.(...).Find(&set)
   ```

2. 在`go/sql`中，如果dest=string, src=string|number|double，会自动将其转化为string类型(`go/sql convert.go 354-361 `)

