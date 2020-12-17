# Golang牵手PostgreSQL"增删改查+不写结构快速扫描字段"快速入门



![What is PostgreSQL](/Users/xb/Downloads/pic/postgresql/What-is-PostgreSQL.png)

## 简介

PostgreSQL(也称postgres)是一款强大的开源对象关系数据库系统(ORDBMS), 历经30年以上的打磨, 具有高可靠性, 强健壮性, 高性能等优点.  详见[官网](https://www.postgresql.org/).

本文主要使用的`github.com/lib/pq`包, 它是一款为Go语言`database/sql`包(sql包是go定义的一套围绕SQL或类似SQL数据库的通用接口, 需要结合具体数据库的驱动一起使用)定制, 纯Go开发的PostgreSQL驱动.

什么是不写结构体?

本文中可以理解为, 查询数据库配置直接返回键值对类型, 或者查询数据返回多行数据, 不需要针对列值申明字段, 分别采用Map和Map数组来接收单行, 和多行数据. 详细原理请参考之前的博文:[Golang连接MySQL执行查询并解析-告别结构体](https://mp.weixin.qq.com/s/3Nb3GFsfaxGKNrWaW2JY5w)

接下来, 咱们一起来实现Golang版本的增删改查(*CRUD*)与单行,多行字段快速扫描解析吧!



## 驱动安装

执行以下命令安装postgresql驱动

```
go get github.com/lib/pq
```



## pq包支持的功能

- SSL
- 作为驱动程序, 与`database/sql`结合处理连接相关操作
- 扫描时间`time.Time`类型, 如 `timestamp[tz]`, `time[tz]`, `date `
- 扫描二进制blobs对象(Blob是内存中的数据缓冲, 用来匹配strign类型的ID和字节切片), 如: `bytea`
- PostgreSQL的`hstore`数据类型支持
- 支持`COPY FROM`
- pq.ParseURL方法用来将urls转化为sql.Open的连接字符串
- 支持许多libpq库兼容的环境变量
- 支持Unix socket
- 支持通知Notifications, 如:`LISTEN`/`NOTIFY`
- 支持`pgpass`
- GSS (Kerberos) 认证



## 连接字符串参数

pq包与`libpq`类似([libpq](https://www.postgresql.org/docs/current/libpq.html)是用C编写的底层接口, 为其他高级语言比如C++,Perl,Python,Tcl和ECPG等提供底层PostgreSQL支持). 建立连接时需要提供连接参数, 一部分支持libpq的参数也支持pq, 额外的, pq允许在连接字符串中指定运行时参数(如:search_path或work_mem), libpq则不能在连接字符串中指定运行时参数, 只能在option参数中指定.

pq包为了兼容`libpq包`, 下面的连接参数都支持

```
* dbname - 需要连接的数据库名
* user - 需要使用的用户名
* password - 该用户的密码
* host - 需要连接的postgresql主机, unix域名套接字以/开始, 默认是localhost
* port - postgresql绑定的端口 (默认5432)
* sslmode - 是否使用SSL (默认是启用(require), libpq包默认不启用SSL)
* fallback_application_name - 失败时,可以提供一个应用程序名来跟踪.
* connect_timeout - 连接最大等待秒数, 0或者不指定, 表示不确定时间的等待
* sslcert - 证书文件位置, 文件中必须包含PEM编码的数据
* sslkey - 密钥文件位置, 文件中必须包含PEM编码的数据
* sslrootcert - 根证书文件位置, 文件中必须包含PEM编码的数据
```

sslmode 支持一下模式

```
* disable - 禁用SSL
* require - 总是使用SSL(跳过验证)
* verify-ca - 总是使用SSL (验证服务器提供的证书是由可信的CA签署的)
* verify-full - 总是使用SSL(验证服务器提供的证书是由受信任的CA签署的，并验证服务器主机名是否与证书中的主机名匹配)
```

更多连接字符串参数请参考[官方文档](http://www.postgresql.org/docs/current/static/libpq-connect.html#LIBPQ-CONNSTRING )

对包含空格的参数, 需要使用单引号, 如:

```
"user=pqgotest password='with spaces'"
```

使用反斜杠进行转义, 如:

```
"user=space\ man password='it\'s valid'"
```

注意: 如果要设置client_encoding连接参数(用于设置连接的编码), 必须设置为"UTF8", 才能与Postgres匹配, 设置为其他值将会报错.

除了上面的参数, 在连接字符串中也可以通过后台设置运行时参数, 详细运行时参数, 请参考[runtime-config](http://www.postgresql.org/docs/current/static/runtime-config.html.)

支持`libpq`的大部分环境变量也支持pq包, 详细环境变量请参考[libpq-envars]( http://www.postgresql.org/docs/current/static/libpq-envars.html). 如果没有设置环境变量且连接字符串也没有提供该参数, 程序会panic崩溃退出, 字符串参数优先级高于环境变量.



## 完整"增删改查"示例代码

```go
package main

import (
	"database/sql"
	"encoding/json"
	"fmt"
	_ "github.com/lib/pq"
	"log"
)

const (
	// Initialize connection constants.
	HOST     = "172.16.xx.xx"
	PORT     = 31976
	DATABASE = "postgres"
	USER     = "postgres"
	PASSWORD = "xxxxx"
)

func checkError(err error) {
	if err != nil {
		panic(err)
	}
}

type Db struct {
	db *sql.DB
}

// 创建表
func (this *Db) CreateTable() {
	// 以水果库存清单表inventory为例
	// Drop previous table of same name if one exists.  如果之前存在清单表, 则删除该表
	_, err := this.db.Exec("DROP TABLE IF EXISTS inventory;")
	checkError(err)
	fmt.Println("Finished dropping table (if existed)")

	// Create table. 创建表, 指定id, name, quantity(数量)字段, 其中id为主键
	_, err = this.db.Exec("CREATE TABLE inventory (id serial PRIMARY KEY, name VARCHAR(50), quantity INTEGER);")
	checkError(err)
	fmt.Println("Finished creating table")
}

// 删除表
func (this *Db) DropTable() {
	// 以水果库存清单表inventory为例
	// Drop previous table of same name if one exists.  如果之前存在清单表, 则删除该表
	_, err := this.db.Exec("DROP TABLE IF EXISTS inventory;")
	checkError(err)
	fmt.Println("Finished dropping table (if existed)")
}

// 增加数据
func (this *Db) Insert() {
	// Insert some data into table. 插入3条水果数据
	sql_statement := "INSERT INTO inventory (name, quantity) VALUES ($1, $2);"
	_, err := this.db.Exec(sql_statement, "banana", 150)
	checkError(err)
	_, err = this.db.Exec(sql_statement, "orange", 154)
	checkError(err)
	_, err = this.db.Exec(sql_statement, "apple", 100)
	checkError(err)
	fmt.Println("Inserted 3 rows of data")
}

// 读数据/查数据
func (this *Db) Read() {
	//读取数据
	// Read rows from table.
	var id int
	var name string
	var quantity int

	sql_statement := "SELECT * from inventory;"
	rows, err := this.db.Query(sql_statement)
	checkError(err)
	defer rows.Close()

	for rows.Next() {
		switch err := rows.Scan(&id, &name, &quantity); err {
		case sql.ErrNoRows:
			fmt.Println("No rows were returned")
		case nil:
			fmt.Printf("Data row = (%d, %s, %d)\n", id, name, quantity)
		default:
			checkError(err)
		}
	}
}


// 更新数据
func (this *Db) Update() {
	// Modify some data in table.
	sql_statement := "UPDATE inventory SET quantity = $2 WHERE name = $1;"
	_, err := this.db.Exec(sql_statement, "banana", 200)
	checkError(err)
	fmt.Println("Updated 1 row of data")
}

// 删除数据
func (this *Db) Delete() {
	// Delete some data from table.
	sql_statement := "DELETE FROM inventory WHERE name = $1;"
	_, err := this.db.Exec(sql_statement, "orange")
	checkError(err)
	fmt.Println("Deleted 1 row of data")
}

// 数据序列化为Json字符串, 便于人工查看
func Data2Json(anyData interface{}) string {
	JsonByte, err := json.Marshal(anyData)
	if err != nil {
		log.Printf("数据序列化为json出错:\n%s\n", err.Error())
		return ""
	}
	return string(JsonByte)
}


//多行数据解析
func QueryAndParseRows(Db *sql.DB, queryStr string) []map[string]string {
	rows, err := Db.Query(queryStr)
	defer rows.Close()
	if err != nil {
		log.Printf("查询出错:\nSQL:\n%s, 错误详情\n", queryStr, err.Error())
		return nil
	}
	cols, _ := rows.Columns() //列名
	if len(cols) > 0 {
		var ret []map[string]string //定义返回的映射切片变量ret
		for rows.Next() {
			buff := make([]interface{}, len(cols))
			data := make([][]byte, len(cols)) //数据库中的NULL值可以扫描到字节中
			for i, _ := range buff {
				buff[i] = &data[i]
			}
			rows.Scan(buff...) //扫描到buff接口中，实际是字符串类型data中

			//将每一行数据存放到数组中
			dataKv := make(map[string]string, len(cols))
			for k, col := range data { //k是index，col是对应的值
				//fmt.Printf("%30s:\t%s\n", cols[k], col)
				dataKv[cols[k]] = string(col)
			}
			ret = append(ret, dataKv)
		}
		log.Printf("返回多元素数组:\n%s", Data2Json(ret))
		return ret
	} else {
		return nil
	}
}

//单行数据解析 查询数据库，解析查询结果，支持动态行数解析
func QueryAndParse(Db *sql.DB, queryStr string) map[string]string {
	rows, err := Db.Query(queryStr)
	defer rows.Close()

	if err != nil {
		log.Printf("查询出错:\nSQL:\n%s, 错误详情\n", queryStr, err.Error())
		return nil
	}
	//rows, _ := Db.Query("SHOW VARIABLES LIKE '%data%'")

	cols, _ := rows.Columns()
	if len(cols) > 0 {
		buff := make([]interface{}, len(cols)) // 临时slice
		data := make([][]byte, len(cols))      // 存数据slice
		dataKv := make(map[string]string, len(cols))
		for i, _ := range buff {
			buff[i] = &data[i]
		}

		for rows.Next() {
			rows.Scan(buff...) // ...是必须的
		}

		for k, col := range data {
			dataKv[cols[k]] = string(col)
			//fmt.Printf("%30s:\t%s\n", cols[k], col)
		}
		log.Printf("返回单行数据Map:\n%s", Data2Json(dataKv))
		return dataKv
	} else {
		return nil
	}
}


func main() {
	// Initialize connection string. 初始化连接字符串, 参数包含主机,端口,用户名,密码,数据库名,SSL模式(禁用),超时时间
	var connectionString string = fmt.Sprintf("host=%s  port=%d user=%s password=%s dbname=%s sslmode=disable connect_timeout=3", HOST, PORT, USER, PASSWORD, DATABASE)

	// Initialize connection object. 初始化连接对象, 驱动名为postgres
	db, err := sql.Open("postgres", connectionString)
	defer db.Close()
	checkError(err)
	postgresDb := Db{
		db: db,
	}
	err = postgresDb.db.Ping() //连通性检查
	checkError(err)
	fmt.Println("Successfully created connection to database")

	postgresDb.CreateTable()         //创建表
	postgresDb.Insert()              //插入数据
	postgresDb.Read()                //查询数据
	QueryAndParseRows(postgresDb.db, "SELECT * from inventory;") //直接查询和解析多行数据
	QueryAndParse(postgresDb.db, "SHOW DateStyle;") //直接查询和解析单行数据
	postgresDb.Update()              //修改/更新数据
	postgresDb.Read()
	postgresDb.Delete()              //删除数据
	postgresDb.Read()
	postgresDb.DropTable()           //清理表
}

```

执行 go run main.go运行结果如下:

```shell

Successfully created connection to database
Finished dropping table (if existed)
Finished creating table
Inserted 3 rows of data
Data row = (1, banana, 150)
Data row = (2, orange, 154)
Data row = (3, apple, 100)
2020/12/15 22:13:33 返回多元素数组:
[{"id":"1","name":"banana","quantity":"150"},{"id":"2","name":"orange","quantity":"154"},{"id":"3","name":"apple","quantity":"100"}]
2020/12/15 22:13:33 返回单行数据Map:
{"DateStyle":"ISO, MDY"}
Updated 1 row of data
Data row = (2, orange, 154)
Data row = (3, apple, 100)
Data row = (1, banana, 200)
Deleted 1 row of data
Data row = (3, apple, 100)
Data row = (1, banana, 200)
Finished dropping table (if existed)
```



## 总结

- 本文对pq驱动包以及连接字符串参数进行了介绍
- 示例代码分别将连接/创建表格/增加行数据/更新行数据/删除行数据封装为不同的方法, 便于灵活使用
- 查询单行或多行数据时, 可以直接使用封装好的方法, 直接传入Db指针和查询语句即可



## 参考文档

https://docs.microsoft.com/en-us/azure/postgresql/connect-go

https://pkg.go.dev/github.com/lib/pq

https://www.postgresql.org/

https://www.postgresql.org/docs/current/libpq.html