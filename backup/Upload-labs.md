#提要
由于upload-labs下载来源不同，可能在靶场搭建时会出现无法上传的报错，可以进入源文件中查看是否有upload文件夹（跟pass01等同级），没有的话创建一个upload文件夹即可。且不同关卡需要的php版本不同，有的关卡需要老版本不带nts的php，建议靶场搭建使用老版本的phpstudy~新人小白在学习过程中也会更新这篇提要

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
，dang当然也可以通过菜刀或者中国蚁剑进行连接操作，连接成功也说明木马上传成功！


