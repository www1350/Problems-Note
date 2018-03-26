## POI

### HSSFWorkbook超过**65536**行报错

错误信息 Invalid row number (65536) outside allowable range (0..65535)

**问题原因**  HSSFWorkbook:是操作Excel2003以前（包括2003）的版本，扩展名是.xls，一张表最大支持**65536**行数据，256列，也就是说一个sheet页，最多导出**6w多条数据**。

**解决方法** 

方案1 改用sxssf（注意很耗费堆空间）

XSSFWorkbook:是操作Excel2007-2010的版本，扩展名是.xlsx对于不同版本的EXCEL文档要使用不同的工具类。它的一张表最大支持**1048576**行，16384列。

用法 http://poi.apache.org/spreadsheet/how-to.html#sxssf

方案2 切分sheet

**注意** 
```java
SXSSFWorkbook wb = new SXSSFWorkbook(100); // keep 100 rows in memory, exceeding rows will be flushed to disk  

 SXSSFWorkbook wb = new SXSSFWorkbook(-1); // turn off auto-flushing and accumulate all rows in memory     
```
上面第一行限制100行后，每100行会被刷入磁盘，刷入磁盘的将无法使用getRow访问。第二行则会关闭自动刷入，注意可能引起oom



每个sheet都会生成一个临时文件，SXSSF会把数据刷到里面去，临时文件可能变得很大。例如，20MB的csv数据临时xml文件将会超过10亿字节。可以让SXSSF用gzip压缩

```java
  SXSSFWorkbook wb = new SXSSFWorkbook(); 
  wb.setCompressTempFiles(true); // temp files will be gzipped
```



## Quartz

### 如何基于Quartz做的集群任务

http://www.importnew.com/22896.html



### Quartz是如何基于数据库做锁的？

核心点在于for update

```java
public static final String SELECT_FOR_LOCK = "SELECT * FROM " 
            + TABLE_PREFIX_SUBST + TABLE_LOCKS + " WHERE " + COL_SCHEDULER_NAME + " = " + SCHED_NAME_SUBST  
            + " AND " + COL_LOCK_NAME + " = ? FOR UPDATE";  

public static final String INSERT_LOCK = "INSERT INTO " 
        + TABLE_PREFIX_SUBST + TABLE_LOCKS + "(" + COL_SCHEDULER_NAME + ", " + COL_LOCK_NAME + ") VALUES ("  
        + SCHED_NAME_SUBST + ", ?)";
```
当不存在记录则插入



## mysql

### 死锁

#### 怎样降低 innodb 死锁几率？

- 尽量使用较低的隔离级别，比如如果发生了间隙锁，你可以把会话或者事务的事务隔离级别更改为 RC(read committed)级别来避免，但此时需要把 binlog_format 设置成 row 或者 mixed 格式
- 精心设计索引，并尽量使用索引访问数据，使加锁更精确，从而减少锁冲突的机会；
- 选择合理的事务大小，小事务发生锁冲突的几率也更小；
- 给记录集显示加锁时，最好一次性请求足够级别的锁。比如要修改数据的话，最好直接申请排他锁，而不是先申请共享锁，修改时再请求排他锁，这样容易产生死锁；
- 不同的程序访问一组表时，应尽量约定以相同的顺序访问各表，对一个表而言，尽可能以固定的顺序存取表中的行。这样可以大大减少死锁的机会；
- 尽量用相等条件访问数据，这样可以避免间隙锁对并发插入的影响；
- 不要申请超过实际需要的锁级别；除非必须，查询时不要显示加锁；
- 对于一些特定的事务，可以使用表锁来提高处理速度或减少死锁的可能。



#### ON DUPLICATE KEY UPDATE或者replace into造成的死锁问题？

**问题描述** RR(REPEATABLE READ)隔离级别下并发执行出现deadlock

**原因** RR级别中，事务A在update后加Gap锁，事务B无法插入新数据，这样事务A在update前后读的数据保持一致，避免了幻读。

##### 什么是Gap锁？ 

RC级别：

| 事务A                                                        | 事务B                                                        |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| begin;                                                       | begin;                                                       |
| select id,class_name,teacher_id from class_teacher where teacher_id=30;idclass_nameteacher_id2初三二班30 |                                                              |
| update class_teacher set class_name='初三四班' where teacher_id=30; |                                                              |
|                                                              | insert into class_teacher values (null,'初三二班',30);commit; |
| select id,class_name,teacher_id from class_teacher where teacher_id=30;idclass_nameteacher_id2初三四班3010初三二班30 |                                                              |

RR级别：

| 事务A                                                        | 事务B                                                        |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| begin;                                                       | begin;                                                       |
| select id,class_name,teacher_id from class_teacher where teacher_id=30;idclass_nameteacher_id2初三二班30 |                                                              |
| update class_teacher set class_name='初三四班' where teacher_id=30; |                                                              |
|                                                              | insert into class_teacher values (null,'初三二班',30);waiting.... |
| select id,class_name,teacher_id from class_teacher where teacher_id=30;idclass_nameteacher_id2初三四班30 |                                                              |
| commit;                                                      | 事务Acommit后，事务B的insert执行。                           |

![mage-20180326151916](/var/folders/cf/lq_f9wkn3gx_l9nghhvyt7240000gn/T/abnerworks.Typora/image-201803261519169.png)

Innodb将这段数据分成几个个区间

- (negative infinity, 5],
- (5,30],
- (30,positive infinity)；

update class_teacher set class_name='初三四班' where teacher_id=30;不仅用行锁，锁住了相应的数据行；同时也在两边的区间，（5,30]和（30，positive infinity），都加入了gap锁。这样事务B就无法在这个两个区间insert进新数据。

update的teacher_id=20是在(5，30]区间，即使没有修改任何数据，Innodb也会在这个区间加gap锁，而其它区间不会影响，事务C正常插入。

如果使用的是没有索引的字段，比如update class_teacher set teacher_id=7 where class_name='初三八班（即使没有匹配到任何数据）',那么会给全表加入gap锁。同时，它不能像上文中行锁一样经过MySQL Server过滤自动解除不满足条件的锁，因为没有索引，则这些字段也就没有排序，也就没有区间。除非该事务提交，否则其它事务无法插入任何数据。

行锁防止别的事务修改或删除，GAP锁防止别的事务新增，行锁和GAP锁结合形成的的Next-Key锁共同解决了RR级别在写数据时的幻读问题。

来自https://tech.meituan.com/innodb-lock.html

**解决方案** 调整隔离级别为RC(read committed)



## 缓存

### 空值异常（缓存穿透）

**问题** 当业务数据为 null 时，无法确定是否已经缓存，会造成缓存无法命中

**方案** 

方案1 对于 null 的数据，需要做特殊处理，比如使用特殊字符串进行替换。缓存使用warpper类进行封装

方案2  缓存设置一个过期时间`cache.expire(key,60*15)`



### key 冲突问题

**问题** key可能冲突

**方案** 使用namespace



### 缓存雪崩

**问题** 由于缓存层承载着大量请求，有效的保护了存储层，但是如果缓存层由于某些原因整体不能提供服务，于是所有的请求都会达到存储层，存储层的调用量会暴增，造成存储层也会挂掉的情况。

**方案** 1.集群保证缓存高可用2.隔离组件限流降级