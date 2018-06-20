## 第8章 缓冲区管理器(buffer manager)

缓冲区管理器管理共享内存和永久存储之间的数据传输，并可能对DBMS的性能产生重大影响。 PostgreSQL缓冲区管理器的工作效率非常高。

在本章中，描述了PostgreSQL缓冲区管理器。 第一部分提供了一个概述，随后的部分描述了以下主题：

- 缓冲区管理器结构
- 缓冲区管理器锁
- 缓冲区管理器如何工作
- 环形缓冲器(Ring buffer)
- 刷新脏页 

**图. 8.1. 缓冲区管理器，存储和后端进程之间的关系**

![Fig. 8.1. Relations between buffer manager, storage, and backend processes.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch8/fig-8-01.png?raw=true)



## 8.1. 概述

本节介绍后续章节中描述所需的关键概念。

### 8.1.1. 缓冲管理器结构

PostgreSQL缓冲区管理器包含缓冲表(buffer table)，缓冲区描述符(buffer descriptors)和缓冲池(buffer pool)，这些将在下一节中介绍。 缓冲池存储数据文件页面，如表和索引，以及空闲空间映射表和可见性映射表。 缓冲池是一个数组，即每个槽存储数据文件的一个页。 缓冲池数组的索引被称为**buffer_id**s。

8.2节和8.3节描述了缓冲区管理器的内部细节。

### 8.1.2. Buffer Tag

在PostgreSQL中，所有数据文件的每一页都可以分配一个唯一的tag，即一个**buffer tag**。 当缓冲区管理器收到请求时，PostgreSQL使用所需页面的buffer_tag。

buffer_tag包含三个值：其page页的relfilenode和fork号及其页面的块号。 例如，buffer_tag'(17721，0，7)'标识第七块中的OID和fork号分别为17721和0的页面(表的fork号为0)。相似地，buffer_tag'(17721，1，3)'标识位于空闲空间映射表的第三个块中的页面，其OID和fork号分别是17721和1(空闲空间映射表的fork号是1)。

### 8.1.3. 后端进程如何读取页面

本小节介绍后端进程如何从缓冲区管理器中读取页面(图8.2)。

**图. 8.2. 后端如何从缓冲区管理器中读取页面**

![Fig. 8.2. How a backend reads a page from the buffer manager.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch8/fig-8-02.png?raw=true)

(1) 当读取表或索引页时，后端进程向缓冲区管理器发送一个请求，该请求包含页面的buffer_tag。

(2) 缓冲区管理器返回存储所请求页面的槽位的buffer_ID。 如果所请求的页面未存储在缓冲池中，则缓冲区管理器将页面从永久存储器加载到其中一个缓冲池插槽，然后返回buffer_ID的插槽。

(3) 后端进程访问buffer_ID的插槽(以读取所需的页面)。

当后端进程修改缓冲池中的页面(例如，通过插入元组)时，尚未刷新到存储器的修改页被称为**脏页dirty page**.。

8.4节描述了缓冲区管理器的工作原理

### 8.1.4. 页面替换算法

当所有缓冲池插槽被占用但所请求的页面未被存储时，缓冲区管理器必须在缓冲池中选择一个页面，该页面将被请求的页面替换。 通常，在计算机科学领域中，*page replacement algorithms*被称为页面替换算法，所选页面被称为**victim page**。

自从计算机科学出现以来，关于页面替换算法的研究一直在进行; 因此，以前已经提出了许多替换算法。 自8.1版以来，PostgreSQL使用**clock sweep**算法，因为它比以前版本中使用的LRU算法更简单，更高效。

8.4.4节描述了clock-sweep的细节。

### 8.1.5. 刷新脏页

脏页最终应该刷新到存储空间; 但是，缓冲区管理器需要帮助才能执行此任务。 在PostgreSQL中，两个后台进程**checkpointer** 和 **background writer** 负责这个任务。

8.6节描述了checkpointer 和 background writer。

 

> :pushpin: *Direct I/O*

> PostgreSQL不支持 direct I/O，尽管有时会讨论它。 如果你想知道更多的细节，请参考关于pgsql-ML和[本文](http://www.postgresql.org/message-id/529E267F.4050700@agliodbs.com)的讨论。

 

## 8.2. 缓冲管理器结构

PostgreSQL缓冲区管理器包含三个层，即缓冲表、缓冲描述符和缓冲池。(图8.3)：

**图. 8.3. 缓冲区管理器的三层结构**

![Fig. 8.3. Buffer manager's three-layer structure.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch8/fig-8-03.png?raw=true)

- **缓冲池** 是一个数组。 每个插槽都存储一个数据文件页面。 数组插槽的索引被称为*buffer_id*s。
- **缓冲描述符** 是一个缓冲区描述符数组。 每个描述符与缓冲池槽位具有一对一的对应关系，并将存储的页面的元数据保存在相应的槽中。
     请注意，使用‘buffer descriptors layer’这一术语是为了方便起见，仅在本文档中使用。
- **缓冲表** 是一个哈希表，它存储了存储页面的 *buffer_tag* 和存储页面的各个元数据的描述符的 *buffer_id*s 之间的关系。

以下小节详细介绍了这些层。

### 8.2.1. 缓冲表

一个缓冲表可以在逻辑上分为三部分：哈希函数(hash function)，哈希桶插槽(hash bucket slots) 和数据项(data entries) (图8.4)。

内置的哈希函数将buffer_tags映射到哈希桶插槽。 即使哈希桶插槽的数量大于缓冲池插槽的数量，也可能发生冲突。 因此，缓冲表使用带有链接列表的单独链接方法来解决冲突。  当数据条目映射到同一个存储槽时，此方法将条目存储在同一个链接列表中，如图8.4所示。

**图 8.4. 缓冲表**

![Fig. 8.4. Buffer table.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch8/fig-8-04.png?raw=true)

数据条目包含两个值：页面的buffer_tag和保存页面元数据的描述符的buffer_id。 例如，数据条目'Tag_A，id = 1'意味着具有buffer_id 1的缓冲区描述符存储标记为Tag_A的页面的元数据。

 

> :pushpin: *Hash function*

> 哈希函数是 [calc_bucket()](https://doxygen.postgresql.org/dynahash_8c.html#ae802f2654df749ae0e0aadf4b5c5bcbd) 和 [hash()](https://doxygen.postgresql.org/rege__dfa_8c.html#a6aa3a27e7a0fc6793f3329670ac3b0cb) 的复合函数。 以下是它作为伪函数的表示。

> ```c
> uint32 bucket_slot = calc_bucket(unsigned hash(BufferTag buffer_tag), uint32 bucket_size)
> ```



请注意，这里没有说明基本操作(查找，插入和删除数据条目)。 这些是非常常见的操作，并在以下各节中进行解释。

### 8.2.2. 缓冲区描述符

本小节描述了缓冲区描述符的结构，下一小节描述了缓冲区描述符层。

缓冲区描述符将存储页面的元数据保存在相应的缓冲池槽位中。 缓冲区描述符结构由 [BufferDesc](javascript:void(0)) 结构定义。 虽然这种结构有许多字段，但主要表现在以下几个方面：

- **tag** 将存储页面的buffer_tag保存在相应的缓冲池槽位中(buffer tag 在第8.1.2节中定义)。
- **buffer_id** 标识描述符(相当于相应缓冲池槽位的buffer_id)。
- **refcount** 保存当前访问相关存储页的PostgreSQL进程的数量。 它也被称为 **pin count**。 当PostgreSQL进程访问存储页面时，其引用计数必须加1(refcount ++)。 访问该页面后，其引用次数必须减少1(refcount--)。
     当引用计数为零时，即相关联的存储页面当前未被访问，该页面被**解除unpinned**锁定; 否则将**锁定pinned**该页。
- **usage_count** 用于保存关联的存储页面被加载到相应的缓冲池槽位中后被访问的次数。请注意，usage_count用于页面替换算法(第8.4.4节)。
- **context_lock** 和 **io_in_progress_lock** 是轻量级锁，用于控制对相关存储页面的访问。这些字段在第8.3.2节中描述。
- **flags** 可以保存关联的存储页面的几种状态。主要状态如下：
     - **dirty bit** 表示存储页是否是脏页。
     - **valid bit** 表示是否可以读取或写入存储的页面(valid)。例如，如果此位有效，则相应的缓冲池槽位将存储一个页面，并且此描述符(valid bit)保存页面元数据;因此，所存储的页面可以被读取或写入。如果此位无效，则此描述符不包含任何元数据;这意味着存储的页面不能被读取或写入，或者缓冲区管理器正在替换已存储的页面。
     - **io_in_progress bit** 表示缓冲区管理器是否正在从/向存储器读取/写入关联的页面。换句话说，该位表示单个进程是否持有该描述符的io_in_progress_lock。
- **freeNext** 是一个指向下一个描述符的指针，用于生成一个freelist，这将在下一小节中介绍。



> :pushpin: 结构BufferDesc在 [src/include/storage/buf_internals.h](https://github.com/postgres/postgres/blob/master/src/include/storage/buf_internals.h) 中定义。



 为了简化以下描述，定义了三个描述符状态：

- **Empty**：当相应的缓冲池插槽不存储页面时(即refcount和usage_count为0)，此描述符的状态为empty。
- **Pinned**：当相应的缓冲池插槽存储页面并且任何PostgreSQL进程正在访问该页面(即refcount和usage_count大于或等于1)时，该锁定缓冲区描述符的状态被锁定。
- **Unpinned**：当相应的缓冲池插槽存储一个页面但没有PostgreSQL进程正在访问该页面(即，usage_count大于或等于1，但是refcount为0)时，该缓冲区描述符的状态为*unpinned*。

每个描述符将具有上述状态之一。 描述符状态相对于特定条件而改变，这在下一小节中进行了描述。

在下面的图中，缓冲区描述符的状态由彩色框表示。

![](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch8/buf-8-01.jpg?raw=true)

另外，脏页被表示为'X'。 例如，一个unpinned缓冲区描述符由![脏描述符由表示](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch8/buf-8-02.jpg?raw=true)表示。

### 8.2.3. 缓冲区描述符层

一组缓冲区描述符形成一个数组。 在本文档中，该数组被称为 *buffer descriptors layer*.。

当PostgreSQL服务器启动时，所有缓冲区描述符的状态都是 *empty*。 在PostgreSQL中，这些描述符包含一个名为**freelist**的链表(图8.5)。



> :exclamation: 请注意，PostgreSQL中的**freelist**与Oracle中的freelists完全不同。 PostgreSQL的freelist只是空缓冲区描述符的链表。 在第5.3.4节中描述的PostgreSQL空闲空间映射中，充当与Oracle中的freelist相同的角色。



**图. 8.5. 缓冲区管理器初始状态**

![Fig. 8.5. Buffer manager initial state.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch8/fig-8-05.png?raw=true)

图8.6显示了如何加载第一页。

(1) 从freelist的顶部检索一个空描述符，并将其锁定(即将其refcount和usage_count增加1)。

(2) 在缓冲表中插入新条目，该条目保存第一页的标记与检索到的描述符的buffer_id之间的关系。 

(3) 将新页面从存储区加载到相应的缓冲池槽位中。 

(4) 将新页面的元数据保存到检索到的描述符中。

第二页和后续页面以类似的方式加载。 其他细节在第8.4.2节中提供。

**图. 8.6. 加载第一页**

![Fig. 8.6. Loading the first page.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch8/fig-8-06.png?raw=true)

 从freelist中检索的描述符总是保存页面的元数据。 换句话说，继续使用非空描述符不会返回到freelist。 但是，当下列情况之一发生时，相关描述符会再次添加到freelist中，并且描述符状态将变为“empty”：

1. 表或索引已被删除。
2. 数据库已被删除。
3. 表或索引已使用VACUUM FULL命令清除。



> :question: *为什么空描述符包含freelist？*

> 生成freelist的原因是立即得到第一个描述符。 这是动态内存资源分配的惯例。 参考这个[描述](https://en.wikipedia.org/wiki/Free_list)。



缓冲区描述符层包含一个无符号的32位整型变量，即**nextVictimBuffer**。 这个变量用在8.4.4节描述的页面替换算法中。

### 8.2.4. 缓冲池

缓冲池是一个存储数据文件页面(如表和索引)的简单数组。 缓冲池数组的索引被称为*buffer_id*s。

缓冲池的插槽大小为8 KB，等于页面的大小。 因此，每个插槽可以存储整个页面。



## 8.3. 缓冲区管理器锁

 缓冲区管理器为许多不同的目的使用许多锁。 本节介绍后续章节中解释所需的锁。



> :exclamation: 请注意，本节中描述的锁是缓冲区管理器的同步机制的一部分; 它们不涉及任何SQL语句和SQL选项。

 

### 8.3.1. 缓冲表锁

**BufMappingLock**保护整个缓冲表的数据完整性。这是一个轻量级的锁，可以在共享和独占模式下使用。在缓冲表中搜索条目时，后端进程保存共享的BufMappingLock。在插入或删除条目时，后端进程拥有排它锁。

BufMappingLock被分割成分区以减少缓冲表中的争用(默认是128个分区)。每个BufMappingLock分区都会保护相应哈希桶插槽的部分。

图8.7显示了分割BufMappingLock效果的典型示例。两个后端进程可以以独占模式同时保存相应的BufMappingLock分区，以便插入新的数据条目。如果BufMappingLock是单个系统范围的锁，则两个进程都应等待另一个进程的处理，具体取决于启动的处理。

**图 8.7. 两个进程以独占模式同时获取BufMappingLock的各个分区以插入新的数据条目**

![Fig. 8.7. Two processes simultaneously acquire the respective partitions of BufMappingLock in exclusive mode to insert new data entries.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch8/fig-8-07.png?raw=true)

缓冲表需要许多其他锁。 例如，缓冲表在内部使用自旋锁spinlock来删除条目。 但是，这些其他锁的描述被省略，因为它们在本文档中不是必需的。



> :pushpin: 默认情况下，BufMappingLock被分成16个单独的锁，直到版本9.4。



### 8.3.2. 每个缓冲区描述符的锁 

每个缓冲区描述符使用两个轻量级锁(**content_lock**和**io_in_progress_lock**)来控制对相应缓冲池插槽中存储页的访问。 当自己的字段的值被检查或改变时，使用自旋锁spinlock。

#### 8.3.2.1. content_lock

content_lock是一个强制访问限制的典型锁。 它可以用于共享和独占模式。

当读取页面时，后端进程获取存储该页面的缓冲区描述符的共享content_lock。

但是，执行以下操作之一时会获取独占的content_lock：

- 将行(即元组)插入存储的页面或更改存储页面中元组的t_xmin/t_xmax字段(t_xmin和t_xmax在第5.2节中描述;简单地说，当删除或更新行时，关联元组的这些字段被更改)。
- 物理地去除元组或者压缩存储页面上的空闲空间(通过分别在第6章和第7章中描述的vacuum处理和HOT执行)。
- 冻结存储页面中的元组(冻结frozen在第5.10.1节和第6.3节中描述)。

官方的[README](https://github.com/postgres/postgres/blob/master/src/backend/storage/buffer/README)文件显示更多细节。

#### 8.3.2.2. io_in_progress_lock

io_in_progress锁用于等待缓冲区上的I/O完成。 当PostgreSQL进程从/向存储器加载/写入页面数据时，该进程在访问存储器时保留相应描述符的独占io_in_progress锁定。

#### 8.3.2.3. spinlock

当选中或更改标志或其他字段(例如refcount和usage_count)时，将使用自旋锁。 下面给出了两个具体的自旋锁使用的例子：

(1) 下面显示了如何锁定缓冲区描述符：

1. 获取缓冲区描述符的自旋锁。
2. 将其refcount和usage_count的值增加1。
3. 释放自旋锁。

```c
LockBufHdr(bufferdesc);    /* Acquire a spinlock */
bufferdesc->refcont++;
bufferdesc->usage_count++;
UnlockBufHdr(bufferdesc); /* Release the spinlock */
```

(2) 下面显示了如何将脏位 dirty bit 设置为'1'：

1. 获取缓冲区描述符的自旋锁。
2. 使用按位操作将脏位设置为'1'。
3. 释放自旋锁。

```c
#define BM_DIRTY             (1 << 0)    /* data needs writing */
#define BM_VALID             (1 << 1)    /* data is valid */
#define BM_TAG_VALID         (1 << 2)    /* tag is assigned */
#define BM_IO_IN_PROGRESS    (1 << 3)    /* read or write in progress */
#define BM_JUST_DIRTIED      (1 << 5)    /* dirtied since write started */

LockBufHdr(bufferdesc);
bufferdesc->flags |= BM_DIRTY;
UnlockBufHdr(bufferdesc);
```
更改其他位以相同的方式执行。

 

> :pushpin:*用原子操作替换缓冲区管理器自旋锁*

> 在版本9.6中，缓冲区管理器的自旋锁将被替换为原子操作。 看到这个[commitfest的结果](https://commitfest.postgresql.org/9/408/)。 如果您想知道详细信息，请参阅此[讨论](http://www.postgresql.org/message-id/flat/2400449.GjM57CE0Yg@dinodell#2400449.GjM57CE0Yg@dinodell)。



## 8.4. 缓冲区管理器如何工作

本节介绍缓冲区管理器的工作方式。 当后端进程想要访问所需页面时，它会调用 *ReadBufferExtended* 函数。

ReadBufferExtended 函数的行为取决于三个逻辑情况。 以下小节将对每种情况进行介绍。 另外，PostgreSQL clock-sweep页面替换算法在最后的小节中进行了描述。

### 8.4.1. 访问存储在缓冲池中的页 

首先描述最简单的情况，即期望的页面已经存储在缓冲池中。 在这种情况下，缓冲区管理器执行以下步骤：

(1) 创建所需页面的buffer_tag(在本例中，buffer_tag为'Tag_C')，并使用哈希函数计算包含创建的buffer_tag的关联条目的哈希桶槽。

(2) 获取BufMappingLock分区，该分区覆盖获取的共享模式下的哈希桶插槽(该锁定将在步骤(5)中释放)。

(3) 查找标签为'Tag_C'的条目并从条目中获取buffer_id。 在这个例子中，buffer_id是2。

(4) 将buffer_id 2的缓冲区描述符锁定，即描述符的refcount和usage_count增加1(8.3.2节描述锁定)。

(5) 释放BufMappingLock。

(6) 使用buffer_id 2访问缓冲池槽位。

**图. 8.8. 访问存储在缓冲池中的页**

![Fig. 8.8. Accessing a page stored in the buffer pool.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch8/fig-8-08.png?raw=true)

然后，当从缓冲池槽位中的页面读取行时，PostgreSQL进程获取相应缓冲区描述符的共享content_lock。 因此，缓冲池槽位可以被多个进程同时读取。

当插入(和更新或删除)行到页面时，Postgres进程获取相应缓冲区描述符的独占content_lock(注意页面的脏位必须设置为1)。

访问页面后，相应缓冲区描述符的refcount值减1。

### 8.4.2. 将页从存储加载到空槽 

在第二种情况下，假定所需页面不在缓冲池中，并且freelist具有空闲元素(空描述符)。在这种情况下，缓冲区管理器执行以下步骤：
(1) 查找缓冲表(我们假设它没有找到)。

1. 创建所需页面的buffer_tag(在本例中，buffer_tag为'Tag_E')并计算哈希桶槽。
2. 以共享模式获取BufMappingLock分区。
3. 查找缓冲表(根据假设未找到)。
4. 释放BufMappingLock。

(2) 从freelist中获取空的缓冲区描述符，并将其锁定。在这个例子中，获得的描述符的buffer_id是4。
(3) 以独占模式获取BufMappingLock分区(该锁将在步骤(6)中释放)。
(4) 创建一个新的数据项，其中包含buffer_tag'Tag_E'和buffer_id 4;将创建的条目插入缓冲表。
(5) 使用buffer_id 4将所需的页面数据从存储器加载到缓冲池槽位中，如下所示：

1. 获取相应描述符的独占锁io_in_progress_lock。
2. 将相应描述符的io_in_progress位设置为1以防止其他进程访问。
3. 将所需的页面数据从存储加载到缓冲池插槽。
4. 更改相应描述符的状态; *io_in_progress*位设置为'0'，并且*valid*位设置为'1'。
5. 释放io_in_progress_lock。

(6) 释放BufMappingLock。
(7) 使用buffer_id 4访问缓冲池槽位。

**图. 8.9. 将存储页面加载到空白插槽**

![Fig. 8.9. Loading a page from storage to an empty slot.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch8/fig-8-09.png?raw=true)

### 8.4.3. 将页面从存储加载到victim缓冲池槽

在这种情况下，假设所有缓冲池插槽都被页面占用，但所需的页面没有被存储。缓冲区管理器执行以下步骤：

(1) 创建所需页面的buffer_tag并查找缓冲表。在这个例子中，我们假设buffer_tag是'Tag_M'(找不到所需的页面)。

(2) 使用 clock-sweep 算法选择 victim 缓冲池，从缓冲表中获取包含victim池槽的buffer_id的旧条目，并将victim池槽锁定在缓冲区描述符层中。在此示例中，victim插槽的buffer_id是5，旧条目是'Tag_F，id = 5'。clock-sweep 在下一小节中描述。

(3) 如果victim页面数据是脏的，则刷新(写入和fsync);否则进入步骤(4)。

在用新数据覆盖之前，脏页必须写入存储器。刷新脏页面的过程如下：

1. 使用buffer_id 5获取描述符的共享锁content_lock和独占锁io_in_progress(在步骤6中发布)。
2. 更改相应描述符的状态; io_in_progress位设置为'1'，just_dirtied位设置为'0'。
3. 根据具体情况，调用XLogFlush()函数将Wal缓冲区上的Wal数据写入当前Wal段文件(细节将被省略；WAL和XLogFlush函数将在第9章中描述)。 
4. 将victim页面数据刷新到存储。
5. 更改相应描述符的状态; io_in_progress位被设置为'0'并且valid位被设置为'1'。
6. 释放io_in_progress和content_lock锁。

(4) 以独占模式获取覆盖包含旧条目的插槽的旧BufMappingLock分区。

(5) 获取新的BufMappingLock分区并将新条目插入缓冲表中：

1. 创建由新的buffer_tag'Tag_M'和victim的buffer_id组成的新条目。
2. 以独占模式获取新的BufMappingLock分区，该分区覆盖包含新条目的插槽。
3. 将新条目插入缓冲表。


**图. 8.10. 将存储页面加载到victim缓冲池槽位** 

![Fig. 8.10. Loading a page from storage to a victim buffer pool slot.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch8/fig-8-10.png?raw=true)

(6) 从缓冲表中删除旧条目，并释放旧的BufMappingLock分区。

(7) 将存储器中所需的页面数据加载到victim缓冲区插槽。 然后，用buffer_id 5更新描述符的标志; 脏位被设置为0并初始化其他位。

(8) 释放新的BufMappingLock分区。

(9) 使用buffer_id 5访问缓冲池槽位。

**图. 8.11. 将存储器中的页面加载到victim缓冲池槽位(接下来的图8.10)**

![Fig. 8.11. Loading a page from storage to a victim buffer pool slot (continued from Fig. 8.10).](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch8/fig-8-11.png?raw=true)

### 8.4.4. 页面替换算法：Clock Sweep

本节的其余部分介绍**clock-sweep**算法。 该算法是低开销的NFU(Not Frequently Used)的变体; 它可以有效地选择使用率较低的页面

将缓冲区描述符想象成一个循环列表(图8.12)。 nextVictimBuffer是一个无符号的32位整数，它总是指向其中一个缓冲区描述符并顺时针旋转。 该算法的伪代码和描述如下：



>:pushpin: *clock-sweep 伪代码:* 

>```c
>    WHILE true
>(1)     Obtain the candidate buffer descriptor pointed by the nextVictimBuffer
>(2)     IF the candidate descriptor is unpinned THEN
>(3)	       IF the candidate descriptor's usage_count == 0 THEN
>            BREAK WHILE LOOP  /* the corresponding slot of this descriptor is victim slot. */
>       ELSE
>	    Decrease the candidate descriptpor's usage_count by 1
>              END IF
>        END IF
>(4)     Advance nextVictimBuffer to the next one
>     END WHILE 
>(5) RETURN buffer_id of the victim
>```

> (1) 获取nextVictimBuffer指向的候选缓冲区描述符。

> (2) 如果候选缓冲区描述符被取消锁定，请继续执行步骤(3); 否则，继续步骤(4)。

> (3) 如果候选描述符的usage_count为0，则选择该描述符的相应槽位作为victim并前进到步骤(5); 否则，将此描述符的usage_count减1并继续步骤(4)。

> (4) 将nextVictimBuffer推进到下一个描述符(如果最后环绕)并返回到步骤(1)。 重复，直到找到victim。

> (5) 返回victim的buffer_id。



图8.12显示了一个具体的例子。 缓冲区描述符显示为蓝色或青色框，并且框中的数字显示每个描述符的usage_count。

**图. 8.12. Clock Sweep.**

![Fig. 8.12. Clock Sweep.](https://github.com/yonj1e/The-Internals-of-PostgreSQL/blob/master/imgs/ch8/fig-8-12.png?raw=true)

1) nextVictimBuffer指向第一个描述符(buffer_id 1); 然而，这个描述符因为被锁定而被跳过。

2) nextVictimBuffer指向第二个描述符(buffer_id 2)。 此描述符未锁定，但其usage_count为2; 因此，usage_count减1，nextVictimBuffer前进到第三个候选。

3) nextVictimBuffer指向第三个描述符(buffer_id 3)。 这个描述符是unpinned的，它的usage_count是0; 因此，它是本轮的victim。

每当nextVictimBuffer扫描一个未锁定的描述符时，它的usage_count减1。因此，如果缓冲池中存在未锁定的描述符，则该算法总是可以通过旋转nextVictimBuffer找到其usage_count为0的victim。



## 8.5. 环形缓冲器 Ring buffer

在读取或写入大表时，PostgreSQL使用环形缓冲区而不是缓冲池。环形缓冲区是一个小的临时缓冲区。当满足下面列出的任何条件时，环形缓冲区将分配给共享内存：

1. Bulk-reading

   当扫描大小超过缓冲池大小的四分之一(shared_buffers/4)的relation时。在这种情况下，环形缓冲区大小为256 KB。

2. Bulk-writing

   当执行下面列出的SQL命令时。在这种情况下，环形缓冲区大小为16 MB。

   - [*COPY FROM*](http://www.postgresql.org/docs/current/static/sql-copy.html) 命令.
   - [*CREATE TABLE AS*](http://www.postgresql.org/docs/current/static/sql-createtableas.html) 命令.
   - [*CREATE MATERIALIZED VIEW*](http://www.postgresql.org/docs/current/static/sql-creatematerializedview.html) 或 [*REFRESH MATERIALIZED VIEW*](http://www.postgresql.org/docs/current/static/sql-refreshmaterializedview.html)命令。
   - [*ALTER TABLE*](http://www.postgresql.org/docs/current/static/sql-altertable.html) 命令。

3. Vacuum-processing

    autovacuum执行vacuum处理时。在这种情况下，环形缓冲区大小为256 KB。

分配的环形缓冲区在使用后立即释放。

环形缓冲区的好处是显而易见的。如果后端进程在不使用环形缓冲区的情况下读取一个巨大的表，则缓冲池中的所有存储页面都将被删除(kicked out)。因此，缓存命中率下降。环形缓冲区避免了这个问题。

 

> :question: 为什么 *bulk-reading* 和 *vacuum* 处理的默认环形缓冲区大小为256 KB？

> 为什么256 KB？答案在位于缓冲区管理器的源目录下的[README](https://github.com/postgres/postgres/blob/master/src/backend/storage/buffer/README)中进行了解释。

> > 对于顺序扫描，使用256 KB的环。这足够小以适应L2缓存，这使得从OS缓存到共享缓存缓存的页面传输效率更高。即使更少，通常也足够了，但环必须足够大以容纳扫描中同时锁定的所有页面。 (snip )



## 8.6. 刷新脏页

除了替换victim页面之外，checkpointer 和 background writer 进程还会将脏页面刷新到存储区域。 两个进程都具有相同的功能(清理脏页面); 然而，他们有不同的角色和行为。

checkpointer进程将检查点记录写入WAL段文件，并在启动检查点时刷新脏页。 第9.7节描述了检查点设置以及何时开始。

background writer进程的作用是减少检查点的密集写的影响。background writer继续一点一点地刷新脏页，对数据库活动影响最小。 默认情况下，后台编写器每200毫秒唤醒一次(由[bgwriter_delay](http://www.postgresql.org/docs/current/static/runtime-config-resource.html#GUC-BGWRITER-DELAY)定义)，并最大限度地刷新[bgwriter_lru_maxpages](http://www.postgresql.org/docs/current/static/runtime-config-resource.html#GUC-BGWRITER-LRU-MAXPAGES)(缺省值为100页)。 