> **GORM = 用 Go 结构体操作关系型数据库（MySQL / PostgreSQL / SQLite / SQL Server）**

1. 约定优于配置

   ```go
   type User struct {
       ID   uint
       Name string
   }
   ```

   表名：`users`

   主键：`id`

   字段名：`name`

   自增主键

2. 结构体即 Schema

   GORM 的 Schema 来自 **Go struct + tag**

   ```go
   type Order struct {
       ID        uint      `gorm:"primaryKey"`
       UserID    uint      `gorm:"index"`
       Amount    float64
       CreatedAt time.Time
   }
   ```

## GORM 的核心对象

###  `*gorm.DB`: 一切操作的入口

```go
db := db.Where("status = ?", 1)
```

`gorm.DB` 是 **不可变对象**

## CRUD 基本操作

### Create

```
db.Create(&user)
```

批量：

```
db.Create(&users)
```

------

###  Read

```
db.First(&user, 1)        // by primary key
db.Where("name = ?", "Tom").Find(&users)
```

### Update

```
db.Model(&user).Update("name", "Jack")
```

批量更新（⚠️ 默认不允许全表）：

```
db.Model(&User{}).
   Where("age > ?", 18).
   Updates(map[string]interface{}{
       "status": 1,
   })
```

------

###  Delete（软删除）

```
db.Delete(&user)
```