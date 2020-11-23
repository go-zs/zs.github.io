---
title: golang版Canal源码阅读
date: 2020-11-04 10:43:25
tags:
---

`Canal`是阿里开源的一款`Java`语言编写的中间件，主要用途是基于MySQL 数据库增量日志解析。[go-mysql](https://github.com/siddontang/go-mysql)是`golang`版本的实现。它不仅支持增量`binlog`消费，也支持全量数据的解析。

<!-- more -->

## 源码结构
```
├── canal    // 核心类
├── client   // 连接数据库的client
├── cmd      // 功能
│   ├── go-binlogparser
│   ├── go-canal
│   ├── go-mysqlbinlog
│   └── go-mysqldump
├── docker
├── driver   
├── dump     // 导出
├── failover
├── mysql
├── notes
├── packet
├── replication  // slave注册,binlog解析
├── schema       // schema解析
├── server
├── test_util
└── utils
```

## Canal类解析

```golang
// Canal can sync your MySQL data into everywhere, like Elasticsearch, Redis, etc...
// MySQL must open row format for binlog
type Canal struct {
	m sync.Mutex

	cfg *Config    // 配置

	parser     *parser.Parser  // 解析器
	master     *masterInfo
	dumper     *dump.Dumper  // 数据导出
	dumped     bool
	dumpDoneCh chan struct{}
	syncer     *replication.BinlogSyncer  // 伪装成从库，同步binlog，事件生产者

	eventHandler EventHandler  // 事件处理，消费者

	connLock sync.Mutex
	conn     *client.Conn  // 数据库连接

	tableLock          sync.RWMutex
	tables             map[string]*schema.Table
	errorTablesGetTime map[string]time.Time

	tableMatchCache   map[string]bool
	includeTableRegex []*regexp.Regexp
	excludeTableRegex []*regexp.Regexp

	delay *uint32

	ctx    context.Context
	cancel context.CancelFunc
}
```

`Canal`中比较重要的是`Dumper`类和`BinlogSyncer`类,`Dumper`用于处理当前数据，`BinlogSyncer`处理增量数据。


## Dumper

```golang
// Unlick mysqldump, Dumper is designed for parsing and syning data easily.
type Dumper struct {
	// mysqldump execution path, like mysqldump or /usr/bin/mysqldump, etc...
	ExecutionPath string  

	Addr     string
	User     string
	Password string
	Protocol string

	// Will override Databases
	Tables  []string
	TableDB string

	Databases []string

	Where   string
	Charset string

	IgnoreTables map[string][]string

	ExtraOptions []string

	ErrOut io.Writer

	masterDataSkipped bool
	maxAllowedPacket  int
	hexBlob           bool
}

// 导出数据
func (d *Dumper) Dump(w io.Writer) error {
...
}

// 导出数据并解析
func (d *Dumper) DumpAndParse(h ParseHandler) error {
  ...
}

```

`Dumper`原理是通过`myqldump`导出数据到`Dump`函数传入的参数`io.Writer`中。

`DumpAndParse`函数接收`ParseHandler`这个`interface`进行数据解析，以此实现导出到不同的数据源。

```golang
// 解析mysqldump数据
type ParseHandler interface {
	// Parse CHANGE MASTER TO MASTER_LOG_FILE=name, MASTER_LOG_POS=pos;
	BinLog(name string, pos uint64) error  // 解析binlog position
	GtidSet(gtidsets string) error  // 解析gtid
	Data(schema string, table string, values []string) error  // 处理行数据
}
```

## BinlogSyncer

```golang
type BinlogSyncer struct {
	m sync.RWMutex

	cfg BinlogSyncerConfig

	c *client.Conn

	wg sync.WaitGroup

	parser *BinlogParser

	nextPos Position

	prevGset, currGset GTIDSet

	running bool

	ctx    context.Context
	cancel context.CancelFunc

	lastConnectionID uint32

	retryCount int
}

// 从指定的postion开始同步
func (b *BinlogSyncer) StartSync(pos Position) (*BinlogStreamer, error) {
  ...
}

// 基于GTID同步
func (b *BinlogSyncer) StartSyncGTID(gset GTIDSet) (*BinlogStreamer, error) {
  ...
}

// BinlogStreamer gets the streaming event.
type BinlogStreamer struct {
	ch  chan *BinlogEvent
	ech chan error
	err error
}

```

`BinlogSyncer`同时支持指定`binlog`位置和`GTID`模式进行同步。它将接收到的`binlog`消息写到`BinlogStreamer`内部的`chan *BinlogEvent`中。 


```golang
type BinlogEvent struct {
	// raw binlog data which contains all data, including binlog header and event body, and including crc32 checksum if exists
	RawData []byte

	Header *EventHeader
	Event  Event
}

func (e *BinlogEvent) Dump(w io.Writer) {
	e.Header.Dump(w)
	e.Event.Dump(w)
}

type Event interface {
	//Dump Event, format like python-mysql-replication
	Dump(w io.Writer)

	Decode(data []byte) error
}
```

写入`BinlogStreamer`中的消息，最后也会被写入到`Event`这个接口`Dump`函数中的`io.Writer`中


