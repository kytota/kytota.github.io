# 提要
由于upload-labs下载来源不同，可能在靶场搭建时会出现无法上传的报错，可以进入源文件中查看是否有upload文件夹（跟pass01等同级），没有的话创建一个upload文件夹即可。且不同关卡需要的php版本不同，有的关卡需要老版本不带nts的php，建议靶场搭建使用老版本的phpstudy~新人小白在学习过程中也会更新这篇提要

# 目录
-[pass01(前端绕过)](#pass01)
-[pass02(MIME绕过)](#pass02)
-[pass03(httpd-conf配置文件和php3)](#pass03)
-[pass04(.htaccess文件)](#pass04)
-[pass05pre](#pass05pre)
-[pass05(.user.ini与php.ini)](#pass05)
-[pass06(大小写绕过)](#pass06)
-[pass07(空格绕过)](#pass07)
-[pass08(点绕过)](#pass08)
-[pass09pre(额外数据流ADS)](#pass09pre)
-[pass09(::$DATA数据流绕过)](#pass09)
-[pass10(双写绕过)](#pass10)
-[pass11pre(空字符)](#pass11pre)
-[pass11(%00截断)](#pass11)
-[pass12(0x00截断)](#pass12)


# pass01
## 源码展示
```html
function checkFile() {
    var file = document.getElementsByName('upload_file')[0].value;
    if (file == null || file == "") {
        alert("请选择要上传的文件!");
        return false;
    }
    //定义允许上传的文件类型
    var allow_ext = ".jpg|.png|.gif";
    //提取上传文件的类型
    var ext_name = file.substring(file.lastIndexOf("."));
    //判断上传文件类型是否允许上传
    if (allow_ext.indexOf(ext_name + "|") == -1) {
        var errMsg = "该文件不允许上传，请上传" + allow_ext + "类型的文件,当前文件类型为：" + ext_name;
        alert(errMsg);
        return false;
    }
}

```
## 分析
### 代码分析
从给出的源码可以看出只涉及前端
### 抓包分析
可以从bp抓包中查出端倪：
先将bp的拦截打开，随后故意上传一个不符合源码要求的文件类型——即不属于jpg,png,gif的文件，点击上传后页面直接弹出警告框，且bp未收到请求包
![QQ20240819-001856](https://github.com/user-attachments/assets/3054b267-70d5-4f9d-ab55-97e902efdca0)
，故可以判断为前端拦截。
## 绕过方法
由于前端只接受gif,png,jpg三种类型的文件，我们可以将自己的一句话木马后缀名改为jpg，
![QQ20240819-002301](https://github.com/user-attachments/assets/f8bd77a1-a2b2-4626-939f-3c6488b01d3c)
打开bp的拦截，随后进行上传，
![QQ20240819-002723](https://github.com/user-attachments/assets/7963deab-ec07-4927-a9ed-c9cb62bbb5d5)
在bp中将filename的用于绕过前端检测的jpg后缀名修改为木马原本的文件类型，如php，随后放行该请求包，回到upload-labs页面，可以看到上传成功
![QQ20240819-002759](https://github.com/user-attachments/assets/a8a0f690-cbb4-490e-96a5-fe45379e1285)
,为了验证，可以访问http://你的ip/upload-labs/你的木马文件名（在bp中修改后的），看到空白的页面一般就是成功了，如：
![QQ20240819-003057](https://github.com/user-attachments/assets/de799a7d-3c7e-49c1-8ce1-424c73e4f272)
，当然也可以通过菜刀或者中国蚁剑进行连接操作，连接成功也说明木马上传成功！




# pass02
## 源码展示
```php
$is_upload = false;
$msg = null;
if (isset($_POST['submit'])) {
    if (file_exists($UPLOAD_ADDR)) {
        if (($_FILES['upload_file']['type'] == 'image/jpeg') || ($_FILES['upload_file']['type'] == 'image/png') || ($_FILES['upload_file']['type'] == 'image/gif')) {
            if (move_uploaded_file($_FILES['upload_file']['tmp_name'], $UPLOAD_ADDR . '/' . $_FILES['upload_file']['name'])) {
                $img_path = $UPLOAD_ADDR . $_FILES['upload_file']['name'];
                $is_upload = true;

            }
        } else {
            $msg = '文件类型不正确，请重新上传！';
        }
    } else {
        $msg = $UPLOAD_ADDR.'文件夹不存在,请手工创建！';
    }
}
```

## 分析
从代码格式就可以看出这是后端的检测了，查看代码可以知道，只放行类型为
```php 
$_FILES['upload_file']['type'] == 'image/jpeg') || ($_FILES['upload_file']['type'] == 'image/png') || ($_FILES['upload_file']['type'] == 'image/gif
```
的文件，那这一句话是什么意思呢？询问gpt可以得知：
### 1 $_FILES['upload_file']['type']
$_FILES['upload_file']['type']是一个PHP超全局数组，用于处理通过HTTP POST方法上传的文件，假如我上传一个名为pass.php的文件夹
```php
$_FILES['upload_file'] = array(
    'name' => 'pass.php',          // 上传文件的原始文件名
    'type' => 'text/plain',        // 文件的 MIME 类型（取决于服务器和浏览器的判断）
    'tmp_name' => '/tmp/phpYzdqkD',// 文件上传后的临时存储路径
    'error' => 0,                  // 上传过程中出现的错误代码，0 表示没有错误
    'size' => 1234                 // 文件的大小（以字节为单位）
);
```
所以这串代码是先选择包含上传文件信息的upload_files，再选择其中的type
### 2 MIME类型
MIME用于描述文件的内容类型，当浏览器收到一个HTML文件时，MIME类型会告诉他这个文件的格式以及他应如何解析和现实内容
MIME类型由两部分组成，用斜杠/分割
#### 主类型（type）：
表示数据的主要类别。例如，text（文本文件）、image（图片文件）、application（应用程序数据）、audio（音频文件）、video（视频文件）等。
#### 子类型（subtype）：
表示主类型中的具体格式。例如，plain 表示纯文本，jpeg 表示 JPEG 图片，json 表示 JSON 数据。
### 3 对应BP
在BP拦截到的请求包中，$_FILES['upload_file']通常在Content-Type或者Content-Disposition部分。
## 绕过策略
经过以上分析，我们知道可以通过BP拦截后修改请求包中的Content-Type或者Content-Disposition来满足源代码的要求即可。
### 方法一
这次没有前端检测，直接上传我们的一句话代码，如ip.php！
打开BP拦截（养成习惯），回到网页选择ip.php木马进行上传，BP拦截后对Content-Type进行修改，改为png,jpg,gif的MIME类型放行
上传成功！
### 方法二
其实可以直接按照Pass01的方法，先将木马后缀修改为png再进行上传，这是浏览器看到的MIME便符合要求了，但是同样的，一个png文件上传后，如果没有.htacess这类文件的帮助，其实是不能承担木马的功能的，所以还需要在BP中修改filename的后缀名为php。这里不做赘述，方法和Pass01一模一样。
### 检测
为了验证上传成功和木马文件内容的正确，继续访问 你的ip/upload-labs/upload/你上传的木马文件，查看到页面一面空白后说明一切正常
![QQ20240819-012404](https://github.com/user-attachments/assets/56f9dd1c-f387-47bf-9536-45f106a8f475)
也可以通过中国蚁剑进行连接测试，连接成功说明任务圆满完成！




# pass03
## 注意
这一关我个人认为是用于衔接下一关（第四关）的，请注意二者相关性，下面会谈到
## 源码分析
```php
$is_upload = false;
$msg = null;
if (isset($_POST['submit'])) {
    if (file_exists($UPLOAD_ADDR)) {
        $deny_ext = array('.asp','.aspx','.php','.jsp');
        $file_name = trim($_FILES['upload_file']['name']);
        $file_name = deldot($file_name);//删除文件名末尾的点
        $file_ext = strrchr($file_name, '.');
        $file_ext = strtolower($file_ext); //转换为小写
        $file_ext = str_ireplace('::$DATA', '', $file_ext);//去除字符串::$DATA
        $file_ext = trim($file_ext); //收尾去空

        if(!in_array($file_ext, $deny_ext)) {
            if (move_uploaded_file($_FILES['upload_file']['tmp_name'], $UPLOAD_ADDR. '/' . $_FILES['upload_file']['name'])) {
                 $img_path = $UPLOAD_ADDR .'/'. $_FILES['upload_file']['name'];
                 $is_upload = true;
            }
        } else {
            $msg = '不允许上传.asp,.aspx,.php,.jsp后缀文件！';
        }
    } else {
        $msg = $UPLOAD_ADDR . '文件夹不存在,请手工创建！';
    }
}

```
分析可知，网站采用黑名单策略，并且防止了后缀名加点和空格的绕过手段（windows会将后缀名后的点和空格去掉，故也用于后缀名绕过），并且还防止了$DATA和大小写绕过，看似无懈可击。此时我们用前面打的方法进行绕过，会发现无法上传。无论如何都无法将php文件送入敌后了。
![QQ20240819-233301](https://github.com/user-attachments/assets/49eefa31-c192-4f63-8e1c-8ab424d8ebe3)

## 绕过策略
此时，黑名单相较于白名单的劣势就体现出来了。网站黑名单只拒绝了四种后缀名的文件，但是可执行文件的后缀名海了去了！如我们想要将php文件送入却无计可施，完全可以用php3,php5,phtml文件绕过黑名单策略。
### 注意
重点来了，这里也是我为什么说这一关更像是 下一关的衔接 ，其本身思路不具备太多可行性，下面会体现出来
为了让网站将php3,php5,phtml文件以解析php的方式解析，我们就要做如下调整：
#### 1
在phpstudy中，将你的php的版本换成一个不带nts的ts版本
![屏幕截图 2024-08-19 233803](https://github.com/user-attachments/assets/2c9e912d-267b-4bb8-b521-15be338dfd55)
nts是非多线程安全，ts是多线程安全，无需深究（因为我也不懂）。
#### 2
在phpstudy中，找到apache的配置文件httpd.conf
![屏幕截图 2024-08-19 235204](https://github.com/user-attachments/assets/16ae242e-09cd-4d3e-a485-68f137baa36e)
（phpstudy中寻找方式如图）
进入文件，搜索 AddType application/x-httpd-php .php ，会找到一行被注释的代码
![QQ20240820-000812](https://github.com/user-attachments/assets/a4520998-9342-47d9-8e9e-d692c98963a7)
这段代码的意思是，将.pho后缀名的文件作为php解析。所以我们将#号删掉，在代码后加上我们想用于绕过的文件的后缀名：php3,php5,phtml。其实这时候你加上jpg和png后缀然后用图片上传木马也可以了，但我们还是按照靶场的思路来。修改此行代码为：
```php
AddType application/x-httpd-php .php .phtml .php3 .php5
```
#### 不可行原因
看了以上两点我们应该会有感想：我修改配置文件是为了上传木马，可为了能上传木马我需要强大到可以修改配置文件。。。这就是这一关的不可行性了，但是这一关的思路：利用配置文件中的addtype和网站的黑名单漏洞，是具有可行性的，下一关我们将用.htaccess文件使这个想法具备可行性。
### 开始绕过
准备好我们的木马文件，老规矩，打开BP的拦截，上传文件，抓包，修改filename的后缀名为php3，放行，会发现文件就上传成功了。验证上传是否成功的方式和前两关一模一样，不再做赘述。

> [!NOTE]
> 最后再次提醒，请读懂下面这行代码：
```php
AddType application/x-httpd-php .php .phtml
```


# pass04
## 源代码展示
```php
$is_upload = false;
$msg = null;
if (isset($_POST['submit'])) {
    if (file_exists($UPLOAD_ADDR)) {
        $deny_ext = array(".php",".php5",".php4",".php3",".php2","php1",".html",".htm",".phtml",".pHp",".pHp5",".pHp4",".pHp3",".pHp2","pHp1",".Html",".Htm",".pHtml",".jsp",".jspa",".jspx",".jsw",".jsv",".jspf",".jtml",".jSp",".jSpx",".jSpa",".jSw",".jSv",".jSpf",".jHtml",".asp",".aspx",".asa",".asax",".ascx",".ashx",".asmx",".cer",".aSp",".aSpx",".aSa",".aSax",".aScx",".aShx",".aSmx",".cEr",".sWf",".swf");
        $file_name = trim($_FILES['upload_file']['name']);
        $file_name = deldot($file_name);//删除文件名末尾的点
        $file_ext = strrchr($file_name, '.');
        $file_ext = strtolower($file_ext); //转换为小写
        $file_ext = str_ireplace('::$DATA', '', $file_ext);//去除字符串::$DATA
        $file_ext = trim($file_ext); //收尾去空

        if (!in_array($file_ext, $deny_ext)) {
            if (move_uploaded_file($_FILES['upload_file']['tmp_name'], $UPLOAD_ADDR . '/' . $_FILES['upload_file']['name'])) {
                $img_path = $UPLOAD_ADDR . $_FILES['upload_file']['name'];
                $is_upload = true;
            }
        } else {
            $msg = '此文件不允许上传!';
        }
    } else {
        $msg = $UPLOAD_ADDR . '文件夹不存在,请手工创建！';
    }
}
```

# 代码分析
相较于第三关，这次网站在前者的基础上把php3等文件后缀名也纳入了黑名单。

# 解决方案
第三关为了使php3,php5,phtml文件被当做php文件解析，我们进入了apache的httpd-conf进行配置文件的修改，修改了代码
```php
AddType application/x-httpd-php
```
并且在其后方加上了.php3 .php5来使这类文件作为php文件被浏览器进行解析。事实上，httpd-conf的功能可以通过.htaccess文件来实现。
## httpd-conf和.htaceess文件的区别
![QQ20240822-181439](https://github.com/user-attachments/assets/f24c2e86-b43d-40c1-b825-d95271243d9c)
(apache官网截的一些讲解)
省流来说，apache中：一、httpd-conf是全局配置文件，而.htaceess是作用于其所在目录及其子目录中的配置文件；二、httpd-conf需要管理员级别的权限才能进行修改，且修改后需要重启服务器才能生效，而.htaccess文件可以通过我们的文本编辑器进行修改，且修改后即刻生效。
## 如何上传.htaccess文件
首先，我们要知道，我们上传.htaccess文件是为了修改网站的配置，让我们上传除了黑名单以外的文件类型时，仍然能够作为php文件进行解析。根据第三关我们知道，这涉及到
```php
AddType application/x-httpd-php .php # 增加这一类型的应用解释
```
这一行代码，我们需要对其进行修改。但是这一配置已经在httpd-conf中，这时候我们再上传一个.htaccess对其进行修改，就涉及到对httpd-conf的override（覆盖）了，而httpd-conf中有一个变量AllowOverride，其值为None时是不允许对httpd-conf进行覆盖重写的，所以进入httpd-conf搜索AllowOverride查看其值，若是None就改为all即可。
## 绕过展示
### 1.写一个.htaccess文件
![QQ20240822-193235](https://github.com/user-attachments/assets/0de71edf-07dc-4db1-8e4e-0e2a5cc5cd79)
### 2.上传.htaccess文件
### 3.准备好后缀名为.htaccess文件中对应后缀名的文件
### 4.上传
可以发现上传成功（如果失败可能是php版本号的问题，参照pass03的版本号选择）



# pass05-pre
## 概念引入
在pass03和pass04，我们了解了.htaccess分布式配置文件和httpd-conf全局配置文件。但有时候网站也会禁止.htaccess文件的上传，使我们无法修改和覆写httpd-conf中的配置。这时候user.ini文件就派上用场了。
## .user.ini文件
打开phpstudy的配置文件，我们可以看到php.ini，这是php的主配置文件。而就像.htaccess之于httpd-conf，.user.ini和php.ini文件之前的关系就是分布式与全局的关系。

## .user.ini文件生效前提
### 一、php版本
(在.user.ini文件的上传与使用中，我们应该尽量用新版本的php进行网站配置)
在你的phpstudy的WWW文件夹根目录下写一个php文件
```php
<?php
phpinfo();
?>
```
![QQ20240824-180145](https://github.com/user-attachments/assets/798b0135-209f-4271-b733-a04404805596)

经过前面几关的学习，我们可以知道这个文件是为了显示php的版本及各种信息。之后在浏览器访问localhost/该文件名。
#### 老版本php(php 5.2.17)
![QQ20240824-180523](https://github.com/user-attachments/assets/312be315-e56b-46c0-a98b-55949a89b601)
#### 新版本php(php7.0.12)
![QQ20240824-180619](https://github.com/user-attachments/assets/56a692c6-cb39-49b8-b510-5541fddf8c0a)

为了使用.user.ini，我们需要server.api值为CGI/FastCGI，从上图可以看成新版本的才符合要求。(server.api可以理解为组件之间的协议，此处无需深究，我也不懂)

### 二、所上传的文件目录中需要有可解析的php文件


# pass05
## 源代码展示
```php
$is_upload = false;
$msg = null;
if (isset($_POST['submit'])) {
    if (file_exists($UPLOAD_ADDR)) {
        $deny_ext = array(".php",".php5",".php4",".php3",".php2",".html",".htm",".phtml",".pHp",".pHp5",".pHp4",".pHp3",".pHp2",".Html",".Htm",".pHtml",".jsp",".jspa",".jspx",".jsw",".jsv",".jspf",".jtml",".jSp",".jSpx",".jSpa",".jSw",".jSv",".jSpf",".jHtml",".asp",".aspx",".asa",".asax",".ascx",".ashx",".asmx",".cer",".aSp",".aSpx",".aSa",".aSax",".aScx",".aShx",".aSmx",".cEr",".sWf",".swf",".htaccess");
        $file_name = trim($_FILES['upload_file']['name']);
        $file_name = deldot($file_name);//删除文件名末尾的点
        $file_ext = strrchr($file_name, '.');
        $file_ext = strtolower($file_ext); //转换为小写
        $file_ext = str_ireplace('::$DATA', '', $file_ext);//去除字符串::$DATA
        $file_ext = trim($file_ext); //首尾去空
        
        if (!in_array($file_ext, $deny_ext)) {
            if (move_uploaded_file($_FILES['upload_file']['tmp_name'], $UPLOAD_ADDR . '/' . $_FILES['upload_file']['name'])) {
                $img_path = $UPLOAD_ADDR . '/' . $file_name;
                $is_upload = true;
            }
        } else {
            $msg = '此文件不允许上传';
        }
    } else {
        $msg = $UPLOAD_ADDR . '文件夹不存在,请手工创建！';
    }
}
```

## 分析绕过策略
注意到，本次网站禁止了.htaccess文件的上传。基于pass04pre的讲解，我们可以使用.user.ini进行绕过
### .user.ini写法
```php
Auto_prepend_file=xxx.xxx #使所有当前目录及其子目录中的php文件优先包含xxx.xxx这个文件（即将xxx.xxx文件加入每个在目录范围内的php文件，跟着php文件一同被以php的方式解析）
```

## 绕过策略
首先，检查我们的靶场文件的upload文件夹中是否已经有readme.php文件，这本应该是靶场内置的，但是由于我们获取靶场的方式各有不同，所以可能有人没有这个php文件（比如我就压根没有upload这个上传文件夹，upload文件夹都是我自己创建的，更别指望里面会有自带的readme.php了），没有这个文件的话我们创建一个空readme.php文件即可（当然不一定非得名字叫readme，举一反三嘛），因为正如pass05pre中所说，.user,ini文件要生效的话，其所在目录必须包含php文件。
![QQ20240824-224350](https://github.com/user-attachments/assets/dcddda10-d53f-4a92-b46a-863dfbcf4f5d)
确保uoload路径下有可用的php文件后，我们开始编写.user.ini
### .user.ini从创建到使用
#### 写法
```php
Auto_prepend_file=pass.jpg #这里我所用的木马名是pass.jpg
```
写好后上传，由于黑名单中未包含.user,ini，所以很顺利的，上传成功。
（上传成功）

#### 运用
由于我的.user.ini中用于被包含的木马文件名为pass.jpg，所以我们将我们的木马文件复制一份并将文件名改为pass.jpg，随后上传。

这时候，我们用浏览器解析readme.php文件时，其会将木马文件pass.jpg一并解析。

## 检验
用中国蚁剑进行连接，发现成功。
![QQ20240824-225216](https://github.com/user-attachments/assets/89b34bd2-4327-4321-9eb2-923d267ab60e)



# pass06
## 源代码展示
```php
$is_upload = false;
$msg = null;
if (isset($_POST['submit'])) {
    if (file_exists($UPLOAD_ADDR)) {
        $deny_ext = array(".php",".php5",".php4",".php3",".php2",".html",".htm",".phtml",".pHp",".pHp5",".pHp4",".pHp3",".pHp2",".Html",".Htm",".pHtml",".jsp",".jspa",".jspx",".jsw",".jsv",".jspf",".jtml",".jSp",".jSpx",".jSpa",".jSw",".jSv",".jSpf",".jHtml",".asp",".aspx",".asa",".asax",".ascx",".ashx",".asmx",".cer",".aSp",".aSpx",".aSa",".aSax",".aScx",".aShx",".aSmx",".cEr",".sWf",".swf",".htaccess");
        $file_name = trim($_FILES['upload_file']['name']);
        $file_name = deldot($file_name);//删除文件名末尾的点
        $file_ext = strrchr($file_name, '.');
        $file_ext = str_ireplace('::$DATA', '', $file_ext);//去除字符串::$DATA
        $file_ext = trim($file_ext); //首尾去空

        if (!in_array($file_ext, $deny_ext)) {
            if (move_uploaded_file($_FILES['upload_file']['tmp_name'], $UPLOAD_ADDR . '/' . $_FILES['upload_file']['name'])) {
                $img_path = $UPLOAD_ADDR . '/' . $file_name;
                $is_upload = true;
            }
        } else {
            $msg = '此文件不允许上传';
        }
    } else {
        $msg = $UPLOAD_ADDR . '文件夹不存在,请手工创建！';
    }
}
```
## 代码分析
我们可以发现一个致命且显眼的漏洞：没有预防大小写，因此本关难度极低

## 绕过策略
对此我们可以用后缀名大小写混写绕过
，太简单，直接贴图~
![QQ20240824-232005](https://github.com/user-attachments/assets/88e155ec-23e1-4e9c-8937-8d0c67efe8dd)
我们先开启BP进行抓包，再上传一个名为ip.php的文件，拦截到请求包后对后缀名进行修改，改为ip.pHP,发现上传成功

## 检验
通过中国蚁剑进行连接
![QQ20240824-232830](https://github.com/user-attachments/assets/22d915d8-3346-4396-8bf2-85d246f72088)
成功！



# pass07
## 源代码展示
```php
$is_upload = false;
$msg = null;
if (isset($_POST['submit'])) {
    if (file_exists($UPLOAD_ADDR)) {
        $deny_ext = array(".php",".php5",".php4",".php3",".php2",".html",".htm",".phtml",".pHp",".pHp5",".pHp4",".pHp3",".pHp2",".Html",".Htm",".pHtml",".jsp",".jspa",".jspx",".jsw",".jsv",".jspf",".jtml",".jSp",".jSpx",".jSpa",".jSw",".jSv",".jSpf",".jHtml",".asp",".aspx",".asa",".asax",".ascx",".ashx",".asmx",".cer",".aSp",".aSpx",".aSa",".aSax",".aScx",".aShx",".aSmx",".cEr",".sWf",".swf",".htaccess");
        $file_name = trim($_FILES['upload_file']['name']);
        $file_name = deldot($file_name);//删除文件名末尾的点
        $file_ext = strrchr($file_name, '.');
        $file_ext = strtolower($file_ext); //转换为小写
        $file_ext = str_ireplace('::$DATA', '', $file_ext);//去除字符串::$DATA
        
        if (!in_array($file_ext, $deny_ext)) {
            if (move_uploaded_file($_FILES['upload_file']['tmp_name'], $UPLOAD_ADDR . '/' . $_FILES['upload_file']['name'])) {
                $img_path = $UPLOAD_ADDR . '/' . $file_name;
                $is_upload = true;
            }
        } else {
            $msg = '此文件不允许上传';
        }
    } else {
        $msg = $UPLOAD_ADDR . '文件夹不存在,请手工创建！';
    }
}
```

## 绕过策略分析
通过网站源码我们可以看到网站的巨大漏洞：没有防止空格绕过！所以同样本关非常简单，在要上传的木马文件名后缀加上空格即可。

## 绕过策略
### 1.打开BP拦截
![QQ20240825-095458](https://github.com/user-attachments/assets/10d7a901-b059-446b-bc0b-140ddfbfe88d)
### 2.上传我们的木马文件
### 3.进入BP中，修改拦截到的请求包，在filename的后缀名后加上空格。
![QQ20240825-095604](https://github.com/user-attachments/assets/52c40b4a-21b7-4058-9509-d867751068fd)
可以看到上传成功。

## 检验
打开我们的网站upload-labs文件的upload文件夹，可以看到上传成功。
![QQ20240825-095928](https://github.com/user-attachments/assets/1258744d-5be1-43a1-ad34-b0d2f1f39c2e)

右键查看文件属性，可以看到我们虽然在后缀名最后加了空格，但是windows操作系统依然将其视为php文件。
![QQ20240825-095941](https://github.com/user-attachments/assets/014f8324-94cc-4efc-b563-d4623bffa7d1)

最后通过中国蚁剑进行连接。
![QQ20240825-100353](https://github.com/user-attachments/assets/775c8bd3-4a0d-4fa3-b280-b440f865fb57)
成功！




# pass08
## 源代码展示
```php
$is_upload = false;
$msg = null;
if (isset($_POST['submit'])) {
    if (file_exists($UPLOAD_ADDR)) {
        $deny_ext = array(".php",".php5",".php4",".php3",".php2",".html",".htm",".phtml",".pHp",".pHp5",".pHp4",".pHp3",".pHp2",".Html",".Htm",".pHtml",".jsp",".jspa",".jspx",".jsw",".jsv",".jspf",".jtml",".jSp",".jSpx",".jSpa",".jSw",".jSv",".jSpf",".jHtml",".asp",".aspx",".asa",".asax",".ascx",".ashx",".asmx",".cer",".aSp",".aSpx",".aSa",".aSax",".aScx",".aShx",".aSmx",".cEr",".sWf",".swf",".htaccess");
        $file_name = trim($_FILES['upload_file']['name']);
        $file_ext = strrchr($file_name, '.');
        $file_ext = strtolower($file_ext); //转换为小写
        $file_ext = str_ireplace('::$DATA', '', $file_ext);//去除字符串::$DATA
        $file_ext = trim($file_ext); //首尾去空
        
        if (!in_array($file_ext, $deny_ext)) {
            if (move_uploaded_file($_FILES['upload_file']['tmp_name'], $UPLOAD_ADDR . '/' . $_FILES['upload_file']['name'])) {
                $img_path = $UPLOAD_ADDR . '/' . $file_name;
                $is_upload = true;
            }
        } else {
            $msg = '此文件不允许上传';
        }
    } else {
        $msg = $UPLOAD_ADDR . '文件夹不存在,请手工创建！';
    }
}
```

## 绕过策略分析
同样的，这个网站源码有一个致命漏洞，没有对后缀名中的“.”进行删除，而windows操作系统会自动无视后缀名中的“.”，所以我们可以通过这个漏网之鱼进行文件上传。

## 绕过策略
老规矩，打开BP拦截；
进行我们文件的上传。
进入BP中修改请求包中filename的后缀名。
![QQ20240825-103010](https://github.com/user-attachments/assets/13793013-0bf7-4d72-b104-de0c22342d34)
上传成功了！

检验步骤同Pass07一样，此处不做赘述。

## 另辟蹊径
其实对于该网站的绕过策略，不论他是否有去除点，去除空格和防止大小写等，我们都可以通过一个固定的方法进行绕过。
观察网站给出的代码，这里展示的是完整地一套去除空格、点，防止大小写的pass05的代码：
```php
$is_upload = false;
$msg = null;
if (isset($_POST['submit'])) {
    if (file_exists($UPLOAD_ADDR)) {
        $deny_ext = array(".php",".php5",".php4",".php3",".php2",".html",".htm",".phtml",".pHp",".pHp5",".pHp4",".pHp3",".pHp2",".Html",".Htm",".pHtml",".jsp",".jspa",".jspx",".jsw",".jsv",".jspf",".jtml",".jSp",".jSpx",".jSpa",".jSw",".jSv",".jSpf",".jHtml",".asp",".aspx",".asa",".asax",".ascx",".ashx",".asmx",".cer",".aSp",".aSpx",".aSa",".aSax",".aScx",".aShx",".aSmx",".cEr",".sWf",".swf",".htaccess");
        $file_name = trim($_FILES['upload_file']['name']);
        $file_name = deldot($file_name);//删除文件名末尾的点
        $file_ext = strrchr($file_name, '.');
        $file_ext = strtolower($file_ext); //转换为小写
        $file_ext = trim($file_ext); //首尾去空
        
        if (!in_array($file_ext, $deny_ext)) {
            if (move_uploaded_file($_FILES['upload_file']['tmp_name'], $UPLOAD_ADDR . '/' . $_FILES['upload_file']['name'])) {
                $img_path = $UPLOAD_ADDR . '/' . $file_name;
                $is_upload = true;
            }
        } else {
            $msg = '此文件不允许上传';
        }
    } else {
        $msg = $UPLOAD_ADDR . '文件夹不存在,请手工创建！';
    }
}
```
观察发现，这里对后缀名的处理经过一轮就结束了，没有for循环。
对此我们完全可以在后缀名后加上“. .”（点+空格+点）进行绕过。
比如我们上传一个"ip.php. ."的木马文件:
首先，去除点——>ip.php. ;(此时后缀名为php+点+空格)
然后，去除空格——>ip.php;(此时后缀名为php+点)
这时候后缀名能去除点和空格的操作就结束了。
这时候上传一个后缀名为php,的文件，而经过上文我们知道windows会无视后缀名中的.，所以php.文件实际上就是php文件，我们的木马上传就成功了。



# pass09pre
## 新概念引入
普通情况下，一个文件只有一个数据流（Data Stream），我们是通过文件名来访问它的，它就是我们双击文件查看到的内容；
但在NTFS文件系统中（一般本地磁盘的 文件系统 就是NTFS，可以在资源管理器中右键磁盘查看），一个文件支持“寄生”额外的数据流（Alternate Data Stream），一般在这个数据流中存放文件的元数据、备份信息、标签等其他数据。正常情况下，即使一个文件被寄生了ADS，普通人也无法访问ADS；需要特定的命令行接口或者编程接口，如：CMD，以及C语言的编译器等；

## ADS示范
### ADS的写入
我们先创建一个txt文件作为实验对象，内容是“web安全”
![QQ20240825-220056](https://github.com/user-attachments/assets/b53a309c-0245-4c08-a0d5-7f831508c16d)

ADS写入有两种方法：
1、将特定内容写入某文件
```cmd
echo "要输入的内容" >> 目标文件:ADS名字
```
我们在存放实验对象1.txt的文件目录下打开cmd，输入上方命令
![QQ20240825-221905](https://github.com/user-attachments/assets/3ff1245b-61c1-4f34-bbce-1dd0d88c9c31)
写入成功，将一行内容“我在学习”输入给了1.txt中我们自定义的ADS，该ADS名字叫word
2、将特定文件写入ADS
```cmd
type 用于加入ADS的文件名 >> 目标文件名:ADS名
```
依旧以1.txt作为实验对象，我们再创建一个2.txt文件用以写入ADS吧：
![QQ20240825-222218](https://github.com/user-attachments/assets/fc7420e0-8cb7-427a-ad25-1396c6939f8b)
2.txt文件内容是“我在学习”，我们将其写入1.txt的另一个自定义ADS
![QQ20240825-222339](https://github.com/user-attachments/assets/0c7ee38f-986b-4b1d-822c-9057baaac618)

### ADS的访问
在我们进行了写入ADS后。直接双击打开或者在cmd中访问目标文件，都不会显示ADS，我们需要在cmd中访问目标文件的目标ADS；
访问的话用一个命令就行了：
用于打开文件的程序（确认添加到了环境变量中，不然默认notepad吧）文件名；例如
```cmd
notepad 1.txt:word
notepad 1.txt:document
```
![QQ20240825-222928](https://github.com/user-attachments/assets/b0c17b8c-b6ff-46f3-bce1-e611416de1a2)



# pass09
## 源代码展示
```php
$is_upload = false;
$msg = null;
if (isset($_POST['submit'])) {
    if (file_exists($UPLOAD_ADDR)) {
        $deny_ext = array(".php",".php5",".php4",".php3",".php2",".html",".htm",".phtml",".pHp",".pHp5",".pHp4",".pHp3",".pHp2",".Html",".Htm",".pHtml",".jsp",".jspa",".jspx",".jsw",".jsv",".jspf",".jtml",".jSp",".jSpx",".jSpa",".jSw",".jSv",".jSpf",".jHtml",".asp",".aspx",".asa",".asax",".ascx",".ashx",".asmx",".cer",".aSp",".aSpx",".aSa",".aSax",".aScx",".aShx",".aSmx",".cEr",".sWf",".swf",".htaccess");
        $file_name = trim($_FILES['upload_file']['name']);
        $file_name = deldot($file_name);//删除文件名末尾的点
        $file_ext = strrchr($file_name, '.');
        $file_ext = strtolower($file_ext); //转换为小写
        $file_ext = trim($file_ext); //首尾去空
        
        if (!in_array($file_ext, $deny_ext)) {
            if (move_uploaded_file($_FILES['upload_file']['tmp_name'], $UPLOAD_ADDR . '/' . $_FILES['upload_file']['name'])) {
                $img_path = $UPLOAD_ADDR . '/' . $file_name;
                $is_upload = true;
            }
        } else {
            $msg = '此文件不允许上传';
        }
    } else {
        $msg = $UPLOAD_ADDR . '文件夹不存在,请手工创建！';
    }
}
```
## 绕过策略分析
可以看出此次网站并没有对文件后缀名进行去除‘::$DATA’字符串的操作；
而在pass09pre中可以知道在加上::$DATA后，网站会认为这是一个数据流而不去检查文件的后缀名；因此我们可以顺利将文件送入后门，随后由于windows不允许后缀名有特殊字符":",所以在送入后门后后缀名会自动去掉‘::$DATA’而还原回原文件；

## 绕过策略
老规矩，先打开BP拦截；
上传木马；
进入BP抓包后修改filename的文件后缀名，加上::$DATA
![QQ20240826-211038](https://github.com/user-attachments/assets/f4ac040e-765b-4719-acae-d5fd9ee5d96a)
可以看到shan上传成功了，我们再进入网站文件上传目录看一看：
![QQ20240826-211208](https://github.com/user-attachments/assets/839cf8be-c360-4899-bd32-1be38283f556)
可以看到，::$DATA被自动过滤掉了，也就是说在windows帮助下，木马被成功还原了~
注意：这是我们想要访问木马url时，文件名不要加上::$DATA哦，因为windows把这个字符串去掉了。
![QQ20240826-211512](https://github.com/user-attachments/assets/dec0c1cb-b8ea-49b7-b6f0-02ce38d83034)




# pass10
## 源代码展示
```php
$is_upload = false;
$msg = null;
if (isset($_POST['submit'])) {
    if (file_exists($UPLOAD_ADDR)) {
        $deny_ext = array("php","php5","php4","php3","php2","html","htm","phtml","jsp","jspa","jspx","jsw","jsv","jspf","jtml","asp","aspx","asa","asax","ascx","ashx","asmx","cer","swf","htaccess");

        $file_name = trim($_FILES['upload_file']['name']);
        $file_name = str_ireplace($deny_ext,"", $file_name);
        if (move_uploaded_file($_FILES['upload_file']['tmp_name'], $UPLOAD_ADDR . '/' . $file_name)) {
            $img_path = $UPLOAD_ADDR . '/' .$file_name;
            $is_upload = true;
        }
    } else {
        $msg = $UPLOAD_ADDR . '文件夹不存在,请手工创建！';
    }
}
```

## 分析
关键在于这一句：
```php
$file_name = str_ireplace($deny_ext,"", $file_name);
```
这句代码的意思是将我们上传的文件名与黑名单中的字符串进行比较，匹配上的就会被替换为空（被删掉）；
由于代码只执行一次，我们可以双写绕过；

## 绕过策略
假如我们想上传一个名为 pass.php的文件，由于网站会用黑名单对他进行匹配，直接上传的话，文件就会变成：pass.
那我们尝试双写呢？
pass.pphphp---->pass.php（将裹在里面的php删掉后，pphphp变成了php）；
非常简单，不做赘述~




# pass11pre
## 空字符
php的许多底层语言都是c，而c有一个可利用的特性：空字符

空字符的作用不止是一个不显示的字符，而且它是一把利刃，可以切掉从左到右自身后面的所有字符
比如：你好【空字符】世界 ---->(c语言)---->你好
“世界”就不见了；致敬传奇心之壁，AT-field！

##空字符的使用
空字符有两种
### %00
用于url编码，我们通常在文件存放路径使用get请求时使用它进行绕过；
### 0x00
用于编程语言，我们通常在文件存放路径使用post请求时使用它进行绕过。




# pass11
## 源代码展示
```php
$is_upload = false;
$msg = null;
if(isset($_POST['submit'])){
    $ext_arr = array('jpg','png','gif');
    $file_ext = substr($_FILES['upload_file']['name'],strrpos($_FILES['upload_file']['name'],".")+1);
    if(in_array($file_ext,$ext_arr)){
        $temp_file = $_FILES['upload_file']['tmp_name'];
        $img_path = $_GET['save_path']."/".rand(10, 99).date("YmdHis").".".$file_ext;

        if(move_uploaded_file($temp_file,$img_path)){
            $is_upload = true;
        }
        else{
            $msg = '上传失败！';
        }
    }
    else{
        $msg = "只允许上传.jpg|.png|.gif类型文件！";
    }
}
```

## 绕过策略分析
我们注意到这句：
```php
$file_ext = substr($_FILES['upload_file']['name'],strrpos($_FILES['upload_file']['name'],".")+1);
```
substr(x,y)我们已经很了解了，就是从x的第y+1个位置开始(包括第y+1个字符)获得字符，比如：
    substr('01234',2);就是从"01234“的第3个位置开始获得字符串，生成：‘234’；
而strrpos(x,y)则是获取字符“y”在字符串"x"的位置，如：
    strrpos('01234',2)就是从‘01234’中找到2出现的位置（从0开始），得到：2；
所以我们上传一个pass.php，就会得到一个用于与白名单匹配的字符串“php”，而由于白名单的特性，之前的绕过就用不了了

但是ちょっと待ってね、我们再看这一句：
```php
$img_path = $_GET['save_path']."/".rand(10, 99).date("YmdHis").".".$file_ext;
```
这是为我们上传的文件分配一个目的路径，而我们可以利用$_GET['save_path']配合空字符对目的路径进行魔改进行绕过；

### 结论
我们可以在save_path进行魔改，在其后面加上pass.php%00    
pass.php是我想让木马叫的名字，而%00是对rand(10, 99).date("YmdHis").".".$file_ext进行截断；
这样文件被上传的最终路径就会变成.../pass.php，这样我们上传一个符合白名单的文件，其最终就会变成名字叫pass.php的文件。

## 绕过策略
打开BP拦截；
将我们的木马后缀名改为jpg，用于白名单放行；
上传；
来到BP进行修改，get请求的内容在第一行请求行内：
![QQ20240901-214658](https://github.com/user-attachments/assets/da3aaa1e-39d3-4fd7-b17b-419d5f7d95ed)
对其进行修改吧：
![QQ20240901-214751](https://github.com/user-attachments/assets/2947cee9-d834-45fd-9873-c46abece305a)
放行！
![QQ20240901-214831](https://github.com/user-attachments/assets/d8dc4d05-632e-4313-b6cb-795e511b1060)
上传成功~
用蚁剑进行连接，也是一样的成功啊~




# pass12
## 源代码分析
```php
$is_upload = false;
$msg = null;
if(isset($_POST['submit'])){
    $ext_arr = array('jpg','png','gif');
    $file_ext = substr($_FILES['upload_file']['name'],strrpos($_FILES['upload_file']['name'],".")+1);
    if(in_array($file_ext,$ext_arr)){
        $temp_file = $_FILES['upload_file']['tmp_name'];
        $img_path = $_POST['save_path']."/".rand(10, 99).date("YmdHis").".".$file_ext;

        if(move_uploaded_file($temp_file,$img_path)){
            $is_upload = true;
        }
        else{
            $msg = "上传失败";
        }
    }
    else{
        $msg = "只允许上传.jpg|.png|.gif类型文件！";
    }
}
```

## 绕过分析
可以看出，此次网站的防御策略的上一关（%00截断）的差别只有上传路径的获取；上一关是使用GET，而此次使用的是POST，虽然只是一个在请求头一个在请求体，但是操作起来还是有一些别的小差别。

## 绕过策略
老规矩，开BP抓包拦截。
上传一个后缀名为白名单内的木马文件。
进入BP查看请求包。
![QQ20240910-170742](https://github.com/user-attachments/assets/9d342bd8-ad51-4a05-8c9e-45b808613b23)
可以看到上传路径的位置。
按照我们的老思路，将save_path进行修改，但注意，在BP请求体是无法直接加空字符的（至少我不会），所以我们先改想要的文件名一个很夸张很具有辨识度的名字：11111111111.php,并在其后面加上一个+；
这时候我们看看这个文件名的16进制如何表示：其中HEX一栏表示16进制，hex中的0x也是16进制的意思（我啰嗦了）。
在BP修改后我们看看这个请求包的hex，并在其中找到我们的上传路径在哪：
![QQ20240910-171331](https://github.com/user-attachments/assets/7201687c-916f-4211-969b-3c4430cd401e)
还是很好找的，一堆31开头，2b结尾。
然后我们将2b改为00；可以看到.php后面的+不见了，鼠标左键选中一下会发现右边隐隐约约还有什么东西，那就是空字符了：
![QQ20240910-171455](https://github.com/user-attachments/assets/23a44b5b-8269-47d8-a6e1-4a8e6828efb7)

到这就可以提前开香槟了，我们直接放行请求包吧。




# pass13（图片马制作）前置
JPEG/JFIF(常见的照片格式):头两个字节为·0xFF·0xD8。
PNG(无损压缩格式):头两个字节为·0x89·0x50。
GIF(支持动画的图像格式):头两个字节为·0x47.0x49。
BMP(Windows位图格式):头两个字节为·0x42·0x4D。
TIFF(标签图像文件格式):头两个字节可以是不同的数值。

另外 图片马的制作，我推荐使用vscode扩展中的hex编辑器，，winHEX应该也可以用，我没有试过。









