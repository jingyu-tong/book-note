- [ORM(Object-Relational Mapping)](#ormobject-relational-mapping)
- [gorm和mysql](#gorm%e5%92%8cmysql)
  - [连接及关闭](#%e8%bf%9e%e6%8e%a5%e5%8f%8a%e5%85%b3%e9%97%ad)
  - [自动迁移](#%e8%87%aa%e5%8a%a8%e8%bf%81%e7%a7%bb)
- [CRUD](#crud)
  - [创建记录](#%e5%88%9b%e5%bb%ba%e8%ae%b0%e5%bd%95)
  - [查询](#%e6%9f%a5%e8%af%a2)
    - [简单查询](#%e7%ae%80%e5%8d%95%e6%9f%a5%e8%af%a2)
    - [where条件查询](#where%e6%9d%a1%e4%bb%b6%e6%9f%a5%e8%af%a2)
    - [高级查询](#%e9%ab%98%e7%ba%a7%e6%9f%a5%e8%af%a2)
  - [更新](#%e6%9b%b4%e6%96%b0)
  - [删除](#%e5%88%a0%e9%99%a4)
    - [删除记录](#%e5%88%a0%e9%99%a4%e8%ae%b0%e5%bd%95)
    - [批量删除](#%e6%89%b9%e9%87%8f%e5%88%a0%e9%99%a4)
    - [软删除](#%e8%bd%af%e5%88%a0%e9%99%a4)
  - [链式操作](#%e9%93%be%e5%bc%8f%e6%93%8d%e4%bd%9c)
# ORM(Object-Relational Mapping)

它的作用是在关系型数据库和业务实体对象之间作一个映射，这样，我们在具体的操作业务对象的时候，就不需要再去和复杂的SQL语句打交道，只需简单的操作对象的属性和方法。

ORM主要是为了解决面向对象与关系数据库存在的互不匹配的现象的技术

# gorm和mysql

## 连接及关闭

```go
db, err = gorm.Open("mysql", "jingyu:123456@(127.0.0.1:3306)/db1?charset=utf8mb4&parseTime=True&loc=Local")
defer db.Close() //退出函数时自动关闭
```

## 自动迁移

自动迁移会在对应的db中，如果已经存在要创建的table，则会验证是否一致
```go
// 自动迁移，在db1中添加user_infos表格，以每个成员为列
db.AutoMigrate(&UserInfo{})
```

# CRUD

GORM 默认会使用名为ID的字段作为表的主键(也可以使用结构体标签自定义)，我们可以自定义Model，也可以匿名嵌套gorm提供的一个简易的model
## 创建记录

```go
u1 := UserInfo{1, "七米", "男", "篮球"}
u2 := UserInfo{2, "沙河娜扎", "女", "足球"}

//利用NewRecord可以判断主键是否存在
db.NewRecord(user) // 主键为空返回`true`

// 创建记录
db.Create(&u1)
db.Create(&u2)
```

通过tag可以设置默认值，需要注意**所有字段的零值, 比如0, "",false或者其它零值，都不会保存到数据库内，但会使用他们的默认值。 如果你想避免这种情况，可以考虑使用指针或实现 Scanner/Valuer接口**

## 查询

### 简单查询

简单的查询如下，还有last等方法没有列出，但大体上类似
```go
// 查询，First返回满足条件的第一个信息
// 类似db.QueryRow
var u = new(UserInfo)
db.First(u) 
fmt.Printf("%#v\n", u)

//Find根据给定的字符串条件进行查询
//如果有多个，可以将第一个参数设为slice
var uu UserInfo
db.Find(&uu, "hobby=?", "足球")
fmt.Printf("%#v\n", uu)
```

### where条件查询

还可以使用where条件对查询进行限制
```go
// Get first matched record
db.Where("name = ?", "jinzhu").First(&user)
//// SELECT * FROM users WHERE name = 'jinzhu' limit 1;

// Get all matched records
db.Where("name = ?", "jinzhu").Find(&users)
//// SELECT * FROM users WHERE name = 'jinzhu';

// <>
db.Where("name <> ?", "jinzhu").Find(&users)
//// SELECT * FROM users WHERE name <> 'jinzhu';

// IN
db.Where("name IN (?)", []string{"jinzhu", "jinzhu 2"}).Find(&users)
//// SELECT * FROM users WHERE name in ('jinzhu','jinzhu 2');

// LIKE
db.Where("name LIKE ?", "%jin%").Find(&users)
//// SELECT * FROM users WHERE name LIKE '%jin%';

// AND
db.Where("name = ? AND age >= ?", "jinzhu", "22").Find(&users)
//// SELECT * FROM users WHERE name = 'jinzhu' AND age >= 22;

// Time
db.Where("updated_at > ?", lastWeek).Find(&users)
//// SELECT * FROM users WHERE updated_at > '2000-01-01 00:00:00';

// BETWEEN
db.Where("created_at BETWEEN ? AND ?", lastWeek, today).Find(&users)
//// SELECT * FROM users WHERE created_at BETWEEN '2000-01-01 00:00:00' AND '2000-01-08 00:00:00';
```

此外，还可以通过struct或Map进行查询(零值不会作为查询条件)
```go
// Struct
db.Where(&User{Name: "jinzhu", Age: 20}).First(&user)
//// SELECT * FROM users WHERE name = "jinzhu" AND age = 20 LIMIT 1;

// Map
db.Where(map[string]interface{}{"name": "jinzhu", "age": 20}).Find(&users)
//// SELECT * FROM users WHERE name = "jinzhu" AND age = 20;

// 主键的切片
db.Where([]int64{20, 21, 22}).Find(&users)
//// SELECT * FROM users WHERE id IN (20, 21, 22);
```

### 高级查询

基于QueryExpr()方法的子查询
```go
db.Where("amount > ?", DB.Table("orders").Select("AVG(amount)").Where("state = ?", "paid").QueryExpr()).Find(&orders)
// SELECT * FROM "orders"  WHERE "orders"."deleted_at" IS NULL AND (amount > (SELECT AVG(amount) FROM "orders"  WHERE (state = 'paid')));
```

选择字段，前面提及的都是`select *`，也就是选择出所有字段，gorm提供了字段选择的方法
```go
db.Select("name, age").Find(&users)
//// SELECT name, age FROM users;

db.Select([]string{"name", "age"}).Find(&users)
//// SELECT name, age FROM users;

db.Table("users").Select("COALESCE(age,?)", 42).Rows()
//// SELECT COALESCE(age,'42') FROM users;
```

排序，第二个参数为是否覆盖
```go
db.Order("age desc, name").Find(&users)
//// SELECT * FROM users ORDER BY age desc, name;

// 多字段排序
db.Order("age desc").Order("name").Find(&users)
//// SELECT * FROM users ORDER BY age desc, name;

// 覆盖排序
db.Order("age desc").Find(&users1).Order("age", true).Find(&users2)
//// SELECT * FROM users ORDER BY age desc; (users1)
//// SELECT * FROM users ORDER BY age; (users2)
```

同样，可以限定数量和偏移
```go
db.Limit(3).Find(&users)
//// SELECT * FROM users LIMIT 3;

// -1 取消 Limit 条件
db.Limit(10).Find(&users1).Limit(-1).Find(&users2)
//// SELECT * FROM users LIMIT 10; (users1)
//// SELECT * FROM users; (users2)

db.Offset(3).Find(&users)
//// SELECT * FROM users OFFSET 3;

// -1 取消 Offset 条件
db.Offset(10).Find(&users1).Offset(-1).Find(&users2)
//// SELECT * FROM users OFFSET 10; (users1)
//// SELECT * FROM users; (users2)
```

## 更新

Save()默认会更新该对象的所有字段，即使你没有赋值,如果只想更新指定的字段，则用update方法


```go
//更新所有字段
db.Save(&user)

// 更新，根据u的主键更新，如果为空，则会更新所有列
// 另有updates可以更新多列
db.Model(&u).Update("hobby", "双色球")
```

## 删除

### 删除记录

**警告**删除记录时，请确保主键字段有值，GORM 会通过主键去删除记录，如果主键为空，GORM 会删除该 model 的所有记录。
```go
// 删除现有记录
db.Delete(&email)
```

### 批量删除

删除所有匹配的记录
```go
db.Where("email LIKE ?", "%jinzhu%").Delete(Email{})
//// DELETE from emails where email LIKE "%jinzhu%";

db.Delete(Email{}, "email LIKE ?", "%jinzhu%")
//// DELETE from emails where email LIKE "%jinzhu%";
```

### 软删除

如果一个 model 有 DeletedAt 字段(gorm.Model自带)，他将自动获得软删除的功能！ 当调用 Delete 方法时， 记录不会真正的从数据库中被删除， 只会将DeletedAt 字段的值会被设置为当前时间

想要进行物理删除，可以如下
```go
db.Unscoped().Delete(&order)
```

## 链式操作

Method Chaining，Gorm 实现了链式操作接口，在我们调用具体的方法前，不会生产query语句，因此可以借助这个特性处理一些通用的逻辑

立即执行方法
* 那些会立即生成SQL语句并发送到数据库的方法, 他们一般是CRUD方法，比如：Create, First, Find, Take, Save, UpdateXXX, Delete, Scan, Row, Rows…  
* 在 GORM 中使用多个立即执行方法时，后一个立即执行方法会复用前一个立即执行方法的条件 (不包括内联条件) 




