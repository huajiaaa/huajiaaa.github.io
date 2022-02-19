---
title: Prometheus tsdb
date: 2020-11-29 14:37:59
tags: ['prometheus', '大监控']
categories: 技术
toc: true
---

Prometheus 把监控数据存在本地磁盘的时间序列数据库中（ time series database, tsdb）, 同时也支持集成远程存储系统.

Prometheus 的 tsdb 经历过两次大升级,  从 v1 升到 v2 , 从 v2 升到 v3, 其中每次升级都比原来的版本有着巨大的改善.

目前最新的版本是 v3 tsdb ,  本文将对其进行介绍,

希望读者在读完本文后, 对 prometheus tsdb 整体设计能有比较清晰的认知、对自己感兴趣的实现细节能有一定的理解.

<!-- more -->

备注: 本文的内容是综合资料介绍以及个人理解总结出来的, 如果有理解不对或不准确的情况, 希望读者们能不吝指出.

# 数据库的选择/设计理念 & Prometheus 的场景需求

## **理念**

**该选择哪款数据库, 或者一个数据库被设计成什么样, 取决与预期怎么使用它(场景).** 

## Prometheus 的场景

### 样本点格式介绍

```
identifier -> (t0, v0), (t1, v1), (t2, v2), (t3, v3), ....
```

其中,

- identifier 是一个 metric name 以及其所有标签的键值对, 例如： requests_total{path="/status", method="GET", instance=”10.0.0.1:80”} 

- ti 表示时间点

- vi 表示 ti 时刻该 identifier 的取值

metric name 也被视为一个标签, 用\ \__${metricName}\_\_ 表示, 因此上述例子也可以表示为：{\_\_name\_\_="requests_total", path="/status", method="GET", instance=”10.0.0.1:80”}

### 场景需求

![](https://raw.githubusercontent.com/huajiaaa/huajiaaa.github.io/master/picture/prometheus%E8%AF%BB%E5%86%99%E7%A4%BA%E6%84%8F%E5%9B%BE.png)

如上图所示, 其中, 横轴是时间 t, 纵轴是 identifier, 平面上的点是 v：

1. 写需求
   1. 特别特别密集. 预期中, 一台 Prometheus 实例每秒钟应该能收集百万级别的样本点
   2. 只是纵向的写
2. 读需求
   1. 相比于写, 读需求很少
   2. 对 Latency 要求较高
   3. 有按时间范围查询的需求, 且一般会涉及多个 identifier 的查询
   4. 越新的数据越有价值, 查询的需求也越高

#### k8s 环境下特有的现象—— series churn

通常, metric 中都会有 instance 标签. 而在 k8s 环境下, 由于 pod 是经常变化的（比如滚动升级时）, 会导致「series churn」（不会翻译就不翻译了）, 如下图所示（即很多老 series 其实是不需要的）：

![](https://raw.githubusercontent.com/huajiaaa/huajiaaa.github.io/master/picture/series-churn.png)

介绍完了 Prometheus 的场景特点以及对数据库的使用需求, 接下来介绍 Prometheus 所使用的 tsdb 是怎样做的.

# 整体架构

![](https://ganeshvernekar.com/blog/img/tsdb1.svg)

其中, 

- (t, v) 即为采集的一个样本点（sample）                                                                
- Head block 在内存中
- 灰色的 block 在磁盘中（并且是不可变的）
- M-map 为磁盘中两个小时以内的数据在内存中的映射
- WAL 是预写日志（Write-ahead logging）



**接下来对监控样本点的生命周期进行简单介绍.**

# 样本点的生命周期

Prometheus 新采集的样本都会存到 Head block, 其中每一个 series 会存到唯一对应的压缩单元(即 chunk) 里. 为了防止 Prometheus 挂了导致内存中的数据丢失, 新采集到的数据还会写到预写日志中.如下图：

![](https://ganeshvernekar.com/blog/img/tsdb2.svg)

备注：样本点在存入 chunk 时, 通过 labels 的哈希值来获得或创建对应的 series chunk. 

**当 chunk 中存了 120 个样本或该 chunk 已满 2h 时, 将创建出一个新的 chunk, 老的 chunk 视为 "已满".**

ps: 如果 15s 采集一次, 则每 30min 会满一次.

如下图所示: 红色的是新创建的 chunk, 黄色的是老的 chunk.

![](https://ganeshvernekar.com/blog/img/tsdb4.svg)

自 Prometheus v2.19.0 之后（我们目前使用的是 v2.20.0）, 当一个 chunk "已满" 时, 它就会被刷新到磁盘中, 并从磁盘中进行内存映射（memory-map）, 仅在内存中存储一个它的引用. 有了内存映射, 可以再需要的时候通过引用动态地将 chunk 加载到内存中, 这是操作系统提供的特性, 参考: https://en.wikipedia.org/wiki/Mmap.

![](https://ganeshvernekar.com/blog/img/tsdb8.svg)

![](https://ganeshvernekar.com/blog/img/tsdb9.svg)

当 Head 中的数据跨度达到 3h 时（mmap 中数据的时间跨度）, 则最久的两个小时的数据(即上图的 1~4)将被压缩成一个 persistent block.

此时, 相关的 WAL 和索引数据也被删除.

再看一次这个图：

![](https://ganeshvernekar.com/blog/img/tsdb1.svg)

每个 block 是按时间序列排序的. 当查询多个 block 中, 会从很多 block 中读出对应的数据, 然后再合并成一个整体的结果, 这个合并过程显然是有代价的.

因此引入了压缩(*compaction*)操作：即将一个或多个 block 合并成一个更大的 block. 在压缩的过程中还可以修改现有数据, 例如删除「已被删除」的数据, 或者重新构造样本块以提高查询性能.

压缩的时机与设定的步长有关: 假设 block 为保存 2h 的数据, 如果步长设置为 3, 则会将三个 2h 的合成一个 6h 的block, 将三个 6h 的合成一个 18h 的block.

当 block 中存储的数据达到了所设置的最大保留时间时, 它们即会被删除.



**以上就是关于监控样本点的生命周期的简单介绍.**

# 磁盘上的数据格式

本节将分别对 Head Block 和 block 进行介绍.

## Head block

### File

上文介绍的 mmap 中的 chunks 保存在名为 chunks_head 的目录下, 文件序列与 WAL 中的相似. 如下图:

```
data
├── chunks_head
|   ├── 000001
|   └── 000002
└── wal
    ├── checkpoint.000003
    |   ├── 000000
    |   └── 000001
    ├── 000004
    └── 000005
```

其中, 文件（比如上图中的 000001）的 最大 size 是 128M. 每个的文件格式如下所示: 

```
┌──────────────────────────────┐
│  magic(0x0130BC91) <4 byte>  │
├──────────────────────────────┤
│    version(1) <1 byte>       │
├──────────────────────────────┤
│    padding(0) <3 byte>       │
├──────────────────────────────┤
│ ┌──────────────────────────┐ │
│ │         Chunk 1          │ │
│ ├──────────────────────────┤ │
│ │          ...             │ │
│ ├──────────────────────────┤ │
│ │         Chunk N          │ │
│ └──────────────────────────┘ │
└──────────────────────────────┘
```

其中. 

- magic number 是可以唯一标识一个文件时 head_chunk 的数字 (类似 Java 中的咖啡宝贝..)
- version 告诉我们如何解码文件中的 chunks (version 怎么告诉我们如何解码？version 其实是编码/格式的版本)
- padding 是为了将来可能需要的选项而预留出来的

备注：magic 在接下来其他的数据格式中会多次出现, 用途一致, 就不赘述了.

### Chunk

一个 chunk 的格式如下所示：

```
┌─────────────────────┬───────────────────────┬───────────────────────┬───────────────────┬───────────────┬──────────────┬────────────────┐
| series ref <8 byte> | mint <8 byte, uint64> | maxt <8 byte, uint64> | encoding <1 byte> | len <uvarint> | data <bytes> │ CRC32 <4 byte> │
└─────────────────────┴───────────────────────┴───────────────────────┴───────────────────┴───────────────┴──────────────┴────────────────┘
```

其中, 

- series ref 是用于访问内存中的序列的 id, 即上文中 mmap 中的引用
- mint 是该 chunk 中 series 的最小时间戳
- max 是该 chunk 中 series 的最大时间戳
- encoding 是压缩该 chunk 时使用的编码
- len 是此后的字节数
- data 是压缩后的数据
- CRC32 是用于检查数据完整性的校验和

Head Block 通过 series ref, 以及 mint、maxt 就可以实现不访问磁盘就选择 chunk.

其中, ref 是 8 bytes, 前 4 个字节告诉了 chunk 存在哪个文件中（file number）， 后四个字节告诉了 chunk 在该文件中的偏移量.

## Persistent Block

```
data
├── 01EM6Q6A1YPX4G9TEB20J22B2R
|   ├── chunks
|   |   ├── 000001
|   |   └── 000002
|   ├── index
|   ├── meta.json
|   └── tombstones
├── chunks_head
|   ├── 000001
|   └── 000002
└── wal
    ├── checkpoint.000003
    |   ├── 000000
    |   └── 000001
    ├── 000004
    └── 000005
```

上图中的 01EM6Q6A1YPX4G9TEB20J22B2R 即是一个 Persistent Block.

其中, 

- meta.json：block 的元信息
- chunks：chunk 的原始数据
- index：block 的索引
- tombstones：删除标记, 用于查询该 block 时排除样本
- 01EM6Q6A1YPX4G9TEB20J22B2R：block id （与 UUID 主要的区别是它是字典序的, 见https://github.com/oklog/ulid）

接下来分别进行介绍.

### meta.json

meta 中包含了整个 block 所需的所有元数据,  如下图：

```
{
    "ulid": "01EM6Q6A1YPX4G9TEB20J22B2R",
    "minTime": 1602237600000,
    "maxTime": 1602244800000,
    "stats": {
        "numSamples": 553673232,
        "numSeries": 1346066,
        "numChunks": 4440437
    },
    "compaction": {
        "level": 1,
        "sources": [
            "01EM65SHSX4VARXBBHBF0M0FDS",
            "01EM6GAJSYWSQQRDY782EA5ZPN"
        ]
    },
    "version": 1
}
```

其中, 

- version：索引格式的版本，告诉我们如何解析该 meta 文件
- ulid：尽管该 block 的目录名是 ULID, 但是实际上已 meta 中的 ulid 为准，目录名可以是任何名称.
- minTime/maxTime：该 block 中所有 chunk 的 min 和 max 时间戳
- stats：该 block 中存储的时间序列、样本和 chunk 的数量
- compaction：该 block 的历史
  - level：该 block 多少代了
  - sources：由哪些 blocks 合并而成, 如果是从 Head block 创建来的, 则 sources 设置为自己的 ULID

### Chunks

```
┌──────────────────────────────┐
│  magic(0x85BD40DD) <4 byte>  │
├──────────────────────────────┤
│    version(1) <1 byte>       │
├──────────────────────────────┤
│    padding(0) <3 byte>       │
├──────────────────────────────┤
│ ┌──────────────────────────┐ │
│ │         Chunk 1          │ │
│ ├──────────────────────────┤ │
│ │          ...             │ │
│ ├──────────────────────────┤ │
│ │         Chunk N          │ │
│ └──────────────────────────┘ │
└──────────────────────────────┘
```

与 Head 中的文件格式差不多, 不赘述了. 但有个区别是 Head 中的 file 最大 size 是 128M, 这里的 file 最大是 512M.

每个 chunk 的格式如下图所示：

```
┌───────────────┬───────────────────┬──────────────┬────────────────┐
│ len <uvarint> │ encoding <1 byte> │ data <bytes> │ CRC32 <4 byte> │
└───────────────┴───────────────────┴──────────────┴────────────────┘
```

与 head_chunk 差不多, 区别是少了 series ref、mint、maxt. 在 Head_chunk 中需要这些附加信息是为了 prometheus 重启时能够创建内存索引, 但是在持久化 block 中, 这些信息在 index 文件中存储了, 因此不再需要.
同样通过 reference 来访问这些 chunk. 

### index

index 是倒排索引, 它包含了查询该 block 所需要的所有信息. 它不与任何其它 block 或外部实体共享数据, 这使得可以在没有任何依赖的情况下读取/查询该 block.(这个结构比较复杂, 但不要慌 : ))

它的宏观视图如下所示：

```
┌────────────────────────────┬─────────────────────┐
│ magic(0xBAAAD700) <4b>     │ version(1) <1 byte> │
├────────────────────────────┴─────────────────────┤
│ ┌──────────────────────────────────────────────┐ │
│ │                 Symbol Table                 │ │
│ ├──────────────────────────────────────────────┤ │
│ │                    Series                    │ │
│ ├──────────────────────────────────────────────┤ │
│ │                 Label Index 1                │ │
│ ├──────────────────────────────────────────────┤ │
│ │                      ...                     │ │
│ ├──────────────────────────────────────────────┤ │
│ │                 Label Index N                │ │
│ ├──────────────────────────────────────────────┤ │
│ │                   Postings 1                 │ │
│ ├──────────────────────────────────────────────┤ │
│ │                      ...                     │ │
│ ├──────────────────────────────────────────────┤ │
│ │                   Postings N                 │ │
│ ├──────────────────────────────────────────────┤ │
│ │              Label Offset Table              │ │
│ ├──────────────────────────────────────────────┤ │
│ │             Postings Offset Table            │ │
│ ├──────────────────────────────────────────────┤ │
│ │                      TOC                     │ │
│ └──────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

其中,

- magic：用来唯一标识某个文件是 index 文件
- version：告诉我们如何解析该文件
- TOC：是该 index 文件的入口处（就像目录一样, 记录了各部分的页码）

#### TOC

```
┌─────────────────────────────────────────┐
│ ref(symbols) <8b>                       │ -> Symbol Table
├─────────────────────────────────────────┤
│ ref(series) <8b>                        │ -> Series
├─────────────────────────────────────────┤
│ ref(label indices start) <8b>           │ -> Label Index 1
├─────────────────────────────────────────┤
│ ref(label offset table) <8b>            │ -> Label Offset Table
├─────────────────────────────────────────┤
│ ref(postings start) <8b>                │ -> Postings 1
├─────────────────────────────────────────┤
│ ref(postings offset table) <8b>         │ -> Postings Offset Table
├─────────────────────────────────────────┤
│ CRC32 <4b>                              │
└─────────────────────────────────────────┘
```

格式如上图所示, 主要是告诉我们各索引开始的位置.

TOC 是固定大小的，因此文件的最后 52 个字节就是 TOC.

#### `Symbol Table`

`符号表, `记录了所有 series 的标签和值的字符串的非重复有序列表.
比如 一个 series 是 {a="y", x="b"}, 则符号会是 "a", "b", "x", "y"  .

```
┌────────────────────┬─────────────────────┐
│ len <4b>           │ #symbols <4b>       │
├────────────────────┴─────────────────────┤
│ ┌──────────────────────┬───────────────┐ │
│ │ len(str_1) <uvarint> │ str_1 <bytes> │ │
│ ├──────────────────────┴───────────────┤ │
│ │                . . .                 │ │
│ ├──────────────────────┬───────────────┤ │
│ │ len(str_n) <uvarint> │ str_n <bytes> │ │
│ └──────────────────────┴───────────────┘ │
├──────────────────────────────────────────┤
│ CRC32 <4b>                               │
└──────────────────────────────────────────┘
```

len：该部分的字节数
symbols：符号的数量
str_i：utf8 编码的字符串, 每个字符串前都有 len 前缀;  随后是字符串的原始值
索引中的其他部分可以为任何字符串引用此符号表, 由此减小 index 的大小. （通过偏移量 str_i 引用, 当需要实际的字符串时, 则通过偏移量从表中获取）

#### Series

```
┌───────────────────────────────────────┐
│ ┌───────────────────────────────────┐ │
│ │   series_1                        │ │
│ ├───────────────────────────────────┤ │
│ │                 . . .             │ │
│ ├───────────────────────────────────┤ │
│ │   series_n                        │ │
│ └───────────────────────────────────┘ │
└───────────────────────────────────────┘
```

此部分存了当前 block 中所有 series 的信息, 按照标签的字典序排序.

每个 series 都是 16 byte 对齐的, 使得 series 开始处的偏移量能够被 16 整除. 因此,将 series 的 id 设置为 offset/16, offset 指向 series 的开头. 每当要访问某个 series 时, 可以直接通过 id * 16 来获取在 index 中的位置.

#### Label Offset Table & Label Index i

这两个是耦合的, 因此应该放在一起介绍. 

但是, 目前这两个已经不再使用, 只是为了向后兼容而编写的. 所以本文暂且没有介绍.

#### Postings Offset Table & Postings i

##### Postings i

posting 其实就是 series id. (之所以叫 posting 其实是因为在倒排索引的 "世界" 里, 文档 id 常被成为 posting, 而在当前的场景下, 一个 series 可以被视为一个文档, 因此把 series id 当做 posting.) 单个 posting 其实代表了一个 posting list, 其格式如下所示:

```
┌────────────────────┬────────────────────┐
│ len <4b>           │ #entries <4b>      │
├────────────────────┴────────────────────┤
│ ┌─────────────────────────────────────┐ │
│ │ ref(series_1) <4b>                  │ │
│ ├─────────────────────────────────────┤ │
│ │ ...                                 │ │
│ ├─────────────────────────────────────┤ │
│ │ ref(series_n) <4b>                  │ │
│ └─────────────────────────────────────┘ │
├─────────────────────────────────────────┤
│ CRC32 <4b>                              │
└─────────────────────────────────────────┘
```

其中, len 和 CRC32 的作用大家都已经很熟悉了. entries 是该 list 中的 posting 数量; 其次是 entries 个有序的 posting（即 series id, 即引用）.

posting list 中存储的内容介绍：

假设有两个时间序列:

1. {a="b", x="y1"} with series ID 120
2. {a="b", x="y2"} with series ID 145

可以看到 a="b" 出现在两个 series 中, ; 而 x="y1", x="y2" 分别各自出现在一个 series 中.

此时 posting list 中会存:

posting 1：   // a="b" 在两个 series 都出现了

- 120
- 145

posting 2：//x ="y1"

- 120

posting 3:  //x ="y2"

- 145

##### Postings Offset Table

```
┌─────────────────────┬──────────────────────┐
│ len <4b>            │ #entries <4b>        │
├─────────────────────┴──────────────────────┤
│ ┌────────────────────────────────────────┐ │
│ │  n = 2 <1b>                            │ │
│ ├──────────────────────┬─────────────────┤ │
│ │ len(name) <uvarint>  │ name <bytes>    │ │
│ ├──────────────────────┼─────────────────┤ │
│ │ len(value) <uvarint> │ value <bytes>   │ │
│ ├──────────────────────┴─────────────────┤ │
│ │  offset <uvarint64>                    │ │
│ └────────────────────────────────────────┘ │
│                    . . .                   │
├────────────────────────────────────────────┤
│  CRC32 <4b>                                │
└────────────────────────────────────────────┘
```

其中, len 和 CRC32 的作用大家都已经很熟悉了. entries 是该表中的条目数.

n 永远是 2, 代表了接下来字符串的数量（label name 和 label value 俩字符串）.

接下来是 label name 和 label value 的原始值. **由于标签对通常并不多, 因此能够负担得起在此处存储原始字符串**，从而避免间接访问符号表（符号表的主要用途在于被 Series 部分使用）. PS：我们公司目前的标签对还是很多的.

offset 代表了该键值对在 postings list 中的偏移量.

以上述例子中,

- name="a", value="b" 的偏移量将指向 posting list 中的 [120, 145]
- name="x", value="y1" 的偏移量将指向 posting list 中的 [120]

Postings Offset Table 中的条目是根据 label name 和 label value 排序的, 因此可以对所需的标签对进行二分查找（这也是此处存储实际字符串以便于更快访问标签值的另一个原因）.



postings list 与 postings offset table 构成了倒排索引.



### tombstones

 Tombstones（墓碑） 是删除标记, 它告诉我们在读取/查询时可以忽略哪个 time series 的哪段时间范围. 

```
┌────────────────────────────┬─────────────────────┐
│ magic(0x0130BA30) <4b>     │ version(1) <1 byte> │
├────────────────────────────┴─────────────────────┤
│ ┌──────────────────────────────────────────────┐ │
│ │                Tombstone 1                   │ │
│ ├──────────────────────────────────────────────┤ │
│ │                      ...                     │ │
│ ├──────────────────────────────────────────────┤ │
│ │                Tombstone N                   │ │
│ ├──────────────────────────────────────────────┤ │
│ │                  CRC<4b>                     │ │
│ └──────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

magic、version、CRC 的作用大家都很熟悉了.

Tombstone 的格式如下：

```
┌────────────────────────┬─────────────────┬─────────────────┐
│ series ref <uvarint64> │ mint <varint64> │ maxt <varint64> │
└────────────────────────┴─────────────────┴─────────────────┘
```

# 其它

还有一些内容也属于 tsdb 中需要了解的范畴, 比如[数据压缩的方法](https://confluence.zhenguanyu.com/www.vldb.org/pvldb/vol8/p1816-teller.pdf)等, 但不是目前必须要知道的内容, 因此本文中暂时没有介绍, 感兴趣的话可以自行去了解.

# 结合上述分析和部分源码, 查询 tsdb 时发生了什么？

## 前置介绍

### 一个 Block 即是一个 NewBlockQuerier.

见下 1424 行.

```
// Querier returns a new querier over the data partition for the given time range.
func (db *DB) Querier(_ context.Context, mint, maxt int64) (storage.Querier, error) {
	var blocks []BlockReader

	db.mtx.RLock()
	defer db.mtx.RUnlock()

	for _, b := range db.blocks {
		if b.OverlapsClosedInterval(mint, maxt) {
			blocks = append(blocks, b)
		}
	}
	if maxt >= db.head.MinTime() {
		blocks = append(blocks, NewRangeHead(db.head, mint, maxt))
	}

	blockQueriers := make([]storage.Querier, 0, len(blocks))
	for _, b := range blocks {
		q, err := NewBlockQuerier(b, mint, maxt)                      //为 block 创建 NewBlockQuerier, 构造函数见下一个代码块
		if err == nil {
			blockQueriers = append(blockQueriers, q)
			continue
		}
		// If we fail, all previously opened queriers must be closed.
		for _, q := range blockQueriers {
			// TODO(bwplotka): Handle error.
			_ = q.Close()
		}
		return nil, errors.Wrapf(err, "open querier for block %s", b)
	}
	return storage.NewMergeQuerier(blockQueriers, nil, storage.ChainedSeriesMerge), nil
}
```

```
// NewBlockQuerier returns a querier against the block reader and requested min and max time range.
func NewBlockQuerier(b BlockReader, mint, maxt int64) (storage.Querier, error) {
	q, err := newBlockBaseQuerier(b, mint, maxt)          // newBlockBaseQuerier 构造函数见下一个代码块
	if err != nil {
		return nil, err
	}
	return &blockQuerier{blockBaseQuerier: q}, nil
}
```

```
func newBlockBaseQuerier(b BlockReader, mint, maxt int64) (*blockBaseQuerier, error) {
	indexr, err := b.Index()
	if err != nil {
		return nil, errors.Wrap(err, "open index reader")
	}
	chunkr, err := b.Chunks()
	if err != nil {
		indexr.Close()
		return nil, errors.Wrap(err, "open chunk reader")
	}
	tombsr, err := b.Tombstones()
	if err != nil {
		indexr.Close()
		chunkr.Close()
		return nil, errors.Wrap(err, "open tombstone reader")
	}

	if tombsr == nil {
		tombsr = tombstones.NewMemTombstones()
	}
	return &blockBaseQuerier{
		mint:       mint,
		maxt:       maxt,
		index:      indexr,
		chunks:     chunkr,
		tombstones: tombsr,
	}, nil
}
```

### Querier 通过调用 Select 方法返回 series set

```
// Querier provides querying access over time series data of a fixed time range.
type Querier interface {
	LabelQuerier

	// Select returns a set of series that matches the given label matchers.
	// Caller can specify if it requires returned series to be sorted. Prefer not requiring sorting for better performance.
	// It allows passing hints that can help in optimising select, but it's up to implementation how this is used if used at all.
	Select(sortSeries bool, hints *SelectHints, matchers ...*labels.Matcher) SeriesSet
}
```

### 在 Select 时, 会根据 PromQL 中的标签条件进行匹配, 最终调用 Postings 方法获得 series ids

```
func (q *blockQuerier) Select(sortSeries bool, hints *storage.SelectHints, ms ...*labels.Matcher) storage.SeriesSet {
	mint := q.mint
	maxt := q.maxt
	p, err := PostingsForMatchers(q.index, ms...)    // 在这一步即获得了 postings, 即 series ids, 见下一个代码块
	if err != nil {
		return storage.ErrSeriesSet(err)
	}
	if sortSeries {
		p = q.index.SortedPostings(p)
	}

	if hints != nil {
		mint = hints.Start
		maxt = hints.End
		if hints.Func == "series" {
			// When you're only looking up metadata (for example series API), you don't need to load any chunks.
			return newBlockSeriesSet(q.index, newNopChunkReader(), q.tombstones, p, mint, maxt)
		}
	}

	return newBlockSeriesSet(q.index, q.chunks, q.tombstones, p, mint, maxt)
}
```

```
// PostingsForMatchers assembles a single postings iterator against the index reader
// based on the given matchers. The resulting postings are not ordered by series.
func PostingsForMatchers(ix IndexReader, ms ...*labels.Matcher) (index.Postings, error) {
	var its, notIts []index.Postings
	// See which label must be non-empty.
	// Optimization for case like {l=~".", l!="1"}.
	labelMustBeSet := make(map[string]bool, len(ms))
	for _, m := range ms {
		if !m.Matches("") {
			labelMustBeSet[m.Name] = true
		}
	}

	for _, m := range ms {
		if labelMustBeSet[m.Name] {
			// If this matcher must be non-empty, we can be smarter.
			matchesEmpty := m.Matches("")
			isNot := m.Type == labels.MatchNotEqual || m.Type == labels.MatchNotRegexp
			if isNot && matchesEmpty { // l!="foo"
				// If the label can't be empty and is a Not and the inner matcher
				// doesn't match empty, then subtract it out at the end.
				inverse, err := m.Inverse()
				if err != nil {
					return nil, err
				}

				it, err := postingsForMatcher(ix, inverse)                          //见下一个代码块
				if err != nil {
					return nil, err
				}
				notIts = append(notIts, it)
			} else if isNot && !matchesEmpty { // l!=""
				// If the label can't be empty and is a Not, but the inner matcher can
				// be empty we need to use inversePostingsForMatcher.
				inverse, err := m.Inverse()
				if err != nil {
					return nil, err
				}

				it, err := inversePostingsForMatcher(ix, inverse)            //见下下个代码块
				if err != nil {
					return nil, err
				}
				its = append(its, it)
			} else { // l="a"
				// Non-Not matcher, use normal postingsForMatcher.
				it, err := postingsForMatcher(ix, m)                           //见下一个代码块
				if err != nil {
					return nil, err
				}
				its = append(its, it)
			}
		} else { // l=""
			// If the matchers for a labelname selects an empty value, it selects all
			// the series which don't have the label name set too. See:
			// https://github.com/prometheus/prometheus/issues/3575 and
			// https://github.com/prometheus/prometheus/pull/3578#issuecomment-351653555
			it, err := inversePostingsForMatcher(ix, m)                    //见下下个代码块
			if err != nil {
				return nil, err
			}
			notIts = append(notIts, it)
		}
	}

	// If there's nothing to subtract from, add in everything and remove the notIts later.
	if len(its) == 0 && len(notIts) != 0 {
		k, v := index.AllPostingsKey()
		allPostings, err := ix.Postings(k, v)                          //最终都会调用 Postings 方法
		if err != nil {
			return nil, err
		}
		its = append(its, allPostings)
	}

	it := index.Intersect(its...)

	for _, n := range notIts {
		it = index.Without(it, n)
	}

	return it, nil
}
```

```
func postingsForMatcher(ix IndexReader, m *labels.Matcher) (index.Postings, error) {
	// This method will not return postings for missing labels.

	// Fast-path for equal matching.
	if m.Type == labels.MatchEqual {
		return ix.Postings(m.Name, m.Value)            //最终都会调用 Postings 方法
	}

	// Fast-path for set matching.
	if m.Type == labels.MatchRegexp {
		setMatches := findSetMatches(m.GetRegexString())
		if len(setMatches) > 0 {
			sort.Strings(setMatches)
			return ix.Postings(m.Name, setMatches...)
		}
	}

	vals, err := ix.LabelValues(m.Name)
	if err != nil {
		return nil, err
	}

	var res []string
	lastVal, isSorted := "", true
	for _, val := range vals {
		if m.Matches(val) {
			res = append(res, val)
			if isSorted && val < lastVal {
				isSorted = false
			}
			lastVal = val
		}
	}

	if len(res) == 0 {
		return index.EmptyPostings(), nil
	}

	if !isSorted {
		sort.Strings(res)
	}
	return ix.Postings(m.Name, res...)
}
```

```
// inversePostingsForMatcher returns the postings for the series with the label name set but not matching the matcher.
func inversePostingsForMatcher(ix IndexReader, m *labels.Matcher) (index.Postings, error) {
	vals, err := ix.LabelValues(m.Name)
	if err != nil {
		return nil, err
	}

	var res []string
	lastVal, isSorted := "", true
	for _, val := range vals {
		if !m.Matches(val) {
			res = append(res, val)
			if isSorted && val < lastVal {
				isSorted = false
			}
			lastVal = val
		}
	}

	if !isSorted {
		sort.Strings(res)
	}
	return ix.Postings(m.Name, res...)
}
```

## 正式开始吧

如 "前置介绍" 中所说, 在查询时最终都会调用 block 的 postings 方法. 接下来分别对 Head Block 中的 postings 方法和 Persistent Block 中的 postings 方法进行介绍.

### Head#Postings

```
// Postings returns the postings list iterator for the label pairs.
func (h *headIndexReader) Postings(name string, values ...string) (index.Postings, error) {
	res := make([]index.Postings, 0, len(values))
	for _, value := range values {
		res = append(res, h.head.postings.Get(name, value))  //直接通过 postings.Get 获得
	}
	return index.Merge(res...), nil
}
```

```
// Get returns a postings list for the given label pair.
func (p *MemPostings) Get(name, value string) Postings {
	var lp []uint64
	p.mtx.RLock()
	l := p.m[name]
	if l != nil {
		lp = l[value]
	}
	p.mtx.RUnlock()

	if lp == nil {
		return EmptyPostings()
	}
	return newListPostings(lp...)
}
```

```
// MemPostings holds postings list for series ID per label pair. They may be written
// to out of order.
// ensureOrder() must be called once before any reads are done. This allows for quick
// unordered batch fills on startup.
type MemPostings struct {
	mtx     sync.RWMutex
	m       map[string]map[string][]uint64
	ordered bool
}
```

总结：Head 的 Postings 方法很简单, 直接通过 mmap 即可获得某个标签键值对的 postings.

### Block#Postings

```
func (r blockIndexReader) Postings(name string, values ...string) (index.Postings, error) {
	p, err := r.ir.Postings(name, values...)        // 调用 Index 的 Postings
	if err != nil {
		return p, errors.Wrapf(err, "block: %s", r.b.Meta().ULID)
	}
	return p, nil
}
```

```
func (r *Reader) Postings(name string, values ...string) (Postings, error) {
	if r.version == FormatV1 {          // 我们现在应该是 V2, 可忽略
		e, ok := r.postingsV1[name]
		if !ok {
			return EmptyPostings(), nil
		}
		res := make([]Postings, 0, len(values))
		for _, v := range values {
			postingsOff, ok := e[v]
			if !ok {
				continue
			}
			// Read from the postings table.
			d := encoding.NewDecbufAt(r.b, int(postingsOff), castagnoliTable)
			_, p, err := r.dec.Postings(d.Get())
			if err != nil {
				return nil, errors.Wrap(err, "decode postings")
			}
			res = append(res, p)
		}
		return Merge(res...), nil
	}

	e, ok := r.postings[name]       // Reader 的内容见下一个代码块, 即获得该 label 在 offset table 中的开始位置
	if !ok {
		return EmptyPostings(), nil
	}

	if len(values) == 0 {
		return EmptyPostings(), nil
	}

	res := make([]Postings, 0, len(values))
	skip := 0
	valueIndex := 0
	for valueIndex < len(values) && values[valueIndex] < e[0].value { // 遍历直至找到开始的位置.
	// 根据上文的宏观介绍, offset table 中 label 的 key 和 value 都是有序的
	// 因此 values[valueIndex] < e[0].value 即代表查询的标签值在当前 block 中不存在/未命中

		// Discard values before the start.
		valueIndex++
	}
	for valueIndex < len(values) {
		value := values[valueIndex]

        //二分查找
		i := sort.Search(len(e), func(i int) bool { return e[i].value >= value })
	 	 
		if i == len(e) {
			// We're past the end.
			break
		}
		if i > 0 && e[i].value != value {
			// Need to look from previous entry.
			i--
		}
		// Don't Crc32 the entire postings offset table, this is very slow
		// so hope any issues were caught at startup.
		d := encoding.NewDecbufAt(r.b, int(r.toc.PostingsTable), nil)
		d.Skip(e[i].off)

		// Iterate on the offset table.
		var postingsOff uint64 // The offset into the postings table.
		for d.Err() == nil {
			if skip == 0 {
				// These are always the same number of bytes,
				// and it's faster to skip than parse.
				skip = d.Len()
				d.Uvarint()      // Keycount.
				d.UvarintBytes() // Label name.
				skip -= d.Len()
			} else {
				d.Skip(skip)
			}
			v := d.UvarintBytes()       // Label value.
			postingsOff = d.Uvarint64() // Offset.
			for string(v) >= value {
				if string(v) == value {  // 如果标签值匹配上了
					// Read from the postings table.
					d2 := encoding.NewDecbufAt(r.b, int(postingsOff), castagnoliTable)
					_, p, err := r.dec.Postings(d2.Get())    // 解码
					if err != nil {
						return nil, errors.Wrap(err, "decode postings")
					}
					res = append(res, p)    // postings 加入 result list
				}
				valueIndex++
				if valueIndex == len(values) {
					break
				}
				value = values[valueIndex]
			}
			if i+1 == len(e) || value >= e[i+1].value || valueIndex == len(values) {
				// Need to go to a later postings offset entry, if there is one.
				break
			}
		}
		if d.Err() != nil {
			return nil, errors.Wrap(d.Err(), "get postings offset entry")
		}
	}

	return Merge(res...), nil
}
```

```
type Reader struct {
	b   ByteSlice
	toc *TOC

	// Close that releases the underlying resources of the byte slice.
	c io.Closer

	// Map of LabelName to a list of some LabelValues's position in the offset table.
	// The first and last values for each name are always present.
	postings map[string][]postingOffset
	// For the v1 format, labelname -> labelvalue -> offset.
	postingsV1 map[string]map[string]uint64

	symbols     *Symbols
	nameSymbols map[uint32]string // Cache of the label name symbol lookups,
	// as there are not many and they are half of all lookups.

	dec *Decoder

	version int
}
```

总结：

- 通过 postings[name] 获得 label name 所对应的 values 的在 offset table 中开始的位置
- 首先根据上述 "开始的位置" 的标签的值过滤掉未命中的 label values（因为有序）
- 遍历所有需要查询的标签值
  - 在 offset table 中进行二分查找, 找到当前标签值在 offset table 中的上界（上界是标签值, 或者前一个是标签值, 或者 miss）
  - 通过 toc 找到 posting offet table， 通过上边找到的 offset 获得对应的 series ids
  - 通过 series ids 反解出原始的标签值
    - 原始标签值与要查找的标签值对比
    - 如果相等则把该 series id 加入到 res 中
    - 不相等则说明二分查找 miss
- merge res

# 参考资料

1. https://prometheus.io/docs/prometheus/latest/storage/
2. https://fabxc.org/tsdb/
3. https://ganeshvernekar.com/blog/prometheus-tsdb-the-head-block
4. https://ganeshvernekar.com/blog/prometheus-tsdb-persistent-block-and-its-index
5. https://github.com/prometheus/prometheus/blob/master/tsdb/docs/format/index.md
6. https://github.com/prometheus/prometheus/blob/master/tsdb/head.go
7. https://github.com/prometheus/prometheus/blob/2c4a4548a8382f7c8966dbfda37f34c43151e316/storage/series.go
8. https://github.com/prometheus/prometheus/tree/a282d2509971174301408c5a5f96946c478dfb0f/tsdb/index