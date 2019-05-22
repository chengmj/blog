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
`ascii(substr((select table_name from information_schema.tables where tables_schema=database()limit 0,1),1,1))=101 --+ `       //substr()函数，ascii()函数
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
