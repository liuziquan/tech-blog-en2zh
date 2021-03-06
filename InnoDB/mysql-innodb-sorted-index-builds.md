---
title: InnoDB 排序索引的构建
tags: InnoDB
categories: InnoDB
---

> 原文作者：Satya Bodapati  
发布日期：2019年5月08日  
关键词： InnoDB, MySQL  
原文链接： https://www.percona.com/blog/2019/05/08/mysql-innodb-sorted-index-builds/    


我们不需要了解MySQL®和Percona Server如何构建索引。然而，如果你对构建过程有一定的了解，那么当你想为数据插入保留适当的空间时，它将会有所帮助。从MySQL5.7开始，开发者改变了他们为InnoDB构建二级索引的方式，应用自底而上的方法，而不是早期版本中使用的自顶而下方法。在这篇文章中，我将通过一个示例演示如何构建InnoDB索引。我将解释如何使用该认识来为参数 ***innodb_fill_factor*** 设置适当的值。

<!-- more -->

## 索引构建过程
要在已包含数据的表上构建索引，在InnoDB中有以下几个阶段：
1. 读取阶段(从聚集索引读取并构建辅助索引条目)
2. 合并排序阶段
3. 插入阶段(将已排序的记录插入辅助索引)

在5.6版本之前，MySQL每次插入一条记录来构建二级索引。这是一种“自上而下”的方法。对插入位置的搜索从根(顶部)开始，并到达相应的叶子页面(底部)。记录被插入到由游标指向的叶子页面上。在查找插入位置和执行页面分割和合并(在根节点和非根节点)方面，是有非常高的代价的。你知道有多少页面会拆分和合并吗，你可以阅读我同事**Marco Tusa**早期的博客[InnoDB页面拆分和合并](https://www.percona.com/blog/2017/04/10/innodb-page-merging-and-page-splitting/)。  

从MySQL 5.7开始，添加索引期间的插入阶段使用“排序索引构建”，也称为“批量加载索引”。在这种方法中，索引是“自底而上”构建的。也就是说，首先构建叶子页面(底部)，然后构建非叶子层，直到根(顶部)。

## 用例  
在这些情况下使用排序索引构建：
- ALTER TABLE t1 ADD INDEX (or CREATE INDEX)
- ALTER TABLE t1 ADD FULLTEXT INDEX
- ALTER TABLE t1 ADD COLUMN, ALGORITHM=INPLACE
- OPTIMIZE TABLE t1

对于最后两个用例，ALTER创建一个中间表。中间表索引(主建索引和辅助索引)是使用“排序索引构建”来构建的。

## 算法  
1. 在0级创建一个页面。还要为这个页面创建一个游标。
2. 使用第0级的游标插入页面，直到填满为止。
3. 一旦页面满了，创建一个同级页面(不会插入同级页面)。
4. 为当前完整页面创建一个节点指针(子页面中的最小键值、子页面号)，并将一个节点指针插入到上面的一层(父页面)。
5. 在上层，检查游标是否已经定位。如果没有，为该级别创建父页面和游标。
6. 在父页面上插入节点指针
7. 如果父页面也满了，将重复步骤3、4、5、6
8. 现在插入到同级页面，并使游标指向同级页面。
9. 在所有插入的最后，每一层都有指针指向最右边的页面。提交所有游标(意味着提交修改页面的小事务，释放所有latch)。

为了简单起见，上面的算法跳过了关于压缩页面和blob(外部存储的blob)处理的细节。

## 自底而上构建索引的过程

通过一个示例，让我们看看如何自底向上构建辅助索引。同样，为了简单起见，假设叶子和非叶子页面中允许的最大记录数为3。
```sql
CREATE TABLE t1 (a INT PRIMARY KEY, b INT, c BLOB);

INSERT INTO t1 VALUES (1, 11, 'hello111');
INSERT INTO t1 VALUES (2, 22, 'hello222');
INSERT INTO t1 VALUES (3, 33, 'hello333');
INSERT INTO t1 VALUES (4, 44, 'hello444');
INSERT INTO t1 VALUES (5, 55, 'hello555');
INSERT INTO t1 VALUES (6, 66, 'hello666');
INSERT INTO t1 VALUES (7, 77, 'hello777');
INSERT INTO t1 VALUES (8, 88, 'hello888');
INSERT INTO t1 VALUES (9, 99, 'hello999');
INSERT INTO t1 VALUES (10, 1010, 'hello101010');
ALTER TABLE t1 ADD INDEX k1(b);
```
InnoDB将主键字段附加到辅助索引。二级索引k1的记录为格式(b, a)，排序阶段后，记录为：  
(11,1), (22,2), (33,3), (44,4), (55,5), (66,6), (77,7), (88,8), (99,9), (1010, 10)

## 初始插入阶段
我们从记录(11,1)开始。  
1. 在0级(叶子级)创建一个页面。
2. 创建页面的游标。
3. 所有插入都转到该页，直到该页满为止。  
![avatar](https://www.percona.com/blog/wp-content/uploads/2019/04/1.png)  

箭头显示游标当前指向的位置。它当前位于第5页，接下来的插入将转到该页。

还有两个空闲槽，因此插入记录(22,2)和(33,3)是很简单的。

![avatar](https://www.percona.com/blog/wp-content/uploads/2019/04/2.png)  

对于下一个记录(44,4)，第5页已满。以下是步骤

## 当页面被填满时索引构建

1. 创建一个同级页面——第6页。
2. 暂时不要插入到同级页面。
3. 在游标处提交页面，即小事务提交、释放latch等。
4. 作为提交的一部分，创建一个节点指针并将其插入父页面[当前级别+ 1]，即在1级。
5. 节点指针的格式是(子页面中的最小键值，子页面号)。第5页的最小键值是(11,1)。在父级插入记录((11,1),5)。  
6. 第1级的父页面还不存在。MySQL创建第7页和指向第7页的游标。 
7. 将 ((11,1),5) 插入第7页。  
8. 现在，返回到第0级并创建从第5页到第6页的链接，反之亦然。
9. 第0级的游标现在指向同级别的第6页。
10. 插入(44,4)到第6页。

![avatar](https://www.percona.com/blog/wp-content/uploads/2019/04/bulk_load_44.png)  

接下来的插入—(55,5)和(66,6)—很简单，它们转到第6页。

![avatar](https://www.percona.com/blog/wp-content/uploads/2019/04/bulk_load_55_66.png)  

插入记录(77,7)类似于(44,4)，只是父页面(第7页)已经存在，并且它还有空间容纳另外两条记录。首先将节点指针(44,4),6)插入第7页，然后将节点指针(77,7)记录到第8页。

![avatar](https://www.percona.com/blog/wp-content/uploads/2019/04/bulk_load_77-1.png)  

插入记录(88,8)和(99,9)非常简单，因为第8页有两个空闲插槽。

![avatar](https://www.percona.com/blog/wp-content/uploads/2019/04/bulk_load_88_99-1.png)  

下一个插入(1010,10)。将节点指针((77,7),8)插入到第1级的父页面(第7页)。

MySQL在0级创建同级页面9。将记录(1010,10)插入到第9页，并将游标标更改为该页。

![avatar](https://www.percona.com/blog/wp-content/uploads/2019/04/Bulk_Load-Page-5.png)  

在所有级别提交游标。在上面的示例中，数据库在第0级提交第9页，在第1级提交第7页。我们现在有一个完整的B+-tree索引，它是自底向上构建的!

## 索引填充因素

全局变量 ***innodb_fill_factor*** 设置B树页面中用于插入的空间量。默认值为100，这意味着使用了整个页面(不包括页头、页尾)。聚集索引具有 ***innodb_fill_factor = 100*** 的豁免权。在这种情况下，聚集索引页空间的1/16保持空闲。即：6.25%的空间预留给未来的DML操作。

值设置为80表示MySQL使用80%的页面进行插入，剩下20%用于将来的更新。

如果 ***innodb_fill_factor*** 设置为100，没有多余的空间留给将来插入辅助索引。如果你希望在添加索引之后表上有更多dml，这可能会导致页面再次拆分和合并。在这种情况下，建议使用80-90之间的值。使用 ***optimization TABLE*** 或 ***ALTER TABLE DROP COLUMN ,ALGORITHM=INPLACE*** 语句，这个变量值还影响索引重建。

你不应使用太低的值，例如：低于50，因为索引会占用更多的磁盘空间。对于较低的值，索引中有更多的页面和索引统计数据抽样可能不是最优的。优化器可能会选择具有次优统计信息的错误查询计划。

## 排序索引构建的优点  

1. 没有页面拆分（表压缩除外）和合并。
2. 不重复搜索插入位置。
3. 插入没有重做日志记录(除了页面分配)，所以重做日志子系统的压力较小。

## 缺点
没有......好吧，有一个，它值得一篇单独的文章^_^，敬请期待。
