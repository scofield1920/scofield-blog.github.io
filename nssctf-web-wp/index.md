# NSSCTF-web-wp


几道web基础题目wp

<!--more-->

### [SWPUCTF 2021 新生赛]easyupload2.0

![image-20230227203926944](https://scofield-1313710994.cos.ap-beijing.myqcloud.com/image-20230227203926944.png)

几个常用的姿势试过了，PHP，Php行不通，最后改成phtml，成功上传

![image-20230227204121456](https://scofield-1313710994.cos.ap-beijing.myqcloud.com/image-20230227204121456.png)

最后蚁剑，成功连接

【复盘】查看upload.php

```
<?php
session_start();
echo "
<meta charset=\"utf-8\">";
if(!isset($_SESSION['user'])){
    $_SESSION['user'] = md5((string)time() . (string)rand(100, 1000));
}
if(isset($_FILES['uploaded'])) 
{
    $target_path  =  "./upload";
    $t_path = $target_path . "/" . basename($_FILES['uploaded']['name']);
    $uploaded_name = $_FILES['uploaded']['name'];
    $uploaded_ext  = substr($uploaded_name, strrpos($uploaded_name,'.') + 1);
    $uploaded_size = $_FILES['uploaded']['size'];
    $uploaded_tmp  = $_FILES['uploaded']['tmp_name'];
 
    if(preg_match("/php|hta|ini/i", $uploaded_ext))
    {
        die("php是不行滴");
    }
    else
    {
        $content = file_get_contents($uploaded_tmp);
        move_uploaded_file($uploaded_tmp, $t_path);
        echo "{$t_path} succesfully uploaded!";
        }
}
else
{
    die("不传🐎还想要f1ag?");
}
?>
```

​	发现用正则表达过滤了php.hta/ini
​	同时，php3，php5，pht，phtml，phps都是php可运行的文件扩展名

### [SWPUCTF 2021 新生赛]easyupload3.0

![image-20230227205104603](https://scofield-1313710994.cos.ap-beijing.myqcloud.com/image-20230227205104603.png)

看赛题标签，提示了.htaccess，果断png+.htaccess组合

![image-20230227205029686](https://scofield-1313710994.cos.ap-beijing.myqcloud.com/image-20230227205029686.png)

### [NISACTF 2022]babyupload

F12发现有目录，进行访问后下载得到upload源码，进行代码审计

```
@app.route('/upload', methods=['POST'])
def upload():
    if 'file' not in request.files:
        return redirect('/')
    file = request.files['file']
    if "." in file.filename:
        return "Bad filename!", 403
    conn = db()
    cur = conn.cursor()
    uid = uuid.uuid4().hex
    try:
        cur.execute("insert into files (id, path) values (?, ?)", (uid, file.filename,))
    except sqlite3.IntegrityError:
        return "Duplicate file"
    conn.commit()

    file.save('uploads/' + file.filename)
    return redirect('/file/' + uid)
```

可知，若上传的文件中包含“.”，则会返回Bad filename!，那我们上传一个没有后缀的文件

![image-20230227212011588](https://scofield-1313710994.cos.ap-beijing.myqcloud.com/image-20230227212011588.png)

```
@app.route('/upload', methods=['POST'])
def upload():
    if 'file' not in request.files:
        return redirect('/')
    file = request.files['file']
    if "." in file.filename:
        return "Bad filename!", 403
    conn = db()
    cur = conn.cursor()
    uid = uuid.uuid4().hex
    try:
        cur.execute("insert into files (id, path) values (?, ?)", (uid, file.filename,))
    except sqlite3.IntegrityError:
        return "Duplicate file"
    conn.commit()

    file.save('uploads/' + file.filename)
    return redirect('/file/' + uid)


@app.route('/file/<id>')
def file(id):
    conn = db()
    cur = conn.cursor()
    cur.execute("select path from files where id=?", (id,))
    res = cur.fetchone()
    if res is None:
        return "File not found", 404

    # print(res[0])

    with open(os.path.join("uploads/", res[0]), "r") as f:
        return f.read()

```

上传后生成一个uuid，并将uuid和文件名存入数据库中，并返回文件的uuid。再通过`/file/uuid`访问文件，通过查询数据库得到对应文件名，在文件名前拼接`uploads/`后读取该路径下上传的文件。

> **绝对路径拼接漏洞**
>
> ​	os.path.join(path,*paths)函数用于将多个文件路径连接成一个组合的路径。第一个函数通常包含了基础路径，而之后的每个参数被当作组件拼接到基础路径之后。
>
> ​	然而，这个函数有一个少有人知的特性，如果拼接的某个路径以 / 开头，那么包括基础路径在内的所有前缀路径都将被删除，该路径将视为绝对路径

当上传的文件名为 /flag ，上传后通过uuid访问文件后，查询到的文件名是 /flag ，那么进行路径拼接时，uploads/ 将被删除，读取到的就是根目录下的 flag 文件。

### [NISACTF 2022]bingdundun~

题目提示可以上传压缩包，关于phar://伪协议，ChatGPT给出了这样的

Phar:// 是 PHP 中的一个伪协议，用于访问 Phar （PHP Archive）文件，Phar 是一种类似于 Zip 的文件格式，用于将多个 PHP 文件打包成一个文件，并可以像一个普通的 PHP 文件一样使用。

> 主要是用于在php中对压缩文件格式的读取。这种方式通常是用来配合文件上传漏洞使用，或者进行进阶的phar反序列化攻击
>
> 用法就是把一句话木马压缩成zip格式，shell.php -> shell.zip，然后再上传到服务器（后续通过前端页面上传也没有问题，通常服务器不会限制上传 zip 文件），再访问：?filename=phar://…/shell.zip/shell.php
>
> 原文链接：https://blog.csdn.net/YangYubo091699/article/details/127351065

所以，上传a.zip(a.php一句话木马)，

![image-20230227223242867](https://scofield-1313710994.cos.ap-beijing.myqcloud.com/image-20230227223242867.png)

根据首页给的提示，构造payload

```
http://1.14.71.254:28403/?bingdundun=phar://6d3604f6ad66f035685c8e4caa342aea.zip/a
```

![image-20230227223354412](https://scofield-1313710994.cos.ap-beijing.myqcloud.com/image-20230227223354412.png)

随后在蚁剑连接，并在根目录发现了flag

### [SWPUCTF 2021 新生赛]jicao

php代码审计

![image-20230228162903097](https://scofield-1313710994.cos.ap-beijing.myqcloud.com/image-20230228162903097.png)

​	分析代码可知，需要用get方法传进一个json，以及通过post方法传入一个id
​	构造数据包如下：

![image-20230228163043675](https://scofield-1313710994.cos.ap-beijing.myqcloud.com/image-20230228163043675.png)

​	回显得到flag

### ![image-20230228182602104](https://scofield-1313710994.cos.ap-beijing.myqcloud.com/image-20230228182602104.png)[SWPUCTF 2021 新生赛]easy_md5

![](https://scofield-1313710994.cos.ap-beijing.myqcloud.com/image-20230228182602104.png)
	进行代码审计，要求name和password的值不同，同时要求两者的md相同，然而进行判断是用的是"=="符号，这是php的弱类型比较
	法一：可以使用带0e开头的数字串进行参数传递，因为PHP会将0e开头的数字转化为0，故此时md5值相等，二来能够给变量的值不相等。
	法二：可以传递数组，如name[]=123，password[]=456，md5不能加密数组，故两个md5返回的都是null

​	（另：若遇到`===`这样的强类型比较，方法一就失效了，方法二仍然有效，或者还可以使用软件fastcoll进行md5碰撞，生成两个字符串使得他们的md5值相同）

​	另：0e开头的数字

> s878926199a
> 0e545993274517709034328855841020
> s155964671a
> 0e342768416822451524974117254469
> s214587387a
> 0e848240448830537924465865611904
> s214587387a
> 0e848240448830537924465865611904
> s878926199a
> 0e545993274517709034328855841020
> s1091221200a
> 0e940624217856561557816327384675
> s1885207154a
> 0e509367213418206700842008763514
> s1502113478a
> 0e861580163291561247404381396064
> s1885207154a
> 0e509367213418206700842008763514
> s1836677006a
> 0e481036490867661113260034900752
> s155964671a
> 0e342768416822451524974117254469
> s1184209335a
> 0e072485820392773389523109082030
> s1665632922a
> 0e731198061491163073197128363787
> s1502113478a
> 0e861580163291561247404381396064
> s1836677006a
> 0e481036490867661113260034900752
> s1091221200a
> 0e940624217856561557816327384675
> s155964671a
> 0e342768416822451524974117254469
> s1502113478a
> 0e861580163291561247404381396064
> s155964671a
> 0e342768416822451524974117254469
> s1665632922a
> 0e731198061491163073197128363787
> s155964671a
> 0e342768416822451524974117254469
> s1091221200a
> 0e940624217856561557816327384675
> s1836677006a
> 0e481036490867661113260034900752
> s1885207154a
> 0e509367213418206700842008763514
> s532378020a
> 0e220463095855511507588041205815
> s878926199a
> 0e545993274517709034328855841020
> s1091221200a
> 0e940624217856561557816327384675
> s214587387a
> 0e848240448830537924465865611904
> s1502113478a
> 0e861580163291561247404381396064
> s1091221200a
> 0e940624217856561557816327384675
> s1665632922a
> 0e731198061491163073197128363787
> s1885207154a
> 0e509367213418206700842008763514
> s1836677006a
> 0e481036490867661113260034900752
> s1665632922a
> 0e731198061491163073197128363787
> s878926199a
> 0e545993274517709034328855841020
> 240610708
> 0e462097431906509019562988736854
> 314282422

使用bp发包，得到flag

![image-20230228184224073](https://scofield-1313710994.cos.ap-beijing.myqcloud.com/image-20230228184224073.png)

### [SWPUCTF 2021 新生赛]Do_you_know_http

![image-20230228184640217](https://scofield-1313710994.cos.ap-beijing.myqcloud.com/image-20230228184640217.png)

根据页面反馈信息，以及题目描述提示，进行抓包修改user-agent并添加xff，在返回信息中，得到location

![image-20230228184610682](https://scofield-1313710994.cos.ap-beijing.myqcloud.com/image-20230228184610682.png)

访问a.php

![image-20230228184803323](https://scofield-1313710994.cos.ap-beijing.myqcloud.com/image-20230228184803323.png)

再次发包，发现新站点
![image-20230228184939746](https://scofield-1313710994.cos.ap-beijing.myqcloud.com/image-20230228184939746.png)

访问后得到flag

### [SWPUCTF 2021 新生赛]babyrce

![image-20230228185051813](C:\Users\Scofield_Lee\AppData\Roaming\Typora\typora-user-images\image-20230228185051813.png)

根据题目提示推测，应该跟cookie有关
抓包修改cookie的值，发包后发现新站点：

![image-20230228185550733](https://scofield-1313710994.cos.ap-beijing.myqcloud.com/image-20230228185550733.png)

```
<?php
error_reporting(0);
highlight_file(__FILE__);
error_reporting(0);
if (isset($_GET['url'])) {
  $ip=$_GET['url'];
  if(preg_match("/ /", $ip)){
      die('nonono');
  }
  $a = shell_exec($ip);
  echo $a;
}
?> nonono
```

​	从代码层面来看，屏蔽了空格（ ），所以要采取手段来绕过对空格的屏蔽，如果传入的url值不包含空格，则通过函数 **$a = shell_exec($ip);**进行远程rce

> 在过滤了空格的系统中，以cat flag.txt为例，系统不允许我们输入空格或输入后被过滤。
>
> ${IFS}
>
> 可使用${IFS}代替空格。
>
> cat${IFS}flag.txt
>
> cat$IFS$1flag.txt
>
> cat${IFS}$1flag.txt

![image-20230228190745205](https://scofield-1313710994.cos.ap-beijing.myqcloud.com/image-20230228190745205.png)

先查看根目录，发现疑似文件
随后使用tac或cat读取文件，得到flag
![image-20230228190903485](https://scofield-1313710994.cos.ap-beijing.myqcloud.com/image-20230228190903485.png)

### [第五空间 2021]WebFTP

![image-20230228193747945](https://scofield-1313710994.cos.ap-beijing.myqcloud.com/image-20230228193747945.png)

​	F12发现用户名为admin，尝试爆破弱口令，可惜没爆破出来，扫描网站后台，发现有.git文件，使用githacker进行扒取，这里复习一下githacker的用法，功能上是比githack要强大的。
​	进入到Githacker文件夹执行`__init__.py`格式如下：

```
┌──(root㉿kali)-[/home/kali/githacker/GitHacker]
└─# python __init__.py --url http://1.14.71.254:28784/.git/ --output-folder result
```

​	扒取到的相关文件会输出到该目录下result文件夹中，查看readme.md文档，发现初始账号和密码

![image-20230228194410519](https://scofield-1313710994.cos.ap-beijing.myqcloud.com/image-20230228194410519.png)

![image-20230228194643435](https://scofield-1313710994.cos.ap-beijing.myqcloud.com/image-20230228194643435.png)

​	进去之后，好像没啥卵用，phpinfo.php文件在目录var/www/html中，但发现无法在ftp里面直接打开。
​	直接访问`http://1.14.71.254:28306/.phpinfo.php`就可以了
​	我觉得倒是做了许多无用功，复盘时返现直接dirsearch返回的结果中除了.git，也包含了README.md文件以及phpinfo.php都可以直接访问，在phpinfo页面搜索flag直接出。

![image-20230228195643826](https://scofield-1313710994.cos.ap-beijing.myqcloud.com/image-20230228195643826.png)

### [SWPUCTF 2021 新生赛]include

​	进入环境，提示传入一个文件，此外并没有发现什么有效信息，dirsearch扫描后台目录，也没啥有效信息，尝试php伪协议

![image-20230228201522848](https://scofield-1313710994.cos.ap-beijing.myqcloud.com/image-20230228201522848.png)

​	构造payload进行get传参：`http://1.14.71.254:28956/?file=flag`

![image-20230228201804240](https://scofield-1313710994.cos.ap-beijing.myqcloud.com/image-20230228201804240.png)

发现有**include_once函数**

include_once()：在脚本执行期间包含并运行指定文件。该函数和include 函数类似，两者唯一的区别是 使用该函数的时候，php会加检查指定文件是否已经被包含过，如果是，则不会再被包含。

于是利用伪协议构造，得到经过base64编码的flag

```
http://1.14.71.254:28956/?file=php://filter/convert.base64-encode/resource=flag.php
```

![image-20230228204810974](https://scofield-1313710994.cos.ap-beijing.myqcloud.com/image-20230228204810974.png)
至于为什么要使用base64进行编码输出

> **常见的php伪协议**
>
> 1）file:// 访问本地文件系统
>
> 2）http:// 访问HTTP(S)网址
>
> 3）ftp:// 访问FTP(S)URL
>
> 4)php:// 访问各个输出输入流
>
> 5)zlib:// 处理压缩流
>
> 6)data:// 读取数据
>
> 7)glob:// 查找匹配的文件路径模式
>
> 8)phar:// PHP归档
>
> 9)rar:// RAR数据压缩

至于为什么要使用base64编码，ChatGPT给出了这样的回答：

> 为了将目标文件内容以文本形式传输给攻击者，攻击者通常会将目标文件内容进行编码，例如使用Base64编码。这是因为HTTP协议只能传输文本数据，对于二进制数据，如图片、音频和视频等，需要进行编码才能在HTTP请求和响应中传输。
>
> 在这种情况下，攻击者可以将目标文件内容进行Base64编码，然后将编码后的结果作为HTTP请求或响应的一部分，从而实现传输。在接收到响应后，攻击者可以将编码后的结果解码，从而获得原始的二进制数据，即目标文件的内容。

(我们读取的是flag.php，而并非.txt文件)

引用一下[头秃的bug](https://blog.csdn.net/L2329794714)师傅在CSDN的笔记：

![image-20230228204934173](https://scofield-1313710994.cos.ap-beijing.myqcloud.com/image-20230228204934173.png)

> **过滤器的分类（及常见过滤器）：**
>
> **string filter(字符过滤器)**
> **string.rot13  (对字符串执行 ROT13 转换)**
>                     例：php://filter/string.rot13/resource=flag.php
> **string.toupper (转大写)**
>                     例：php://filter/string.toupper/resource=flag.php
> **string.tolower (转小写)**
> **string.strip_tags (去除 HTML 和 PHP 标记，尝试返回给定的字符串 str 去除空字符、HTML 和 PHP 标记后的结果)**
>                     例：php://filter/string.strip_tags/resource=flag.php
> **conversion filter (转换过滤器)**
>             convert.base64-encode & convert.base64-decode (base64加密 base64解密)
>                     例：php://filter/convert.base64-encode/resource=flag.php
>             convert.quoted-printable-encode & convert.quoted-printable-decode(在字符串后加=0A,及恢复)
> **compression filter(压缩过滤)**
>             zlib.deflate & zlib.inflate(压缩 解压)
>                     例：php://filter/zlib.deflate/resource=flag.php                                                       php://filter/zlib.inflate/resource=flag.php
>             bzip2.compress & bzip2.decompress(zip 压缩 解压)
> **encryption filter(加密过滤器)** 

摘自：https://blog.csdn.net/L2329794714/article/details/121288668

### [SWPUCTF 2021 新生赛]easyrce

灰常简单的rce，
<img src="https://scofield-1313710994.cos.ap-beijing.myqcloud.com/image-20230228205604826.png" alt="image-20230228205604826" style="zoom:67%;" />

在phpinfo中未发现flag，随后构造：

```
http://1.14.71.254:28678/?url=system("ls /);
```

![image-20230228205746590](https://scofield-1313710994.cos.ap-beijing.myqcloud.com/image-20230228205746590.png)

```
http://1.14.71.254:28678/?url=system(%22tac%20/flllllaaaaaaggggggg%22);
```

得到flag

### [SWPUCTF 2021 新生赛]ez_unserialize

<img src="https://scofield-1313710994.cos.ap-beijing.myqcloud.com/image-20230228210051445.png" alt="image-20230228210051445" style="zoom:67%;" />

没啥有效信息，扫描一下后台，发现

![image-20230228210325967](https://scofield-1313710994.cos.ap-beijing.myqcloud.com/image-20230228210325967.png)

进入c145s.php，得到源代码
<img src="https://scofield-1313710994.cos.ap-beijing.myqcloud.com/image-20230228210349597.png" alt="image-20230228210349597" style="zoom:80%;" />

经典php反序列化，ChatGPT讲的很明确了

> 1. 禁用错误报告。`error_reporting(0);` 可以禁用 PHP 的错误报告，以确保攻击者无法通过错误信息来了解代码的运行情况。
> 2. 显示 `cl45s.php` 的源代码。`show_source("cl45s.php");` 用于将 `cl45s.php` 的源代码显示在页面上，这可以帮助攻击者更好地了解代码的运行情况和漏洞点。
> 3. 定义 `wllm` 类。这个类具有两个公共属性 `$admin` 和 `$passwd`，分别用于存储用户名和密码。类中定义了两个方法，一个是构造函数 `__construct()`，用于初始化 `$admin` 和 `$passwd` 的值；另一个是析构函数 `__destruct()`，当对象被销毁时执行。
> 4. 处理传入的参数。`$p = $_GET['p'];` 用于获取 URL 参数中名为 `p` 的值。
> 5. 反序列化。`unserialize($p);` 将传入的参数 `$p` 进行反序列化处理，即将字符串转换为 PHP 对象。
> 6. 调用析构函数。由于在反序列化时会调用对象的构造函数，因此在对象被销毁时会自动调用析构函数 `__destruct()`。在析构函数中，如果 `$admin` 的值为 `"admin"`，`$passwd` 的值为 `"ctf"`，则会包含一个名为 `flag.php` 的文件，并将其中的 `$flag` 变量输出到页面上。否则，会将 `$admin` 和 `$passwd` 的值输出到页面上，并输出一条提示信息。

传入以下参数得到flag

```
http://1.14.71.254:28814/cl45s.php?p=O:4:%22wllm%22:2:{s:5:%22admin%22;s:5:%22admin%22;s:6:%22passwd%22;s:3:%22ctf%22;}
```

[复习php反序列化]

> ##### 什么是反序列化漏洞
>
> 当程序在进行反序列化时，会自动调用一些函数，例如__wakeup(),__destruct()等函数，但是如果传入函数的参数可以被用户控制的话，用户可以输入一些恶意代码到函数中，从而导致反序列化漏洞。
>
> ##### PHP魔术方法
>
> 魔术方法是PHP面向对象中特有的特性。它们在特定的情况下被触发，都是以双下划线开头，利用魔术方法可以轻松实现PHP面向对象中重载（Overloading即动态创建类属性和方法）。 问题就出现在重载过程中，执行了相关代码。

以下是一些常见的php魔术方法：

```
1、__get、__set

这两个方法是为在类和他们的父类中没有声明的属性而设计的

__get( $property ) 当调用一个未定义的属性时访问此方法

__set( $property, $value ) 给一个未定义的属性赋值时调用

这里的没有声明包括访问控制为proteced,private的属性（即没有权限访问的属性）

2、__isset、__unset

__isset( $property ) 当在一个未定义的属性上调用isset()函数时调用此方法

__unset( $property ) 当在一个未定义的属性上调用unset()函数时调用此方法

与__get方法和__set方法相同，这里的没有声明包括访问控制为proteced,private的属性（即没有权限访问的属性）

3、__call

__call( $method, $arg_array ) 当调用一个未定义(包括没有权限访问)的方法是调用此方法

4、__autoload

__autoload 函数，使用尚未被定义的类时自动调用。通过此函数，脚本引擎在 PHP 出错失败前有了最后一个机会加载所需的类。

注意: 在 __autoload 函数中抛出的异常不能被 catch 语句块捕获并导致致命错误。

5、__construct、__destruct

__construct 构造方法，当一个对象被创建时调用此方法，好处是可以使构造方法有一个独一无二的名称，无论它所在的类的名称是什么，这样你在改变类的名称时，就不需要改变构造方法的名称

__destruct 析构方法，PHP将在对象被销毁前（即从内存中清除前）调用这个方法

默认情况下,PHP仅仅释放对象属性所占用的内存并销毁对象相关的资源.，析构函数允许你在使用一个对象之后执行任意代码来清除内存，当PHP决定你的脚本不再与对象相关时，析构函数将被调用.

在一个函数的命名空间内，这会发生在函数return的时候，对于全局变量，这发生于脚本结束的时候，如果你想明确地销毁一个对象，你可以给指向该对象的变量分配任何其它值，通常将变量赋值勤为NULL或者调用unset。

6、__clone

PHP5中的对象赋值是使用的引用赋值，使用clone方法复制一个对象时，对象会自动调用__clone魔术方法，如果在对象复制需要执行某些初始化操作，可以在__clone方法实现。

7、__toString

__toString方法在将一个对象转化成字符串时自动调用，比如使用echo打印对象时，如果类没有实现此方法，则无法通过echo打印对象，否则会显示：Catchable fatal error: Object of class test could not be converted to string in，此方法必须返回一个字符串。

在PHP 5.2.0之前，__toString方法只有结合使用echo() 或 print()时 才能生效。PHP 5.2.0之后，则可以在任何字符串环境生效（例如通过printf()，使用%s修饰符），但 不能用于非字符串环境（如使用%d修饰符）

从PHP 5.2.0，如果将一个未定义__toString方法的对象 转换为字符串，会报出一个E_RECOVERABLE_ERROR错误。

8、__sleep、__wakeup

__sleep 串行化的时候用

__wakeup 反串行化的时候调用

serialize() 检查类中是否有魔术名称 __sleep 的函数。如果这样，该函数将在任何序列化之前运行。它可以清除对象并应该返回一个包含有该对象中应被序列化的所有变量名的数组。

使用 __sleep 的目的是关闭对象可能具有的任何数据库连接，提交等待中的数据或进行类似的清除任务。此外，如果有非常大的对象而并不需要完全储存下来时此函数也很有用。

相反地，unserialize() 检查具有魔术名称 __wakeup 的函数的存在。如果存在，此函数可以重建对象可能具有的任何资源。使用 __wakeup 的目的是重建在序列化中可能丢失的任何数据库连接以及处理其它重新初始化的任务。

9、__set_state

当调用var_export()时，这个静态 方法会被调用（自PHP 5.1.0起有效）。本方法的唯一参数是一个数组，其中包含按array(’property’ => value, …)格式排列的类属性。

10、__invoke

当尝试以调用函数的方式调用一个对象时，__invoke 方法会被自动调用。PHP5.3.0以上版本有效

11、__callStatic

它的工作方式类似于 __call() 魔术方法，__callStatic() 是为了处理静态方法调用，PHP5.3.0以上版本有效，PHP 确实加强了对 __callStatic() 方法的定义；它必须是公共的，并且必须被声明为静态的。

同样，__call() 魔术方法必须被定义为公共的，所有其他魔术方法都必须如此。
```
