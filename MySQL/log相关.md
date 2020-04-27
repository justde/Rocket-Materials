# MySQL中常见的三种log

## Bin log

### 什么是bin log

记录了数据库表结构和表数据的变更，比如`update/delete/insert/create`等操作，不会记录`select`操作

### bin log的结构

存储每条变更的SQL和事务id等，还有执行的时间

###  bin log是否完整

完整的bin log结尾

- statement格式 以commit结尾
- row格式最后以XID 结尾

### 用途

复制和恢复数据

- MySQL一般是一主多从结构，从服务器与主服务器数据保持一致，就是通过bin log来实现的
- 主数据库挂了可以通过bin log恢复

## Undo log

回滚和多版本并控制保证了隔离性

数据修改的时候，不但记录了redo log，还记录了undo log，如果因为某些原因导致事务回滚，可以使用undo log进行回滚

undo log主要存储了逻辑日志，比如insert一台记录，undo log记录了一条delete日志。可以保证原子性，一个事务内多个操作，要么全部执行，要么全部不执行

undo log存储着修改之前的数据，相当于前一个版本，mvcc实现的是读写不阻塞，读的时候只要返回前一个版本的数据就行

##  Redo log

