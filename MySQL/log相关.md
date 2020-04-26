# MySQL中常见的三种log

## Bin log

### 什么是bin log

记录了数据库表结构和表数据的变更，比如`update/delete/insert/create`等操作，不会记录`select`操作

### bin log的结构

存储每条变更的SQL和事务id等