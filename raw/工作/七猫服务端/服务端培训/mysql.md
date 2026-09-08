- 了解MySQL基本原理：
    
    - 【Mysql是什么？架构是怎么样的？-哔哩哔哩】 [https://b23.tv/9CDeE33](https://b23.tv/9CDeE33)
        
- 了解基本设计范式：
    
    - 【数据库第一范式，第二范式，第三范式讲解-哔哩哔哩】 [https://b23.tv/I85cMed](https://b23.tv/I85cMed)
        
- 熟悉DDL/DML 基本语法：
    
    - DDL/DML：【从零玩转MySQL-基础篇-哔哩哔哩】 [https://b23.tv/FULZ72d](https://b23.tv/FULZ72d)
        
    - 菜鸟教程： https://www.runoob.com/mysql/mysql-create-database.html
        
- 熟悉常用字段数据类型：
    
    - 菜鸟教程： https://www.runoob.com/mysql/mysql-data-types.html
        

目标：能创建一个简单 书籍表、章节表（注意字段类型选择），写出基本增删改查语句

- B+树索引：可以加速查找数据页的树
- 辅助索引：
- 进程内缓存：Buffer Pool，通过直接IO模式，绕过操作系统的缓存机制，直接从磁盘读写数据
- 自适应哈希索引：提升读性能
- change Buffer是什么：将写操作收集起来的地方。（不太懂）
- undo log: 更新buffer Pool数据页的时候，会用旧数据生成undo log记录，并会根据buffer pool的刷盘机制，不定时的写入到磁盘的undo log文件中
- redo log：将事务中更新数据行的操作都写入到redo log Buffer内存中，然后在事务提交的时候，进行redo log刷磁盘，将数据固化到redo log文件中，数据库进程奔溃重启后，就能通过redo Log File，找到历史操作记录，重做数据，保证了事务里的多行数据变更，要么都成功，要么都失败。
- redo Log File 是顺序写入的，Buffer Pool的内存数据是随机分配在磁盘各处的，顺序写磁盘的性能是随机写的几十倍。
- innoDB：平时写的sql语句，最终都会转换成InnoDb提供的接口函数调用。
- server层是什么：sql语句和InnoDB存储引擎之间的中间层。它提供了连接管理、分析器、优化器、执行器。server层和存储引擎层共同构成了一个完整的数据库。
- binlog是什么：server层会将历史上的所有变更操作都记录到磁盘上的日志文件中，误删表就可以用binlog来恢复数据。redo log和bin log有什么区别？
- 数据库查询更新流程：
	- 读操作：InnoDB存储引擎会先检查Buffer Pool是否存在所需的B+树数据页，如果存在则直接返回数据，如果没有就从磁盘中读取数据页，加载到buffer Pool中，再返回数据，若查询的数据是热点数据还会将数据页加入到自适应哈希索引中，加速后续的查询
	- 写操作：会先将数据写入到Buffer Pool中，并生成相应的Undo log记录，以便在事务回滚的时候，恢复数据的原始状态，接下来会将写操作记录到Redo Log Buffer中，这些redo log会周期性的写入到磁盘中的redo log文件中，就算数据库崩溃了，已提交的事务也不会丢失。对于 辅助索引 的更新操作，InnoDB会将这些更新暂时存储在change Buffer中，等到相关的索引页被读取到Buffer Pool时，再进行实际的更新操作，从而减少磁盘IO。同时所有的变更都会被记录到server层的binlog中，以便进行数据恢复。
- HDFS
Mysql是什么？ 数据页是什么？ Mysql数据页为什么是16KB. B+树是什么？ 数据页和索引页是什么？ Buffer Pool是什么？ 自适应哈希索引是什么？ Change Buffer是什么？ Undo log是什么？ Redo log是什么？ InnoDB是什么？ Myisam是什么？ Mysql Server是什么？ Binlog是什么？ 有Redo log为什么还要有binlog？ Mysql连接器是什么 Mysql优化器是什么 Mysql执行器是什么 Mysql分析器是什么？ Mysql执行计划是什么？ 数据库查询流程 数据库更新流程


数据库范式：1NF、2NF、3NF
- 函数依赖
	- 完全
	- 部分
- 1NF： 一列一个值
- 2NF：消除部分函数依赖，把多余的部分拿出来，单独创建一张表 
- 3NF：消除传递函数依赖，把多余的传递函数依赖拆分出来，单独建一张表



