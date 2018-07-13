## 第9章 预写式日志 - WAL{docsify-ignore}  

**事务日志**是数据库的重要组成部分，因为即使发生系统故障，也要求所有数据库管理系统都不会丢失任何数据。它是数据库系统中所有更改和操作的历史记录，以确保没有数据由于故障(如电源故障或导致服务器崩溃的其他服务器故障)而丢失。由于日志包含有关每个已执行事务的足够信息，因此在服务器崩溃的情况下，数据库服务器应能够通过重播事务日志中的更改和操作来恢复数据库集群。

在计算机科学领域，**WAL**是**Write Ahead Logging**的缩写，它是将变更和操作写入事务日志的协议或规则，而在PostgreSQL中，WAL是**Write Ahead Log**的缩写。在这里，该术语用作事务日志的同义词，也用于指与向事务日志(WAL)写入操作相关的实现机制。虽然这有点令人困惑，但在本文档中采用了PostgreSQL定义。

WAL机制在版本7.1中首次实现以减少服务器崩溃的影响。它还使实现时间点恢复(PITR)和流式复制(SR)成为可能，这两种方法分别在第10章和第11章中介绍。

虽然对于使用PostgreSQL的系统集成和管理来说，理解WAL机制是必不可少的，但是由于该机制的复杂性，不可能简单地对其描述进行总结。因此，对PostgreSQL中Wal的完整解释如下。在第一节中，介绍了WAL的总体情况，介绍一些重要的概念和关键词。在随后的章节中，将介绍以下主题： 

- WAL(事务日志)的逻辑和物理结构
- WAL数据的内部布局
- 写入WAL数据
- WAL writer 进程
- 检查点处理
- 数据库恢复处理
- 管理WAL段文件
- 连续归档

## 9.1. 概述

我们来看看WAL机制的概况。为了阐明WAL一直在处理的问题，第一部分展示了如果PostgreSQL没有实现WAL，发生崩溃时会发生什么。第二部分介绍了一些关键概念，并对本章的主要内容、WAL数据的编写和数据库恢复处理进行了概述。最后一小节完成了WAL的概述，增加了另一个关键概念。

在本节中，为了简化描述，使用了仅包含一个页面的表TABLE_A。

### 9.1.1. 没有WAL的插入操作

如第8章所述，为了提供对关系页面的有效访问，每个DBMS都实现共享缓冲池。

假设我们在PostgreSQL的TABLE_A中插入了一些数据元组，这些元组没有实现WAL特征; 这种情况如图9.1所示。

**图. 9.1. 没有WAL的插入操作**

![Fig. 9.1. Insertion operations without WAL.](https://github.com/yonj1e/interdb/blob/master/imgs/ch9/fig-9-01.png?raw=true)

(1)发出第一条INSERT语句，PostgreSQL将TABLE_A的页面从数据库集群加载到内存中的共享缓冲池中，并将一个元组插入到页面中。此页面不会立即写入数据库集群。(正如第8章所述，修改过的页面通常被称为脏页。)

(2)发出第二个INSERT语句，PostgreSQL在缓冲池的页面插入一个新的元组。此页面尚未写入存储。

(3)如果操作系统或PostgreSQL服务器由于任何原因(例如电源故障)而崩溃，则所有插入的数据都将丢失。

所以，没有WAL的数据库容易受到系统故障的影响。

### 9.1.2. 插入操作和数据库恢复

为了在不影响性能的前提下处理系统故障，PostgreSQL支持WAL。在本小节中，描述了一些关键字和关键概念，然后描述了WAL数据的写入和数据库的恢复。

PostgreSQL将所有修改作为历史数据写入永久存储器，以便为失败做好准备。在PostgreSQL中，历史数据被称为**XLOG记录**或**WAL数据**。

通过插入，删除或提交操作等更改操作将XLOG记录写入内存中的**WAL缓冲区**。当事务提交/中止时，它们会立即写入存储器中的**WAL段文件**。(具体而言，在其他情况下可能会写入XLOG记录，细节将在第9.5节中介绍)。XLOG记录的**LSN (Log Sequence Number)** 表示其记录写入事务日志的位置。记录的LSN被用作XLOG记录的唯一ID。

顺便说一下，当我们考虑数据库系统如何恢复时，可能会有一个问题; PostgreSQL开始从哪个点恢复？答案是**REDO点**;也就是在最新**检查点**开始的时候编写XLOG记录的位置(PostgreSQL中的检查点在9.7节中描述)。实际上，数据库恢复处理与检查点处理有很强的关联，并且这两种处理都是不可分割的。



> :pushpin: WAL和检查点在7.1版本中同时实现。



随着主要关键字和概念的引入基本完成，从现在起将描述有WAL的插入元组。参见图9.2和下面的描述。(另请参阅此[slide](http://www.slideshare.net/suzuki_hironobu/fig-902))

**图. 9.2. 有WAL的插入操作**

![Fig. 9.2. Insertion operations with WAL.](https://github.com/yonj1e/interdb/blob/master/imgs/ch9/fig-9-02.png?raw=true)



> :pushpin: Notation
>
> ‘*TABLE_A's LSN*’ 显示TABLE_A的page-header的pd_lsn的值。 ‘*page's LSN*’ 是一样的。



(1)checkpointer 进程，定期执行检查点设置。每当 checkpointer开始时，它就会将一个名为检查点记录的XLOG记录写入当前的WAL段。该记录包含最新REDO点的位置。

(2)发出第一条INSERT语句，PostgreSQL将TABLE_A的页面加载到共享缓冲池中，向页面中插入一个元组，并将此语句的XLOG记录创建并写入位于LSN_1的WAL缓冲区中，并更新TABLE_A的LSN从LSN_0到LSN_1。

在这个例子中，这个XLOG记录是一对头数据和整个元组。

(3)当这个事务提交时，PostgreSQL创建这个提交操作的XLOG记录并将其写入WAL缓冲区，然后将来自LSN_1的WAL缓冲区上的所有XLOG记录写入WAL段文件并将其刷新。

(4)发布第二个INSERT语句，PostgreSQL在页面中插入一个新的元组，并创建此元组的XLOG记录并将其写入LSN_2的WAL缓冲区，并将TABLE_A的LSN从LSN_1更新为LSN_2。

(5)当这个语句的事务提交时，PostgreSQL以和步骤(3)相同的方式运行。

(6)想象一下，当操作系统发生故障时。即使共享缓冲池中的所有数据都已丢失，但页面的所有修改都将作为历史数据写入WAL段文件。

以下说明显示了如何将数据库集群恢复到崩溃前的状态。没有必要做任何特殊的事情，因为PostgreSQL会通过重新启动自动进入恢复模式。参见图9.3(和这张[slide](http://www.slideshare.net/suzuki_hironobu/fig-903))。PostgreSQL将依次从REDO点读取并重放适当的WAL段文件中的XLOG记录。

**图 9.3. 使用WAL进行数据库恢复**

![Fig. 9.3. Database recovery using WAL.](https://github.com/yonj1e/interdb/blob/master/imgs/ch9/fig-9-03.png?raw=true)

(1)PostgreSQL从相应的WAL段文件中读取第一条INSERT语句的XLOG记录，将TABLE_A的页面从数据库集群加载到共享缓冲池中。

(2)在重放XLOG记录之前，PostgreSQL应将XLOG记录的LSN与相应页面的LSN进行比较，这将在9.8节中描述。下面显示了重放XLOG记录的规则。

如果XLOG记录的LSN大于页面的LSN，则将XLOG记录的数据部分插入页面，并将页面的LSN更新为XLOG记录的LSN。另一方面，如果XLOG记录的LSN较小，除了读取下一个WAL数据外没有其他任何操作。

在本例中，由于XLOG记录的LSN(LSN_1)大于TABLE_A的LSN(LSN_0)，所以XLOG记录会重播。那么TABLE_A的LSN从LSN_0更新为LSN_1。

(3)PostgreSQL以相同的方式重放剩余的XLOG记录。

PostgreSQL可以根据时间顺序重放按照WAL段文件编写的XLOG记录来恢复自身。因此，PostgreSQL的XLOG记录显然是**REDO日志**。

 

> :pushpin: PostgreSQL不支持UNDO日志。

 

虽然写XLOG记录肯定要有一定的消耗，但与编写整个修改过的页面相比，这算不了什么。我们确信我们能得到更大的好处，即系统的容错性。

### 9.1.3. 全页写(Full-Page writes)

假设存储中的TABLE_A的页面数据已经损坏，因为操作系统在background-writer进程写入脏页时失败。由于XLOG记录无法在损坏的页面上重放，因此我们需要一个附加的功能。

PostgreSQL支持称为**全页写full-page writes**的功能来处理这类故障。如果启用，PostgreSQL在每个检查点之后的每个页面首次更改期间将一对头数据和整个页面写为XLOG记录; 默认启用。在PostgreSQL中，包含整个页面的这种XLOG记录被称为**备份块 backup block**(或 **full-page image**)。

让我们再次描述元组的插入，但启用了full-page-writes。参见图9.4和下面的描述。

**图. 9.4. 全页写**

![Fig. 9.4. Full page writes.](https://github.com/yonj1e/interdb/blob/master/imgs/ch9/fig-9-04.png?raw=true)

(1)checkpointer启动检查点进程。

(2)插入第一个INSERT语句时，虽然PostgreSQL的操作方式与前一小节几乎相同，但这个XLOG记录是此页面的备份块(即它包含整个页)，因为这是在最新的检查点之后首次写入该页面。

(3)当这个事务提交时，PostgreSQL的运行方式与前一小节相同。

(4)插入第二个INSERT语句时，PostgreSQL的操作方式与上一小节相同，因为此XLOG记录不是备份块。

(5)当这个语句的事务提交时，PostgreSQL的操作方式与前一小节相同。

(6)为了演示整页写入的有效性，在这里我们考虑由于在background-writer将其写入HDD中时发生操作系统故障而导致存储器上的TABLE_A页面被损坏的情况。

重新启动PostgreSQL服务器以修复损坏的集群。参见图9.5和下面的描述。

**图. 9.5. 数据库恢复与备份块**

![Fig. 9.5. Database recovery with backup block.](https://github.com/yonj1e/interdb/blob/master/imgs/ch9/fig-9-05.png?raw=true)

(1)PostgreSQL读取第一个INSERT语句的XLOG记录，并将损坏的TABLE_A页面从数据库集群加载到共享缓冲池中。在本例中，XLOG记录是一个备份块，因为根据全页写的写入规则，每个页面的第一个XLOG记录始终是其备份块。

(2)当一个XLOG记录是它的备份块时，应用另一个重放规则：记录的数据部分(即页面本身)将被覆盖到页面上，而不管这两个LSN的值如何，并且页面的LSN被更新到XLOG记录的LSN。

在这个例子中，PostgreSQL将记录的数据部分覆盖到损坏的页面上，并将TABLE_A的LSN更新为LSN_1。通过这种方式，损坏的页面被其备份块恢复。

(3)由于第二个XLOG记录是非备份块，因此PostgreSQL的操作方式与前一小节中的说明相同。

即使发生了一些数据写入失败，PostgreSQL也可以恢复。(当然，如果发生文件系统或介质故障，这不适用。)

## 9.2. 事务日志和WAL段文件

从逻辑上讲，PostgreSQL将XLOG记录写入事务日志中，该事务日志是一个8字节长的虚拟文件(16 ExaByte)。

由于事务日志容量实际上是无限的，所以可以说8字节地址空间足够大，我们不可能处理8字节长的文件。因此，PostgreSQL中的事务日志被分成16MB的文件，每个文件被称为WAL段。参见图9.6。



> :pushpin: *WAL 段文件大小* 
>
> 在版本11或更高版本中，当通过initdb命令创建PostgreSQL集群时，可以使用[--wal-segsize](https://www.postgresql.org/docs/11/static/app-initdb.html)选项来配置WAL段文件的大小



**图. 9.6. 事务日志和WAL段文件**

![Fig. 9.6. Transaction log and WAL segment files](https://github.com/yonj1e/interdb/blob/master/imgs/ch9/fig-9-06.png?raw=true)

WAL段的文件名是十六进制的24位数字，命名规则如下：

​	WAL segment file name = timelineId + (uint32)$\frac{LSN−1}{16M∗256}$ + (uint32)($\frac{LSN−1}{16M}$)%256

 

> :pushpin: *timelineId*
>
> PostgreSQL的WAL包含**timelineId**(4字节无符号整数)的概念，该概念用于第10章中描述的时间点恢复(PITR)。但是，本章中timelineId固定为0x00000001，因为此概念在 以下说明。

 

 第一个WAL段文件是000000010000000000000001.如果第一个填写了XLOG记录，则会提供第二个000000010000000000000002。后续文件按照升序排列，0000000100000000000000FF填充后，将提供下一个000000010000000100000000。通过这种方式，每当最后两位数字结转时，中间的8位数字就增加一位。

同样，在0000000100000001000000FF填满后，将提供000000010000000200000000等等。



> :pushpin: *pg_xlogfile_name / pg_walfile_name*
>
> 使用内置函数pg_xlogfile_name(版本9.6或更低版本)或pg_walfile_name(版本10或更高版本)，我们可以找到包含指定LSN的WAL段文件名。一个例子如下所示：
>
>```sql
>testdb=# SELECT pg_xlogfile_name('1/00002D3E');  # In version 10 or >later, "SELECT pg_walfile_name('1/00002D3E');"
>     pg_xlogfile_name     
>--------------------------
> 000000010000000100000000
>(1 row)
>```




## 9.3. WAL段的内部结构

WAL段是一个16 MB的文件，它在内部分为8192个字节(8 KB)的页面。第一页有一个由结构[XLogLongPageHeaderData](javascript:void(0)) 定义的头数据，而所有其他页面的头具有由结构[XLogPageHeaderData](javascript:void(0)) 定义的页面信息。在页面头之后，XLOG记录按从低到高的顺序从头开始写入每个页面。参见图9.7。

图. 9.7. WAL段文件的内部布局

![Fig. 9.7. Internal layout of a WAL segment file.](https://github.com/yonj1e/interdb/blob/master/imgs/ch9/fig-9-07.png?raw=true)

XLogLongPageHeaderData结构和XLogPageHeaderData结构在[src/include/access/xlog_internal.h](https://github.com/postgres/postgres/blob/master/src/include/access/xlog_internal.h)中定义。两种结构的解释被省略，因为在以下描述中不需要这些解释。

## 9.4. XLOG记录的内部结构

XLOG记录包括通用头部分和相关数据部分。第一小节描述头结构; 剩余的两个小节分别解释9.4或更早版本和9.5版本中数据部分的结构。(数据格式在版本9.5中已更改。)

### 9.4.1. XLOG记录的头部分

All XLOG records have a general header portion defined by the structure XLogRecord. Here, the structure of 9.4 and earlier versions is shown in the following, though it is changed in version 9.5.

所有的XLOG记录都有一个由XLogRecord结构定义的通用头部分。这里，9.4和更早版本的结构如下所示，尽管它在版本9.5中进行了更改。

```c
typedef struct XLogRecord
{
   uint32          xl_tot_len;   /* total len of entire record */
   TransactionId   xl_xid;       /* xact id */
   uint32          xl_len;       /* total len of rmgr data */
   uint8           xl_info;      /* flag bits, see below */
   RmgrId          xl_rmid;      /* resource manager for this record */
   /* 2 bytes of padding here, initialize to zero */
   XLogRecPtr      xl_prev;      /* ptr to previous record in log */
   pg_crc32        xl_crc;       /* CRC for this record */
} XLogRecord;
```

除了两个变量之外，大多数变量都比较明显以至于不需要描述。

**xl_rmid**和**xl_info**都是与资源管理器相关的变量，这些资源管理器是与WAL功能相关的操作的集合，例如写入和重放XLOG记录。每个PostgreSQL版本都会增加资源管理器的数量，版本10包含以下内容：

| Operation                              | Resource manager                                           |
| -------------------------------------- | ---------------------------------------------------------- |
| Heap tuple operations                  | RM_HEAP, RM_HEAP2                                          |
| Index operations                       | RM_BTREE, RM_HASH, RM_GIN, RM_GIST, RM_SPGIST, RM_BRIN     |
| Sequence operations                    | RM_SEQ                                                     |
| Transaction operations                 | RM_XACT, RM_MULTIXACT, RM_CLOG, RM_XLOG, RM_COMMIT_TS      |
| Tablespace operations                  | RM_SMGR, RM_DBASE, RM_TBLSPC, RM_RELMAP                    |
| replication and hot standby operations | RM_STANDBY, RM_REPLORIGIN, RM_GENERIC_ID, RM_LOGICALMSG_ID |

以下是资源管理器在以下方面的一些代表性示例：

- 如果发出INSERT语句，则其XLOG记录的头变量xl_rmid和xl_info分别设置为'RM_HEAP'和'XLOG_HEAP_INSERT'。恢复数据库集群时，根据xl_info选择的RM_HEAP函数heap_xlog_insert()会重放此XLOG记录。
- 虽然它与UPDATE语句类似，但XLOG记录的头部变量xl_info设置为'XLOG_HEAP_UPDATE'，并且RM_HEAP的函数heap_xlog_update()在数据库恢复时重放其记录。
- 事务提交时，其XLOG记录的头变量xl_rmid和xl_info分别设置为'RM_XACT'和'XLOG_XACT_COMMIT'。恢复数据库集群时，函数xact_redo_commit()会重放此记录。

在版本9.5或更高版本中，一个变量(xl_len)已经从[XLogRecord](javascript:void(0)) 结构中移除以优化XLOG记录格式，该格式将大小减小了几个字节。

 

> :pushpin: 版本9.4或更低版本中的XLogRecord结构在[src/include/access/xlog.h](https://github.com/postgres/postgres/blob/REL9_4_STABLE/src/include/access/xlog.h)中定义，版本9.5或更高版本中的XLogRecord结构在[src/include/access/xlogrecord.h](https://github.com/postgres/postgres/blob/master/src/include/access/xlogrecord.h)中定义。
>
> heap_xlog_insert和heap_xlog_update在[src/backend/access/heap/heapam.c](https://github.com/postgres/postgres/blob/master/src/backend/access/heap/heapam.c)中定义; 而函数xact_redo_commit在[src/backend/access/transam/xact.c](https://github.com/postgres/postgres/blob/master/src/backend/access/transam/xact.c)中定义。

 

### 9.4.2. XLOG记录的数据部分(9.4或更低版本)

XLOG记录的数据部分分为备份块(整个页面)或非备份块(通过操作不同的数据)。

**图. 9.8. XLOG记录(版本9.4或更早版本)的示例**

![Fig. 9.8. Examples of XLOG records  (version 9.4 or earlier).](https://github.com/yonj1e/interdb/blob/master/imgs/ch9/fig-9-08.png?raw=true)

下面介绍XLOG记录的内部结构，给出一些具体示例。

#### 9.4.2.1. 备份块 backup block

图9.8(a)显示了备份块。它由两个数据结构和一个数据对象组成，如下所示：

1. 结构XLogRecord(头部分)
2. 结构 [BkpBlock](javascript:void(0))
3. 整个页面除了它的空闲空间外

[BkpBlock](javascript:void(0)) 包含用于在数据库集群中识别此页面的变量(即，包含此页面的关系的relfilenode和fork号以及此页面的块号)以及此页面的空闲空间的起始位置和长度。

#### 9.4.2.2. 非备份块 non-backup block

在非备份块中，数据部分的布局根据每个操作而不同。这里，INSERT语句的XLOG记录被解释为一个代表性的例子。参见图9.8(b)。在这种情况下，INSERT语句的XLOG记录由两个数据结构和一个数据对象组成，如下所示：

1. 结构XLogRecord(标题部分)
2. 结构 [xl_heap_insert](javascript:void(0))
3. 插入的元组 - 准确地说，从元组中移除几个字节

结构 [xl_heap_insert](javascript:void(0)) 包含用于标识数据库集群(即包含此元组的表的relfilenode，以及此元组的tid)中插入的元组的变量以及此元组的可见性标志。



> :pushpin: 在结构xl_heap_header的源代码注释中描述了从插入的元组中移除几个字节的原因：
>
> > 我们不在WAL中存储插入或更新元组的全部固定部分(HeapTupleHeaderData); 我们可以通过重构WAL记录中其他位置可用的字段来节省几个字节，或者干脆不需要重构。

 

在此显示另一个示例。参见图9.8(c)。检查点记录的XLOG记录非常简单; 它由两个数据结构组成，如下所示：

1. 结构XLogRecord(头部分)
2. 包含检查点信息的检查点结构(详见9.7节)

 

> :pushpin: xl_heap_header结构在[src/include/access/htup.h](https://github.com/postgres/postgres/blob/master/src/include/access/htup.h)中定义，而CheckPoint结构在[src/include/catalog/pg_control.h](https://github.com/postgres/postgres/blob/master/src/include/catalog/pg_control.h)中定义。

### 9.4.3. XLOG记录的数据部分(版本9.5或更高版本)

在版本9.4或更低版本中，没有XLOG记录的通用格式，因此每个资源管理器都必须定义自己的格式。在这种情况下，维护源代码和实现与WAL相关的新功能变得越来越困难。为了解决这个问题，在9.5版本中引入了一种不依赖资源管理器的通用结构化格式。

XLOG记录的数据部分可以分为两部分：头和数据。参见图9.9。

**图. 9.9. 常见的XLOG记录格式**

![Fig. 9.9. Common XLOG record format.](https://github.com/yonj1e/interdb/blob/master/imgs/ch9/fig-9-09.png?raw=true)

头部分包含零个或多个[XLogRecordBlockHeaders](javascript:void(0)) 和零个或一个 [XLogRecordDataHeaderShort](javascript:void(0)) (或XLogRecordDataHeaderLong); 它必须至少包含其中的一个。当其记录存储备份块时，XLogRecordBlockHeader包含[XLogRecordBlockImageHeader](javascript:void(0))，如果其块被压缩，还包括 [XLogRecordBlockCompressHeader](javascript:void(0))。

数据部分由零个或多个块数据和零个或一个主数据组成，分别对应XLogRecordBlockHeader和XLogRecordDataHeader。

 

> :pushpin: *WAL 压缩*
>
> 在版本9.5或更高版本中，可以使用LZ压缩方法通过设置参数wal_compression = enable来压缩XLOG记录内的备份块。在这种情况下，将添加结构XLogRecordBlockCompressHeader。
>
> 此功能有两个优点和一个缺点。优点是减少了写入记录和抑制WAL段文件消耗的I/O成本。缺点是耗费大量CPU资源来压缩。

 

**图. 9.10. XLOG记录(版本9.5或更高版本)的示例**

![Fig. 9.10. Examples of XLOG records  (version 9.5 or later).](https://github.com/yonj1e/interdb/blob/master/imgs/ch9/fig-9-10.png?raw=true)

一些具体的例子如前一小节所示。

#### 9.4.3.1. 备份块

INSERT语句创建的备份块如图9.10(a)所示。它由四个数据结构和一个数据对象组成，如下所示：

1. 结构XLogRecord(头部分)
2. 结构XLogRecordBlockHeader包含一个LogRecordBlockImageHeader
3. 结构XLogRecordDataHeaderShort
4. 备份块(块数据)
5. 结构xl_heap_insert(主数据)

XLogRecordBlockHeader包含变量来标识数据库集群中的块(relfilenode，fork号和块号); XLogRecordImageHeader包含此块的长度和偏移号。(这两个头结构一起可以存储相同的BkpBlock数据，直到版本9.4使用。)

XLogRecordDataHeaderShort存储记录的主要数据xl_heap_insert结构的长度。(见下文。)

 

> :pushpin: 除了在一些特殊情况下(例如在逻辑解码和推测性插入speculative insertions中)，不包含包含备份块的XLOG记录的主数据。当这个记录被重放时，它被忽略，这是冗余数据。今后可能会有所改进。
>
> 另外，备份块记录的主要数据取决于创建这些记录的语句。例如，UPDATE语句附加xl_heap_lock或xl_heap_updated。

#### 9.4.3.2. 非备份块

接下来，由INSERT语句创建的非备份块记录将被描述如下(参见图9.10(b))。它由四个数据结构和一个数据对象组成，如下所示：

1. 结构XLogRecord(头部分)
2. 结构XLogRecordBlockHeader
3. 结构XLogRecordDataHeaderShort
4. 一个插入的元组(确切地说，一个xl_heap_header结构和一个插入的数据整体)
5. 结构 [xl_heap_insert](javascript:void(0))(主数据)

XLogRecordBlockHeader包含三个值(relfilenode，fork number和block number)，用于指定插入元组的块以及所插入元组的数据部分的长度。XLogRecordDataHeaderShort包含新的xl_heap_insert结构的长度，这是该记录的主数据。

新的xl_heap_insert只包含两个值：该块内的该元组的偏移号和可见性标志;它变得非常简单，因为XLogRecordBlockHeader存储了旧数据中包含的大部分数据。

作为最后一个例子，图9.10(c)显示了检查点记录。它由三个数据结构组成，如下所示：

1. 结构XLogRecord(头部分)
2. 包含主数据长度的结构XLogRecordDataHeaderShort
3. 结构CheckPoint(主数据)

 

> :pushpin: 结构xl_heap_header在[src/include/access/htup.h](https://github.com/postgres/postgres/blob/master/src/include/access/htup.h) 中定义，CheckPoint结构在[src/include/catalog/pg_control.h](https://github.com/postgres/postgres/blob/master/src/include/catalog/pg_control.h)中定义。

 

虽然新格式对我们来说有点复杂，但是它为资源管理器的解析器设计得很好，而且许多类型的XLOG记录的大小通常比前一个小。主要结构的尺寸显示在图。9.8和9.10，所以你可以计算这些记录的大小并相互比较。(新检查点的大小大于前一个，但它包含更多变量。)

## 9.5. 写XLOG记录

完成热身练习后，现在我们已经准备好了解写XLOG记录。所以我会在本节中尽可能精确地解释它。

首先，发出以下语句来研究PostgreSQL内核：

```sql
testdb=# INSERT INTO tbl VALUES ('A');
```

通过发出上述语句，调用内部函数exec_simple_query(); exec_simple_query()的伪代码如下所示：

```c
exec_simple_query() @postgres.c

(1) ExtendCLOG() @clog.c                  /* Write the state of this transaction
                                           * "IN_PROGRESS" to the CLOG.
                                           */
(2) heap_insert()@heapam.c                /* Insert a tuple, creates a XLOG record,
                                           * and invoke the function XLogInsert.
                                           */
(3)   XLogInsert() @xlog.c (9.5 or later, xloginsert.c)
                                          /* Write the XLOG record of the inserted tuple
                                           *  to the WAL buffer, and update page's pd_lsn.
                                           */
(4) finish_xact_command() @postgres.c     /* Invoke commit action.*/   
      XLogInsert() @xlog.c  (9.5 or later, xloginsert.c)
                                          /* Write a XLOG record of this commit action 
                                           * to the WAL buffer.
                                           */
(5)   XLogWrite() @xlog.c                 /* Write and flush all XLOG records on 
                                           * the WAL buffer to WAL segment.
                                           */
(6) TransactionIdCommitTree() @transam.c  /* Change the state of this transaction 
                                           * from "IN_PROGRESS" to "COMMITTED" on the CLOG.
                                           */
```

在下面的段落中，为了理解写XLOG记录，将解释伪代码的每一行; 也参见图1和2。9.11和9.12。

(1)函数ExtendCLOG()将此事务的状态'IN_PROGRESS'写入(内存中)CLOG中。

(2)函数heap_insert()将堆元组插入共享缓冲池的目标页面，创建此页的XLOG记录，并调用函数XLogInsert()。

(3)函数XLogInsert()将由heap_insert()创建的XLOG记录写入LSN_1的WAL缓冲区，然后将修改后的页面的pd_lsn从LSN_0更新为LSN_1。

(4)为提交此事务而调用的函数finish_xact_command()创建此提交操作的XLOG记录，然后函数XLogInsert()将此记录写入LSN_2的WAL缓冲区中。

**图. 9.11. XLOG记录的写入顺序**

![Fig. 9.11. Write-sequence of XLOG records.](https://github.com/yonj1e/interdb/blob/master/imgs/ch9/fig-9-11.png?raw=true)

(5)函数XLogWrite()写入并刷新WAL缓冲区上的所有XLOG记录到WAL段文件。

如果参数wal_sync_method设置为'open_sync'或'open_datasync'，则会同步写入记录，因为该函数使用open()系统调用写入所有记录，并指定了标志O_SYNC或O_DSYNC。如果参数设置为'fsync'，'fsync_writethrough'或'fdatasync'，则将执行相应的系统调用-fsync()，fcntl()与F_FULLFSYNC选项或fdatasync()。无论如何，确保所有XLOG记录都写入存储。

(6)函数TransactionIdCommitTree()在CLOG上将此事务的状态从'IN_PROGRESS'更改为'COMMITTED'。

**图. 9.12. XLOG记录的写入顺序(从图9.11继续)**

![Fig. 9.12. Write-sequence of XLOG records (continued from Fig. 9.11).](https://github.com/yonj1e/interdb/blob/master/imgs/ch9/fig-9-12.png?raw=true)

在上面的示例中，提交操作导致将XLOG记录写入WAL段，但发生以下任何一种情况时可能会导致写入操作：

1. 一个正在运行的事务已经提交或已经中止。

2. WAL缓冲区已经写满了许多元组。(WAL缓冲区大小设置为参数[wal_buffers](https://www.postgresql.org/docs/current/static/runtime-config-wal.html#GUC-WAL-BUFFERS)。)
3. WAL编写器进程定期写入。(请参阅下一节。)

如果出现上述情况之一，WAL缓冲区上的所有WAL记录都会写入WAL段文件，而不管它们的事务是否已提交。

理所当然，DML(数据操作语言)操作编写XLOG记录，但非DML操作也是如此。如上所述，提交操作写入一个包含提交事务标识的XLOG记录。另一个例子可能是编写包含该检查点一般信息的XLOG记录的检查点操作。此外，SELECT语句在特殊情况下创建XLOG记录，尽管它通常不创建它们。例如，如果在SELECT语句处理过程中通过HOT(堆只有元组)删除不必要的元组并对页中必需的元组进行碎片整理，则修改页的XLOG记录将写入WAL缓冲区。

## 9.6. WAL Writer 进程

WAL writer 是后台进程，用于定期检查WAL缓冲区，并将所有未写入的XLOG记录写入WAL段。此过程的目的是避免XLOG记录的突发写入。如果此过程尚未启用，那么当一次提交大量数据时，XLOG记录的写入可能成为瓶颈。

WAL writer默认工作，不能被禁用。检查间隔由配置参数wal_writer_delay设置，默认值为200毫秒。

## 9.7. PostgreSQL中的检查点进程

在PostgreSQL中，检查点(后台)进程执行检查点操作； 其过程在发生以下情况之一时开始：

1. 自上一个检查点结束已经超过checkpoint_timeout设置的间隔时间(默认间隔为300秒(5分钟))。
2. 在版本9.4或更低版本中，自上一个检查点结束WAL段文件数量已经超过checkpoint_segments设置的文件数量(默认值为3)。
3. 在版本9.5或更高版本中，pg_xlog(版本10或更高版本，pg_wal)中的WAL段文件的总大小已超过参数max_wal_size的值(默认值为1GB(64个文件))。
4. PostgreSQL服务器以*smart*或*fast*模式停止。

当超级用户手动发出CHECKPOINT命令时，其进程也会执行此操作。

 

> :pushpin: 在9.1版本或更早的版本中，background writer进程同时进行了检查点和写脏页。



在以下小节中，将介绍检查点的概要和保存当前检查点的元数据的pg_control文件。

### 9.7.1. 检查点处理概述

检查点处理包括两个方面：准备数据库恢复以及刷共享缓冲池上的脏页。在本小节中，将重点介绍前者的内部处理。见图9.13和下面的描述。

**图. 9.13. PostgreSQL检查点的内部处理**

![Fig. 9.13. Internal processing of PostgreSQL's checkpoint.](https://github.com/yonj1e/interdb/blob/master/imgs/ch9/fig-9-13.png?raw=true)

(1)检查点处理开始后，REDO点存储在内存中; REDO点是在最新检查点开始的时刻写XLOG记录的位置，也是数据库恢复的起点。

(2)该检查点的XLOG记录(即检查点记录)被写入WAL缓冲区。记录的数据部分由结构 [CheckPoint](javascript:void(0)) t定义，该结构包含若干变量，如步骤(1)中存储的REDO点。

另外，写入检查点记录的位置实际上称为检查点。

(3)共享存储中的所有数据(例如，clog等的内容)被刷新到存储器。

(4)共享缓冲池中的所有脏页逐渐被写入并刷新到存储器中。

(5)pg_control文件被更新。该文件包含基本信息，例如检查点记录已经写入的位置(又名检查点位置checkpoint locatio)。稍后介绍这个文件的细节。

为了从数据库恢复的角度总结以上描述，检查点创建包含REDO点的检查点记录，并将检查点位置等存储到pg_control文件中。因此，PostgreSQL可以通过从pg_control文件提供的REDO点(从检查点记录中获取)重放WAL数据来恢复自身。

### 9.7.2. pg_control 文件

由于pg_control文件包含检查点的基本信息，因此对于数据库恢复来说，这是非常重要的。如果它已损坏或无法读取，恢复过程无法启动，无法获得起点。

尽管pg_control文件存储了超过40个项目，但下一部分中需要三个项目：

- **State** - 启动最新检查点时数据库服务器的状态。共有七种状态：start up'*是系统启动的状态; *'shut down'*是系统通过关机命令正常关机的状态; *'in production'* 是系统运行的状态;等等。
- **Latest checkpoint location** - LSN最新检查点记录的位置。
- **Prior checkpoint location** - LSN先前检查点记录的位置。请注意，它在版本11中已弃用;细节描述如下。

pg_control文件存储在base目录下的global子目录中;其内容可以使用pg_controldata工具显示。

```sql
postgres> pg_controldata  /usr/local/pgsql/data
pg_control version number:            937
Catalog version number:               201405111
Database system identifier:           6035535450242021944
Database cluster state:               in production
pg_control last modified:             Fri May 18 15:16:38 2018
Latest checkpoint location:           0/C000F48
Prior checkpoint location:            0/C000E70

... snip ...
```

 

> :pushpin: 在PostgreSQL 11中删除 prior checkpoint
>
> PostgreSQL 11或更高版本将只存储包含最新检查点或更新的WAL段; 将不存储包含先前检查点的旧段文件，以减少用于保存pg_xlog(pg_wal)子目录下的WAL段文件的磁盘空间。详细请参阅[此主题](http://www.postgresql-archive.org/Remove-secondary-checkpoint-tt5989050.html)。

 

## 9.8. PostgreSQL中的数据库恢复

PostgreSQL实现基于redo日志的恢复功能。如果数据库服务器崩溃，PostgreSQL通过从REDO点顺序地重放WAL段文件中的XLOG记录来恢复数据库集群。

在本节之前，我们已经多次讨论过数据库恢复，所以我将描述关于恢复的两件事，这两件事还没有解释过。

首先是PostgreSQL如何开始恢复过程。当PostgreSQL启动时，它首先读取pg_control文件。以下是从这一点开始的恢复处理的细节。参见图9.14和下面的描述。

**图. 9.14. 恢复过程的细节**

![Fig. 9.14. Details of the recovery process.](https://github.com/yonj1e/interdb/blob/master/imgs/ch9/fig-9-14.png?raw=true)

(1) PostgreSQL在启动时读取pg_control文件的所有项目。如果状态项目处于 *'in production'* 状态，PostgreSQL将进入恢复模式，因为这意味着数据库没有正常停止;如果 *'shut down'*，它将进入正常启动模式。

(2) PostgreSQL从相应的WAL段文件中读取最新的检查点记录，该记录位于pg_control文件中，并从记录中获取REDO点。如果最新的检查点记录无效，PostgreSQL会先读取它之前的记录。如果两个记录都不可读，它会自行恢复。(请注意，PostgreSQL 11后不再存储以前的检查点 prior checkpoint)

(3) 资源管理器从REDO点开始依次读取和重放XLOG记录，直到它们到达最新WAL段的最后一个点。当一个XLOG记录被重放并且它是一个备份块时，它将被覆盖在相应的表页上，而不管它的LSN如何。否则，只有当此记录的LSN大于相应页面的pd_lsn时，才会重放(非备份块)的XLOG记录。

第二点是关于LSN的比较：为什么应当比较非备份块的LSN和相应页面的pd_lsn。与前面的例子不同，它将用一个强调两个LSN之间比较的需要的特定例子来解释。参见图。9.15和9.16。(请注意，WAL缓冲区被省略以简化描述。)

**图. 9.15. 在background writer工作期间的插入操作**

![Fig. 9.15. Insertion operations during the background writer working.](https://github.com/yonj1e/interdb/blob/master/imgs/ch9/fig-9-15.png?raw=true)

(1) PostgreSQL将一个元组插入到TABLE_A中，并在LSN_1处写入一个XLOG记录。

(2) background-writer进程将TABLE_A的页面写入存储器。此时，此页面的pd_lsn为LSN_1。

(3) PostgreSQL在TABLE_A中插入一个新的元组，并在LSN_2处写入一个XLOG记录。修改后的页面尚未写入存储。

与概述中的示例不同，在此场景中，TABLE_A的页面曾被写入存储。

使用 immediate 模式关机，然后启动。

**图. 9.16. 数据库恢复**

![Fig. 9.16. Database recovery.](https://github.com/yonj1e/interdb/blob/master/imgs/ch9/fig-9-16.png?raw=true)

(1) PostgreSQL加载第一个XLOG记录和TABLE_A的页面，但不重放它，因为此记录的LSN不大于TABLE_A的LSN(两个值均为LSN_1)。事实上，一眼就可以清楚看到没有必要重放它。

(2) 接下来，PostgreSQL重放第二个XLOG记录，因为此记录的LSN(LSN_2)大于当前TABLE_A的LSN(LSN_1)。

从这个例子可以看出，如果非备份块的重放顺序不正确，或者非备份块重复出现多次，数据库集群将不再一致。总之，非备份块的redo(replay)操作不是幂等 *idempotent* 的。因此，要保留正确的重放顺序，当且仅当其LSN大于相应页面的pd_lsn时，才会重放非备份块记录。

另一方面，由于备份块的redo操作是幂等的，所以无论LSN如何，备份块都可以重放任意次数。

## 9.9. WAL段文件管理

PostgreSQL将XLOG记录写入存储在pg_xlog子目录(版本10或更高版本，pg_wal子目录中)中的一个WAL段文件中，并且在老版本已经填满时切换为新版本。WAL文件的数量取决于几个配置参数以及服务器活动。另外，他们的管理策略在9.5版本中得到了改进。

在下面的小节中，描述了WAL段文件的切换和管理。

### 9.9.1. WAL段文件切换

发生以下某种情况时会发生WAL段切换：

1. WAL段文件已经填满。
2. 调用函数[pg_switch_xlog](http://www.postgresql.org/docs/current/static/functions-admin.html#FUNCTIONS-ADMIN-BACKUP)。
3. [archive_mode](http://www.postgresql.org/docs/current/static/runtime-config-wal.html#GUC-ARCHIVE-MODE)已启用，并且超过了设置为[archive_timeout](http://www.postgresql.org/docs/current/static/runtime-config-wal.html#GUC-ARCHIVE-TIMEOUT)的时间。

切换文件通常会被回收(重新命名并重新使用)以供将来使用，但如果不必要，它可能会在以后删除。

### 9.9.2. WAL段文件管理(9.5版本或更高版本)

每当checkpoint开始时，PostgreSQL就会估计并准备下一个检查点周期所需的WAL段文件的数量。这种估计是根据以前检查点周期中消耗的文件数量来进行的。它们从包含先前REDO点的段开始计数，并且该值在min_wal_size(默认情况下，80 MB，即5个文件)和max_wal_size(1 GB，即64个文件)之间。如果checkpoint开始，必要的文件将被保留或回收，而不必要的文件将被删除。

图9.17给出了一个具体的例子。假设在checkpoint开始之前有六个文件，WAL_3包含之前的REDO点，PostgreSQL估计需要五个文件。在这种情况下，WAL_1将重命名为WAL_7以便回收，WAL_2将被删除。

 

> :pushpin: 可以删除比包含先前REDO点的文件早的文件，因为从第9.8节中描述的恢复机制可以看出，它们将永远不会被使用。

 

**图. 9.17. 在checkpoint时回收和移除WAL段文件**

![Fig. 9.17. Recycling and removing WAL segment files at a checkpoint.](https://github.com/yonj1e/interdb/blob/master/imgs/ch9/fig-9-17.png?raw=true)

如果由于WAL活动中的峰值而需要更多文件，则会在WAL文件的总大小小于max_wal_size时创建新文件。例如，在图9.18中，如果WAL_7已经填满，则WAL_8是新创建的。

**图. 9.18. 创建WAL段文件**

![Fig. 9.18. Creating WAL segment file.](https://github.com/yonj1e/interdb/blob/master/imgs/ch9/fig-9-18.png?raw=true)

WAL文件的数量根据服务器活动自适应地更改。如果WAL数据写入量不断增加，则WAL段文件的估计数量以及WAL文件的总大小也逐渐增加。在相反的情况下(即WAL数据写入的数量减少)，这些减少。

如果WAL文件的总大小超过max_wal_size，则将启动checkpoint。图9.19说明了这种情况。通过checkpoint，将创建一个新的REDO点，最后一个REDO点将是之前的点; 然后不必要的旧文件将被回收。通过这种方式，PostgreSQL将始终保存数据库恢复所需的WAL段文件。

**图. 9.19. checkpoint和回收WAL段文件**

![Fig. 9.19. Checkpointing and recycling WAL segment files.](https://github.com/yonj1e/interdb/blob/master/imgs/ch9/fig-9-19.png?raw=true)

配置参数[wal_keep_segments](http://www.postgresql.org/docs/current/static/runtime-config-replication.html#GUC-WAL-KEEP-SEGMENTS)和[复制槽](http://www.postgresql.org/docs/current/static/warm-standby.html#STREAMING-REPLICATION-SLOTS)功能也会影响WAL段文件的数量。

### 9.9.3. WAL段文件管理(9.4版本或更早版本)

WAL段文件的数量主要由以下三个参数控制：checkpoint_segments，checkpoint_completion_target和wal_keep_segments。其数量通常超过((2 + checkpoint_completion_target)×checkpoint_segments + 1)或(checkpoint_segments + wal_keep_segments + 1)个文件。根据服务器活动的不同，此数字可能暂时变为(3×checkpoint_segments + 1)个文件。复制插槽也会影响它们的数量。

如第9.7节所述，checkpoint 处理在checkpoint_segments文件的数量被消耗时发生。因此，确保WAL文件中始终包含两个或更多REDO点，因为文件数总是大于2×checkpoint_segments。如果超时发生，情况也是如此。因此，PostgreSQL将始终保存足够的WAL段文件(有时甚至超过必需的)以进行恢复。

 

> :pushpin: 在版本9.4或更早的版本中，参数checkpoint_segments是一个棘手的问题。如果设置的数量较少，则checkpoint会频繁发生，这会导致性能下降，而如果设置的数量较多，则WAL文件始终需要巨大的磁盘空间，其中一些并不总是必需的。
>
> 在V9.5中，WAL文件的管理策略得到了改进，并且checkpoint_segments已经过时。因此，上述折衷问题也已得到解决。

 

## 9.10. 连续归档和归档日志

**连续归档Continuous Archiving**是一种功能，可在WAL段切换时将WAL段文件复制到归档区域，并由归档(后台)进程执行。复制的文件称为**归档日志archive log**。此功能通常用于第10章中介绍的热物理备份和PITR(时间点恢复Point-in-Time Recovery)。

归档区域的路径设置通过配置参数archive_command。例如，使用以下参数，每当每个段切换时，WAL段文件都被复制到目录*'/home/postgres/archives/'* 中：

```shell
archive_command = 'cp %p /home/postgres/archives/%f'
```

其中，占位符 *%p* 被复制到WAL段，而 *%f* 是归档日志。

**图. 9.20. 连续归档**

![Fig. 9.20. Continuous archiving.](https://github.com/yonj1e/interdb/blob/master/imgs/ch9/fig-9-20.png?raw=true)

------

当切换WAL段文件WAL_7时，该文件作为 *Archive log 7* 被复制到归档区域。



参数archive_command可以设置任何Unix命令和工具，因此您可以通过设置scp命令或任何文件备份工具而不是普通的复制命令将存档日志传输到其他主机。

 

> :exclamation: PostgreSQL不会清理已创建的归档日志，因此在使用此功能时应正确管理日志。如果你什么都不做，归档日志的数量会不断增加。
>
> [pg_archivecleanup](http://www.postgresql.org/docs/current/static/pgarchivecleanup.html)实用程序是归档日志管理的有用工具之一。

