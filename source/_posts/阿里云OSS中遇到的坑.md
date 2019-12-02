---
title: 阿里云OSS中遇到的坑
tags: ['javascript','c#']
date: 2016-06-22 16:18:00
categories: [前端]
---

最近项目开发用了阿里云OSS存储与下载文件，遇到不少的坑。为了防止人老健忘，特意记录下开发过程。

先上介绍：[阿里云OSS](https://www.aliyun.com/product/oss/)
OSS的申请就不细说了，总之要有关键的Access Key ID、Access Key Secret
此外访问OSS也需要buket，OSS访问域名记为host

本文章基于前端上传回调方案，参考链接：[服务端签名直传并设置上传回调](https://help.aliyun.com/document_detail/31927.html?spm=5176.doc32016.6.201.hFMYGU)

1.  上传
上传首先得有文件

`<input type='file' id='fff' />;`

写个简单事件(假装有jquery~)
``` javascript
$('#fff').change(
    var file=this.files[0];
);
```

好了有文件了，但是OSS不是任何人都能上传的，需要获得授权。我这里是通过后台获取，然而官方并没有c#的文档...
于是我把php的demo用c#写了遍
``` c#
string id = '你的id';
string key = '你的key';
string host = '你的host';
string callback = '你的callback地址';
int expire = 600;   //600秒
string dir = '自定义目录/';   
long GB = 1024 * 1024 * 1024;
long maxlength = 2 * GB; //最大2G

ObjGetOssSignCallback rslt = new ObjGetOssSignCallback();
//回调主体，这里可以传参给回调地址
var callback_param = new
{
        callbackUrl = callback,
        callbackBody = 'filename=${object}&size=${size}&mimeType=${mimeType}&height=${imageInfo.height}&width=${imageInfo.width}',
            callbackBodyType = 'application/x-www-form-urlencoded'
};
string callback_string = JsonConvert.SerializeObject(callback_param);
string base64_callback_body = callback_string.ToBase64();

//过期时间 
DateTime now = DateTime.Now;
DateTime end = now.AddSeconds(expire);
string expiration = end.GMT_iso8601();

//文件设置
var conditions = new List<dynamic>
{
        new List<dynamic>{'content-length-range',0,maxlength},  //文件大小范围
        new List<dynamic>{'starts-with','$key',dir}             //路径
};

//计算policy和signature
var arr = new
{
        expiration = expiration,
        conditions = conditions
};
string policy = JsonConvert.SerializeObject(arr);
string base64_policy = policy.ToBase64();
string string_to_sign = base64_policy;
string signature = BaseTools.hash_hmac(string_to_sign, key, true);

var result = new
{
        accessid = id,
        host = host,
        policy = base64_policy,
        signature = signature,
        expire = end.TimeStamp(),
        callback = base64_callback_body,
        dir = dir
};
rslt.Sign = result;
rslt.CallBackUrl = callback;
return rslt;
```

以下是上面使用过的一些工具方法
``` c#
// Base64
public static string ToBase64(this string str)
{
        byte[] b = Encoding.UTF8.GetBytes(str);
        return Convert.ToBase64String(b);
}

// 转成iso8601格式的日期
public static string GMT_iso8601(this DateTime d)
{
        return string.Format('{0}Z', d.ToString('s'));
}

// HASH，sha1算法
public static string hash_hmac(string signatureString, string secretKey, bool raw_output = true)
{
    var enc = Encoding.UTF8;
    HMACSHA1 hmac = new HMACSHA1(enc.GetBytes(secretKey));
    hmac.Initialize();

    byte[] buffer = enc.GetBytes(signatureString);
    if (raw_output)
    {
        byte[] b = hmac.ComputeHash(buffer);
        return Convert.ToBase64String(b);
    }
    else
    {
        return BitConverter.ToString(hmac.ComputeHash(buffer)).Replace('-', '').ToLower();
    }
}
```

我用的mvc，返回的格式如下：
``` json
{
    "Sign": {
    "accessid": "xxxxxxxxxxxxxxx",
    "host": "http://file.xxxxxxx.com",
    "policy": "eyJleHBpcmF0aW9uIjoiMjAxNi0wNi1yMlQxNjo0MDozOaoiLCJjb25kaXRpb25zIjpbWyJjb250ZW50LWxlamd0aC1yYW5nZSIsMCwyMTQ3NDgzNjQ4XSxbInN0YXJ0cy13arRoIiwiJGtleSIsIm16UHJval9kaXIvIl1dfQ==",
    "signature": "u868e090GtFyHlCtCtw6b5xb5gU=",
    "expire": 1466584839,
    "callback": "eyJjYWxsYmFja1VybCI6Imh0dHA6Ly9tYWxsLmlzdGFybGlmZS5jb20vQ29tbW9uL1VwbG9hZENhbGxiYWNrIiwiY2FsbGJhY2tCb2R5IjoiZmlsZW5hbWU9JHtvYmplY3R9JnNpemU9JHtzaXplfSZtaW1lVHlwZT0ke21pbWVUeXBlfSZoZWlnaHQ9JHtpbWFnZUluZm8uaGVpZ2h0fSZ3aWR0aD0ke2ltYWdlSW5mby53aWR0aH0mdWlkPSIsImNhbGxiYWNrQm9keVR5cGUiOiJhcHBsaWNhdGlvbi94LXd3dy1mb3JtLXVybGVuY29kZWQifQ==",
    "dir": "自定义目录/"
    },
    "CallBackUrl": "http://xxxx.xxxxxxx.com/controller/uploadcallback"
}
```

要上传文件到OSS，需要这几个值：accessid，policy，callback，signature
项目要求使用ajax上传文件，于是就用到了FormData对象(为什么又不使用jquery传formdata了呢因为我懒...)
``` javascript
var xhr = new XMLHttpRequest();
var frm = new FormData();
frm.append('key', url);    //此为文件保存名，例 'xinjianwenjianjia/asdasd.jpg'，不是文件名
//frm.append('Content-Disposition', 'attachment');    //如需下载就要设置此表单域，而且要注意顺序不能放在下面！血淋淋的坑..
frm.append('policy', policy);
frm.append('OSSAccessKeyId', accessid);
frm.append('success_action_status', 200);    //不要在意这种细节~
frm.append('callback', callback);
frm.append('signature', signature);
frm.append('file', file);    //文件对象

xhr.open('POST', oss.host);
xhr.send(frm);
```

好了完成了~个鬼啊，我们要监控上传状态或者要写进度条事件，哪能这么快完结(xhr在创建之后就能添加事件了)
``` js
//进度条事件
xhr.upload.addEventListener('progress', function (e) {
    //console.log(e);
    if (e.lengthComputable) {
        var progress = Math.round(e.loaded * 100 / e.total);
        console.log(progress);
    }
}, false);
//完成事件
xhr.addEventListener('load', function (e) {
    console.log(e.target.responseText);
}, false);
//error事件
xhr.addEventListener('error', function (e) {
    console.log(e);
    alert('上传出错！');
}, false);
//暂停上传后的事件
xhr.addEventListener('abort', function (e) {
    console.log(e);
}, false);
```

上传过程到此结束！
参考文档：[Post Object](https://help.aliyun.com/document_detail/31988.html?spm=5176.doc31979.6.263.T1TJji)

1.  下载
下载分两种，公开的和私有的。如果在OSS管理控制台中设置了buket为公开的，那么用host和上面上传的key值拼成url就能直接访问 `Demo: http://file.asdasd.com/xinjianwenjianjia/asdasd.jpg`

然而我被坑这么久明显是因为需要访问私有的文件...
先放文档：[下载参考](https://help.aliyun.com/document_detail/31857.html?spm=5176.doc31855.6.167.MJcFyR)
很容易看到要通过url访问私有资源需要加一些参数，文件的保存路径(也是上面的key)记为objectName，OSSAccessKeyId，Expires，Signature
objectName是访问文件的唯一标识。OSSAccessKeyId是开发者的Access Key ID，不用担心泄密，Access Key Secret才是最秘密的东西
Expires是过期时间，时间戳。Signature是加密后的签名
由此可见，重点是如何生成Signature签名，其余的字段都是已有或者很容易获得的
``` c#
Signature = base64(hmac-sha1(AccessKeySecret,
            VERB + '\n” 
            + Content-MD5 + '\n' 
            + Content-Type + '\n' 
            + Date + '\n' 
            + CanonicalizedOSSHeaders
            + CanonicalizedResource))
```

这是官方的Signature生成方案，其中Content-MD5和Content-Type和CanonicalizedOSSHeaders和CanonicalizedResource不用填写，有的时候填了还会报错...(理解不能，求原因)
因为这是用AccessKeySecret加密，涉及到安全问题最好放在后台生成，以下是我从php里转写成c#的代码(既视感)：
``` c#
//必须传入objectName，生成Signature必须
//objectName不能以'/'开头
if (string.IsNullOrEmpty(objectName))
    return null;
var result = new ObjGetOssDownloadSign();
string id = 'xxxxxxxx';
string key = 'xxxxxxxxxxxxxxxx';
string host = '你的host';
string bucket = '/yourbuket/';
int expire = 600;

DateTime now = DateTime.Now;
DateTime end = now.AddSeconds(expire);

string txt = 'GET\n' +      //method
            '\n' +          //Content-MD5
            '\n' +          //Content-Type text/html 没什么卵用
            end.TimeStamp() + '\n' +    //注意啊这是时间戳！千万别用官方文档上的GMT时间啊！
            bucket + objectName;        //bucket+ObjectName
//加密
string hash = BaseTools.hash_hmac(txt, key, true);

//encode一下
hash = System.Web.HttpUtility.UrlEncode(hash, System.Text.Encoding.UTF8);

result.Sign = new
{
    OSSAccessKeyId = id,
    Expires = end.TimeStamp(),
    Signature = hash
};
result.DownloadUrl = host + obj.ObjectName + '?OSSAccessKeyId=' + id + '&Expires=' + end.TimeStamp() + '&Signature=' + hash;
return result;
```

也扔在mvc里，返回的数据格式如下(注意传参数，objectName)：
``` json
{
    "Sign": {
    "OSSAccessKeyId": "xxxxxxxxx",
    "Expires": 1466587469,
    "Signature": "Vz3AxkNVaWDh9HsDhrglbq6JiUI%3d"
    },
    "DownloadUrl": "http://file.xxxx.com/asd/sadsda.png?OSSAccessKeyId=MHkBVLh0E5gGQkka&Expires=1466587469&Signature=Vz3AxkNVaWDh9HsDhrglbq6JiUI%3d"
}
```

哦对了，扔个转时间戳的方法
``` c#
//转时间戳
public static long TimeStamp(this DateTime d)
{
    DateTime DateStart = new DateTime(1970, 1, 1, 8, 0, 0); //天朝时区UTC+8
    return Convert.ToInt32((d - DateStart).TotalSeconds);
}
```

理论上返回的DownloadUrl就是访问文件的路径啦~然而这个链接一旦过期就会失效需要重新申请，前端也可以用Expires判断是否过期提醒用户刷新
但是坑还没完结，因为图片类型(image/jpg,png,gif)和PDF类型(目前只用过这两种)在点击链接的时候是默认打开的，项目需要的是让用户点击下载文件，怎么办呢~
这需要上传的时候多传一个字段，上面也写过了
``` javascript
frm.append('Content-Disposition', 'attachment;filename=asdasd.png');
```

这样上传的图片用`<img src='xxx' alt='' />`还是能正常显示，但是直接打开链接的话就变成下载了。filename后面的就是文件下载时的重命名
这个坑就坑在append这个字段的时候不能放在最下边，一开始放在下面试了半天死活不能下载后来灵机一动移上去居然就成功了...成功了...功了...了....
(不能理解阿里云的思路)
完 End

附一个自己封装的oss工具吧，别嫌我简陋...
``` js
var OSSFile = function () {
    var FileModel = function (objname, option) {
        if (!objname) {
            return null;
        }
        this.objname = objname;
        option = option || {};
        this.ispublic = option.ispublic || 0;
        this.url = option.url || null;
        this.expire = option.ex || 0;
        this.type = option.type || null;
    };
    FileModel.prototype.getLink = function (callback) {
        var _self = this;
        $.post('http://xxx.com/api/web/GetOssDownloadSign', {
            ObjectName: _self.objname,
            Type: _self.type,
            IsPublic: _self.ispublic    //项目有两个库，一个公开一个私有~
        }, function (data) {
            console.log(data);
            if (data.Sign) {
                _self.expire = data.Sign.Expires;
            }
            _self.url = data.DownloadUrl;
            if (typeof callback === 'function') {
                callback(data.DownloadUrl, data.Sign ? data.Sign.Expires : null);
            }
        });
    }
    //获取后缀名
    var get_suffix = function (filename) {
        pos = filename.lastIndexOf('.')
        suffix = ''
        if (pos != -1) {
            suffix = filename.substring(pos)
        }
        return suffix;
    }
    //随机字符串
    var random_string = function (len) {
        len = len || 32;
        var chars = 'ABCDEFGHJKMNPQRSTWXYZabcdefhijkmnprstwxyz2345678';
        var maxPos = chars.length;
        var pwd = '';
        for (i = 0; i < len; i++) {
            pwd += chars.charAt(Math.floor(Math.random() * maxPos));
        }
        return pwd;
    }

    return {
        createModel: function (objname, option) {
            return new FileModel(objname, option);
        },
        upload: function (file, option, callback, progress, error) {
            var xhr = null, url = null, option = option || {};
            if (typeof file !== 'undefined' && file.type && file.name && file.size) {
                xhr = new XMLHttpRequest();
                xhr.upload.addEventListener('progress', function (e) {
                    if (typeof progress === 'function') {
                        progress(e);
                    }
                }, false);
                xhr.addEventListener('load', function (e) {
                    if (typeof callback === 'function') {
                        callback(e.target.responseText, url);
                    }
                }, false);
                xhr.addEventListener('error', function (e) {
                    console.log(e);
                    alert('上传出错！')
                    if (typeof error === 'function') {
                        error(e);
                    }
                }, false);
                xhr.addEventListener('abort', function (e) {
                    console.log(e);
                }, false);
                //项目有两个库，一个公开一个私有~
                $.post('http://xxx.com/api/web/GetOssSignCallback', { IsPublic: option.ispublic || 0 }, function (data) {
                    console.log(data);
                    var oss = data.Sign;

                    var suffix = get_suffix(file.name);
                    var name = random_string(32) + suffix;

                    var frm = new FormData();
                    url = oss.dir + name;
                    frm.append('key', url);
                    frm.append('Content-Disposition', 'attachment;filename=' + (option.downloadname ? option.downloadname + suffix : name));
                    frm.append('policy', oss.policy);
                    frm.append('OSSAccessKeyId', oss.accessid);
                    frm.append('success_action_status', 200);
                    frm.append('callback', oss.callback);
                    frm.append('signature', oss.signature);
                    frm.append('file', file);

                    xhr.open('POST', oss.host);
                    xhr.send(frm);
                });
            }
            return xhr;
        },
        get_suffix: function (filename) {
            return get_suffix(filename);
        },
        random_string: function (len) {
            return random_string(len);
        }
    }
}();
```

调用示例(好吧我承认一开始写的已经忘了)：
``` javascript
$('#fff').change(function () {
    var _ = this;
    var file = _.files[0];
    xhr = OSSFile.upload(file, { ispublic: 0, downloadname: '这是重命名' }, function (txt, url) {
        console.log(txt, url);
    }, function (e) {
        if (e.lengthComputable) {
            var progress = Math.round(e.loaded * 100 / e.total);
            console.log(progress);
        }
    });
})
```

上面是上传，下面是获取下载或者访问链接：
``` javascript
var model = OSSFile.createModel('xinjianwenjianjia/FMS8yhfAky5sskFK.png', { ispublic: 0 });
model.getLink(function (u) {
    console.log(u);
    //$('#nice>img').attr('src', u);
});
```
完2.0