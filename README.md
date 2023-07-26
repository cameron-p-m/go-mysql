# go-mysql

A pure go library to handle MySQL network protocol and replication.

![semver](https://img.shields.io/github/v/tag/go-mysql-org/go-mysql)
![example workflow](https://github.com/go-mysql-org/go-mysql/actions/workflows/ci.yml/badge.svg)
![gomod version](https://img.shields.io/github/go-mod/go-version/go-mysql-org/go-mysql/master)

## How to migrate to this repo
To change the used package in your repo it's enough to add this `replace` directive to your `go.mod`:
```
replace github.com/siddontang/go-mysql => github.com/go-mysql-org/go-mysql v1.7.0
```

v1.7.0 - is the last tag in repo, feel free to choose what you want.

## Changelog
This repo uses [Changelog](CHANGELOG.md).

---
# Content
* [Replication](#replication)
* [Incremental dumping](#canal)
* [Client](#client)
* [Fake server](#server)
* [Failover](#failover)
* [database/sql like driver](#driver)

## Replication

Replication package handles MySQL replication protocol like [python-mysql-replication](https://github.com/noplay/python-mysql-replication).

You can use it as a MySQL replica to sync binlog from master then do something, like updating cache, etc...

### Example

```go
import (
	"github.com/go-mysql-org/go-mysql/replication"
	"os"
)
// Create a binlog syncer with a unique server id, the server id must be different from other MySQL's. 
// flavor is mysql or mariadb
cfg := replication.BinlogSyncerConfig {
	ServerID: 100,
	Flavor:   "mysql",
	Host:     "127.0.0.1",
	Port:     3306,
	User:     "root",
	Password: "",
}
syncer := replication.NewBinlogSyncer(cfg)

// Start sync with specified binlog file and position
streamer, _ := syncer.StartSync(mysql.Position{binlogFile, binlogPos})

// or you can start a gtid replication like
// streamer, _ := syncer.StartSyncGTID(gtidSet)
// the mysql GTID set likes this "de278ad0-2106-11e4-9f8e-6edd0ca20947:1-2"
// the mariadb GTID set likes this "0-1-100"

for {
	ev, _ := streamer.GetEvent(context.Background())
	// Dump event
	ev.Dump(os.Stdout)
}

// or we can use a timeout context
for {
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
	ev, err := streamer.GetEvent(ctx)
	cancel()

	if err == context.DeadlineExceeded {
		// meet timeout
		continue
	}

	ev.Dump(os.Stdout)
}
```

The output looks:

```
=== RotateEvent ===
Date: 1970-01-01 08:00:00
Log position: 0
Event size: 43
Position: 4
Next log name: mysql.000002

=== FormatDescriptionEvent ===
Date: 2014-12-18 16:36:09
Log position: 120
Event size: 116
Version: 4
Server version: 5.6.19-log
Create date: 2014-12-18 16:36:09

=== QueryEvent ===
Date: 2014-12-18 16:38:24
Log position: 259
Event size: 139
Salve proxy ID: 1
Execution time: 0
Error code: 0
Schema: test
Query: DROP TABLE IF EXISTS `test_replication` /* generated by server */
```

## Canal 

Canal is a package that can sync your MySQL into everywhere, like Redis, Elasticsearch. 

First, canal will dump your MySQL data then sync changed data using binlog incrementally. 

You must use ROW format for binlog, full binlog row image is preferred, because we may meet some errors when primary key changed in update for minimal or noblob row image. 

A simple example:

```go
package main

import (
	"github.com/go-mysql-org/go-mysql/canal"
	"github.com/siddontang/go-log/log"
)

type MyEventHandler struct {
	canal.DummyEventHandler
}

func (h *MyEventHandler) OnRow(e *canal.RowsEvent) error {
	log.Infof("%s %v\n", e.Action, e.Rows)
	return nil
}

func (h *MyEventHandler) String() string {
	return "MyEventHandler"
}

func main() {
	cfg := canal.NewDefaultConfig()
	cfg.Addr = "127.0.0.1:3306"
	cfg.User = "root"
	// We only care table canal_test in test db
	cfg.Dump.TableDB = "test"
	cfg.Dump.Tables = []string{"canal_test"}

	c, err := canal.NewCanal(cfg)
	if err != nil {
		log.Fatal(err)
	}

	// Register a handler to handle RowsEvent
	c.SetEventHandler(&MyEventHandler{})

	// Start canal
	c.Run()
}
```

You can see [go-mysql-elasticsearch](https://github.com/siddontang/go-mysql-elasticsearch) for how to sync MySQL data into Elasticsearch. 

## Client

Client package supports a simple MySQL connection driver which you can use it to communicate with MySQL server. 

### Example

```go
import (
	"github.com/go-mysql-org/go-mysql/client"
)

// Connect MySQL at 127.0.0.1:3306, with user root, an empty password and database test
conn, _ := client.Connect("127.0.0.1:3306", "root", "", "test")

// Or to use SSL/TLS connection if MySQL server supports TLS
//conn, _ := client.Connect("127.0.0.1:3306", "root", "", "test", func(c *Conn) {c.UseSSL(true)})

// Or to set your own client-side certificates for identity verification for security
//tlsConfig := NewClientTLSConfig(caPem, certPem, keyPem, false, "your-server-name")
//conn, _ := client.Connect("127.0.0.1:3306", "root", "", "test", func(c *Conn) {c.SetTLSConfig(tlsConfig)})

conn.Ping()

// Insert
r, _ := conn.Execute(`insert into table (id, name) values (1, "abc")`)

// Get last insert id
println(r.InsertId)
// Or affected rows count
println(r.AffectedRows)

// Select
r, err := conn.Execute(`select id, name from table where id = 1`)

// Close result for reuse memory (it's not necessary but very useful)
defer r.Close()

// Handle resultset
v, _ := r.GetInt(0, 0)
v, _ = r.GetIntByName(0, "id")

// Direct access to fields
for _, row := range r.Values {
	for _, val := range row {
		_ = val.Value() // interface{}
		// or
		if val.Type == mysql.FieldValueTypeFloat {
			_ = val.AsFloat64() // float64
		}
	}   
}
```

Tested MySQL versions for the client include:
- 5.5.x
- 5.6.x
- 5.7.x
- 8.0.x

### Example for SELECT streaming (v1.1.1)
You can use also streaming for large SELECT responses.
The callback function will be called for every result row without storing the whole resultset in memory.
`result.Fields` will be filled before the first callback call.

```go
// ...
var result mysql.Result
err := conn.ExecuteSelectStreaming(`select id, name from table LIMIT 100500`, &result, func(row []mysql.FieldValue) error {
    for idx, val := range row {
    	field := result.Fields[idx]
    	// You must not save FieldValue.AsString() value after this callback is done.
    	// Copy it if you need.
    	// ...
    }
    return nil
}, nil)

// ...
```

### Example for connection pool (v1.3.0)

```go
import (
    "github.com/go-mysql-org/go-mysql/client"
)

pool := client.NewPool(log.Debugf, 100, 400, 5, "127.0.0.1:3306", `root`, ``, `test`)
// ...
conn, _ := pool.GetConn(ctx)
defer pool.PutConn(conn)

conn.Execute() / conn.Begin() / etc...
```

## Server

Server package supplies a framework to implement a simple MySQL server which can handle the packets from the MySQL client. 
You can use it to build your own MySQL proxy. The server connection is compatible with MySQL 5.5, 5.6, 5.7, and 8.0 versions,
so that most MySQL clients should be able to connect to the Server without modifications.

### Example

Minimalistic MySQL server implementation:

```go
package main

import (
	"log"
	"net"

	"github.com/go-mysql-org/go-mysql/server"
)

func main() {
	// Listen for connections on localhost port 4000
	l, err := net.Listen("tcp", "127.0.0.1:4000")
	if err != nil {
		log.Fatal(err)
	}

	// Accept a new connection once
	c, err := l.Accept()
	if err != nil {
		log.Fatal(err)
	}

	// Create a connection with user root and an empty password.
	// You can use your own handler to handle command here.
	conn, err := server.NewConn(c, "root", "", server.EmptyHandler{})
	if err != nil {
		log.Fatal(err)
	}

	// as long as the client keeps sending commands, keep handling them
	for {
		if err := conn.HandleCommand(); err != nil {
			log.Fatal(err)
		}
	}
}

```

Another shell

```
$ mysql -h127.0.0.1 -P4000 -uroot
Your MySQL connection id is 10001
Server version: 5.7.0

MySQL [(none)]>
// Since EmptyHandler implements no commands, it will throw an error on any query that you will send
```

> ```NewConn()``` will use default server configurations:
> 1. automatically generate default server certificates and enable TLS/SSL support.
> 2. support three mainstream authentication methods **'mysql_native_password'**, **'caching_sha2_password'**, and **'sha256_password'**
>    and use **'mysql_native_password'** as default.
> 3. use an in-memory user credential provider to store user and password.
>
> To customize server configurations, use ```NewServer()``` and create connection via ```NewCustomizedConn()```.


## Failover

Failover supports to promote a new master and let replicas replicate from it automatically when the old master was down.

Failover supports MySQL >= 5.6.9 with GTID mode, if you use lower version, e.g, MySQL 5.0 - 5.5, please use [MHA](http://code.google.com/p/mysql-master-ha/) or [orchestrator](https://github.com/outbrain/orchestrator).

At the same time, Failover supports MariaDB >= 10.0.9 with GTID mode too. 

Why only GTID? Supporting failover with no GTID mode is very hard, because replicas can not find the proper binlog filename and position with the new master.
Although there are many companies use MySQL 5.0 - 5.5, I think upgrade MySQL to 5.6 or higher is easy. 

## Driver

Driver is the package that you can use go-mysql with go database/sql like other drivers. A simple example:

```go
package main

import (
	"database/sql"

	_ "github.com/go-mysql-org/go-mysql/driver"
)

func main() {
	// dsn format: "user:password@addr?dbname"
	dsn := "root@127.0.0.1:3306?test"
	db, _ := sql.Open(dsn)
	db.Close()
}
```

We pass all tests in https://github.com/bradfitz/go-sql-test using go-mysql driver. :-)

## Donate

If you like the project and want to buy me a cola, you can through: 

|PayPal|微信|
|------|---|
|[![](https://www.paypalobjects.com/webstatic/paypalme/images/pp_logo_small.png)](https://paypal.me/siddontang)|[![](https://github.com/siddontang/blog/blob/master/donate/weixin.png)|

## Feedback

go-mysql is still in development, your feedback is very welcome. 


Gmail: siddontang@gmail.com
