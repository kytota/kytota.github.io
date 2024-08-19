# 提要
由于upload-labs下载来源不同，可能在靶场搭建时会出现无法上传的报错，可以进入源文件中查看是否有upload文件夹（跟pass01等同级），没有的话创建一个upload文件夹即可。且不同关卡需要的php版本不同，有的关卡需要老版本不带nts的php，建议靶场搭建使用老版本的phpstudy~新人小白在学习过程中也会更新这篇提要

# 目录
-[pass01](#第一关)
-[pass02](#第二关)
-[pass03](#第三关)


# 第一关
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

# 第二关
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
'```
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

# 第三关
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


