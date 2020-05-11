# 简述

go没有直接定义支持数据库的驱动，而是定义了数据库相关的接口，在database/sql中

这样的好处是只要是按照标准接口开发的代码， 以后需要迁移数据库时，不需要任何修改

这里首先了解下接口，然后再了解链接数据库的第三方包

# 接口

## sql.Register

`func Register(name string, driver driver.Driver)`  
首先是注册函数，通过Register，构建了一个map，将name映射到各自的driver中

## driver.Driver

```go
type Driver interface {
    Open(name string) (Conn, error)
}
```

Driver是一个定义了open的接口，会返回一个数据库的Conn链接(这个链接只能用于一个goroutine，否则会乱套)

## driver.Conn

```go
type Conn interface {
    Prepare(query string) (Stmt, error)
    Close() error
    Begin() (Tx, error)
}
```
Conn是一个接口，定义了三个函数，Prepare函数返回与当前连接相关的执行 Sql 语句的准备状态，可以进行查询、删除等操作。

Close 函数关闭当前的连接，执行释放连接拥有的资源等清理工作。因为驱动实现了 database/sql 里面建议的 conn pool，所以你不用再去实现缓存 conn 之类的，这样会容易引起问题。

Begin 函数返回一个代表事务处理的 Tx，通过它你可以进行查询，更新等操作，或者对事务进行回滚、递交。

## driver.Stmt

Stmt对应一个准备好的Conn，同样只能用于一个goroutine中

```go
type Stmt interface {
    Close() error
    NumInput() int
    Exec(args []Value) (Result, error)
    Query(args []Value) (Rows, error)
}
```
Close 函数关闭当前的链接状态，但是如果当前正在执行 query，query 还是有效返回 rows 数据。

NumInput 函数返回当前预留参数的个数，当返回 >=0 时数据库驱动就会智能检查调用者的参数。当数据库驱动包不知道预留参数的时候，返回 -1。

Exec 函数执行 Prepare 准备好的 sql，传入参数执行 update/insert 等操作，返回 Result 数据

Query 函数执行 Prepare 准备好的 sql，传入需要的参数执行 select 操作，返回 Rows 结果集

## driver.Result和driver.Rows

Result是Exec返回的值，是update等操作对应的接口

```go
type Result interface {
    LastInsertId() (int64, error)
    RowsAffected() (int64, error)
}
```
LastInsertId 函数返回由数据库执行插入操作得到的自增 ID 号。

RowsAffected 函数返回 query 操作影响的数据条目数。

Rows 是执行查询返回的结果集接口定义

```go
type Rows interface {
    Columns() []string
    Close() error
    Next(dest []Value) error
}
```
Columns 函数返回查询数据库表的字段信息，这个返回的 slice 和 sql 查询的字段一一对应，而不是返回整个表的所有字段。  
Close 函数用来关闭 Rows 迭代器。  
Next 函数用来返回下一条数据，把数据赋值给 dest。dest 里面的元素必须是 driver.Value 的值除了 string，返回的数据里面所有的 string 都必须要转换成 [] byte。如果最后没数据了，Next 函数最后返回 io.EOF。

## driver.Value

Value是一个空接口，可以容纳任意类型的数据，但是drive 的 Value 是驱动必须能够操作的 Value，Value 要么是 nil，要么是下面的任意一种

```go
int64
float64
bool
[]byte
string   [*]除了Rows.Next 返回的不能是 string.
time.Time
```

## sql.DB

database/sql 在 database/sql/driver 提供的接口基础上定义了一些更高阶的方法，用以简化数据库操作，同时内部还建议性地实现一个 conn pool。

```go
type DB struct {
    driver   driver.Driver
    dsn      string
    mu       sync.Mutex // protects freeConn and closed
    freeConn []driver.Conn
    closed   bool
}
```

实现上其实也很简陋，就是在freeConn中查询是否有可以复用的链接，有的话取出来复用

# mysql驱动

这里采用的是[mysql驱动](https://github.com/go-sql-driver/mysql)

首先，我们需要导入接口，以及驱动的包，如下
```go
import (
    "database/sql"
    "fmt"
    // "time"

    _ "github.com/go-sql-driver/mysql" //这里的init函数，会调用Register注册mysql
)
```

然后调用sql.Open()链接数据库，注意，这里只检查参数，并不会真正链接，返回的就是DB的实例，这个是带有连接池的
`db, err := sql.Open("mysql", "astaxie:astaxie@/test?charset=utf8")`

然后就可以进行CRUD了

## 插入、更新、删除(insert/update/delete)

根据我们之前所说对于insert/update等操作，需要先调用Prepare，获得准备好的stmt状态，然后调用Exec执行，返回Result，代码如下
```go
// 插入数据
stmt, err := db.Prepare("INSERT userinfo SET username=?,department=?,created=?") //利用？占位符，后续Exec需要填上

res, err := stmt.Exec("astaxie", "研发部门", "2012-12-09")
checkErr(err)

id, err := res.LastInsertId()

// 更新数据
stmt, err = db.Prepare("update userinfo set username=? where uid=?")

res, err = stmt.Exec("astaxieupdate", id)

affect, err := res.RowsAffected()

 // 删除数据
stmt, err = db.Prepare("delete from userinfo where uid=?")

res, err = stmt.Exec(id)

affect, err = res.RowsAffected()
```

## 查询

跟之前需要更改表单的内容不同，select不需要预先prepare，遍历查询结果代码如下
```go
 // 查询数据
rows, err := db.Query("SELECT * FROM userinfo")

// 非常重要：关闭rows释放持有的数据库链接
defer rows.Close()

for rows.Next() {
    var uid int
    var username string
    var department string
    var created string
    err = rows.Scan(&uid, &username, &department, &created)
    fmt.Println(uid)
    fmt.Println(username)
    fmt.Println(department)
    fmt.Println(created)
}

//还有只返回一行(Row)的db.QueryRow等操作
//Row类型，Scan内部会自己关闭链接，不需要再额外Close了，但是Rows类型必须要显示的Close，来释放连接池的资源
```









