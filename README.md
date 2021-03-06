# dbx: A database library for MySQL/SQLite/Cassandra/Scylladb by golang.

What is dbx? 

> **dbx = DB + Cache**

It is a golang database library that supports transparent caching of all table data. When the memory is large enough, caching services such as Memcached and Redis are no longer needed.

Moreover, the speed of reading cache is quite fast, and the local test QPS reaches 3.5 million + / seconds, which can effectively simplify the application-side business logic code.

It supports MySQL/Sqlite3 and free nesting of structures.

Its implementation principle is to automatically scan the table structure, determine the primary key and self-adding column, and cache data according to the row by the primary key, manage the cache transparently according to the row, the upper layer only needs to operate according to the ordinary ORM style API.



# Supporting caching, high performance read KV cached full table data
After a simple test (with small data), Sqlite3 can be queried directly at a speed of 3w+/s, and 350 w+/s after opening the cache is much faster than Redis (because it has no network IO).
It then supports caching (generally for small tables, ensuring that memory can be opened when it can be put down)
```golang
db.Bind("user", &User{}, true)
db.Bind("group", &Group{}, true)
db.EnableCache(true)
```


# Support nesting, avoid inefficient reflection
Golang is a static language. Reflections are often used to implement more complex functions, but they slow down severely when they are not used properly. Practice has found that we should try our best to use digital index instead of string index, such as Field () performance is about 50 times that of FieldByName ().
Most DB libraries do not support nesting because reflection is slow and complex, especially when there are too many nested layers. Fortunately, through hard work, DBX effectively implements the nesting support of unlimited layers, and the performance is good.

```golang
type Human struct {
	Age int64     `db:"age"`
}
type User struct {
	Human
	Uid        int64     `db:"uid"`
	Gid        int64     `db:"gid"`
	Name       string    `db:"name"`
	CreateDate time.Time `db:"createDate"`
}
```


# API overview
by golang's reflective feature, it can achieve the convenience close to script language level. As follows:
```golang

// Open database
db, err = dbx.Open("mysql", "root@tcp(localhost)/test?parseTime=true&charset=utf8")

// insert one
db.Table("user").Insert(u1)

// find one
db.Table("user").Where("uid=?", 1).One(&user)

// find one by primary key
db.Table("user").WherePK(1).One(u2)

// update one by primary key
db.Table("user").Update(u2)

// delete one by primary key
db.Table("user").WherePK(1).Delete()

// find multi rows
db.Table("user").Where("uid>?", 1).All(&userList)

// find multi rows by IN()
db.Table("user").Where("uid IN(?)", []int{1, 2, 3}).All(&userList)

```

# Log output to specified stream
Free redirection of log data streams.
```golang
// output error information generated by DB to standard output (console)
db.Stderr = os.Stdout

// output the error information generated by DB to the specified file
db.Stderr = dbx.OpenFile("./db_error.log") 

// default: redirect the output of DB (mainly SQL statements) to "black hole" (no information such as executed SQL statements is output)
db.Stdout = ioutil.Discard

// default: Output from DB (mainly SQL statements) to standard output (console)
db.Stdout = os.Stdout
```

# Compatible with native methods
Sometimes we need to call the native interface to achieve more complex purposes.
```golang
// customize complex SQL to get a single result (native)
var uid int64
err = db.QueryRow("SELECT uid FROM user WHERE uid=?", 2).Scan(&uid)
if err != nil {
	panic(err)
}
fmt.Printf("uid: %v\n", uid)
db.Table("user").LoadCache() // Customization requires manual refresh of the cache
```

# Cassandra/Scylladb Use case
```
db, err = dbx.Open("cql", "root@tcp(192.168.0.129:9042)/btc")
dbx.Check(err)
defer db.Close()
more: example/test_cql/main.go
```

# MySQL/SQLite Use case
```golang
package main

import (
	"github.com/xiuno/dbx"
	"fmt"
	"os"
	"time"
)

type User struct {
	Uid        int64     `db:"uid"`
	Gid        int64     `db:"gid"`
	Name       string    `db:"name"`
	CreateDate time.Time `db:"createDate"`
}

func main() {

	var err error
	var db *dbx.DB

	// db, err = dbx.Open("sqlite3", "./db1.db?cache=shared&mode=rwc&parseTime=true&charset=utf8") // sqlite3
	db, err = dbx.Open("mysql", "root@tcp(localhost)/test?parseTime=true&charset=utf8")            // mysql
	dbx.Check(err)
	defer db.Close()

	// output to
	db.Stdout = os.Stdout // output sql to os.Stdout
	db.Stderr = dbx.OpenFile("./db_error.log") // output error to specified file
	// db.Stdout = ioutil.Discard // default: output sql to black hole

	// argument setting:
	db.SetMaxIdleConns(10)
	db.SetMaxOpenConns(10)
	// db.SetConnMaxLifetime(time.Second * 5)

	// create table
	_, err = db.Exec(`DROP TABLE IF EXISTS user;`)
	_, err = db.Exec(`CREATE TABLE user(
		uid        INT(11) PRIMARY KEY AUTO_INCREMENT,
		gid        INT(11) NOT NULL DEFAULT '0',
		name       TEXT             DEFAULT '',
		createDate DATETIME         DEFAULT CURRENT_TIMESTAMP
		);
	`)
	dbx.Check(err)

	// enable cache, optional, usually only for small tables to enable the cache, more than 10W rows, not recommended to open!
	db.Bind("user", &User{}, true)
	db.EnableCache(true)

	// insert one
	u1 := &User{1, 1, "jack", time.Now()}
	_, err = db.Table("user").Insert(u1)
	dbx.Check(err)

	// read one
	u2 := &User{}
	err = db.Table("user").WherePK(1).One(u2)
	dbx.Check(err)
	fmt.Printf("%+v\n", u2)
	
	// read one, check exists
	err = db.Table("user").WherePK(1).One(u2)
	dbx.Check(err)
	if dbx.NoRows(err) {
		panic("not found.")
	}
	fmt.Printf("%+v\n", u2)

	// update one
	u2.Name = "jack.ma"
	_, err = db.Table("user").Update(u2)
	dbx.Check(err)

	// delete one
	_, err = db.Table("user").WherePK(1).Delete()
	dbx.Check(err)
	
	// Where condition + update
	_, err = db.Table("user").WhereM(dbx.M{{"uid", 1}, {"gid", 1}}).UpdateM(dbx.M{{"Name", "jet.li"}})
	dbx.Check(err)

	// insert multi
	for i := int64(0); i < 5; i++ {
		u := &User{
			Uid: i,
			Gid: i,
			Name: fmt.Sprintf("name-%v", i),
			CreateDate: time.Now(),
		}
		_, err := db.Table("user").Insert(u)
		dbx.Check(err)
	}

	// fetch multi
	userList := []*User{}
	err = db.Table("user").Where("uid>?", 1).All(&userList)
	dbx.Check(err)
	for _, u := range userList {
		fmt.Printf("%+v\n", u)
	}

	// multi update
	_, err = db.Table("user").Where("uid>?", 3).UpdateM(dbx.M{{"gid", 10}})
	dbx.Check(err)

	// multi delete
	_, err = db.Table("user").Where("uid>?", 3).Delete()
	dbx.Check(err)

	// count()
	n, err := db.Table("user").Where("uid>?", -1).Count()
	dbx.Check(err)
	fmt.Printf("count: %v\n", n)

	// sum()
	n, err = db.Table("user").Where("uid>?", -1).Sum("uid")
	dbx.Check(err)
	fmt.Printf("sum(uid): %v\n", n)

	// max()
	n, err = db.Table("user").Where("uid>?", -1).Max("uid")
	dbx.Check(err)
	fmt.Printf("max(uid): %v\n", n)

	// min()
	n, err = db.Table("user").Where("uid>?", -1).Min("uid")
	dbx.Check(err)
	fmt.Printf("min(uid): %v\n", n)

	// Customize complex SQL to get a single result (native)
	var uid int64
	err = db.QueryRow("SELECT uid FROM user WHERE uid=?", 2).Scan(&uid)
	dbx.Check(err)
	fmt.Printf("uid: %v\n", uid)
	
	
	
	// db.Table("user").LoadCache() // Customization requires manual refresh of the cache

	// Customize complex SQL to get multiple (native)
	var name string
	rows, err := db.Query("SELECT `uid`, `name` FROM `user` WHERE 1 ORDER BY uid DESC")
	dbx.Check(err)
	rows.Close()
	for rows.Next() {
		rows.Scan(&uid, &name)
		fmt.Printf("uid: %v, name: %v\n", uid, name)
	}
	db.Table("user").LoadCache() // Customization requires manual refresh of the cache

	return
}

```


[中文文档](README_cn.md)
