# 第7章：数据库和文件系统：盗用和滥用

## 摘要

既然我们已经讨论了用于主动服务询问、命令和控制以及其他恶意活动的大多数通用网络协议，现在让我们将重点放到同样重要的主题上：数据掠夺。

虽然数据掠夺可能不像初始利用、横向网络运动或特权升级那样令人兴奋，但它是整个攻击链的一个关键方面。毕竟，我们经常需要数据来执行其他活动。通常，这些数据对攻击者来说是有实际价值的。虽然黑客攻击一个组织是令人兴奋的，但数据本身往往是攻击者有利可图的奖品，也是组织的致命损失。

根据研究，在2016年，一次入侵可能会给组织造成大约400万到700万美元的损失。IBM的一项研究估计，每一个被偷的记录会给组织带来129到355美元的损失。见鬼，黑帽黑客可以通过在地下市场上以每张7到80美元的价格出售信用卡来赚大钱（[http://online.wsj.com/public/resources/documents/secureworks\_hacker\_annualreport.pdf\*](http://online.wsj.com/public/resources/documents/secureworks_hacker_annualreport.pdf*) ）。

仅Target漏洞就造成了4000万张卡片的泄露。在某些情况下，Target卡的售价高达每张135美元（[http://www.businessinsider.com/heres-what-happened-to-your-target-data-that-was-hacked-2014-10/](http://www.businessinsider.com/heres-what-happened-to-your-target-data-that-was-hacked-2014-10/)）。那是非常赚钱的。我们绝不提倡这种行为，但那些道德有问题的人却仍然通过盗窃数据赚钱。

关于该行业的丰富知识和对在线文章的丰富参考——让我们拭目以待！在本章中，将学习如何配置和建立各种SQL和NoSQL数据库，以及如何通过Go连接并和这些数据库交互。我们还将演示如何创建数据库和文件系统数据挖掘程序，用于搜索有趣信息的关键指示符。

## 使用Docker配置数据库

在这一节中，将安装各种数据库，然后将本章掠夺示例中使用的数据插入到数据库中。如果可能的话，在Ubuntu 18.04 VM上使用Docker。_Docker_ 是个软件容器平台，能够轻松地部署和管理应用程序。可以直接捆绑部署应用程序及其依赖。容器与操作系统分开，以防止对主机平台的污染。这是很漂亮的设计。

对于本章，为使用数据库将会用到各种预构建的Docker镜像。还未安装的话就先安装Docker。在\*[https://docs.docker.com/engine/installation/linux/ubuntu/](https://docs.docker.com/engine/installation/linux/ubuntu/) 可找到Ubuntu的安装说明。

### 安装**MongoDB**并写入数据

_MongoDB_是在本章中唯一用到NoSQL数据库。不像传统的关系数据库，MongoDB无法通过SQL进行通信。相反，MongoDB使用易于理解的JSON语法检索和管理数据。需要用专门的整本书来解释MongoDB，而全部的解释显然超出了本书的范围。现在，安装Docker镜像，并插入假数据。

和传统的SQL数据库不一样，MongoDB是_schema-less_，也就是它不用遵循用于组织表格数据的预定义的严格规则系统。这也就是为什么在清单7-1中只有 insert 命令而没有定义模式。首先，使用下面的命令安装MongoDB的Docker镜像：

```text
$ docker run --name some-mongo -p 27017:27017 mongo
```

该命令从Docker仓库中下载名为 mongo的镜像，然后启动名为 some-mongo（给实例起的名字是任意的） 的新实例，并且将本地的27017端口映射到容器的27017端口。端口映射是关键，因为直接从操作系统访问数据库实例。没有的话就无法访问。

通过列出所有运行中的容器来检查容器是否自动启动：

```text
$ docker ps
```

万一容器没有自动启动，执行下面的命令：

```text
$ docker start some-mongo
```

start 命令应该能让容器运行。

容器运行后使用 `run` 命令传递MongoDB客户端连接到 MongoDB 实例，用这种方式就可以和数据库交互来插入数据：

```text
$ docker run -it --link some-mongo:mongo --rm mongo sh \ 
-c 'exec mongo "$MONGO_PORT_27017_TCP_ADDR:$MONGO_PORT_27017_TCP_PORT/store"'
>
```

这个神奇的命令运行一个一次性的，第二个安装了MongoDB客户端二进制文件的Docker容器——所以不必在主机的操作系统上安装二进制文件——并使用它连接到名为some-mongo的 Docker容器中的MongoDB实例。在这个例子中，连接到名为 test 的数据库。

清单7-1，将数组文档插入到 transactions 集合中。

```javascript
> db.transactions.insert([ 
    {
        "ccnum" : "4444333322221111", 
        "date" : "2019-01-05", 
        "amount" : 100.12,
        "cvv" : "1234",
        "exp" : "09/2020" 
    },
    {
        "ccnum" : "4444123456789012", 
        "date" : "2019-01-07",
        "amount" : 2400.18, 
        "cvv" : "5544", 
        "exp" : "02/2021"
    }, 
    {
        "ccnum" : "4465122334455667", 
        "date" : "2019-01-29", 
        "amount" : 1450.87,
        "cvv" : "9876",
        "exp" : "06/2020" 
    }
]);
```

清单 7-1: 将 transactions 插入到 MongoDB 集合 \([https://github.com/blackhat-go/bhg/ch-7/db/seed-mongo.js/](https://github.com/blackhat-go/bhg/ch-7/db/seed-mongo.js/)\)

就是这样!现在，已经创建了MongoDB数据库实例，并插入了一个包含三个用于查询的虚假文档的 transactions 集合。稍后将介绍查询部分，但是首先，应该知道如何安装和插入传统SQL数据库。

### 安装**PostgreSQL 和 MySQL**并写入数据

_PostgreSQL_ （也叫 _Postgres_） 和 _MySQL_  是两个最常见的、最知名的、企业级质量的开源关系数据库管理系统，并且都有官方的Docker镜像。因为它们比较相似，并且安装步骤也有些重复，因此一起介绍安装。

与上一节中的MongoDB示例大致相同，首先下载并允许合适的Docker镜像：

```text
$ docker run --name some-mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=password \ -d mysql
$ docker run --name some-postgres -p 5432:5432 -e POSTGRES_PASSWORD=password \ -d postgres
```

容器构建成功后确认是否在运行，如果没在运行中的话，通过**docker start** **name** 命令启动。

接下来，使用相应的客户端连接到容器中——同样，使用Docker镜像来避免在主机上安装任何额外的文件——然后继续创建数据库并写入数据。清单7-2中是MySQL相关的逻辑。

```text
$ docker run -it --link some-mysql:mysql --rm mysql sh -c \
'exec mysql -h "$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" \
-uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD"'
mysql> create database store;
mysql> use store;
mysql> create table transactions(ccnum varchar(32), date date, amount float(7,2),
    -> cvv char(4), exp date);
```

清单 7-2: 创建并初始化 MySQL 数据库

与下面的清单一样，该清单启动一个一次性Docker shell，执行适当的数据库客户端。创建并连接到名为 store 的数据库，然后创建 transactions 表。这两个清单是相同的，只是针对不同的数据库系统进行了修改。、

清单7-3是Postgres的逻辑，和MySQL 的语法稍微不同。

```text
$ docker run -it --rm --link some-postgres:postgres postgres psql \ 
-h postgres -U postgres
postgres=# create database store;
postgres=# \connect store
store=# create table transactions(ccnum varchar(32), date date, amount money, cvv char(4), exp date);
```

清单 7-3: 创建并初始化 Postgres 数据库

MySQL 和 Postgres 的插入 transactions 的语法是一样的。例如，清单7-4中可以看到如何向MySQL的transactions集合中插入三个文档。

```text
mysql> insert into transactions(ccnum, date, amount, cvv, exp) values
    -> ('4444333322221111', '2019-01-05', 100.12, '1234', '2020-09-01');
mysql> insert into transactions(ccnum, date, amount, cvv, exp) values
    -> ('4444123456789012', '2019-01-07', 2400.18, '5544', '2021-02-01');
mysql> insert into transactions(ccnum, date, amount, cvv, exp) values
    -> ('4465122334455667', '2019-01-29', 1450.87, '9876', '2019-06-01');
```

清单 7-4: 向 MySQL 的 transactions 插入数据 \([https://github.com/blackhat-go/bhg/ch-7/db/seed-pg-mysql.sql/](https://github.com/blackhat-go/bhg/ch-7/db/seed-pg-mysql.sql/)\)

尝试向Postgres数据库中插入同样的三个文档数据。

## 用Go连接并查询数据库

既然现在已经多个测试数据库，那么就用Go构建客户端来连接和查询这些数据库。我们将讨论分为两个主题：一个是MongoDB，另一个是传统SQL数据库。

### 查询MongoDB

尽管有优秀的标准SQL包，但Go并没有维护类似的与NoSQL数据库交互的包。相反，需要依赖第三方包来方便交互。与其检阅第三方包的实现，不如只专注于MongoDB。我们将使用 _mgo_ \(发音为 _mango_\) 这个DB驱动程序。

使用下面的命令安装 _mgo_ 驱动：

```text
$ go get gopkg.in/mgo.v2
```

现在可以和 _store_ 集合（等同于表）建立连接并查询，所需要的代码比稍后创建的SQL示例代码还要少（参见清单7-6）。

```go
package main

import (
    "fmt"
    "log"

    mgo "gopkg.in/mgo.v2"
)

type Transaction struct {
    CCNum      string  `bson:"ccnum"`
    Date       string  `bson:"date"`
    Amount     float32 `bson:"amount"`
    Cvv        string  `bson:"cvv"`
    Expiration string  `bson:"exp"`
}

func main() {
    session, err := mgo.Dial("127.0.0.1")
    if err != nil {
        log.Panicln(err)
    }
    defer session.Close()

    results := make([]Transaction, 0)
    if err := session.DB("store").C("transactions").Find(nil).All(&results); err != nil {
        log.Panicln(err)
    }
    for _, txn := range results {
        fmt.Println(txn.CCNum, txn.Date, txn.Amount, txn.Cvv, txn.Expiration)
    }
}
```

清单 7-6: 连接并查询 MongoDB 数据库 \([https://github.com/blackhat-go/bhg/ch-7/db/mongo-connect/main.go/](https://github.com/blackhat-go/bhg/ch-7/db/mongo-connect/main.go/)\)

先定义 `Transaction` 类型，相当于 `store` 集合中的单个文档。MongoDB中数据表示的内部机制是二进制JSON。基于此，使用标记来显式定义要在二进制JSON数据中使用的元素名称。

在 `main()` 函数中，调用 `mgo.Dial()` 通过建立与数据库的连接来创建session，测试来确定没有发生错误，延迟调用来关闭session。然后使用 `session` 变量来查询 `store` 数据库，从 `transactions` 集合中检索所有记录。将结果保存在名为 `results`的 `Transaction` 类型的切片中。其背后原理是结构体的标记用于将二进制JSON解析到定义的类型中。最后，循环遍历结果集并打印。这个例子和下节的SQL示例都输出类似下面的内容：

```text
$ go run main.go
4444333322221111 2019-01-05 100.12 1234 09/2020 
4444123456789012 2019-01-07 2400.18 5544 02/2021 
4465122334455667 2019-01-29 1450.87 9876 06/2020
```

### 查询SQL数据库

Go中有一个 `database/sql` 的标准包，该包定义了用于与SQL和类似SQL的数据库进行交互的接口。基本的实现包括像连接池和事务之类的功能。遵循此接口的数据库驱动会自动继承这些功能，并且由于API在驱动之间保持一致，因此它们基本上可以互换。无论使用的是Postgres，MSSQL，MySQL还是其他驱动程序，代码中的函数调用和实现都是相同的。在客户端改动很少的代码就能方便地切换后端的数据库。当然，驱动实现了针对于特定数据库的功能，并使用不同的SQL语法，但是函数调用几乎相同。

基于此，只展示连接到一个SQL数据库——MYSQL——而将其他SQL数据库留给读者作为练习。首先使用下面的命令安装驱动：

```text
$ go get github.com/go-sql-driver/mysql
```

然后，创建简单的客户端连接到数据库，并从 `transactions`表中查询数据。代码见清单7-7。

```go
package main

import (
    "database/sql"
    "fmt"
    "log"

    _ "github.com/go-sql-driver/mysql"
)

func main() {
    db, err := sql.Open("mysql", "root:password@tcp(127.0.0.1:3306)/store")
    if err != nil {
        log.Panicln(err)
    }
    defer db.Close()

    var (
        ccnum, date, cvv, exp string
        amount                float32
    )
    rows, err := db.Query("SELECT ccnum, date, amount, cvv, exp FROM transactions")
    if err != nil {
        log.Panicln(err)
    }
    defer rows.Close()
    for rows.Next() {
        err := rows.Scan(&ccnum, &date, &amount, &cvv, &exp)
        if err != nil {
            log.Panicln(err)
        }
        fmt.Println(ccnum, date, amount, cvv, exp)
    }
    if rows.Err() != nil {
        log.Panicln(err)
    }
}
```

清单 7-7: 连接并查询 MySQL 数据库 \([https://github.com/blackhat-go/bhg/ch-7/db/mysql-connect/main.go/](https://github.com/blackhat-go/bhg/ch-7/db/mysql-connect/main.go/)\)

代码从导入Go的 `database/sql` 包开始。这样一来，就可以使用Go中出色的标准SQL库与数据库进行交互。同样导入MySQL数据库驱动。前面的下划线表示是匿名导入包，也就是不包括包的导出的类型，但是驱动将自己注册到 `sql` 包中，以便MySQL驱动本身处理函数调用。

接下来调用 `sql.Open` 和数据库建立连接。第一个参数指明要使用那种驱动——在本例中使用 `mysql` 驱动，第二个参数指明连接的字符串。然后查询数据库，使用SQL语句查询 `transactions` 表中的所有行，然后循环遍历这些行，随后将数据读入变量并打印值。

这就是查询MySQL数据库所需要做的。使用不同的后端数据库仅需要对代码进行以下较小更改：

1. 导入正确的数据库驱动。
2. 修改传入到 `sql.Open()` 的参数。
3. 将SQL语法调整为后端数据库所需的风格。

在几种可用的数据库驱动程序中，许多是纯Go语言，而少数驱动使用 `cgo` 进行一些基础交互。访问 `https://github.com/golang/go/wiki/SQLDrivers/` 查看可用驱动程序列表。

## 构建数据库挖掘器

在本节中，将创建一个检查数据库模式（例如列名）的工具，以确定其中的数据是否值得窃取。例如，假设希望查找密码、哈希、社保号和信用卡号。与其构建庞大功能来挖掘各种后端数据库，不如为每个数据创建单独的功能——实现一个定义的接口，以确保实现之间的一致性。对于本示例，这种灵活性可能有些过高，但是这是创建可重用和可移植性代码的好机会。

接口应该小巧，由最基本的类型和函数组成，并且它应该需要实现一个方法来检索数据库模式。清单7-8是 _dbminer.go_ ，定义了数据库旷工接口。

```go
package dbminer

import (
    "fmt"
    "regexp"
)

type DatabaseMiner interface {
    GetSchema() (*Schema, error)
}

type Schema struct {
    Databases []Database
}

type Database struct {
    Name   string
    Tables []Table
}

type Table struct {
    Name    string
    Columns []string
}

func Search(m DatabaseMiner) error {
    s, err := m.GetSchema()
    if err != nil {
        return err
    }

    re := getRegex()
    for _, database := range s.Databases {
        for _, table := range database.Tables {
            for _, field := range table.Columns {
                for _, r := range re {
                    if r.MatchString(field) {
                        fmt.Println(database)
                        fmt.Printf("[+] HIT: %s\n", field)
                    }
                }
            }
        }
    }
    return nil
}

func getRegex() []*regexp.Regexp {
    return []*regexp.Regexp{
        regexp.MustCompile(`(?i)social`),
        regexp.MustCompile(`(?i)ssn`),
        regexp.MustCompile(`(?i)pass(word)?`),
        regexp.MustCompile(`(?i)hash`),
        regexp.MustCompile(`(?i)ccnum`),
        regexp.MustCompile(`(?i)card`),
        regexp.MustCompile(`(?i)security`),
        regexp.MustCompile(`(?i)key`),
    }
}

func (s Schema) String() string {
    var ret string
    for _, database := range s.Databases {
        ret += fmt.Sprint(database.String() + "\n")
    }
    return ret
}

func (d Database) String() string {
    ret := fmt.Sprintf("[DB] = %+s\n", d.Name)
    for _, table := range d.Tables {
        ret += table.String()
    }
    return ret
}

func (t Table) String() string {
    ret := fmt.Sprintf("    [TABLE] = %+s\n", t.Name)
    for _, field := range t.Columns {
        ret += fmt.Sprintf("       [COL] = %+s\n", field)
    }
    return ret
}
/* Extranneous code omitted for brevity */
```

清单 7-8: 实现数据库挖掘器 \([https://github.com/blackhat-go/bhg/ch-7/db/dbminer/dbminer.go/](https://github.com/blackhat-go/bhg/ch-7/db/dbminer/dbminer.go/)\)

代码先定义 `DatabaseMiner` 接口。只有 `GetSchema()` 这一个方法，实现接口的任何类型都需要该方法。因为每个后端数据库可能有各自的逻辑来检索数据库模式，所以期望每个功能都以对所使用的后端数据库和驱动统一的方式来实现逻辑。

接下来定义 `Schema` 类型，由在此处定义的一些类型组成。使用 `Schema` 在逻辑上表示数据库模式，即数据库，表，列。可以会注意到接口中定义的 `GetSchema()` 函数，其实现的返回值为 `*Schema` 。

现在定义了 `Search()` 这个函数，其中包含大部分逻辑。调用`Search()`函数时需要传入 `DatabaseMiner` 实例，并且在 `m` 变量中保存挖掘器值。函数通过调用 `m.GetSchema()` 检索数据库模式开始。函数然后循环遍历整个模式，在正则表达式（regex）值列表中搜索与之匹配的列名。如果找到匹配的话，数据库模式和匹配的字段被打印到屏幕上。

最后定义 `getRegex()` 函数。该函数通过使用Go的 `regexp` 包来过滤正则字符串，然后返回这些值的切片。正则表达式列表由不区分大小写的字符串组成，这些字符串与常见或有趣的字段名称（例如`ccnum`，`ssn`和`password`）匹配。

有了数据库挖掘器的接口，就可以创建实现特定的功能。从MongoDB数据库挖掘器开始。

### 实现MongoDB数据库挖掘器

清单7-9中的MongoDB的功能程序实现了清单7-8中定义的接口，同时还集成了在清单7-6中构建的数据库连接代码。

```go
package main

import (
    "os"

    "gopkg.in/mgo.v2"
    "gopkg.in/mgo.v2/bson"
    "github.com/blackhat-go/bhg/ch-7/db/dbminer"
)

type MongoMiner struct {
    Host    string
    session *mgo.Session
}

func New(host string) (*MongoMiner, error) {
    m := MongoMiner{Host: host}
    err := m.connect()
    if err != nil {
        return nil, err
    }
    return &m, nil
}

func (m *MongoMiner) connect() error {
    s, err := mgo.Dial(m.Host)
    if err != nil {
        return err
    }
    m.session = s
    return nil
}

func (m *MongoMiner) GetSchema() (*dbminer.Schema, error) {
    var s = new(dbminer.Schema)

    dbnames, err := m.session.DatabaseNames()
    if err != nil {
        return nil, err
    }

    for _, dbname := range dbnames {
        db := dbminer.Database{Name: dbname, Tables: []dbminer.Table{}}
        collections, err := m.session.DB(dbname).CollectionNames()
        if err != nil {
            return nil, err
        }

        for _, collection := range collections {
            table := dbminer.Table{Name: collection, Columns: []string{}}

            var docRaw bson.Raw
            err := m.session.DB(dbname).C(collection).Find(nil).One(&docRaw)
            if err != nil {
                return nil, err
            }

            var doc bson.RawD
            if err := docRaw.Unmarshal(&doc); err != nil {
                if err != nil {
                    return nil, err
                }
            }

            for _, f := range doc {
                table.Columns = append(table.Columns, f.Name)
            }
            db.Tables = append(db.Tables, table)
        }
        s.Databases = append(s.Databases, db)
    }
    return s, nil
}

func main() {

    mm, err := New(os.Args[1])
    if err != nil {
        panic(err)
    }
    if err := dbminer.Search(mm); err != nil {
        panic(err)
    }
}
```

清单 7-9: 创建 MongoDB 数据库挖掘器 \([https://github.com/blackhat-go/bhg/ch-7/db/mongo/main.go/](https://github.com/blackhat-go/bhg/ch-7/db/mongo/main.go/)\)

从导入定义了`Database Miner`接口的 `dbminer` 包开始。然后定义了用于实现接口的 `MongoMiner` 类型。为了方便起见，定义 `New()` 函数来创建 `MongoMiner` 类型的新实例，该函数调用 `connect()` 和数据库建立连接。该逻辑的整体实质上是启动代码，并 以类似于清单7-6中所讨论的方式连接到数据库。

代码中最有趣的部分是实现 `GetSchema()` 接口方法。与清单7-6中之前的MongoDB示例代码不同，现在检查MongoDB的元数据，先获取数据库名字，然后循环遍历数据库来检索每个数据库的集合名字。函数的最后检索原始文档，与典型的MongoDB查询不同，此处使用懒惰解析。这样可以将记录显式地解析到通用结构，如此以来就可以检查字段名称。如果不使用懒惰解析，必须定义明确的类型，可能使用 `bson` 属性，以便指示代码如何将数据解析到定义的结构体中。在这种情况下，无需知道（或关心）结构体的字段类型，而只需要字段名称（而不是数据），因此，这是可以在事先不了解数据结构的情况下解析结构化数据的方法。

`main()` 函数将MongoDB实例的IP地址作为其唯一参数，调用 `New()` 函数来启动，然后调用 `dbminer.Search()`，传入 `MongoMiner` 实例。回想在接收到的 `DatabaseMiner` 实例时 `dbmine . search()` 调用 `GetSchema()`；这将调用该函数的 `MongoMiner` 实现，从而创建 `dbminer`，然后根据清单7-8中的正则表达式列表搜索。

当运行程序时，将得到以下输出:

```text
$ go run main.go 127.0.0.1 
[DB] = store
    [TABLE] = transactions 
        [COL] = _id
        [COL] = ccnum 
        [COL] = date 
        [COL] = amount 
        [COL] = cvv 
        [COL] = exp
[+] HIT: ccnum
```

匹配成功了！看起来可能不美观，但它完成了任务——成功地定位了有一个名为 `ccnum` 字段的数据库集合。

在下一部分中，构建完MongoDB实现后，将对MySQL后端数据库执行相同的操作。

### 实现**MySQL**数据库挖掘器

要使MySQL正常执行，需要检查 `information _schema.columns` 表。该表持有所有的数据库及其结构的元数据，包括表明和列名。要最简单的使用数据，使用下面的SQL查询，该查询删除了与某些内置MySQL数据库无关的信息，这些信息与掠夺工作无关：

```text
SELECT TABLE_SCHEMA, TABLE_NAME, COLUMN_NAME FROM columns
    WHERE TABLE_SCHEMA NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys') 
    ORDER BY TABLE_SCHEMA, TABLE_NAME
```

该查询产生类似于以下内容的结果：

```text
+--------------+--------------+-------------+ 
| TABLE_SCHEMA |  TABLE_NAME  | COLUMN_NAME | 
+--------------+--------------+-------------+
| store        | transactions | ccnum       |
| store           | transactions | date        |
| store        | transactions | amount      |
| store        | transactions | cvv         |
| store        | transactions | exp         |
--snip--
```

尽管使用查询来检索模式信息非常直接，但是代码的复杂性来自逻辑上在定义 `GetSchema()` 函数时对每行进行区分和分类。例如，连续的输出行可能属于或不属于同一数据库或表，因此将行和正确的 `dbminer .Database` 和 `dbminer.Table` 实例关联起来变的有点棘手，

清单7-10定义了实现。

```go
package main

import (
    "database/sql"
    "fmt"
    "log"
    "os"

    _ "github.com/go-sql-driver/mysql"
    "github.com/blackhat-go/bhg/ch-7/db/dbminer"
)

type MySQLMiner struct {
    Host string
    Db   sql.DB
}

func New(host string) (*MySQLMiner, error) {
    m := MySQLMiner{Host: host}
    err := m.connect()
    if err != nil {
        return nil, err
    }
    return &m, nil
}

func (m *MySQLMiner) connect() error {

    db, err := sql.Open("mysql", fmt.Sprintf("root:password@tcp(%s:3306)/information_schema", m.Host))
    if err != nil {
        log.Panicln(err)
    }
    m.Db = *db
    return nil
}

func (m *MySQLMiner) GetSchema() (*dbminer.Schema, error) {
    var s = new(dbminer.Schema)

    sql := `SELECT TABLE_SCHEMA, TABLE_NAME, COLUMN_NAME FROM columns
    WHERE TABLE_SCHEMA NOT IN ('mysql', 'information_schema', 'performance_schema', 'sys')
    ORDER BY TABLE_SCHEMA, TABLE_NAME`
    schemarows, err := m.Db.Query(sql)
    if err != nil {
        return nil, err
    }
    defer schemarows.Close()

    var prevschema, prevtable string
    var db dbminer.Database
    var table dbminer.Table
    for schemarows.Next() {
        var currschema, currtable, currcol string
        if err := schemarows.Scan(&currschema, &currtable, &currcol); err != nil {
            return nil, err
        }

        if currschema != prevschema {
            if prevschema != "" {
                db.Tables = append(db.Tables, table)
                s.Databases = append(s.Databases, db)
            }
            db = dbminer.Database{Name: currschema, Tables: []dbminer.Table{}}
            prevschema = currschema
            prevtable = ""
        }

        if currtable != prevtable {
            if prevtable != "" {
                db.Tables = append(db.Tables, table)
            }
            table = dbminer.Table{Name: currtable, Columns: []string{}}
            prevtable = currtable
        }
        table.Columns = append(table.Columns, currcol)
    }
    db.Tables = append(db.Tables, table)
    s.Databases = append(s.Databases, db)
    if err := schemarows.Err(); err != nil {
        return nil, err
    }

    return s, nil
}

func main() {
    mm, err := New(os.Args[1])
    if err != nil {
        panic(err)
    }
    defer mm.Db.Close()

    if err := dbminer.Search(mm); err != nil {
        panic(err)
    }
}
```

清单 7-10: 创建 MySQL 数据库挖掘器 \([https://github.com/blackhat-go/bhg/ch-7/db/mysql/main.go/](https://github.com/blackhat-go/bhg/ch-7/db/mysql/main.go/)\)

快速浏览下代码的话会发现和前一部分中MongoDB的示例代码非常非常相似。实际上，`main()` 函数是相同的。

启动函数也非常相似——只需要将和MongoDB交互的逻辑改为和MySQL交互。注意，这是连接到 `information_schema` 数据库的逻辑，以便能够检查数据库模式。

代码的大部分复杂性都位于 `GetSchema()` 的实现中。尽管使用单个数据库查询能够检索到数据库模式信息，然后循环遍历结果，检查每一行，以便确定存在哪些数据库，每个数据库中有哪些表，每个表中有哪些行。与MongoDB实现不同，无法使用带有属性标签的 **JSON / BSON** 来将数据编组和解编到复杂的结构；可以维护变量来跟踪当前行中的信息，并将其与上一行中的数据进行比较，以确定您是否遇到了新的数据库或表。这不是最优雅的解决方案，但是这也能工作。

接下来，检查当前行的和前一行的数据库名字是否不同。如果是的话就创建新的 `miner.Database` 实例。如果这不是第一次循环，就将表和数据库添加到 `miner.Schema` 实例中。使用类似的逻辑来追踪并将 `miner.Table` 实例添加到当前的 `miner.Database` 中。最后，将每一列添加到 `miner.Table` 。

现在，针对 Docker MySQL 实例运行程序，确认是否工作正常，如下所示：

```text
$ go run main.go 127.0.0.1 
[DB] = store
    [TABLE] = transactions 
        [COL] = ccnum
        [COL] = date
        [COL] = amount [COL] = cvv
        [COL] = exp 
[+] HIT: ccnum
```

输出应该与MongoDB输出几乎一样。这是因为 `dbminer.Schema` 没有产生任何输出—— `dbminer .Search()` 函数是。这就是使用接口的威力。可以有关键特性的特定实现，但仍然可以使用单一的标准函数以可预测的、可用的方式处理数据。

## 掠夺文件系统

在本节中，构建个文件系统工具，其能够递归地遍历用户提供的文件系统路径，并与匹配一系列有趣的文件名，这些文件名在后面的开发的练习中会有用。这些文件可能包含个人身份信息、用户名、密码、系统登录和密码数据库文件。

该工具专门针对于文件名而不是文件内容，脚本实际上非常简单，使用的是Go的 `path/filepath` 包中的标准函数，该包非常容易地遍历目录结构。该工具的代码参见清单 7-11。

```go
package main

import (
    "fmt"
    "log"
    "os"
    "path/filepath"
    "regexp"
)

var regexes = []*regexp.Regexp{
    regexp.MustCompile(`(?i)user`),
    regexp.MustCompile(`(?i)password`),
    regexp.MustCompile(`(?i)kdb`),
    regexp.MustCompile(`(?i)login`),
}

func walkFn(path string, f os.FileInfo, err error) error {
    for _, r := range regexes {
        if r.MatchString(path) {
            fmt.Printf("[+] HIT: %s\n", path)
        }
    }
    return nil
}

func main() {
    root := os.Args[1]
    if err := filepath.Walk(root, walkFn); err != nil {
        log.Panicln(err)
    }
}
```

清单 7-11: 遍历并搜索文件系统 \([https://github.com/blackhat-go/bhg/ch-7/filesystem/main.go/](https://github.com/blackhat-go/bhg/ch-7/filesystem/main.go/)\)

和数据库挖掘的实现相对比，文件系统的设置和逻辑似乎有点太简单了。与创建数据库实现的方式类似，定义一个正则表达式列表来识别有趣的文件名。为了使代码最少，我们将列表限制为少数几个条目，但是可以扩展列表以适应更多实际用途。

接下来，定义 `walkFn()` 函数，该函数的参数为文件路径和一些额外的参数，该函数循环遍历正则表达式列表并检查匹配，显示到标准输出。`walkFn()` 函数被用在 `main()` 函数中，并作为参数传递给 `filepath.Walk()` 。`Walk()` 函数有两个参数，一个根路径和一个函数（这里是 `walkFn()`）。并从作为根路径提供的值开始递归遍历目录结构，为遇到的每个目录和文件调用 `walkFn()` 。

工具完成后，回到桌面并创建以下目录结构：

```text
$ tree targetpath/ 
targetpath/
--- anotherpath
-    --- nothing.txt
-    --- users.csv
--- file1.txt
--- yetanotherpath 
    --- nada.txt
    --- passwords.xlsx 
2 directories, 5 files
```

在同一个 `targetpath` 目录下运行你的工具会产生以下输出，确认了代码可以出色地执行：

```text
$ go run main.go ./somepath
[+] HIT: somepath/anotherpath/users.csv
[+] HIT: somepath/yetanotherpath/passwords.xlsx
```

这就是全部的内容。可以通过包含额外的或更特定的正则表达式来改进示例代码。此外，我们鼓励您通过仅对文件名而不对目录应用正则表达式检查来改进代码。我们鼓励您进行的另一项增强功能是查找和标记具有最近修改或访问时间的特定文件。该元数据可以引导您获得更重要的内容，包括用作关键业务流程一部分的文件。

## 总结

在这一章，深入研究了数据库交互和文件系统遍历，即使用了Go的原生包，又使用了第三方库来和数据库元数据和文件系统交互。作为攻击者，这些资源经常包含有价值的信息，并且我们创建了各种实用程序，使我们可以搜索这些有趣的信息。

在下一章中，将了解实际的数据包处理。具体来说，将学习如何嗅探和操作网络包。

