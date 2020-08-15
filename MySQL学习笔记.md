#### MySQL架构原理和存储机制

##### MySQL体系架构

![image-20200810005611488](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200810005611488.png)

###### 连接层

客户端连接器：与MySQL服务处建立连接

###### 服务层

服务层是MySQL Server的核心，主要包含系统管理和控制工具、连接池、SQL接口、解析器、查询优化器、缓存等6个部分

- 连接池：负责存储和管理客户端的连接，一个线程负责一个连接
- 系统管理和控制工具：包含备份恢复、安全管理、集群管理等
- SQL接口：用于接收客户端发送的SQL语句，以及返回查询结果
- 解析器：负责将请求的SQL解析生成一个“解析树”，然后根据MySQL规则判断解析树是否合法（词法分析、语法分析）
- 查询优化器：将“解析树”转化为执行计划，然后与执行引擎交互
- 缓存：包含表缓存、记录缓存、权限缓存、引擎缓存等

###### 存储引擎层

存储引擎负责数据的存储与提取，与底层系统文件进行交互

###### 系统文件层

系统文件层负责将数据和日志存储在文件系统上，以及与存储引擎交互。主要包含日志文件、数据文件、配置文件、pid文件、socket文件等

- 日志文件

  - 错误日志error log

    默认开启，show variables like '%log_error%'

  - 通用查询日志general query log

    记录一般查询语句，show variables like '%general%'

  - 二进制日志binary log

    记录更改操作（不记录select、show等不修改的SQL），包括语句的发生时间、执行时长等，主要用于主从复制和数据库恢复

    show variables like '%log_bin%'; //是否开启

    show variables like '%binlog%' //参数查看

    show binary logs; //查看日志文件

  - 慢查询日志slow query log

    记录所有执行时间超时的查询SQL，默认10秒

    show variables like '%slow_query%'; //是否开启

    show variables like '%long_query_time%'; //时长

- 配置文件

  用于存放MySQL的配置文件，如my.ini，my.cnf

- 数据文件

  db.opt文件：记录数据库默认使用的字符集和校验规则

  .frm文件：存储与表相关的元数据信息，包括表结构的定义信息。每一张表都会有一个.frm文件

  .MYD：MyISAM存储引擎专用，存放表的数据。每一张表都会有一个.MYD文件

  .MYI文件：MyISAM存储引擎专用，存放索引相关信息。每一张表都会有一个.MYI文件

  .ibd文件和.ibdata文件：存放InnoDb的数据文件（包括索引）。InnoDB存储引擎有两种表空间方式：独享表空间和共享表空间。独享表空间使用.ibd文件来存放数据，且每一张表对应一个.ibd文件。共享表空间使用.ibdata文件，所有表共同使用一个或多个（可配置）.ibdata文件

  .ibdata1文件：系统表空间数据文件，存储表的元数据、undo日志等

  .ib_logfile0\.ib_logfile1文件：redo log日志文件

- pid文件

  pid文件是mysqld应用程序在unix环境下的一个进程文件，存放自己的进程id

- socket文件

  socket文件也是unix环境下才有的，用户在unix环境下的客户端连接可以不通过TCP/IP而直接使用unix socket来连接SQL

##### MySQL运行机制

![image-20200812234721865](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200812234721865.png)

###### 建立连接 

- 客户端通过客户端/服务器通信协议与MySQL建立连接

- MySQL客户端与服务端的通信方式是“半双工”

- 对于每一个MySQL的连接，时刻都有一个线程状态标识当前这个连接正在做什么

  通讯机制：

  - 全双工：同一时刻既能发又能收
  - 半双工：某一时刻，要么发送数据，要么接收数据，不能同时
  - 单工：只能发送或接收数据

  线程状态：

  show processlist; //查看当前用户正在运行的线程信息

  - id：线程ID
  - user：启动该线程的用户
  - Host：发送请求的客户端IP和端口号
  - db：当前命令在哪个库执行
  - Command：该线程正在执行的操作命令
    - create db：创建库
    - drop db：删除库
    - execute：正在执行一个PreparedStatement
    - close Stmt：正在关闭一个PreparedStatement
    - Query：正在执行一个语句
    - sleep：正在等待客户端发送语句
    - quit：正在退出
    - shutdown：正在关闭服务器
  - Time：表示该线程处于当前状态的时间，单位秒
  - state：线程状态
    - update：正在搜索匹配记录，进行修改
    - sleeping：正在等待客户端发送新请求
    - starting：正在执行请求处理
    - checking table：正在检查数据表
    - closing table：正在将表中数据刷新到磁盘中
    - locked：被其他查询锁住了记录
    - sending data：正在处理select查询，同时将结果发送给客户端
  - info：一般记录线程执行的语句，默认显示前100个字符。查看完整信息使用show full processlist

###### 查询缓存

如果开启了查询缓存，在查询过程中会先查询缓存，如果匹配到完全相同的SQL语句，则直接将缓存结果返回。如果没有开启缓存或者没有匹配成功，则SQL会将由解析器进行词法语法解析，生成解析树

- 缓存select查询的结果和SQL语句
- 执行select查询时，先查询缓存，判断是否存在可用的记录集
- 即使开启查询缓存，以下SQL也不会进行缓存
  - 查询语句使用SQL_NO_CACHE
  - 查询的结果大于query_cache_limit设置
  - 查询中有一些不确定的参数，比如now()
  - show variables like '%query_cache%' //查看查询缓存是否启用，空间大小，限制等
  - show status like 'Qcache%' //查看更详细的缓存参数，可用缓存空间，缓存块等

###### 解析器

将客户端发送的SQL进行语法解析，生成解析树。预处理器根据一些MySQL规则进一步检查”解析树“是否合法，例如检查数据表和数据列是否存在，还会解析名字和别名，看看它们是否有歧义，最后生成新的”解析树“

###### 查询优化器

根据“解析树”生成最优的执行计划。MySQL使用很多优化策略生成最优的执行计划，可以分为两类：静态优化（编译时优化），动态优化（运行时优化）

- 等价变换策略

  例：5=5 and a>5 改为 a>5

- 优化count、min、max等函数

  Innodb引擎min函数只需要找索引的最左边，max函数只需要找索引最右边

  MyISAM引擎count(*)，不需要计算，直接返回

- 提前终止查询

  使用了limit查询，获取limit所需的数据，就不在继续遍历后面数据

- in的优化

  MySQL对in查询，会先进行排序，再采用二分法查找数据

###### 执行引擎

负责执行SQL语句，此时查询执行引擎会根据SQL语句中表的存储引擎类型与底层存储引擎进行交互，得到查询结果并返回给客户端。若开启了查询缓存，这时会将SQL语句和结果完整得保存到查询缓存中

- 如果开启了查询缓存，先将查询结果做缓存操作
- 返回结果过多，采用增量模式返回

##### MySQL存储引擎

存储引擎在MySQL的体系架构中位于第三层，负责MySQL中数据的存储和提取。它是根据MySQL提供的文件访问层抽象接口定制的一种文件访问机制

使用show engines命令可以查看当前数据库支持的引擎信息

![image-20200813225259659](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200813225259659.png)

注：MySQL5.5之前默认采用MyISAM存储引擎，从5.5开始采用InnoDB存储引擎

各个存储引擎的特点：

- InnoDB：支持事务，事务安全
- MyISAM：不支持事务和外键，访问速度快
- Memory：利用内存创建表，访问速度非常快，因为数据在内存，而且默认使用Hash索引，但是一旦关闭，数据就会丢失
- Archive：归档类型引擎，仅能支持insert和select语句
- CSV：以CSV文件进行数据存储，由于文件限制，所有列必须强制指定not null，另外CSV引擎也不支持索引和分区，适合做数据交换的中间表
- BlackHole：黑洞，只进不出，进来消失，所有插入数据都不会保存
- Federated：可以访问远端MySQL数据库中的表。一个本地表，不保存数据，访问远程包内容
- MRG_MyISAM：一组MyISAM表的组合，这些MyISAM表必须结构相同，Merge表本身没有数据，对Merge操作可以对一组MyISAM表进行操作

###### InnoDB和MyISAM对比

- 事务和外键

  InnoDB支持事务和外键，具有安全性和完整性，适合大量insert或update操作

  MyISAM不支持事务和外键，它具有告诉存储和检索，适合大量的select查询操作

- 锁机制

  InnoDB支持行级锁，锁定指定记录，基于索引来加锁实现

  MyISAM支持表级锁，锁定整张表

- 索引结构

  InnoDB使用聚集索引，索引和技术在一起存储，既缓存索引，也缓存记录

  MyISAM使用非聚集索引，索引和记录分开

- 并发处理能力

  InnoDB读写阻塞与隔离级别有关，可以采用多版本并发控制MVCC来支持高并发

  MyISAM使用表锁，会导致写操作并发率低，读之间并不阻塞，读写阻塞

- 存储文件

  InnoDB表对应两个文件，一个.frm表结构文件，一个.ibd数据文件。InnoDB表最大支持64TB

  MyISAM表对应三个文件，一个.frm表结构文件，一个MYD表数据文件，一个.MYI索引文件。从MySQL5.0开始默认限制是256TB

- 适用场景

  InnoDB

  - 需要事务支持
  - 行级锁对高并发有很好的适应能力
  - 适合数据更新较为频繁的场景
  - 数据一致性要求较高
  - 硬件设备内存较大，可以利用InnoDB较好的缓存能力来提高内存利用率，减少磁盘IO

  MyISAM

  - 不需要事务支持
  - 表锁导致并发能力相对较弱
  - 适合以读为主，数据修改相对较少的场景
  - 数据一致性要求不高

- 总结

  两种引擎如何选择？

  - 是否需要事务？有InnoDB
  - 是否存在并发修改？有，InnoDB
  - 是否追求快速查询，且数据修改较少？是，MyISAM
  - 在绝大多数情况下，推荐使用InnoDB

###### InnoDB存储结构

下图为InnoDb引擎架构图，主要分为内存结构和磁盘结构两大部分

![image-20200814001650395](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814001650395.png)

- 内存结构

  - Buffer Pool：缓冲池，简称BP。BP以Page页为单位，默认大小16K，BP底层采用链表管理Page

    - Page管理机制

      Page的分类：

      - free page：空闲页，未被使用
      - clean page：已被使用的页，且数据没有被修改过
      - dirty page：脏页，已被使用且数据被修改过，页中的数据与磁盘的数据不一致

      针对上诉三种类型，InnoDB通过三种链表结构来维护和管理

      - free list：空闲缓冲区，管理free page
      - flush list：需要刷新到磁盘的缓冲区，管理dirty page。内部page按修改时间排序。脏页即存在于flush链表，页存在LRU链表中，但是两种互不影响，LRU链表负责管理page的可用性和释放，而flush链表负责管理脏页的刷盘操作
      - LRU list：正在使用的缓冲区，管理clean page和dirty page，缓冲区以midpoint为基点，前面链表称为new列表去，存放经常访问的数据，占63%；后面的链表称为old列表区，存放使用较少的数据，占37%

    - 改进型LRU算法

      普通LRU：末尾淘汰法，链头插入，链尾删除

      改进型LRU：链表分为new和old两部分，加入元素时并不是从表头插入，而是从中间midpoint位置插入，如果数据很快被访问，那么page就会向new列表头部移动；如果数据没有被访问，会逐步向old尾部移动，等待淘汰

    - Buffer Pool配置参数

      show variables like '%innodb_page_size%';	//查看page页大小

      show variables like '%innodb_old%'; //查看LRU list中old列表参数

      show variables like '%innodb_buffer%';	//查看buffer pool参数

      建议：将innodb_buffer_pool_size设置为总内存大小的60%-80%，innodb_buffer_pool_instances可以设置为多个，这样可以避免缓存争夺

  - Change Buffer：写缓存区，简称CB。在进行DML操作时，如果BP没有其相应的Page数据，并不会立刻将磁盘页加载到缓冲池，而是在CB中记录缓冲变更，等未来数据被读取时，再将数据合并恢复到BP中

    ChangeBudder占用BufferPool空间，默认占25%，最大允许占50%，可以根据读写业务量来进行调整，参数innodb_change_buffer_max_size

    写缓冲区仅适用于非唯一普通索引页：如果索引设置成唯一，在进行修改时，innoDB需要进行唯一性校验，因此必须查询磁盘，做一次IO操作。会直接将记录查询到Buffer Pool中，然后再BufferPool中修改，不会再Change Buffer上操作

  - Adaptive Hash Index：自适应哈希索引，用于优化对BP数据的查询。InnoDB会监控表索引的查找，会根据访问频率和模式来为某些页建立哈希索引

  - Log Buffer：日志缓冲区，用来保存要写入磁盘（redo/undo）上的数据，日志缓冲区的内容定期刷新到磁盘log文件中。日志缓冲区满时会自动将其刷新到磁盘，当遇到BLOB或多行更新的大事务操作时，增加日志缓冲区可以节省磁盘IO

    innodb_flush_log_at_trx_commit参数控制日志刷新行为，默认为1

    - 0：每隔1秒写日志文件和刷盘操作，最多丢失1秒数据
    - 1：事务提交立刻写日志文件和刷盘，数据不丢失，但IO频繁
    - 2：事务提交立刻写日志文件，每隔1秒钟进行刷盘操作

- 磁盘结构

  - Tablespaces表空间：用于存储表结构和数据。表空间又分为系统表空间，独立表空间、通用表空间、临时表空间、Undo表空间等

    - The System Tablespace系统表空间：共享，该空间的数据文件通过innodb_data_file_path控制，默认值ibdata1:12M:autoextend（文件名ibdata1，12M，自动扩展）

    - File-Per_Table Tablespaces独立表空间：当innodb_file_per_table开启时，表被创建在独立表空间中，否则，将创建于系统表空间，默认开启。每个表空间由一个.ibd数据文件代表，支持动态和压缩行格式

    - General Tablespaces通用表空间：通用表空间为通过create tablespaces语法创建的共享表空间

    - Undo Tablespaces撤销表空间：由一个或多个包含Undo日志文件组成。在MySQL5.7之前Undo占用系统表空间，从5.7开始将Undo分离出来。InnoDB使用的undo表空间通过innodb_undo_tablespaces配置选项控制，默认为0

      参数为0表示使用系统表空间ibdata1；大于0表示使用undo表空间undo_001、undo_002等

    - Temporary临时表空间：分为session temporary tablespaces和global temporary tablespaces两种

      session：存储用户创建的临时表个磁盘内部的临时表

      global：存储用户临时表的回滚段

      MySQL服务器正常关闭或异常终止，临时表空间都会被清除，每次启动时重新创建

  - InnoDB Data Dictionary数据字典

    InnoDb数据字典由内部系统表组成，这些表包含用于查找表、索引和表字段等对象的元数据。元数据物理上位于InnoDB系统表空间中。由于历史原因，数据字典元数据在一定程度上与InnoDB表元数据文件（.frm）中存储的信息重叠

  - Doublewrite Buffer双写缓冲区

    位于系统表空间，时一个存储区域。在BufferPage的page页刷新到磁盘前，会先将数据存在Doublewrite缓冲区

  - Redo Log重做日志

    重做日志时一种基于磁盘的数据结构，用于在崩溃恢复期间更正不完整事务写入的数据。MySQL以循环方式写入重做日志文件，记录InnoDB中所有对Buffer Pool修改的日志。当出现实例故障，导致数据未能更新到数据文件，则数据库重启时须redo，重新把数据更新到数据文件。读写事务在执行的过程中，都会不断地产生redo_log。默认情况下，重做日志在磁盘上由两个名为ib_logfile0和ib_logfile1的文件物理表示

  - Undo Log撤销日志

    撤销日志在事务开始之前对数据的备份，用于例外情况时回滚事务。撤销日志属于逻辑日志，根据每行记录进行记录

###### InnoDB线程模型

![image-20200814013954943](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814013954943.png)

- IO Thread

  在InnoDB中使用了大量的AIO来做读写处理，这样可以极大提高数据库的性能。在InnoDB1.0版本之前公有4个IO Thread，分别是write、read、insert Buffer和log thread，后来版本将read thread和write thread分别增大到了4个，一共有10个

  - read thread：负责读取操作，将数据从磁盘加载到缓存page页，共4个
  - write thread：负责写操作，将缓存脏页刷新到磁盘，共4个
  - log thread：负责将日志缓冲区内容刷新到磁盘，共1个
  - insert buffer thread：负责将写缓冲内容刷新到磁盘，共1个

- Purge Thread

  事务提交之后，其使用的undo日志将不再需要，因此需要Purge Thread回收已经分配的undo页

  show variables like '%innodb_purge_threads%'

- Page Cleaner Thread

  将脏数据刷新到磁盘，脏数据刷盘后相应的redo log也就可以覆盖了，既可以同步数据，又能达到redo log循环使用的目的。会调用write thread线程处理

  show variables like '%innodb_page_cleaners%'

- Master Thread

  Master Thread是innodb的主线程，负责调度其他线程，优先级最高。且负责将缓冲池中的数据异步刷新到磁盘，保证数据的一致性。包含：脏页的刷新、undo页回收、redo日志刷新、合并写缓冲等。内部有两个主处理，分别是每隔1秒和每隔10秒的处理

  每1秒的操作：

  - 刷新日志缓冲区，刷到磁盘
  - 合并写缓冲区数据，根据IO读写压力来决定是否操作
  - 刷新脏页数据到磁盘，根据脏页比例达到75%才操作

  每10秒的操作：

  - 刷新脏页数据到磁盘
  - 合并写缓冲区数据
  - 刷新日志缓冲区
  - 删除无用的undo页

###### InnoDB数据文件

- InnoDB文件存储结构

  ![image-20200814015852158](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814015852158.png)

  InnoDB数据文件存储结构：

  .ibd数据文件->Segment段->Extent区->Page页->Row行

  - Tablespace

    表空间，用于存储多个ibd数据文件，用于存储表的记录和索引。一个文件包含多个段

  - Segment

    段，用于管理多个Extent。分为数据段（leaf node segment）、索引段（non-leaf node segment）、回滚段（rollback segment）。一个表至少会有两个segment，一个管理数据，一个管理索引。每创建一个索引，会多两个segment

  - Extent

    区，一个区固定包含64个连续的页，大小为1M。当表空间不足，需要分配新的页资源时，不会一页一页分，直接分配一个区

  - Page

  - 页，用于存储多个Row行记录，大小为16K。包含很多种页类型，比如数据页、undo页、系统页、事务数据页、大的BLOB对象页

  - Row

    行，包含了记录的字段值，事务ID、滚动指针、字段指针等信息

  Page是文件的基本单位，无论何种类型的page，都是由page header、page trailer和page body组成，如下图所示

  ![image-20200814021028103](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814021028103.png)

- InnoDB文件存储格式

  ![image-20200814021702820](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814021702820.png)

  row_format为redundant、compact，文件格式为antelope；row_format为dynamic、compressed，文件格式为barracuda

  通过information_schema查看指定表的文件格式:

  ```
  select * from information_schema.innodb_sys_tables;
  ```

- File文件格式

  目前InnoDB只支持两种文件格式：antelope和barracuda

  antelope：最原始的InnoDB文件格式，它支持两种行格式：compact和redundant，MySQL5.6及其之前的版本默认格式为antelope

  barracude：新的文件格式，支持InnoDB的所有行格式

  innodb_file_format可以配置InnoDB文件格式

- Row行格式

  表的行格式决定了它的物理存储，这会影响查询和DML操作的性能

  InnoDB支持四种行格式：redundant、compact、dynmic和compressed

  ![image-20200814082551942](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814082551942.png)

  每个表的数据分成若干页来存储，页采用B树结构存储，如果某些字段过长，无法存储在B树节点中，这时候会被单独分配空间，此时被称为溢出页，该字段被称为页外列

  - redundant

    将变长列值的前768字节存储在B树节点的索引记录中，其余的存储在溢出页上。针对大于等于786字节的固定长度的字段InnoDB会将其转换为变长字段，以便使用页外存储

  - compact

    与redundent相比，compact减少了约20%的行存储空间，但代价是增加了某些操作的CPU使用量。如果系统性能受缓存命中率和磁盘速度限制，那么compact格式可能更快。如果系统性能受CPU速度限制，那么compact格式可能会慢一些

  - dynamic

    将可变长度的列值完全存储在页外，而索引记录只包含指向溢出页的20字节指针。大于或等于768字节的固定长度字段会转换为可变长度字段。dynamic支持大索引前缀，最多可以为3072字节，可通过innodb_large_prefix参数控制

  - compressed

    与dynamic提高相同的存储特性和功能，但增加了对表和索引数据压缩的支持

###### Undo Log

undo：以撤销为目的，返回某个状态的操作

undo log：事务开始之前，将要修改的表记录保存到undo日志中，当事务回滚或数据库崩溃时，可以利用undo日志撤销未提交事务对数据库产生的影响

undo log的产生和销毁：产生在事务开始之前；销毁在事务提交后，但不会立即删除undo log，innodb会将该事务对应的undo log日志放入到删除列表中，后面通过后台线程purge thread进行回收处理

undo log属于逻辑日志，记录的是一个变化的过程。例如执行一个delete，undo log会记录一个insert

undo log存储：采用段的方式管理和记录。在innodb数据文件中包含一种rollback segment回滚段，内部包含1024个undo log segment。可以通过下面一组参数来控制undo log存储

![image-20200814085938877](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814085938877.png)

undo log的作用：

- 实现事务的原子性

  事务处理过程中，如果出现了错误或者用户执行了回滚，MySQL可以利用undo log将数据恢复到事务开始之前的状态

- 实现多版本并发控制

  事务未提交之前，undo log保存未提交前的数据，可作为数据旧版本快照供其他并发事务进行快照读

###### Redo Log和Binlog

redo log和Binlog是MySQL日志系统中非常重要的两种机制

redo log日志

- 介绍

  redo：以恢复为目的，在数据库出错时重现操作

  redo log：重做日志。事务中修改的数据

  产生和释放：随着事务操作的执行，就会生成redo log；在事务提交时会将产生的redo log写入log buffer，但不会立刻写入磁盘；等事务操作的脏页写入磁盘之后，redo log占用的空间就可以被重用

- 工作原理

  redo log是为了实现事务的持久性而出现的产物

  ![image-20200814092208414](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814092208414.png)

- 写入机制

  redo log文件内容以顺序循环的方式写入文件

  ![image-20200814092347670](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814092347670.png)

  - write pos

    当前记录的位置，一边写一边后移，写到末尾就回溯到开头

  - checkPoint

    当前要擦除的位置，擦除记录前要把记录更新到数据文件

- 相关配置参数

  每个InnoDB存储引擎至少有1个重做日志文件组，每个文件组至少有两个重做日志文件，默认为ib_logfile0和ib_logfile1。可以通过下面一组参数控制redo log存储

  ![image-20200814095319125](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814095319125.png)

- redo buffer持久化到redo log的策略，通过innodb_flush_log_at_trx_commit设置

  - 0：每秒提交，可能丢失一秒内的事务数据。由后台master线程每隔一秒执行一次操作

    redo buffer --> OS cache --> flush cache to disk

  - 1默认值：每次事务提交时执行redo buffer --> OS cache --> flush cache to disk，最安全，性能最差

  - 2：每次事务提交时执行redo buffer --> OS cache，然后由后台Master线程每隔1秒执行OS cache --> flush cache to disk的操作

    ![image-20200814100537269](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814100537269.png)

Binlog日志

- 记录模式

  Binlog是记录表结构变更和表数据修改的二进制日志，不记录select和show这类操作。以事件形式记录，还包含语句所执行的消耗时间

  使用场景：

  - 主从复制：在主库中开启Binlog功能，这样主库就可以把Binlog传递给从库，从库拿到Binlog后实现数据恢复，达到主从数据一致性

  - 数据恢复：通过mysqlbinlog工具来恢复数据

    Bin log文件名默认为“主机名_binlog-序列号”格式，也可以在配置文件中指定名称

  文件记录格式：

  - row

    记录每一行数据被修改的情况，然后再slave端进行同样的修改

    优点：能清除记录每一行数据的修改细节

    缺点：批量操作，会产生大量的日志

  - statement

    记录被修改数据的SQL，slave进行SQL语句的执行

    优点：日志量小，减少磁盘IO，提升存储和恢复速度

    缺点：在某些情况下会导致主从数据不一致，比如last_insert_id()、now()等

  - mixed

    以上两种模式的混合使用，一般会使用statement模式保存binlog，对于statement模式无法复制的操作使用row，MySQL会根据执行的SQL语句选择记录模式

- 文件结构

  MySQL的Binlog记录的是对数据库的修改操作，用来表示修改操作的数据结构是Log event。不同的修改操作对应不同的log event。比较常用的log event有：Query event、Row event、Xid event

  log event结构如下：

  ![image-20200814102337819](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814102337819.png)

  

- 文件操作

  - 状态查看
  
    show variables like 'log_bin'
  
  - 开启binlog功能
  
    修改配置文件my.cnf或my.ini，重启服务
  
    ```
    #log-bin=ON 
    #log-bin-basename=mysqlbinlog 
    binlog-format=ROW 
    log-bin=mysqlbinlog
    
    ```
  
  - 使用show binlog events命令
  
    ```
    show binary logs; //等价于show master logs; 
    show master status; 
    show binlog events;
    show binlog events in 'mysqlbinlog.000001'
    ```
  
  - 使用mysqlbinlog命令
  
    ```
    mysqlbinlog "文件名" 
    mysqlbinlog "文件名" > "test.sql"
    
    ```
  
  - 使用binlog恢复数据
  
    ```
    //按指定时间恢复 
    mysqlbinlog --start-datetime="2020-04-25 18:00:00" --stopdatetime="2020-04-26 00:00:00" mysqlbinlog.000002 | mysql -uroot -p1234 
    
    //按事件位置号恢复 
    mysqlbinlog --start-position=154 --stop-position=957 mysqlbinlog.000002 | mysql -uroot -p1234
    
    ```
  
    mysqldump：定期全部备份数据库数据；mysqlbinlog：增量备份和恢复操作
  
  - 删除binlog文件
  
    ```
    purge binary logs to 'mysqlbinlog.000001'; //删除指定文件
    purge binary logs before '2020-04-28 00:00:00'; //删除指定时间之前的文件 
    reset master; //清除所有文件
    ```
  
    expire_logs_days参数来启动自动清理功能。默认值为0表示没启用；设置为1表示满1天binlog文件自动删除掉

redo log VS Binlog

- redo log属于innodb引擎功能；Binlog属于mysql自带功能且以二进制形式记录
- redo log属于物理日志，记录该数据页变更状态内容；Binlog属于逻辑日志，记录更新过程
- redo log日志循环写，日志空间大小是固定的；Binlog追加写
- redo log用于服务器异常宕机后事务数据自动恢复；Binlog用于主从复制和数据恢复使用，Binlog没有自动crash-safe能力

#### MySQL索引存储机制和工作原理

##### 索引类型

- 索引存储结构划分：B Tree索引、Hash索引、fulltext全文索引、R tree索引
- 应用层次划分：普通索引、唯一索引、主键索引、复合索引
- 索引键值类型划分：主键索引、辅助索引（二级索引）
- 数据存储和索引键值逻辑关系划分：聚集索引（聚簇索引）、非聚集索引（非聚簇索引）

###### 普通索引

这是最基本的索引类型，基于普通字段建立的索引，没有任何限制

创建普通索引的方法如下：
	CREATE INDEX <索引的名字> ON tablename (字段名); 

​	ALTER TABLE tablename ADD INDEX [索引的名字] (字段名); 

​	CREATE TABLE tablename ( [...], INDEX [索引的名字] (字段名) 

###### 唯一索引

与"普通索引"类似，不同的就是：索引字段的值必须唯一，但允许有空值 。在创建或修改表时追加唯一 约束，就会自动创建对应的唯一索引

创建唯一索引的方法如下：
	CREATE UNIQUE INDEX <索引的名字> ON tablename (字段名); 		  	ALTER TABLE tablename ADD UNIQUE INDEX [索引的名字] (字段名); 	CREATE TABLE tablename ( [...], UNIQUE [索引的名字] (字段名) ; 

###### 主键索引

它是一种特殊的唯一索引，不允许有空值。在创建或修改表时追加主键约束即可，每个表只能有一个主键

创建主键索引的方法如下：
	CREATE TABLE tablename ( [...], PRIMARY KEY (字段名) ); 

​	ALTER TABLE tablename ADD PRIMARY KEY (字段名); 

###### 复合索引

用户可以在多个列上建立索 引，这种索引叫做组复合索引（组合索引）。复合索引可以代替多个单一索引，相比多个单一索引复合 索引所需的开销更小

索引同时有两个概念叫做窄索引和宽索引，窄索引是指索引列为1-2列的索引，宽索引也就是索引列超 过2列的索引，设计索引的一个重要原则就是能用窄索引不用宽索引，因为窄索引往往比组合索引更有效

创建组合索引的方法如下：
	CREATE INDEX <索引的名字> ON tablename (字段名1，字段名2...); 	ALTER TABLE tablename ADD INDEX [索引的名字] (字段名1，字段名2...);

​	CREATE TABLE tablename ( [...], INDEX [索引的名字] (字段名1，字段名2...) );

复合索引使用注意事项：
	何时使用复合索引，要根据where条件建索引，注意不要过多使用索引，过多使用会对更新操作效 率有很大影响。 如果表已经建立了(col1，col2)，就没有必要再单独建立（col1）；

​	如果现在有(col1)索引，如果查 询需要col1和col2条件，可以建立(col1,col2)复合索引，对于查询有一定提高。 

###### 全文索引

查询操作在数据量比较少时，可以使用like模糊查询，但是对于大量的文本数据检索，效率很低。如果 使用全文索引，查询速度会比like快很多倍。在MySQL 5.6 以前的版本，只有MyISAM存储引擎支持全 文索引，从MySQL 5.6开始MyISAM和InnoDB存储引擎均支持
创建全文索引的方法如下：
		CREATE FULLTEXT INDEX <索引的名字> ON tablename (字段名); 		ALTER TABLE tablename ADD FULLTEXT [索引的名字] (字段名); 		CREATE TABLE tablename ( [...], FULLTEXT KEY [索引的名字] (字段名) ; 

全文索引使用语法：

```
select * from user  where match(name) against('aaa');
```

全文索引使用注意事项：

- 全文索引必须在字符串、文本字段上建立

- 全文索引字段值必须在小字符和大字符之间的才会有效。（innodb：3-84；myisam：484） 

- 全文索引字段值要进行切词处理，按syntax字符进行切割，例如b+aaa，切分成b和aaa 

- 全文索引匹配查询，默认使用的是等值匹配，例如a匹配a，不会匹配ab,ac。如果想匹配可以在布 尔模式下搜索a*

  ```
  select * from user  where match(name) against('a*' in boolean mode);
  ```

##### 索引原理

索引是存储引擎用于快速查找记录的一种数据结构，需要开辟空间和数据维护

- 索引是物理数据页存储，在数据文件中（InnoDB，ibd文件），利用数据页(page)存储
- 索引可以加快检索速度，但是同时也会降低增删改操作速度，索引维护需要代价

###### 二分查找法

在有序数组中查找指定数据的搜索算法

优点：等值查询，范围查询性能优秀

缺点：更新数据、新增数据、删除数据维护成本高

###### Hash结构

底层由Hash表实现，根据键值<key,value>存储数据的结构。适合单值查询

优点：根据key查询value性能好

缺点：范围查询需要全表扫描

###### B+Tree结构

- B-Tree结构

  - 索引值和data数据分布在整棵树结构中 

  - 每个节点可以存放多个索引值及对应的data数据 

  - 树节点中的多个索引值从左到右升序排列

    ![](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814123528106.png)

  B树的搜索：从根节点开始，对节点内的索引值序列采用二分法查找，如果命中就结束查找。没有 命中会进入子节点重复查找过程，直到所对应的的节点指针为空，或已经是叶子节点了才结束

- B+Tree结构

  - 非叶子节点不存储data数据，只存储索引值，这样便于存储更多的索引值 

  - 叶子节点包含了所有的索引值和data数据

  - 叶子节点用指针连接，提高区间的访问性能

    ![image-20200814123716229](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814123716229.png)

    相比B树，B+树进行范围查找时，只需要查找定位两个节点的索引值，然后利用叶子节点的指针进行遍历即可。而B树需要遍历范围内所有的节点和数据，显然B+Tree效率高

###### 聚簇索引和辅助索引

聚簇索引和非聚簇索引：B+Tree的叶子节点存放主键索引值和行记录就属于聚簇索引；如果索引值和行记录分开存放就属于非聚簇索引

主键索引和辅助索引：B+Tree的叶子节点存放的是主键字段值就属于主键索引；如果存放的是非主键值 就属于辅助索引（二级索引）

在InnoDB引擎中，主键索引采用的就是聚簇索引结构存储

- 聚簇索引（聚集索引）

  聚簇索引是一种数据存储方式，InnoDB的聚簇索引就是按照主键顺序构建 B+Tree结构。B+Tree 的叶子节点就是行记录，行记录和主键值紧凑地存储在一起。 这也意味着 InnoDB 的主键索引就 是数据表本身，它按主键顺序存放了整张表的数据，占用的空间就是整个表数据量的大小

  InnoDB的表要求必须要有聚簇索引： 

  - 如果表定义了主键，则主键索引就是聚簇索引 
  - 如果表没有定义主键，则第一个非空unique列作为聚簇索引 
  - 否则InnoDB会从建一个隐藏的row-id作为聚簇索引 

- 辅助索引 

  InnoDB辅助索引，也叫作二级索引，是根据索引列构建 B+Tree结构。但在 B+Tree 的叶子节点中 只存了索引列和主键的信息。二级索引占用的空间会比聚簇索引小很多， 通常创建辅助索引就是 为了提升查询效率。一个表InnoDB只能创建一个聚簇索引，但可以创建多个辅助索引

  **![image-20200814124633242](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814124633242.png)**

- 非聚簇索引

  与InnoDB表存储不同，MyISAM数据表的索引文件和数据文件是分开的，被称为非聚簇索引结构

  ![image-20200814124709527](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814124709527.png)

##### 索引分析与优化

###### explain

![image-20200814125026945](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814125026945.png)

- select_type：查询的类型

  - SIMPLE ： 表示查询语句不包含子查询或union 
  - PRIMARY：表示此查询是外层的查询 
  - UNION：表示此查询是UNION的第二个或后续的查询
  - DEPENDENT UNION：UNION中的第二个或后续的查询语句，使用了外面查询结果 
  - UNION RESULT：UNION的结果 
  - SUBQUERY：SELECT子查询语句 
  - DEPENDENT SUBQUERY：SELECT子查询语句依赖外层查询的结果

- type：存储引擎查询数据时采用的方式

  - ALL：表示全表扫描，性能差
  - index：表示基于索引的全表扫描，先扫描索引再扫描全表数据
  - range：表示使用索引范围查询。使用>、>=、<、<=、in等等
  - ref：表示使用非唯一索引进行单值查询
  - eq_ref：一般情况下出现在多表join查询，表示前面表的每一个记录，都只能匹配后面表的一 行结果
  - const：表示使用主键或唯一索引做等值查询，常量查询
  - NULL：表示不用访问表，速度快

- possible_keys：表示查询时可能会使用到的索引

- key：示查询时真正使用到的索引

- rows：MySQL查询优化器会根据统计信息，估算SQL要查询到结果需要扫描多少行记录

- key_len：表示查询使用的索引的字节数量

  key_len的计算规则如下： 

  - 字符串类型 

    字符串长度跟字符集有关：latin1=1、gbk=2、utf8=3、utf8mb4=4

     char(n)：n*字符集长度 

    varchar(n)：n * 字符集长度 + 2字节 

  - 数值类型 

    TINYINT：1个字节 

    SMALLINT：2个字节

     MEDIUMINT：3个字节 

    INT、FLOAT：4个字节 

    BIGINT、DOUBLE：8个字节 

  - 时间类型 

    DATE：3个字节 

    TIMESTAMP：4个字节 

    DATETIME：8个字节 

  - 字段属性 

    NULL属性占用1个字节，如果一个字段设置了NOT NULL，则没有此项

- Extra：额外的信息

  - Using where ：表示查询需要通过索引回表
  - Using index ：表示查询需要通过索引，索引就可以满足所需数据
  - Using ﬁlesort ：表示查询出来的结果需要额外排序，数据量小在内存，大的话在磁盘，因此有Using ﬁlesort 建议优化
  - Using temprorary ：查询使用到了临时表，一般出现于去重、分组等操作

###### 回表查询

InnoDB索引有聚簇索引和辅助索引。聚簇索引的叶子节点存储行记录有且只有一个。辅助索引的叶子节点存储的是主键值和索引字段值，通过辅助索引无法直接定位行记录，通常情况下，需要扫描两遍索引树。先通过辅助索引定位主键值，然后再通过聚簇索引定位行记录，这就叫做回表查询，它的性能比扫一遍索引树低
总结：通过索引查询主键值，然后再去聚簇索引查询记录信息 

###### 覆盖索引

只需要在一棵索引树上就能获得SQL所需的所有列数据，无需回表，速度更快，这就叫做索引覆盖

实现索引覆盖常见的方法就是：将被查询的字段，建立到组合索引

###### 最左前缀原则

复合索引使用时遵循左前缀原则，左前缀顾名思义，就是左优先，即查询中使用到左边的列， 那么查询就会使用到索引

###### like查询

面试题：MySQL在使用like模糊查询时，索引能不能起作用？ 

回答：MySQL在使用Like模糊查询时，索引是可以被使用的，只有把%字符写在后面才会使用到索引。
select * from user where name like '%o%';  //不起作用 

select * from user where name like 'o%';  //起作用 

select * from user where name like '%o';  //不起作用

###### null查询

面试题：如果MySQL表的某一列含有NULL值，那么包含该列的索引是否有效？

对MySQL来说，NULL是一个特殊的值，从概念上讲，NULL意味着“一个未知值”，它的处理方式与其他 值有些不同。比如：不能使用=，<，>这样的运算符，对NULL做算术运算的结果都是NULL，count时 不会包括NULL行等，NULL比空字符串需要更多的存储空间等

NULL列需要增加额外空间来记录其值是否为NULL。对于MyISAM表，每一个空列额外占用一位，四舍 五入到接近的字节

虽然MySQL可以在含有NULL的列上使用索引，但NULL和其他数据还是有区别的，不建议列上允许为 NULL。最好设置为NOT NULL，并给一个默认值，比如0和 ‘’ 空字符串等。如果是datetime类型，也可以 设置系统当前时间或某个固定的特殊值，例如'1970-01-01 00:00:00'

##### 索引与排序

MySQL查询支持ﬁlesort和index两种方式的排序，ﬁlesort是先把结果查出，然后在缓存或磁盘进行排序 操作，效率较低。使用index是指利用索引自动实现排序，不需另做排序操作，效率会比较高

ﬁlesort有两种排序算法：双路排序和单路排序

- 双路排序：需要两次磁盘扫描读取，终得到用户数据。第一次将排序字段读取出来，然后排序；第二 次去读取其他字段数据

- 单路排序：从磁盘查询所需的所有列数据，然后在内存排序将结果返回。如果查询数据超出缓存 sort_buﬀer，会导致多次磁盘读取操作，并创建临时表，后产生了多次IO，反而会增加负担。解决方 案：少使用select *；增加sort_buﬀer_size容量和max_length_for_sort_data容量

  如果我们Explain分析SQL，结果中Extra属性显示Using ﬁlesort，表示使用了ﬁlesort排序方式，需要优 化。如果Extra属性显示Using index时，表示覆盖索引，也表示所有操作在索引上完成，也可以使用 index排序方式，建议大家尽可能采用覆盖索引

##### 查询优化

###### 慢查询定位

- 开启慢查询日志

  ```
  SHOW VARIABLES LIKE 'slow_query_log%'
  ```

  通过如下命令开启慢查询日志：

  ```
  SET global slow_query_log = ON; 
  SET global slow_query_log_file = 'OAK-slow.log'; 
  SET global log_queries_not_using_indexes = ON; 
  SET long_query_time = 10;
  
  ```

  - long_query_time：指定慢查询的阀值，单位秒。如果SQL执行时间超过阀值，就属于慢查询，需要 记录到日志文件中
  - log_queries_not_using_indexes：表示会记录没有使用索引的查询SQL。前提slow_query_log 的值为ON

- 慢查询原因总结

  - 全表扫描：explain分析type属性all 
  - 全索引扫描：explain分析type属性index 
  - 索引过滤性不好：与索引字段选型、数据量和状态、表设计有关
  - 频繁的回表查询开销：尽量少用select *，使用覆盖索引 

#### MySQL事务和锁工作原理

##### ACID特性

###### 原子性

原子性：事务是一个原子操作单元，其对数据的修改，要么全都执行，要么全都不执行。
修改---》Buﬀer Pool修改---》刷盘。可能会有下面两种情况： 

- 事务提交了，如果此时Buﬀer Pool的脏页没有刷盘，如何保证修改的数据生效？ Redo 
- 如果事务没提交，但是Buﬀer Pool的脏页刷盘了，如何保证不该存在的数据撤销？Undo 

###### 持久性

持久性：指的是一个事务一旦提交，它对数据库中数据的改变就应该是永久性的，后续的操作或故障不应该对其有任何影响

###### 隔离性

隔离性：指的是一个事务的执行不能被其他事务干扰，即一个事务内部的操作及使用的数据对其他的并 发事务是隔离的

InnoDB 支持的隔离性有 4 种，隔离性从低到高分别为：读未提交、读提交、可重复读、可串行化

锁和多版本控制（MVCC）技术就是用于保障隔离性的

###### 一致性

一致性：指的是事务开始之前和事务结束之后，数据库的完整性限制未被破坏。一致性包括两方面的内 容，分别是约束一致性和数据一致性

###### WAL技术

WAL的全称为Write-Ahead Logging，先写日志，再写磁盘

![image-20200814132448096](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814132448096.png)

##### 事务控制的演进

###### 并发事务

事务的并发处理可能会带来一些问题，比如：更新丢失、脏读、不可重复读、幻读等

- 更新丢失

  当两个或多个事务更新同一行记录，会产生更新丢失现象，分为回滚覆盖和提交覆盖

  回滚覆盖：一个事务回滚操作，将其他事务已提交的数据覆盖了

  提交覆盖：一个事务提交，将其他事务以提交的数据给覆盖了

- 脏读

  一个事务读到了另一个事务修改但未提交的事务

- 不可重复读

  一个事务中多次读取同一行记录，数据不一致

- 幻读

  一个事务中多次按相同条件查询，结果不一致

###### 排队

按顺序执行所有事务操作。特点：强一致性、处理性能低

###### 排他锁

如果事务之间涉及相同的数据项时，会使用排他锁，先进入的事务独占数据项以后，其他事务被阻塞，等待前面的事务释放锁

![image-20200814195343589](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814195343589.png)

###### 读写锁

读写锁就是进一步细化锁的颗粒度，区分读操作和写操作，让读和读之间不加锁，这样下面的两个事务 就可以同时被执行了

![image-20200814195515405](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814195515405.png)

###### MVCC

多版本控制MVCC，也就是Copy on Write的思想。MVCC除了支持读和读并行，还支持读和写、写和读 的并行，但为了保证一致性，写和写是无法并行的

![image-20200814195604264](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814195604264.png)

在事务1开始写操作的时候会copy一个记录的副本，其他事务读操作会读取这个记录副本，因此不会影 响其他事务对此记录的读取，实现写和读并行

- MVCC概念

  MVCC多版本控制，是指在数据库中为了实现高并发的数据访问，对数据进行多版本处理，并通过事务的可见性来保证事务能看到相应的数据版本

  如何生成多版本？每次事务修改操作之前，都会在Undo日志中记录修改之前的数据状态和事务号，该备份记录可用于其他事务的读取，也可进行数据回滚

- MVCC实现原理

  目前MVCC只在 Read Commited 和 Repeatable Read 两种隔离级别下工作

  在 MVCC 并发控制中，读操作可以分为两类: 快照读（Snapshot Read）与当前读 （Current Read）

  - 快照读：读取的是记录的快照版本（有可能是历史版本），不用加锁（select） 
  - 当前读：读取的是记录的新版本，并且当前读返回的记录，都会加锁，保证其他事务不会再并发修改这条记录（select... for update 或lock in share mode，insert/delete/update） 

MVCC已经实现了读读、读写、写读并发处理，如果想进一步解决写写冲突，可以采用下面两种方案：

- 乐观锁 
- 悲观锁 

##### 事务隔离级别

![image-20200814200523400](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814200523400.png)

- 读未提交

  Read Uncommitted 读未提交：解决了回滚覆盖类型的更新丢失，但可能发生脏读现象，也就是 可能读取到其他会话中未提交事务修改的数据

- 已提交读

  Read Committed 读已提交：只能读取到其他会话中已经提交的数据，解决了脏读。但可能发生 不可重复读现象，也就是可能在一个事务中两次查询结果不一致

- 可重复读

  Repeatable Read 可重复读：解决了不可重复读，它确保同一事务的多个实例在并发读取数据 时，会看到同样的数据行。不过理论上会出现幻读，简单的说幻读指的的当用户读取某一范围的数 据行时，另一个事务又在该范围插入了新行，当用户在读取该范围的数据时会发现有新的幻影行

- 可串行化

  Serializable 串行化：所有的增删改查串行执行。它通过强制事务排序，解决相互冲突，从而解决幻读的问题。这个级别可能导致大量的超时现象的和锁竞争，效率低下

数据库的事务隔离级别越高，并发问题就越小，但是并发处理能力越差（代价）。读未提交隔离级别 低，并发问题多，但是并发处理能力好。以后使用时，可以根据系统特点来选择一个合适的隔离级别， 比如对不可重复读和幻读并不敏感，更多关心数据库并发处理能力，此时可以使用Read Commited隔离级别

事务隔离级别，针对Innodb引擎，支持事务的功能。像MyISAM引擎没有关系

###### 事务隔离级别和锁的关系

- 事务隔离级别是SQL92定制的标准，相当于事务并发控制的整体解决方案，本质上是对锁和MVCC使 用的封装，隐藏了底层细节
- 锁是数据库实现并发控制的基础，事务隔离性是采用锁来实现，对相应操作加不同的锁，就可以防 止其他事务同时对数据进行读写操作
- 对用户来讲，首先选择使用隔离级别，当选用的隔离级别不能解决并发问题或需求时，才有必要在开发中手动的设置锁

MySQL默认隔离级别：可重复读

Oracle、SQLServer默认隔离级别：读已提交

##### 锁机制和实战

###### 锁分类

锁有不同的分类:

- 从操作的粒度可分为表级锁、行级锁和页级锁

  - 表级锁：每次操作锁住整张表。锁定粒度大，发生锁冲突的概率高，并发度低。应用在 MyISAM、InnoDB、BDB 等存储引擎中

  - 行级锁：每次操作锁住一行数据。锁定粒度小，发生锁冲突的概率低，并发度高。应 用在InnoDB 存储引擎中

  - 页级锁：每次锁定相邻的一组记录，锁定粒度界于表锁和行锁之间，开销和加锁时间界于表 锁和行锁之间，并发度一般。应用在BDB 存储引擎中

    ![image-20200814201620739](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814201620739.png)

- 从操作的类型可分为读锁和写锁

  - 读锁（S锁）：共享锁，针对同一份数据，多个读操作可以同时进行而不会互相影响

  - 写锁（X锁）：排他锁，当前写操作没有完成前，它会阻断其他写锁和读锁

    IS锁、IX锁：意向读锁、意向写锁，属于表级锁，S和X主要针对行级锁。在对表记录添加S或X锁之 前，会先对表添加IS或IX锁。 S锁：事务A对记录添加了S锁，可以对记录进行读操作，不能做修改，其他事务可以对该记录追加 S锁，但是不能追加X锁，需要追加X锁，需要等记录的S锁全部释放。 X锁：事务A对记录添加了X锁，可以对记录进行读和修改操作，其他事务不能对记录做读和修改操作

- 从操作的性能可分为乐观锁和悲观锁

  - 乐观锁：一般的实现方式是对记录数据版本进行比对，在数据更新提交的时候才会进行冲突 检测，如果发现冲突了，则提示错误信息
  - 悲观锁：在对一条数据修改的时候，为了避免同时被其他人修改，在修改数据之前先锁定， 再修改的控制方式。共享锁和排他锁是悲观锁的不同实现，但都属于悲观锁范畴

###### 行锁原理

在InnoDB引擎中，我们可以使用行锁和表锁，其中行锁又分为共享锁和排他锁。InnoDB行锁是通过对 索引数据页上的记录加锁实现的，主要实现算法有 3 种：Record Lock、Gap Lock 和 Next-key Lock

- RecordLock锁：锁定单个行记录的锁。（记录锁，RC、RR隔离级别都支持） 

- GapLock锁：间隙锁，锁定索引记录间隙，确保索引记录的间隙不变。（范围锁，RR隔离级别支 持） 

- Next-key Lock 锁：记录锁和间隙锁组合，同时锁住数据，并且锁住数据前后范围。（记录锁+范 围锁，RR隔离级别支持） 

  在RR隔离级别，InnoDB对于记录加锁行为都是先采用Next-Key Lock，但是当SQL操作含有唯一索引 时，Innodb会对Next-Key Lock进行优化，降级为RecordLock，仅锁住索引本身而非范围

   1）select ... from 语句：InnoDB引擎采用MVCC机制实现非阻塞读，所以对于普通的select语句， InnoDB不加锁

   2）select ... from lock in share mode语句：追加了共享锁，InnoDB会使用Next-Key Lock锁进行处 理，如果扫描发现唯一索引，可以降级为RecordLock锁
   3）select ... from for update语句：追加了排他锁，InnoDB会使用Next-Key Lock锁进行处理，如果扫 描发现唯一索引，可以降级为RecordLock锁

   4）update ... where 语句：InnoDB会使用Next-Key Lock锁进行处理，如果扫描发现唯一索引，可以 降级为RecordLock锁

  5）delete ... where 语句：InnoDB会使用Next-Key Lock锁进行处理，如果扫描发现唯一索引，可以降 级为RecordLock锁
   6）insert语句：InnoDB会在将要插入的那一行设置一个排他的RecordLock锁

###### 悲观锁

悲观锁（Pessimistic Locking），是指在数据处理过程，将数据处于锁定状态，一般使用数据库的锁机 制实现。从广义上来讲，前面提到的行锁、表锁、读锁、写锁、共享锁、排他锁等，这些都属于悲观锁范畴

- 表级锁

  表级锁每次操作都锁住整张表，并发度低。常用命令如下:

  ```
  # 手动增加表锁
  lock table 表名称 read|write,表名称2 read|write;
  # 查看表上加过的锁
  show open tables;
  #删除表锁
  unlock tables;
  ```

  表级读锁：当前表追加read锁，当前连接和其他的连接都可以读操作；但是当前连接增删改操作 会报错，其他连接增删改会被阻塞

  表级写锁：当前表追加write锁，当前连接可以对表做增删改查操作，其他连接对该表所有操作都 被阻塞（包括查询）

  总结：表级读锁会阻塞写操作，但是不会阻塞读操作。而写锁则会把读和写操作都阻塞

- 共享锁

  共享锁又称为读锁，简称S锁。共享锁就是多个事务对于同一数据可以共享一把锁，都能访问到数 据，但是只能读不能修改。使用共享锁的方法是在select ... lock in share mode，只适用查询语句

  总结：事务使用了共享锁（读锁），只能读取，不能修改，修改操作被阻塞

- 排他锁

- 排他锁又称为写锁，简称X锁。排他锁就是不能与其他锁并存，如一个事务获取了一个数据行的排 他锁，其他事务就不能对该行记录做其他操作，也不能获取该行的锁。 使用排他锁的方法是在SQL末尾加上for update，innodb引擎默认会在update，delete语句加上 for update。行级锁的实现其实是依靠其对应的索引，所以如果操作没用到索引的查询，那么会锁住全表记录

  总结：事务使用了排他锁（写锁），当前事务可以读取和修改，其他事务不能修改，也不能获取记录 锁（select... for update）。如果查询没有使用到索引，将会锁住整个表记录

###### 乐观锁

乐观锁是相对于悲观锁而言的，它不是数据库提供的功能，需要开发者自己去实现。在数据库操作时， 想法很乐观，认为这次的操作不会导致冲突，因此在数据库操作时并不做任何的特殊处理，即不加锁， 而是在进行事务提交时再去判断是否有冲突了

乐观锁实现的关键点：冲突的检测

- 悲观锁和乐观锁都可以解决事务写写并发，在应用中可以根据并发处理能力选择区分，比如对并发率要 求高的选择乐观锁；对于并发率要求低的可以选择悲观锁

#### MySQL集群架构及相关原理

##### 集群架构设计

###### 架构设计理念

在集群架构设计中,主要遵循以下三个维度:

- 可用性
- 扩展性
- 一致性

###### 可用性设计

保证好可用的方法是冗余,但是数据冗余会带来数据一致性问题

实现高可用的方案有以下几种架构模式:

- 主从模式

  简单灵活，但是写操作高可用需要自行处理

- 双主模式

  互为主从，有双主双写、双主单写两种方式，建议使用双主单写 

###### 扩展性设计

扩展性主要围绕着读操作扩展和写操作扩展展开

- 如何扩展以提高读性能
  - 加从库
  - 分库分表
- 如何扩展以提高写性能 
  - 分库分表

###### 一致性设计

一致性主要考虑集群中各数据库数据同步以及同步延迟问题。可以采用的方案如下：

- 不使用从库 

  扩展读性能问题需要单独考虑，否则容易出现系统瓶颈

- 增加访问路由层 

  先得到主从同步长时间t，在数据发生修改后的t时间内，先访问主库

##### 主从模式

###### 适用场景

MySQL主从模式是指数据可以从一个MySQL数据库服务器主节点复制到一个或多个从节点。MySQL 默 认采用异步复制方式，从节点可以复制主数据库中的所有数据库，或者特定的数据库，或者特定的表

![image-20200814204936178](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814204936178.png)

mysql主从复制用途:

- 实时灾备，用于故障切换（高可用） 
- 读写分离，提供查询服务（读扩展） 
- 数据备份，避免影响业务（高可用）

主从部署必要条件:

- 从库服务器能连通主库
- 主库开启binlog日志（设置log-bin参数） 
- 主从server-id不同 

###### 实现原理

- 主从复制

  ![image-20200814205423834](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814205423834.png)

  主从复制整体分为以下三个步骤:

  - 主库将数据库的变更操作记录到Binlog日志文件中 
  - 从库读取主库中的Binlog日志文件信息写入到从库的Relay Log中继日志中
  - 从库读取中继日志信息在从库中进行Replay,更新从库数据信息 

  mysql主从复制存在的问题:

  - 主库宕机后，数据可能丢失

  - 从库只有一个SQL Thread，主库写压力大，复制很可能延时

    解决方案:

    - 半同步复制---解决数据丢失的问题
    - 并行复制----解决从库复制延迟的问题 

- 半同步复制

  为了提升数据安全，MySQL让Master在某一个时间点等待Slave节点的 ACK（Acknowledge character）消息，接收到ACK消息后才进行事务提交，这也是半同步复制的基础，MySQL从5.5版本开 始引入了半同步复制机制来降低数据丢失的概率

  ![image-20200814205811985](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814205811985.png)

  主库等待从库写入 relay log 并返回 ACK 后才进行事务Commit

###### 并行复制

MySQL 5.6版基于库的并行复制

MySQL 5.7是基于组提交的并行复制

slave-parallel-type可配置基于库还是基于表:Database   logical_clock

MySQL 5.7中组提交的并行复制究竟是如何实现的？

MySQL 5.7是通过对事务进行分组，当事务提交时，它们将在单个操作中写入到二进制日志中。如果多 个事务能同时提交成功，那么它们意味着没有冲突，因此可以在Slave上并行执行，所以通过在主库上 的二进制日志中添加组提交信息。
MySQL 5.7的并行复制基于一个前提，即所有已经处于prepare阶段的事务，都是可以并行提交的。这 些当然也可以在从库中并行提交，因为处理这个阶段的事务都是没有冲突的。在一个组里提交的事务， 一定不会修改同一行。这是一种新的并行复制思路，完全摆脱了原来一直致力于为了防止冲突而做的分 发算法，等待策略等复杂的而又效率底下的工作。
InnoDB事务提交采用的是两阶段提交模式。一个阶段是prepare，另一个是commit

并行复制配置与调优

- binlog_transaction_dependency_history_size

  用于控制集合变量的大小

- binlog_transaction_dependency_tracking

  用于控制binlog文件中事务之间的依赖关系,即last_commited值

  - COMMIT_ORDERE: 基于组提交机制 
  - WRITESET: 基于写集合机制 
  - WRITESET_SESSION: 基于写集合，比writeset多了一个约束，同一个session中的事务 last_committed按先后顺序递增 

- transaction_write_set_extraction

  用于控制事务的检测算法，参数值为：OFF、 XXHASH64、MURMUR32

- master_info_repository

  开启MTS功能后，务必将参数master_info_repostitory设置为TABLE，这样性能可以有50%~80% 的提升。这是因为并行复制开启后对于元master.info这个文件的更新将会大幅提升，资源的竞争 也会变大

- slave_parallel_workers

  若将slave_parallel_workers设置为0，则MySQL 5.7退化为原单线程复制，但将 slave_parallel_workers设置为1，则SQL线程功能转化为coordinator线程，但是只有1个worker 线程进行回放，也是单线程复制。然而，这两种性能却又有一些的区别，因为多了一次 coordinator线程的转发，因此slave_parallel_workers=1的性能反而比0还要差

- slave_preserve_commit_order

  MySQL 5.7后的MTS可以实现更小粒度的并行复制，但需要将slave_parallel_type设置为 LOGICAL_CLOCK，但仅仅设置为LOGICAL_CLOCK也会存在问题，因为此时在slave上应用事务的 顺序是无序的，和relay log中记录的事务顺序不一样，这样数据一致性是无法保证的，为了保证事 务是按照relay log中记录的顺序来回放，就需要开启参数slave_preserve_commit_order

  要开启enhanced multi-threaded slave其实很简单，只需根据如下设置：

  ```
  slave-parallel-type=LOGICAL_CLOCK 
  slave-parallel-workers=16 
  slave_pending_jobs_size_max = 2147483648 slave_preserve_commit_order=1 master_info_repository=TABLE relay_log_info_repository=TABLE 
  relay_log_recovery=ON
  ```

并行复制监控

在使用了MTS后，复制的监控依旧可以通过SHOW SLAVE STATUS\G，但是MySQL 5.7在 performance_schema库中提供了很多元数据表，可以更详细的监控并行复制过程

```
mysql> show tables like 'replication%'; 
+---------------------------------------------+ 
| Tables_in_performance_schema (replication%) | 
+---------------------------------------------+ 
| replication_applier_configuration           | 
| replication_applier_status                  | 
| replication_applier_status_by_coordinator   | 
| replication_applier_status_by_worker        |
| replication_connection_configuration        | 
| replication_connection_status               | 
| replication_group_member_stats              | 
| replication_group_members                   | 
+---------------------------------------------+

```

通过replication_applier_status_by_worker可以看到worker进程的工作情况：

```
select * from replication_applier_status_by_worker; 
```

###### 读写分离

###### 读写分离引入时机

![image-20200814224130758](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814224130758.png)

在应用中可以在从库追加多个索引来优化查询，主库这些索引可以不加，用于提升写效率
读写分离架构也能够消除读写锁冲突从而提升数据库的读写性能。使用读写分离架构需要注意：主从同 步延迟和读写分配机制问题

###### 主从同步延迟

使用读写分离架构时，数据库主从同步具有延迟性，数据一致性会有影响，对于一些实时性要求比较高 的操作，可以采用以下解决方案

- 写后立刻读 

  在写入数据库后，某个时间段内读操作就去主库，之后读操作访问从库

- 二次查询 

  先去从库读取数据，找不到时就去主库进行数据读取。该操作容易将读压力返还给主库，为了避免 恶意攻击，建议对数据库访问API操作进行封装，有利于安全和低耦合

- 根据业务特殊处理 

  根据业务特点和重要程度进行调整，比如重要的，实时性要求高的业务数据读写可以放在主库。对 于次要的业务，实时性要求不高可以进行读写分离，查询时去从库查询

###### 读写分离落地

读写路由分配机制是实现读写分离架构关键的一个环节，就是控制何时去主库写，何时去从库读。目 前较为常见的实现方案分为以下两种：

- 基于编程和配置实现（应用端）

  在代码中封装数据库的操作，代码中可以根据操作类型进行路由分配，增删改时操作主库， 查询时操作从库

- 基于服务器端代理实现（服务器端）

  ![image-20200814224521218](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814224521218.png)

  中间件代理一般介于应用服务器和数据库服务器之间，从图中可以看到，应用服务器并不直接进入 到master数据库或者slave数据库，而是进入MySQL proxy代理服务器。代理服务器接收到应用服 务器的请求后，先进行判断然后转发到后端master和slave数据库

目前有很多性能不错的数据库中间件，常用的有MySQL Proxy、MyCat以及Shardingsphere等

- MySQL Proxy：是官方提供的MySQL中间件产品可以实现负载平衡、读写分离等
- MyCat：MyCat是一款基于阿里开源产品Cobar而研发的，基于 Java 语言编写的开源数据库中间件
- ShardingSphere：ShardingSphere是一套开源的分布式数据库中间件解决方案，它由ShardingJDBC、Sharding-Proxy和Sharding-Sidecar（计划中）这3款相互独立的产品组成。已经在2020 年4月16日从Apache孵化器毕业，成为Apache顶级项目

##### 双主模式

###### 适用场景

双主模式是指两台服务器互为主 从，任何一台服务器数据变更，都会通过复制应用到另外一方的数据库中

![image-20200814224716636](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814224716636.png)

使用双主双写还是双主单写？

建议大家使用双主单写，因为双主双写存在以下问题：

- ID冲突 

  在A主库写入，当A数据未同步到B主库时，对B主库写入，如果采用自动递增容易发生ID主键的冲 突。 可以采用MySQL自身的自动增长步长来解决，例如A的主键为1,3,5,7...，B的主键为2,4,6,8... ，但 是对数据库运维、扩展都不友好

- 更新丢失 

  同一条记录在两个主库中进行更新，会发生前面覆盖后面的更新丢失

![image-20200814224847968](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814224847968.png)

随着业务发展，架构会从主从模式演变为双主模式，建议用双主单写，再引入高可用组件，例如 Keepalived和MMM等工具，实现主库故障自动切换

###### MMM架构

MMM（Master-Master Replication Manager for MySQL）是一套用来管理和监控双主复制，支持双 主故障切换 的第三方软件。MMM 使用Perl语言开发，虽然是双主架构，但是业务上同一时间只允许一 个节点进行写入操作。下图是基于MMM实现的双主高可用架构

![image-20200814224933452](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814224933452.png)

###### MHA架构

MHA（Master High Availability）是一套比较成熟的 MySQL 高可用方案，也是一款优秀的故障切换和 主从提升的高可用软件。在MySQL故障切换过程中，MHA能做到在30秒之内自动完成数据库的故障切 换操作，并且在进行故障切换的过程中，MHA能在大程度上保证数据的一致性，以达到真正意义上的 高可用。MHA还支持在线快速将Master切换到其他主机，通常只需0.5－2秒

目前MHA主要支持一主多从的架构，要搭建MHA，要求一个复制集群中必须少有三台数据库服务器

![image-20200814225027378](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814225027378.png)

###### 主备切换

主备切换是指将备库变为主库，主库变为备库，有可靠性优先和可用性优先两种策略:

- 可靠性优先

  主备切换过程一般由专门的HA高可用组件完成，但是切换过程中会存在短时间不可用，因为在切 换过程中某一时刻主库A和从库B都处于只读状态。如下图所示:

  ![image-20200814225203995](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814225203995.png)

  主库由A切换到B，切换的具体流程如下： 

  - 判断从库B的Seconds_Behind_Master值，当小于某个值才继续下一步
  -  把主库A改为只读状态（readonly=true） 
  - 等待从库B的Seconds_Behind_Master值降为 0 
  - 把从库B改为可读写状态（readonly=false）
  -  把业务请求切换至从库B 

- 可用性优先

  不等主从同步完成， 直接把业务请求切换至从库B ，并且让从库B可读写 ，这样几乎不存在不可用时间，但可能会数据不一致

  ![image-20200814225330780](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814225330780.png)

  如上图所示，在A切换到B过程中，执行两个INSERT操作，过程如下： 

  - 主库A执行完 INSERT c=4 ，得到 (4,4) ，然后开始执行 主从切换

  - 主从之间有5S的同步延迟，从库B会先执行 INSERT c=5 ，得到 (4,5) 

  - 从库B执行主库A传过来的binlog日志 INSERT c=4 ，得到 (5,4) 

  - 主库A执行从库B传过来的binlog日志 INSERT c=5 ，得到 (5,5)

  -  此时主库A和从库B会有 两行 不一致的数据 

    通过上面介绍了解到，主备切换采用可用性优先策略，由于可能会导致数据不一致，所以大多数情 况下，优先选择可靠性优先策略。在满足数据可靠性的前提下，MySQL的可用性依赖于同步延时 的大小，同步延时越小，可用性就越高

##### 分库分表

使用分库分表时，主要有垂直拆分和水平拆分两种拆分模式，都属于物理空间的拆分

垂直拆分：由于表数量多导致的单个库大。将表拆分到多个库中

水平拆分：由于表记录多导致的单个库大。将表记录拆分到多个表中

###### 拆分方式

- 垂直拆分

  垂直拆分又称为纵向拆分，垂直拆分是将表按库进行分离，或者修改表结构按照访问的差异将某些 列拆分出去。应用时有垂直分库和垂直分表两种方式，一般谈到的垂直拆分主要指的是垂直分库

- 水平拆分

  水平拆分又称为横向拆分。 相对于垂直拆分，它不再将数据根据业务逻辑分类，而是通过某个字 段（或某几个字段），根据某种规则将数据分散至多个库或表中，每个表仅包含数据的一部分

###### 分片策略

分片（Sharding）就是用来确定数据在多台存储设备上分布的技术。Shard这个词的意思是“碎片”，如 果将一个数据库当作一块大玻璃，将这块玻璃打碎，那么每一小块都称为数据库的碎片（Database Sharding）。将一个数据库打碎成多个的过程就叫做分片，分片是属于横向扩展方案

分片：表示分配过程，是一个逻辑上概念，表示如何实现

分库分表：表示分配结果，是一个物理上概念，表示最终实现的结果

数据库扩展方案:

- 横向扩展：一个库变多个库，加机器数量 
- 纵向扩展：一个库还是一个库，优化机器性能，加高配CPU或内存

数据分片是根据指定的分片键和分片策略将数据水平拆分，拆分成多个数据片后分散到多个数据存储节 点中。分片键是用于划分和定位表的字段，一般使用ID或者时间字段。而分片策略是指分片的规则，常 用规则有以下几种

- 基于范围分片

  根据特定字段的范围进行拆分，比如用户ID、订单时间、产品价格等

  优点：新的数据可以落在新的存储节点上，如果集群扩容，数据无需迁移

  缺点：数据热点分布不均，数据冷热不均匀，导致节点负荷不均

- 哈希取模分片

  整型的Key可直接对设备数量取模，其他类型的字段可以先计算Key的哈希值，然后再对设备数量 取模。假设有n台设备，编号为0 ~ n-1，通过Hash(Key) % n就可以确定数据所在的设备编号。该 模式也称为离散分片

  优点：实现简单，数据分配比较均匀，不容易出现冷热不均，负荷不均的情况

  缺点：扩容时会产生大量的数据迁移，比如从n台设备扩容到n+1，绝大部分数据需要重新分配和 迁移

- 一致性哈希分片

  一致性Hash是将数据按照特征值映射到一个首尾相接的Hash环上，同时也将节点（按照IP地址或 者机器名Hash）映射到这个环上.对于数据，从数据在环上的位置开始，顺时针找到的第一个节 点即为数据的存储节点。Hash环示意图与数据的分布如下

  ![image-20200814233819644](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814233819644.png)

  一致性Hash在增加或者删除节点的时候，受到影响的数据是比较有限的，只会影响到Hash环相邻的节 点，不会发生大规模的数据迁移

###### 扩容方案

当系统用户进入了高速增长期时，即便是对数据进行分库分表，但数据库的容量，还有表的数据量也总 会达到天花板。当现有数据库达到承受极限时，就需要增加新服务器节点数量进行横向扩容

首先来思考一下，横向扩展会有什么技术难度？

![image-20200814234353521](C:\Users\x1850\AppData\Roaming\Typora\typora-user-images\image-20200814234353521.png)

- 数据迁移问题
- 分片规则改变
- 数据同步、时间点、数据一致性

遇到上述问题时,可以使用以下两种方案:

- 停机扩容

  这是一种很多人初期都会使用的方案，尤其是初期只有几台数据库的时候。停机扩容的具体步骤如下：

  - 站点发布一个公告，例如：“为了为广大用户提供更好的服务，本站点将在今晚00:00-2:00之间升 级，给您带来不便抱歉"
  - 时间到了，停止所有对外服务
  - 新增n个数据库，然后写一个数据迁移程序，将原有x个库的数据导入到新的y个库中。比如分片 规则由%x变为%y
  -  数据迁移完成，修改数据库服务配置，原来x个库的配置升级为y个库的配置 
  - 重启服务，连接新库重新对外提供服务

  回滚方案：万一数据迁移失败，需要将配置和数据回滚

  优点：简单

  缺点：

  停止服务，缺乏高可用 

  程序员压力山大，需要在指定时间完成 

  如果有问题没有及时测试出来启动了服务，运行后发现问题，数据会丢失一部分，难以回滚
  适用场景：
  小型网站 

  大部分游戏 

  对高可用要求不高的服务

- 平滑扩容

  数据库扩容的过程中，如果想要持续对外提供服务，保证服务的可用性，平滑扩容方案是好的选择。 平滑扩容就是将数据库数量扩容成原来的2倍，比如：由2个数据库扩容到4个数据库

