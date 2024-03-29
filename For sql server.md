# sql server
### 1.查询数据文件空间使用情况  
不同的统计方法，给出的结果会不一样，因为统计的单位不一样。有按区(Extent)为单位，也有按页(Page)为单位的方式。因为每个区有8个页，而8个页面不一定都被使用到。  
如果某个区，还有几个没被用到的页，如果按照区来统计，那这8个页面都算是使用了的空间。如果按照页面来统计，没有使用的页面就可以不算。anyway，取决于统计方法。    
#### 1.1 按照区统计：  
`DBCC ShowFileStats
GO`
这个命令直接从GAM和SGAM这样的系统分配页面上读取区分信息，直接计算出数据库文件里有多少区已被分配。优点是能够快速准确地计算出一个数据库数据文件去的总数和已使用过的区的数目。  

#### 1.2 按照页面统计： 
`EXEC sp_spaceused;`  --有时不准确，因为非实时    
`EXEC sp_spaceused @updateusage = N'TRUE';` --更具实时性的写法，基本准确。但是若数据库在繁忙工作时，最好不要执行该任务，因为会很耗资源。    
<img width="272" alt="屏幕截图 2022-05-21 175701" src="https://user-images.githubusercontent.com/32427537/169646440-d7ba2126-da3e-4d34-9cc7-96af9c8be5b7.png">  
`DBCC ShowContig`  --最精确的方法，但是要对数据库进行扫描，而扫描过程中，sql server要加锁。分3种扫描模式，分别为Limited,Sampled,Detailed. 依次得到的结果越精准，当然，伴随着越来越大的扫描范围。毕竟，天下没有免费的午餐。同理，数据库处于高峰期应避免使用该方法。  


### 2.查看日志文件使用情况
`DBCC SqlPerf(LogSpace);`    
会显示所有数据库的日志大小及使用情况。  
相对查数据文件的使用情况，超简单，而且不会对sql server带来负担，而且返回结果总是准确的。    

### 3.查看数据库状态信息
`select * from sys.databases;`  
`SELECT name FROM SYS.MASTER_FILES;`  
注意：对于 tempdb，sys.master_files显示初始 tempdb 大小。 这些值用作在启动 tempdb 时创建 tempdb 的SQL Server。 因此，tempdb 增长时，不会反映在视图中。 若要获取 tempdb 文件的当前大小，请查询 tempdb.sys.database_files。（官网说明）
`SELECT * FROM sys.database_files;`  
查询mdf ndf ldf file具体的摆放位置  

### 4.查询数据库文件的默认位置
用sql server management studio图形界面，查询数据库文件中的数据、日志、备份    
在object explorer,右击实例，选择“属性”，选择“数据库设置”，可以看到数据、日志、备份的默认设置位置  
(在硬盘充足的情况下，将数据文件和日志文件分别存储到不同的物理硬盘上，可增加安全性，提高数据库的性能）  

### 5.查询日志备份相关设置
`select * from msdb.dbo.backupset where database='testdb' order by backup_start_date desc;`    
type表示备份的类型，D:全量备份，L表示日志备份，i表示差异备份。  
（根据需求选择需要的列，如果只想看最后一次备份，可用SSMS右击DB的属性来看）    

### 6.查看sql server 版本
`select @@version;` 

### 7.实例与数据库
每台服务器可以有多个实例，每个实例也可以有多个数据库。  
SQL server中实例和数据库的主要区别在于，实例是sqlservr.exe可执行文件的副本，该文件作为操作系统服务运行，而数据库是将数据存储在表中的系统数据集合。  
[官网instance说明](https://docs.microsoft.com/en-us/sql/database-engine/configure-windows/database-engine-instances-sql-server?view=sql-server-2017)

### 8.CDC
变更数据捕获（Change Data Capture ，简称 CDC）记录 SQL Server 表的插入、更新和删除活动。使用变更数据捕获可以更有效跟踪表对象DML历史操作，对 ETL 等数据转移也非常有用  
变更数据捕获的更改数据源为 SQL Server 事务日志。当对表启用变更数据捕获时，系统将生成一个与该表结构类似的CDC表(like cdc.dbo_tablenameA_CT,在对应数据库下，展开system table即可看到开了CDC功能的表)。当对源表进行插入、更新和删除 时，在事务日志会记录相关操作信息。变更数据捕获代理使用异步进程读取事务日志，将相关操作结果应用到副本表（捕获实例表）中，这样就完成了对源表操作的记录跟踪。  
一些参考资料：    
https://sqlzealots.com/2019/03/02/how-do-we-set-up-change-data-capture-in-sql-server/    
https://sqlzealots.com/category/change-data-capture/  
https://docs.microsoft.com/en-us/previous-versions/sql/sql-server-2008/dd266396(v=sql.100)   
https://hevodata.com/learn/sql-server-cdc/#understanding_sql_server_cdc    
https://blog.csdn.net/kk185800961/article/details/45749333  
https://hevodata.com/learn/sql-server-cdc/#understanding_sql_server_cdc  
开启CDC导致事务日志空间被占满的原因：  
https://blog.csdn.net/woailyoo0000/article/details/106481601   
https://www.likecs.com/show-306009460.html  

开启CDC后，可能会把放事务日志的硬盘撑爆。记得设定定时任务，backup 事务日志，然后及时清走。  
CDC table字段介绍：  
![image](https://user-images.githubusercontent.com/32427537/197345869-b2749dec-f1ba-466c-a9b1-a3f8392fad09.png)

https://medium.com/%E3%84%9A%E5%8C%97%E7%9A%84%E6%89%80%E8%A6%8B%E6%89%80%E8%81%9E/docker-with-mssql-cdc-80148241d415

### 8.recovery mode
有三种recovery mode:简单，完整，大容量日志。  
默认是完整模式，大多数情况，在生产不建议用简单模式（simple）。当然根据生产的情况而定，我这边有试过可不用完整（full）模式的，直接用simple（智能的一匹），如果有生产事故，可不用到事务日志，用另一个软件搭配来恢复丢失的一段数据。  






