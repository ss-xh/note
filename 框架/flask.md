# flask

## 1. 基础

### 1.1 快速开始

```python
from flask import Flask

"""
app=Flask(__name__) 表示:
1.创建一个 Flask的实例对象
2.将__name__传递给Flask,即将本模块(这个文件)所在的目录作为整个项目的工程目录

-------------------------------
1.static_url_path="/flask",url访问静态资源的前缀, 如 http://127.0.0.1:7000/flask/index.html ,(只能以 /flask开头,而不是 flask)
这个index.html指的就是静态目录下的index.html文件

2.static_folder="static" ,静态文件的文件夹,默认 static,访问 http://127.0.0.1:7000/flask/index.html 
就是找静态目录 static下的index.html文件

3.template_folder="templates"  ,模板目录,默认 templates
"""
app = Flask(
    __name__,
    static_url_path="/flask",
    static_folder="static",
    template_folder="templates"
)


@app.route("/")  # 利用装饰器,将装饰器的参数值与视图函数 index 绑定
def access_index():
    return "index"


if __name__ == '__main__':
    app.run()
```



### 1.2 读取配置文件

```python
from flask import Flask

app = Flask(
    __name__,
    static_url_path="/flask",
    static_folder="static",
    template_folder="templates"
)


class MyConfig(object):
    LOG_LEVEL = "info"
    PORT = 5000


# 获取自定义的类对象中的属性值,并添加到app对象中
app.config.from_object(MyConfig)

# 从app对象中读取值,app.config可以直接看成字典,通过get()取值
print(app.config.get("PORT"))


@app.route("/")
def index():
    # 直接返回的是数据,而不是对应的视图
    return "hello world"


@app.route("/config")
def get_app_config():
    # 如果是在其他的模块文件中,无法直接引用当前 app对象,而又想使用 app.config
    # 此时引入的 current_app 就是全局的 app对象
    from flask import current_app
    print(current_app.config.get("LOG_LEVEL"))
    return "..."


if __name__ == '__main__':
    # 0.0.0.0 允许局域网内所有主机访问
    app.run(host="0.0.0.0", port=80, debug=True)
```



### 1.3 路由

#### 1.3.1 请求方式

```python
from flask import Flask

app = Flask(
    __name__,
    static_url_path="/flask",
    static_folder="static",
    template_folder="templates"
)


@app.route("/")
def index():
    """
    app.url_map:
        Map([<Rule '/flask/<filename>' (HEAD, GET, OPTIONS) -> static>,
        <Rule '/' (HEAD, GET, OPTIONS) -> index>])

    app.url_map显示所有的路由信息,
    '/' (HEAD, GET, OPTIONS) -> index> 表示 访问 127.0.0.1/ 会交给视图函数 index 处理
    这个函数支持的访问方式有 HEAD, GET, OPTIONS
    """
    print(app.url_map)
    return "hello world"


"""
1.定义视图函数的请求方式,不区分大小写
2.如果只支持一种请求方式,methods的数据类型也应该是 list
3.如果一个url对应多个完全相同(请求方式,路径完全相同)的视图函数,哪个视图函数在 app.url_map 的前面,就执行哪个
4.如果多个请求的路径相同,可以通过指定不同的请求方式找到对应的视图函数 
"""


@app.route("/center", methods=["POST", "delete"])
def center1():
    return "center post delete"


@app.route("/center", methods=["get"])
def center2():
    return "center get"


# 一个视图函数可以对应多个请求的路径
@app.route("user1")
@app.route("user2")
def user():
    return "user"


if __name__ == '__main__':
    app.run(host="0.0.0.0", port=80, debug=True)
```



#### 1.3.2 重定向

```python
from flask import Flask, redirect, url_for

app = Flask(
    __name__,
    static_url_path="/flask",
    static_folder="static",
    template_folder="templates"
)


@app.route("/index")
def index():
    return "hello world"


@app.route("/login")
def login():
    # 通过视图函数名 index 找到对应的 请求url => /index
    url = url_for("index")

    # 将请求重定向到另外一个视图函数
    return redirect(url)


if __name__ == '__main__':
    app.run(host="0.0.0.0", port=80, debug=True)
```



#### 1.3.3 url参数提取

```python
from flask import Flask

app = Flask(
    __name__,
    static_url_path="/flask",
    static_folder="static",
    template_folder="templates"
)


@app.route("/")
def index():
    return "hello world"


@app.route("/goods/<int:goods_id>")
def get_goods(goods_id):
    """
    <int:goods_id> 这种语法格式,在flask被称为转换器,
    默认转换器可以将数据转换成 int,float,str 格式的数据类型
    即访问 127.0.0.1/goods/11 可以自动将11 赋值给 goods_id,且 11的数据类型是int
    如果访问 127.0.0.1/goods/11xx 那么 11xx转换成int会失败,返回404页面到客户端
    """
    print(type(goods_id))  # <class 'int'>
    return f"goods_id:{goods_id}"


@app.route("/user/<user_id>")
def get_user(user_id):
    """
    如果不指定数据类型<user_id>,则使用默认转换器将 user_id转成 str数据类型
    """
    return f"user_id:{user_id},type:{type(user_id)}"


if __name__ == '__main__':
    app.run(host="0.0.0.0", port=80, debug=True)
```



#### 1.3.4 自定义转换器

```python
from flask import Flask

app = Flask(
    __name__,
    static_url_path="/flask",
    static_folder="static",
    template_folder="templates"
)

"""
使用自定义转换器的步骤:
1.导入转换器基类  from werkzeug.routing import BaseConverter
2.自定义转换器,需要继承 BaseConverter
3.将自定义的转换器加入到 app对象当中
4.使用自定义转换器
"""
from werkzeug.routing import BaseConverter


class ReConverter(BaseConverter):
    def __init__(self, url_map, regex):
        # 调用父类的初始化方法
        super(ReConverter, self).__init__(url_map)
        # 变量名 regex是固定的,表示正则表达式
        self.regex = regex


# 自定义转换器的名称(re)任意字符串都可以
app.url_map.converters["re"] = ReConverter

"""
@app.route("/register/<re(r'1[35789]\d{9}'):phone_number>")使用自定义转换器, <自定义转换器别名:接收的url参数的名称>
re 表示自定义转换器的的别名 
创建自定义转换器实例对象的时候会传给 会将 app.url_map, r'1[35789]\d{9}' 传递给 ReConverter.__init__(...)
即:self.regex = r'1[35789]\d{9}'
"""

# 127.0.0.1/register/13123456789
@app.route("/register/<re(r'1[35789]\d{9}'):phone_number>")
def register(phone_number):
    return f"phone_number:{phone_number}"


@app.route("/")
def index():
    return "hello world"


if __name__ == '__main__':
    app.run(host="0.0.0.0", port=80, debug=True)
```

---

```python
from flask import Flask, url_for, redirect

app = Flask(
    __name__,
    static_url_path="/flask",
    static_folder="static",
    template_folder="templates"
)

from werkzeug.routing import BaseConverter


class ReConverter(BaseConverter):
    def __init__(self, url_map, regex):
        super().__init__(url_map)
        self.regex = regex

    def to_python(self, value):
        # phone_number 的值会先经过这个方法,可以根据自己的需求对value做一些个性化定制
        value = "119"
        return value

    def to_url(self, value):
        # 次数的url 是 url_for() 函数传递过来的值,即 value = 13123456789
        value = "13111111111"
        return value


app.url_map.converters["re"] = ReConverter


@app.route("/register/<re(r'1[35789]\d{9}'):phone_number>")
def register(phone_number):
    return f"phone_number:{phone_number}"


@app.route("/login")
def login():
    """
    1.查找视图函数名 register找到对应的url,将 phone_number 的值和找到的url拼接起来
    2.url_for(...)  -> /register/13123456789
    3.此处的 phone_number 这个变量名需要与 定义视图函数 变量名一致 /register/<re(...):phone_number>
    4.因为使用了自定义转换器,所以调用 url_for() 的过程中 会调用 自定义转换器的 to_url方法
    5.因为转换器改变了 value的值 所以最终url -> /register/13111111111 
    """
    url = url_for("register", phone_number="13123456789")
    return redirect(url)


@app.route("/")
def index():
    return "hello world"


if __name__ == '__main__':
    app.run(host="0.0.0.0", port=80, debug=True)
```



### 1.4 request

#### 1.4.1 form

```python
@app.route("/register/form")
def register1():
    """
    获取form表单 数据
    <form action="" method="post" enctype="multipart/form-data">
        <label>
            <input type="text" name="username"/>
        </label>
        <label>
            <input type="password" name="password"/>
        </label>
        <label>
            <input type="file" name="files">
        </label>
        <label>
            <input type="submit"/>
        </label>
</form>
    """
    # 如果有多个同名参数 username,通过get方式只能获取第一个 username对应的值
    username = request.form.get("username", None)
    password = request.form.get("password", None)
    if username and password:
        return f"{username}:{password}"
    return "username or password is None"
```



#### 1.4.2 json

```python
@app.route("/register/json")
def register2():
    """
    {
        "username": "123xx",
        "password": "xxx"
    }
    request.content-type =  "application/json"
    """
    print(request.data)
    if request.data:
        return request.data
    return "nothing to requests"
```



#### 1.4.3 param

```python
@app.route("/register/param")
def register3():
    """
    127.0.0.1/register/param?username=linux&age=18

    url传入查询参数,不仅是支持get请求方式,其他任意请求方式都支持的
    传入查询参数的同时,也可以传入表单数据或者json数据

    如果有多个同名参数 username,通过get方式只能获取第一个 username对应的值
    """
    # username = request.args["username"]
    username = request.args.get("username", None)
    age = request.args.get("age", None)
    if username and age:
        return f"username:{username}:age{age}"
    return "nothing to requests"
```



#### 1.4.4 获取多个同名参数的值

```python
@app.route("/register/same_param")
def register():
    """
    获取多个同名参数的值
    127.0.0.1/register/same_param?username=linux&age=18&age=120
    <form>
        <input type="text" name="username"/> 111
        <input type="text" name="username"/> 222
    </form>
    """
    usernames = request.form.getlist("username")
    ages = request.args.getlist("age")

    return f"{usernames}:{ages}"  # ['111', '222']:['18', '120']
```



#### 1.4.5 常用的请求参数

```python
from flask import Flask, request

app = Flask(
    __name__,
    static_url_path="/flask",
    static_folder="static",
    template_folder="templates"
)


@app.route("/register", methods=["get", "post"])
def register():
    """
    request.headers.environ:dict=
    {
        'wsgi.version': (1, 0), 'wsgi.url_scheme': 'http',
        'wsgi.input': <_io.BufferedReader name=8>,
        'wsgi.errors': <_io.TextIOWrapper name='<stderr>' mode='w' encoding='utf-8'>,
        'wsgi.multithread': True, 'wsgi.multiprocess': False,
        'wsgi.run_once': False, 'werkzeug.socket': <socket.socket fd=8, family=AddressFamily.AF_INET,
        type=SocketKind.SOCK_STREAM, proto=0, laddr=('127.0.0.1', 80), raddr=('127.0.0.1', 57791)>,
        'SERVER_SOFTWARE': 'Werkzeug/3.0.2', 'REQUEST_METHOD': 'GET',
        'SCRIPT_NAME': '', 'PATH_INFO': '/register', 'QUERY_STRING': '',
        'REQUEST_URI': '/register', 'RAW_URI': '/register', 'REMOTE_ADDR': '127.0.0.1',
        'REMOTE_PORT': 57791, 'SERVER_NAME': '0.0.0.0', 'SERVER_PORT': '80',
        'SERVER_PROTOCOL': 'HTTP/1.1',
        'HTTP_USER_AGENT': 'PostmanRuntime-ApipostRuntime/1.1.0',
        'HTTP_CACHE_CONTROL': 'no-cache', 'HTTP_ACCEPT': '*/*',
        'HTTP_ACCEPT_ENCODING': 'gzip, deflate, br',
        'HTTP_CONNECTION': 'keep-alive', 'HTTP_HOST': '127.0.0.1',
        'werkzeug.request': <Request 'http://127.0.0.1/register' [GET]>,
        'werkzeug.debug.preserve_context': <built-in method append of list object at 0x101fef600>
    }
    """
    environ = request.headers.environ  # 获取请求头中所有的参数
    print(type(environ))  # dict
    print(request.cookies)  # cookies
    print(request.method)  # 请求方式
    print(request.user_agent)  # user-agent
    print(request.base_url)  # base_url
    print(request.host_url)  # 请求的url
    return request.cookies


if __name__ == '__main__':
    app.run(host="0.0.0.0", port=80, debug=True)
```



#### 1.4.6 文件上传

```python
from flask import Flask, request

app = Flask(
    __name__,
    static_url_path="/flask",
    static_folder="static",
    template_folder="templates"
)


@app.route("/upload", methods=["get", "post"])
def register():
    """
    <input type="file" name="image"/>
    <input type="file" name="image"/>

    request.files.getlist 可以获取多个同名文件的列表
    file.filename 获取上传文件的文件名
    """
    if request.method == "POST":
        list_file = request.files.getlist("image")
        if len(list_file) > 0:
            for file in list_file:
                # file.save(f"{file.filename}") # 快速保存文件
                save_file(file.filename, file.read())  # 自定义方法保存文件
            return "success"
    return "fail"


def save_file(file_name, content):
    with open(file_name, "wb") as f:
        f.write(content)


if __name__ == '__main__':
    app.run(host="0.0.0.0", port=80, debug=True)

```



### 1.5 response

#### 1.5.1 abort

```python
from flask import Flask, request, abort

app = Flask(
    __name__,
    static_url_path="/flask",
    static_folder="static",
    template_folder="templates"
)


@app.route("/login", methods=["post", "get"])
def login():
    """
    调用abort函数 可以立即终止当前视图函数的执行,
    在abort函数里面传递的状态码,可以传递给前端,
    传递的状态码必须是 http标准状态码
    """
    username = request.form.get("username", None)
    password = request.form.get("password", None)
    if username and password:
        return "success"
    abort(401)  # 传递 401 状态码给前端


if __name__ == '__main__':
    app.run(host="0.0.0.0", port=80, debug=True)
```



#### 1.5.2 自定义错误信息

```python
from flask import Flask, request, abort

app = Flask(
    __name__,
    static_url_path="/flask",
    static_folder="static",
    template_folder="templates"
)


@app.route("/login", methods=["post", "get"])
def login():
    username = request.form.get("username", None)
    password = request.form.get("password", None)
    if username and password:
        return "success"
    abort(401)


@app.errorhandler(401)
def handler_error_page(error_info):
    # error_info 表示是错误信息,flask调用这个方法时会自动将错误信息对象 传递过来
    # abort(401) 的时候,下面的信息会直接返回给前端
    return f"你已经被禁止访问{error_info}"


@app.errorhandler(404)
def handler_error_page(error_info):
    return f"页面找不到{error_info}"


if __name__ == '__main__':
    app.run(host="0.0.0.0", port=80, debug=True)
```



#### 1.5.3 自定义响应信息

```python
from flask import Flask, make_response

app = Flask(
    __name__,
    static_url_path="/flask",
    static_folder="static",
    template_folder="templates"
)


# 第一种
@app.route("/index")
def index():
    # 自定义返回信息格式: 响应体, 状态码, 响应头(键值对用)
    # return "自定义响应体信息", 200, [("sever", "my_server"), ("Set-Cookie", "123cookies")]
    # return "自定义响应体信息", 200, {"sever": "my_server", "Set-Cookie": "123cookies"}
    return "自定义响应体信息", 200


# 第二种
@app.route("/login")
def login():
    resp = make_response("响应体信息")
    resp.status_code = 200
    resp.headers["server"] = "my_server"
    return resp


if __name__ == '__main__':
    app.run(host="0.0.0.0", port=80, debug=True)
```



#### 1.5.4 返回json数据

```python
import json

from flask import Flask, make_response, jsonify

app = Flask(
    __name__,
    static_url_path="/flask",
    static_folder="static",
    template_folder="templates"
)


@app.route("/index")
def index():
    data = {
        "name": "xx",
        "ag4": "123"
    }

    json_str = json.dumps(data)
    resp = make_response(json_str)
    resp.status_code = 200
    resp.headers["Content-Type"] = "application/json"
    return resp


@app.route("/login")
def login():
    data = {
        "name": "xx",
        "ag4": "123"
    }
    """
    jsonify 帮我们做了两步操作
    1.将数据dict数据类型转成 json字符串类型
    2.设置响应头 headers["Content-Type"]="application/json"
    """
    # return jsonify(name="xxx", sex="women") # jsonify会自动 将字典参转成json字符串
    return jsonify(data)


if __name__ == '__main__':
    app.run(host="0.0.0.0", port=80, debug=True)
```



#### 1.5.5 cookies

```python
import json

from flask import Flask, make_response, request, jsonify

app = Flask(
    __name__,
    static_url_path="/flask",
    static_folder="static",
    template_folder="templates"
)


@app.route("/set_cookies")
def set_cookies():
    resp = make_response()
    # 可以重复设置多个cookie,如果不设置cookie的有效期,默认浏览器关闭时,cookie过期
    resp.set_cookie("name", "linux")
    resp.set_cookie("age", "18")

    # max_age =120 设置cookie的有效期是120秒
    resp.set_cookie("gender", "man", max_age=120)
    return resp


@app.route("/get_cookies")
def get_cookies():
    # 获取指定key的cookies值
    print(request.cookies.get("name"))
    # 返回所有的cookies信息
    return jsonify(request.cookies)


@app.route("/del_cookies")
def del_cookies():
    resp = make_response()
    # 删除指定的cookie,无法做到直接让浏览器删除,
    # 而是指定当前要删除的cookie的过期时间为当前时间
    resp.delete_cookie("gender")
    return resp


@app.route("/set_cookies_by_headers")
def set_cookies_by_headers():
    response = make_response()
    # 通过请求头设置 cookie
    response.headers["set-cookie"] = "gender=woman; Expires=Sat, 27 Apr 2024 07:55:38 GMT; Max-Age=400; Path=/"
    return response


if __name__ == '__main__':
    app.run(host="0.0.0.0", port=80, debug=True)
```



#### 1.5.6 session

```python
from flask import Flask, session, jsonify

app = Flask(
    __name__,
    static_url_path="/flask",
    static_folder="static",
    template_folder="templates"
)

# 使用 session 必须设置 app.config["SECRET_KEY"] = "随机字符串"
# flask会利用 SECRET_KEY 设置的字符串对需要保存的 session信息进行加密,保存到浏览器的 cookie信息中
# 获取session的值时候,加密后的cookie信息,用 SECRET_KEY 解密得到相应的保存的信息
app.config["SECRET_KEY"] = "chinesexxx"


@app.route("/login")
def login():
    # 利用 SECRET_KEY 加密session保存的信息,然后返回给浏览器,交给cookie保存
    session["username"] = "linux"
    session["user_id"] = "123"
    return ""


@app.route("/order")
def order():
    # 利用 SECRET_KEY 解密从前端拿到的 cookie 信息,获取对应的 session保存的信息
    username = session.get("username", None)
    user_id = session.get("user_id", None)
    return jsonify(username=username, user_id=user_id)


if __name__ == '__main__':
    app.run(host="0.0.0.0", port=80, debug=True)
```



## 2.进阶

### 2.1 请求钩子

#### 2.1.1 exp2

> 钩子函数本身不区分视图函数,不区分路径,只要加上对应的钩子,都会执行对应的钩子函数

```python
from flask import Flask

app = Flask(
    __name__,
    static_url_path="/flask",
    static_folder="static",
    template_folder="templates"
)


@app.before_request
def handler_before_request():
    print("每次请求前执行这个钩子函数")


@app.after_request
def handler_after_request(response):
    print("每个请求对应的视图函数无异常,才会在视图函数结束后执行这个钩子函数")
    # 这个response 是视图函数的返回值对象,会自动传递到 钩子函数中
    return response


@app.teardown_request
def handler_teardown_request(response):
    print("无论请求对应的视图函数是否发生异常,每次请求后都会执行这个钩子函数 ")
    return response


@app.route("/")
def index():
    print("index视图函数被执行...")

    return "index"


@app.route("/order")
def order():
    return "order"


def user():
    return "user"


if __name__ == '__main__':
    app.run(host="0.0.0.0", port=80)
```



#### 2.1.2 exp2

> 通过自定义规则,对不同的视图函数,在钩子函数中进行逻辑上的区分

```python 
from flask import Flask, request, url_for, session, abort

app = Flask(
    __name__,
    static_url_path="/flask",
    static_folder="static",
    template_folder="templates"
)

app.config["SECRET_KEY"] = "123"


@app.before_request
def handler_before_request():
    # 对请求访问的是 user,order函数进行权限校验 
    if request.path in [url_for("user"), url_for("order")]:
        username = session.get("username", None)
        if not username:
            abort(401)


@app.route("/login")
def login():
    session["username"] = "linux"
    return "login success"


@app.route("/")
def index():
    print("index视图函数被执行...")
    return "index"


@app.route("/order")
def order():
    return "order"


@app.route("/user")
def user():
    return "user"


if __name__ == '__main__':
    app.run(host="0.0.0.0", port=80)
```



### 2.2 上下文对象

```python
"""
g：global全局对象
g对象是专门用来保存用户的数据的
g对象在一次请求中的所有的代码的地方，都是可以使用的
突破变量存储位置限制，为数据传递添加了新的方式，比如我们在before_request产生一个数据在后面需要使
用，可以保存在g对象中，在其他视图函数中就可以使用这个数据．

request：请求对象，可以获取客户端提交过来的所有请求信息

session：会话技术，服务端会话技术的接口

current_app： 使用获取当前app需要注意，一定要在程序初始化完成之后
"""
```



```python
from flask import Flask, request, g

app = Flask(
    __name__,
    static_url_path="/flask",
    static_folder="static",
    template_folder="templates"
)

app.config["SECRET_KEY"] = "123"


@app.route('/login')
def index():
    """
    request={
        "线程ID_1":{"form":"...","args":"...","cookies":""}
        "线程ID_2":{"form":"...","args":"...","cookies":""}
        ...
    }

    1.request 是一个全局变量,也可以看成线程局部变量,一个请求对应一个线程ID

    2.当某个视图函数执行 request.args[form,cookies..],request这个全局变量
    会先拿到当前请求的线程ID 到 request 这个全局变量里面寻找 对应线程ID的key对应的 value

    3.通过不同的线程ID,就能在全局变量request中拿到不同的请求对应的值

    4. 当一个请求结束(线程结束)的时候,会自动销毁全局request对应的线程ID的key-value

    5.session这个上下文对象,在不同请求中获取不同值的操作与request相同
    """
    username = request.args.get("username", None)
    g.username = username  # g看成是一个空对象,可以通过对象赋值的方式,在同一个线程中传递值
    show()
    return "success"


def show():
    # 全局对象g,在一次请求中生效,当请求结束时,对应的g对象自动销毁
    # 不同的 g 对象 对应不同的请求
    print(g.get("username"))


if __name__ == '__main__':
    app.run(host="0.0.0.0", port=80)

```



### 2.2 flask-script

> flask == 2.3.3
>
> flask-script == 2.0.6
>
> ```python
> # 出现这种错误 ModuleNotFoundError: No module named 'flask._compat',需要修改 flask_script 模块的 __init__ 文件
> # from flask._compat import text_type
> from flask_script._compat import text_type
> ```

```python
from flask import Flask
from flask_script import Manager

app = Flask(
    __name__,
    static_url_path="/flask",
    static_folder="static",
    template_folder="templates"
)

# 将app对象交给 manager对象管理
manager = Manager(app)

if __name__ == '__main__':
    # 通过 manager对象 启动 flask
    manager.run()

```

```bash
# 通过命令行启动 flask
python main.py runserver -h 0.0.0.0 -p 80 

# 查看帮助信息
python main.py --help
python main.py runserver --help
```



### 2.3 模版

#### 2.3.1 取值操作

```python
from flask import Flask, render_template

app = Flask(
    __name__,
    static_url_path="/flask",
    static_folder="static",
    template_folder="templates"
)


@app.route("/")
def index():
    # render_template(模块的名称,渲染数据) 渲染数据通过字典参数传递
    data = {
        "sex": "gender",
        "hobby": "basketball"
    }
    return render_template("user.html", name="linux", age=18, data=data)


@app.route("/list")
def get_dict_list():
    data = {
        "info": {"country": "china", "gov": "beijing"},
        "hobby": ["music", "ball", "game"]
    }
    return render_template("center.html", data=data)


if __name__ == '__main__':
    app.run(host="0.0.0.0", port=80)

```

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
/ ---------
<p>name:{{name}}</p>
<p>age:{{age}}</p>
<p>{{data.sex}}</p>
<p>{{data.hobby}}</p>

/list ---------
<p>{{data.info.country}}</p>
<p>{{data.hobby[1]}}</p>
</body>
</html>
```



#### 2.3.2 自定义模版过滤器

```python
from flask import Flask, render_template

app = Flask(
    __name__,
    static_url_path="/flask",
    static_folder="static",
    template_folder="templates"
)


@app.route("/")
def index():
    # render_template(模块的名称,渲染数据) 渲染数据通过字典参数传递
    data = {
        "nums": [i for i in range(10)]
    }
    return render_template("my_li.html", data=data)


# 自定义模版过滤器
def my_li1(li):
    return li[::2]


# 将自定义的模版过滤器添加到 app对象当中
app.add_template_filter(my_li1, "my_li1")


@app.template_filter("my_li2")  # 直接通过装饰器,将模版过滤器加入到 app对象当中  
def my_li2(li):
    return li[::2]


if __name__ == '__main__':
    app.run(host="0.0.0.0", port=80)

```

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
使用模版过滤器:
{{data.nums | my_li}}
<!--输出结果
使用模版过滤器: [0, 2, 4, 6, 8]
-->
</body>
</html>
```



### 2.4 蓝图

#### 2.4.1 循环导入的问题

![image-20240428135824955](./assets/image-20240428135824955.png)

```python
"""
t1作为主程序运行,t2是注册视图函数,如果直接运行t1,就会造成循环导入的问题
因为 t1必须先创建才能有 app 的实例对象,在 t1当中导入t2 则需要 app对象 ,t2又需要t1的app实例对象,来注册路由
最后就会造成 t1等到 app实例对象,t2等待着 t1的app实例对象传过来
"""
```



#### 2.4.2 解决方案(无蓝图)

![image-20240428140425783](./assets/image-20240428140425783.png)

```python
"""
先导入 t2的函数,然后拿装饰器对象,装饰 index 函数
让它变成视图函数,这样就不会存在循环导入的问题

@app.route("/") 是一个三层函数嵌套的闭包,拿到 "/" 参数,返回一个两层嵌套的闭包
app.route("/") 执行的结果返回的就是一个两层函数嵌套的闭包
app.route("/")(index) 将index函数的内存地址传给闭包,达到装饰的效果
"""
```



#### 2.4.3 蓝图的概念

```python
"""
蓝图：用于实现单个应用的视图、模板、静态文件的集合。

蓝图就是模块化处理的类。

简单来说，蓝图就是一个存储操作路由映射方法的容器，主要用来实现客户端请求和URL相互关联的功能。
在Flask中，使用蓝图可以帮助我们实现模块化应用的功能。

蓝图的运行机制：

蓝图是保存了一组将来可以在应用对象上执行的操作。注册路由就是一种操作，当在程序实例上调用route装饰器注册路由时，这个操作将修改对象的url_map路由映射列表。当我们在蓝图对象上调用route装饰器注册 路由时，它只是在内部的一个延迟操作记录列表defered_functions中添加了一个项。当执行应用对象的 register_bolueprintO 方法时，应用对象从蓝图对象的 defered_ functions 列表中取出每一项，即调用应用对象 的 add_urL_rule()方法，这将会修改程序实例的路由映射列表。
"""
```

 

#### 2.4.4 蓝图基本使用

![image-20240428141449269](./assets/image-20240428141449269.png)



#### 2.4.5 目录中使用蓝图

![image-20240428144241603](./assets/image-20240428144241603.png)

![image-20240428144300504](./assets/image-20240428144300504.png)

![image-20240428144315309](./assets/image-20240428144315309.png)

![image-20240428144335322](./assets/image-20240428144335322.png)





### 2.5 ORM

```python
"""
对应的版本:
Flask==2.3.3
Flask-Migrate==4.0.4
Flask-SQLAlchemy==3.0.3
PyMySQL==1.0.2
SQLAlchemy==2.0.5.post1 
"""
```



#### 2.5.1 框架搭建

##### 2.5.1.1 模块拆分

基于下面3.1模块拆分的步骤新增配置模块

![image-20240428170059935](./assets/image-20240428170059935.png)



- database_settings.py

```python
class SqliteSettings(object):
    """
    sqlite数据库配置类
    sqlite数据库不需要驱动,也不需要用户名和密码
    """
    SQLALCHEMY_DATABASE_URI = 'sqlite:///sqlite3.db'
    SQLALCHEMY_TRACK_MODIFICATIONS = False  # 禁止对象追踪
```

- extends.py

```python
"""
这个模块专门用于导入插件
"""

from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate

db = SQLAlchemy()
migrate = Migrate()


def init_extends(app):
    """
    将插件与 全局Flask()实例对象绑定
    :param app: 全局flask实例对象
    :return: 全局flask实例对象
    """
    db.init_app(app)
    migrate.init_app(app=app, db=db)
```

- config.\__init__.py

```python
from .database_settings import SqliteSettings
from .extends import init_extends, db, migrate
```

- app.\__init__.py

```python
"""
初始化操作,创建 flask应用
"""

from flask import Flask
from .config import SqliteSettings, init_extends, db, migrate
from .views import user_blue # 因为 导入 views 用到了模块 db,所以这个 只能写到最下面,不然就造成了循环导入


def create_app():
    app = Flask(
        __name__,
        static_url_path="/static",
        static_folder="static",
        template_folder="templates"
    )
    # 注册蓝图
    app.register_blueprint(blueprint=user_blue)

    # 添加sqlite数据库的配置信息
    app.config.from_object(SqliteSettings)

    # 将插件与app对象绑定
    init_extends(app)

    return app

```



##### 2.5.1.2 创建模型类

- models.py

```python
"""
配置模型类
"""

from . import db


class User(db.Model):
    """
    db.Column：表示字段 db.Integer：表示整数
    primary_key=True ： 主键
    autoincrement=True： 自动递增
    db.String（30）：varchar（30）可变字符中
    unique=True ：唯一约束
    index=True ： 普通索引
    default : 默认值
    nullable: 是否允许为空
    """
    # 表名
    __tablename__ = 'tb_user'
    # 定义表字段
    id = db.Column(db.Integer, primary_key=True, autoincrement=True)
    name = db.Column(db.String(30), unique=True, index=True)
    age = db.Column(db.Integer, default=18)
    sex = db.Column(db.Boolean, default=True)
    salary = db.Column(db.Float, default=12345.5, nullable=False)
```



##### 2.5.1.3 数据迁移

```python
"""
执行数据迁移前,要确保 views.py 导入了 models.py模块,因为flask执行数据迁移的时候,会默认找 继承 db.Model的类
进入主程序所在的目录 App.py
flask db init 创建迁移文件夹migrates，只调用一次
flask db migrate 生成迁移文件
flask db upgrade 执行迁移文件中的升级
flask db downgrade 执行迁移文件中的降级
"""
```

- `flask db init`

![image-20240428172723896](./assets/image-20240428172723896.png)





- `flask db migrate`

![image-20240428172929441](./assets/image-20240428172929441.png)



- `flask db upgrade`

![image-20240428173516142](./assets/image-20240428173516142.png)





##### 2.5.1.4 重新执行数据迁移

```bash
# 如果修改了模型了的属性,并且希望同步到数据库表当中,那么需要重新执行数据迁移和升级操作
flask db migrate # 生成迁移文件
flask db upgrade # 执行迁移文件中的升级

# 执行完成后会形成一个新的版本号,且数据库字段与模型类已保持一致 
```

![image-20240428174233874](./assets/image-20240428174233874.png)



##### 2.5.1.5 数据库版本降级

```bash
# 如果不满意刚才的数据库修改操作,可以通过降级操作,恢复数据库到上一个版本,
# 模型类不会自动同步,所以需要删除模型类刚才添加或修改的属性,保持模型类和数据库字段同步
flask db downgrade # 执行迁移文件中的降级
```





#### 2.5.2 单表CRUD

##### 2.5.2.1 CUD

```python
"""
# 添加mysql数据的配置
class MysqlSettings(object):
    SQLALCHEMY_DATABASE_URI = "mysql+pymysql://root:root@192.168.10.254:3306/python_test"
    SQLALCHEMY_TRACK_MODIFICATIONS = False  # 禁止对象追踪
    
app.config.from_object(MysqlSettings)

....
flask db init
flask db migrate
flask db upgrade
"""
```

```python
"""
配置路由和视图函数
"""
from flask import Blueprint
from .models import User
from . import db

# 创建蓝图对象
user_blue = Blueprint("user", __name__)


# 新增数据
@user_blue.route("/add_one", methods=["post"])
def add_user():
    user = User()
    user.name = "linux"
    user.age = 18
    user.sex = True
    user.salary = 12345.6

    try:
        db.session.add(user)
        db.session.commit()
        return "success"
    except Exception as e:
        db.session.rollback()
        return f"error:{e}"


# 一次新增多条数据
@user_blue.route("/add_more", methods=["post"])
def add_users():
    list_user = list()

    for i in range(10):
        user = User()
        user.name = f"linux_{i}"
        user.age = i
        user.sex = True
        user.salary = 12345.6
        list_user.append(user)

    try:
        db.session.add_all(list_user)
        db.session.commit()
        return "success"
    except Exception as e:
        db.session.rollback()
        db.session.flush()
        return f"error:{e}"

# 删除记录
@user_blue.route("/delete", methods=["post"])
def delete_users():
    user = User.query.first()  # 获取这个表中的第一条数据
    try:
        db.session.delete(user)
        db.session.commit()
        return "success"
    except Exception as e:
        db.session.rollback()
        db.session.flush()
        return f"error:{e}"

# 修改记录
@user_blue.route("/update", methods=["post"])
def update_users():
    user = User.query.first()  # 获取这个表中的第一条数据
    user.name = "china"
    try:
        db.session.commit()
        return "success"
    except Exception as e:
        db.session.rollback()
        db.session.flush()
        return f"error:{e}"

```

##### 2.5.2.2 R

```python
"""
过滤器

filter() 把过滤器添加到原查询上，返回一个新查询
filter_by()把等值过滤器添加到原查询上，返回一个新查询


limit() 使用指定的值限制原查询返回的结果数量，返回一个新查询

offset() 偏移原查询返回的结果，返回一个新查询

order_by() 根据指定条件对原查询结果进行排序，返回一个新查询 
group_by（） 根据指定条件对原查询结果进行分组，返回一个新查询
常用查询

al1() 以列表形式返回查询的所有结果，返回列表

first()返回查询的第一个结果，如果没有结果，则返回 None
first_or_404() 返回查询的第一个结果，如果没有结果，则终止请求，返回 404 错误响应 
get() 返回指定主键对应的行，如果没有对应的行，则返回 None

get_or_404() 返回指定主键对应的行，如果没找到指定的主键，则终止请求，返回 404 错误响应

count()返回查询结果的数量
paginate() 返回一个 Paginate 对象，它包含指定范围内的结果
"""
```

```python
@user_blue.route("/query_all")
def query_all():
    users = User.query.all()  # 返回User类对应的的这张表的所有数据
    print(users)  # <class:list>
    print(User.query, type(User.query))  # flask_sqlalchemy   查询类

    return "success"


@user_blue.route("/query_filter_by")
def query_filter_by():
    u1 = User.query.filter_by(age=7)  # 获取age=7的对象集合
    print(list(u1))

    u2 = User.query.first()
    print(u2)  # 取出第一条数据
    return "success"


@user_blue.route("/query_filter")
def query_filter():
    users = User.query.filter()  # 返回查询集
    print(users)
    user = users.get(11)  # 拿到查询集后,获取主键 是11的 User对象
    print(user, type(user))  # <class 'App.models.User'>

    u2 = User.query.filter(User.age == 7)  # 查询 age == 8的对象
    print(u2)

    u3 = User.query.filter(User.age > 3)  # 查询 age > 3的对象
    print(list(u3))  # 将查询的结果集 变成list,方便取出对象

    # 多条件查询,first() 返回查询的第一条数据
    u4 = User.query.filter(User.age > 1, User.name == "linux_6").first()
    print(u4)

    # all()获取查询的结果集全部数据
    u5 = User.query.filter(User.age > 7, User.name.like("%linux%")).all()
    print(u5)

    # 返回查询结果的数量, User.age.in_([3, 5, 7]) 年龄等于 3或5或7
    u6 = User.query.filter(User.age.in_([3, 5, 7])).count()
    print(u6)

    # 查询年龄在 9~100岁之间的数据
    u7 = User.query.filter(User.age.between(9, 100)).all()
    print(u7)

    # name 包含 linux,且 age 等于3
    u8 = User.query.filter(User.name.contains("linux"), User.age == 3).all()
    print(u8)

    # 取前3条数据
    u9 = User.query.filter().limit(3).all()
    print(u9)

    # 跳过前3条数据
    u10 = User.query.filter().offset(3).all()
    print(u10)

    # 按照 age排序,默认升序
    u11 = User.query.filter().order_by("age").all()
    print(u11)

    # 按照 age排序,降序排列
    from sqlalchemy import desc
    u12 = User.query.filter().order_by(desc("age")).all()
    print(u12)

    from sqlalchemy import or_, and_, not_
    # and_
    User.query.filter(and_(User.age > 3, User.name.contains("linux_1"))).all()
    # User.query.filter(User.age > 8, User.name = "linux").all()

    # or_
    u13 = User.query.filter(or_(User.age > 8, User.age < 3)).all()
    print(u13)

    # not_
    u14 = User.query.filter(not_(or_(User.age > 8, User.age < 3))).all()
    print(u14)

    # User.name 以linux开头
    u15 = User.query.filter(User.name.startswith("linux")).all()
    print(u14)

    # User.name 以linux 结尾
    u16 = User.query.filter(User.name.endswith("linux")).all()
    print(u14)

    return "success"
```



##### 2.5.2.3 分页

```python
@user_blue.route("/query_page")
def query_page():
    page_num = 2  # 取第2页的数据
    page_count = 5  # 每页5条数据
    """
    第1页 1 2 3 4 5
    第2页 6 7 8 9 10 
    第3页 11 12 13 14 15
    """
    offset = (page_num - 1) * page_count  # 跳过前 offset 条

    u1 = User.query.filter().offset(offset).limit(page_count).all()
    print(u1)
    return "success" 
```

```python
@user_blue.route("/query_paginate")
def query_paginate():
    # 这两个数据正常来说应该是接受前端的传值
    current_page = 1
    page_count = 5

    # User.query.paginate() 返回分页对象,未进行条件过滤,默认拿到整张表数据
    paginate = User.query.filter(User.age > 3).paginate(page=current_page, per_page=page_count, error_out=False)

    # 返回当前页的所有内容
    print(paginate.items)

    # 判断是否还有下一页
    print(paginate.has_next)  # class:bool

    # 判断是否还有上一页
    print(paginate.has_prev)  # class:bool

    # 返回当前页码
    print(paginate.page)

    # 返回总页数
    print(paginate.pages)

    # 返回每页的数量
    print(paginate.per_page)

    # 返回总记录数
    print(paginate.total)
    return "success"
```



#### 2.5.3 一对多表CRUD

##### 2.5.3.1 构建一对多关系的模型类

```python
"""
配置模型类
"""

from . import db


class User(db.Model):
    """
    db.Column：表示字段 db.Integer：表示整数
    primary_key=True ： 主键
    autoincrement=True： 自动递增
    db.String（30）：varchar（30）可变字符中
    unique=True ：唯一约束
    index=True ： 普通索引
    default : 默认值
    nullable: True 允许为空
    """
    # 表名
    __tablename__ = 'tb_user'
    # 定义表字段
    id = db.Column(db.Integer, primary_key=True, autoincrement=True)
    name = db.Column(db.String(30), unique=True, index=True)
    age = db.Column(db.Integer, default=18)
    sex = db.Column(db.Boolean, default=True)
    salary = db.Column(db.Float, default=12345.5, nullable=True)

    # 建立关系映射,
    # orders不是属于数据库表中的字段,而是单纯的一个类属性,是为了方便查询操作
    # db.relationship("Order",...) "Order" 表示建立关联类的名字
    # backref="user", 相当于给order对象新增一个属性 user
    # 拿到 order对象后,可以直接通过 order.user 获取order对应的 User对象
    # lazy=True 表示懒加载 ,当调用  orders.user 才会真正的建立关联关系
    orders = db.relationship("Order", backref="user", lazy=True)

    def __str__(self):
        return f"name:{self.name},age:{self.age},sex:{self.sex}"


class Order(db.Model):
    # 表名
    __tablename__ = 'tb_order'
    # 定义表字段
    order_id = db.Column(db.Integer, primary_key=True, autoincrement=True)
    goods_name = db.Column(db.String(30), unique=True)
    goods_price = db.db.Column(db.Float, default=12345.5, nullable=False)
    buy_date = db.Column(db.Integer, nullable=False)  # int 日期存时间戳
    user_id = db.Column(db.Integer, db.ForeignKey(User.id))

    def __str__(self):
        return f"order_id:{self.order_id},goods_name:{self.goods_name},buy_date:{self.buy_date}"

```

```bash
# 执行构建数据库表的命令
# 如果是第一次执行,需要先生成迁移文件  flask db init
flask db migrate
flask db upgrade 
```



##### 2.5.3.2 crud

```python
@order_blue.route("/")
def query_all():
    o1 = Order.query.filter().all()
    print(o1)
    return "success"


@order_blue.route("/add_more")
def add_more():
    orders = list()
    for i in range(3):
        order = Order()
        order.goods_name = f"phone_{i}"
        order.goods_price = float(i)
        order.buy_date = int(time.time())
        order.user_id = 9
        orders.append(order)

    for i in range(4, 8):
        order = Order()
        order.goods_name = f"phone_{i}"
        order.goods_price = float(i)
        order.buy_date = int(time.time())
        order.user_id = 10

        orders.append(order)

    for i in range(9, 13):
        order = Order()
        order.goods_name = f"phone_{i}"
        order.goods_price = float(i)
        order.buy_date = int(time.time())
        order.user_id = 11

        orders.append(order)

    db.session.add_all(orders)
    db.session.commit()

    return "success"


@order_blue.route("/update")
def update():
    order = Order.query.filter(Order.goods_name.like("%phone%", Order.user_id == 9)).first()

    if order:
        order.goods_name = "chinese_phone"
        order.goods_price = 100.11
        order.user_id = 10

        db.session.commit()
    return "success"


@order_blue.route("/delete")
def delete_order():
    order = Order.query.first()
    if order:
        db.session.delete(order)
        db.session.commit()
    return "success"


@order_blue.route("/query")
def query_order():
    # 查询用户对应的所有订单信息
    user = User.query.first()
    for order in user.orders:
        print(order)

    # 返回某个订单对应的用户信息
    order = Order.query.first()
    """
    # 因为 Order与User有外键关联,可以直接通过Order对象.user
    # 在User类中定义了:  orders = db.relationship("Order", backref="user", lazy=True)
    # backref="user" ->> order.user  backref="xxx" -> oder.xxx
    """
    print(order.user)

    return "success"
```



#### 2.5.4 多对多CRUD

##### 2.5.4.1 增加多对多模型类

```python
"""
配置模型类
"""

from . import db

# 多对多的关系中,建立的中间表
# "collect", 创建中间表的表名
# db.Column("user_id" ,...) user_id指的是 中间表的表名 
# db.ForeignKey("tb_user.id") tb_user指的是User类中的 __tablename__的值,id指的是User.id属性
collect = db.Table(
    "collect",
    db.Column("user_id", db.Integer, db.ForeignKey("tb_user.id"), primary_key=True),
    db.Column("movie_id", db.Integer, db.ForeignKey("tb_movie.id"), primary_key=True),
)


class User(db.Model):
    """
    db.Column：表示字段 db.Integer：表示整数
    primary_key=True ： 主键
    autoincrement=True： 自动递增
    db.String（30）：varchar（30）可变字符中
    unique=True ：唯一约束
    index=True ： 普通索引
    default : 默认值
    nullable: True 允许为空
    """
    # 表名
    __tablename__ = 'tb_user'
    # 定义表字段
    id = db.Column(db.Integer, primary_key=True, autoincrement=True)
    name = db.Column(db.String(30), unique=True, index=True)
    age = db.Column(db.Integer, default=18)
    sex = db.Column(db.Boolean, default=True)
    salary = db.Column(db.Float, default=12345.5, nullable=True)

    # 建立关系映射,
    # orders不是属于数据库表中的字段,而是单纯的一个类属性,是为了方便查询操作
    # db.relationship("Order",...) "Order" 表示建立关联类的名字
    # backref="user", 相当于给order对象新增一个属性 user
    # 拿到 order对象后,可以直接通过 order.user 获取order对应的 User对象
    # lazy=True 表示懒加载 ,当调用  orders.user 才会真正的建立关联关系
    orders = db.relationship("Order", backref="user", lazy=True)

    def __str__(self):
        return f"name:{self.name},age:{self.age},sex:{self.sex}"


class Movie(db.Model):
    __tablename__ = 'tb_movie'
    id = db.Column(db.Integer, primary_key=True, autoincrement=True)
    name = db.Column(db.String(30), unique=True)

    """
    lazy属性： 懒加载，可以延迟在使用关联属性的时候才建立关联
    lazy='dynamic'：会返回一个query 矿象（查询集），可以继续使用其他查询方法，如aLL（）
    lazy='select'： 首次访问到属性的时候，就会全部加载该属性的数据。 
    lazy='joined'：在矿关联的两个表进行join操作，从而获取到所有相关的矿象 
    lazy=True： 返回一个可用的列表矿象，同select
    """

    # 两张表多对多的关联关系 中 db.relationship(...) 只需要在其中一个表中增加这行代码就ok了
    # backref="movies" 这个属性是给 User类用的,用于直接 User类的实例对象.movies
    # Movie().users 可以获取到 所有关联的user对象
    # User().movies 可以获取到 所有关联的 movie对象
    # secondary=collect 指明中间表是collect ,最上面中间表已经通过 db.Table(...)创建出来了
    users = db.relationship("User", backref="movies", lazy=True, secondary=collect)

    def __str__(self):
        return f"movie_name:{self.name}"


class Order(db.Model):
    # 表名
    __tablename__ = 'tb_order'
    # 定义表字段
    order_id = db.Column(db.Integer, primary_key=True, autoincrement=True)
    goods_name = db.Column(db.String(30), unique=True)
    goods_price = db.db.Column(db.Float, default=12345.5, nullable=False)
    buy_date = db.Column(db.Integer, nullable=False)  # int 日期存时间戳
    user_id = db.Column(db.Integer, db.ForeignKey(User.id))

    def __str__(self):
        return f"order_id:{self.order_id},goods_name:{self.goods_name},buy_date:{self.buy_date}"

```



##### 2.5.4.2 CRUD

```python
@movie_blue.route("/add_more")
def add_more():
    movies = list()
    for i in range(10):
        movie = Movie()
        movie.name = f"朝花夕拾{i}"
        movies.append(movie)
    db.session.add_all(movies)
    db.session.commit()
    return "success"


@movie_blue.route("/query")
def query():
    # 查询一部电影有多少用户收藏
    movie = Movie.query.filter().first()
    print(movie.users)

    # 查询一个用户收藏了多少部电影
    user = User.query.filter().first()
    print(user.movies)
    return "success"


@movie_blue.route("/del")
def delete():
    # 级联删除,删除指定用户,会删除中间表跟这个用户相关的数据
    user = User.query.filter().first()
    db.session.delete(user)
    db.session.commit()
    return "success"
```







---





## 3.tip

### 3.1 模块拆分

```python
"""
本示例程序启动的流程分析
1.from App import create_app,这句话会先执行 App模块下的 __init__.py

2.执行 __init__.py的代码
from .views import user_blue 从 __init__.py的统计目录下执行 user_blue 模块
拿到就能从 user模块中拿到蓝图对象和 交给蓝图管理的路由信息

3.在执行视图模块 views中, 会执行 from .models import * 会导入同级目录下的 models.py下的所有模型类

4.__init__.py文件执行 from .views import user_blue 这句话后,就已经拿到了了蓝图对象,
蓝图管理的信息,模型类对象

5.__init__.py def create_app()... 整个执行完成后,就能拿到 创建 app实例对象的这个方法的内存地址

6.从主程序 create_app() 创建一个 app,就能得到一个 flask 的实例对象

7.最后全局的app实例对象就只创建了一次,模块之间也不存在循环依赖的问题
"""
```



![image-20240428145839295](./assets/image-20240428145839295.png)

#### 3.1.1 启动模块 app.py

```python
from App import create_app

app = create_app()


def main():
    app.run(host="0.0.0.0", port=80, debug=True)


if __name__ == '__main__':
    main()
```



#### 3.1.2 App模块 __init__.py

```python
"""
初始化操作,创建 flask应用
"""

from flask import Flask
from .views import user


def create_app():
    app = Flask(
        __name__,
        static_url_path="/static",
        static_folder="static",
        template_folder="templates"
    )
    # 注册蓝图
    app.register_blueprint(blueprint=user)

    return app

```



#### 3.1.3 视图函数模块 views.py

```python
"""
配置路由和视图函数
"""
from flask import Blueprint
from .models import *

# 创建蓝图对象
user = Blueprint("user", __name__)


@user.route("/")  # 将路由信息添加到蓝图中
def index():
    return "hello index"

```



#### 3.1.4 模型类模块 models.py

```python
class AA():
    pass
```





### 3.2 插件

#### 3.2.1 flask-caching

` pip install flask-caching `

```python
"""
这个模块专门用于导入插件
"""

from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate
from flask_caching import Cache

db = SQLAlchemy()
migrate = Migrate()
cache = Cache(config={
    "CACHE_TYPE": "simple"  # 指定缓存数据的类型,可以保存到内存,redis...
})


def init_extends(app):
    """
    将插件与 全局Flask()实例对象绑定
    :param app: 全局flask实例对象
    :return: 全局flask实例对象
    """
    db.init_app(app)
    migrate.init_app(app=app, db=db) 
    cache.init_app(app=app)

```

```python
@movie_blue.route("query_all")
@cache.cached(timeout=10)  # 10秒后缓存的内容才会失效
def query_all():
    movies = Movie.query.filter().all()
    print(movies)
    time.sleep(3)  # 模拟查询数据库的时间为3 秒
    return "success"
```



#### 3.2.2 flask-mail & flask-redis

`pip install flask-mail `

##### 3.2.2.1 邮箱配置类

```python
class QQMailSettings(object):
    """
    QQ邮箱配置
    """
    MAIL_SERVER = 'smtp.qq.com'
    MAIL_USE_SSL = True
    MAIL_PORT = 465
    MAIL_USERNAME = "xxxx-xxx@foxmail.com"
    MAIL_PASSWORD = "xxxxxx"
    MAIL_DEFAULT_SENDER = "xxxx-xxx@foxmail.com"

    
class RedisSettings(object):
    # root 表示redis数据库的密码
    REDIS_URL = "redis://:root@192.168.10.254:6379/0"    
```

##### 3.2.2.2 加载插件

```python
"""
这个模块专门用于导入插件
"""

from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate
from flask_caching import Cache
from flask_mail import Mail
from flask_redis import FlaskRedis

db = SQLAlchemy()
migrate = Migrate()
cache = Cache(config={
    "CACHE_TYPE": "simple"  # 指定缓存数据的类型,可以保存到内存,redis...
})
mail = Mail()
redis = FlaskRedis()


def init_extends(app):
    """
    将插件与 全局Flask()实例对象绑定
    :param app: 全局flask实例对象
    :return: 全局flask实例对象
    """
    db.init_app(app)
    migrate.init_app(app=app, db=db)
    cache.init_app(app=app)
    mail.init_app(app=app)
    redis.init_app(app=app)

```

##### 3.2.2.3 将配置添加到 app中

```python
"""
初始化操作,创建 flask应用
"""

from flask import Flask
from .config import SqliteSettings, MysqlSettings, QQMailSettings, RedisSettings, init_extends, db, migrate

# .views模块中,从当前的模块 __init__.py 中导入了 db,
# 所以当前的(from .views import user_blue)导入操作只能放到导入db操作的下面,避免循环导入
from .views import user_blue, order_blue, movie_blue, pay_blue, mail_blue


def create_app():
    app = Flask(
        __name__,
        static_url_path="/static",
        static_folder="static",
        template_folder="templates"
    )
    # 注册蓝图
    app.register_blueprint(blueprint=user_blue, url_prefix="/user")
    app.register_blueprint(blueprint=order_blue, url_prefix="/order")
    app.register_blueprint(blueprint=movie_blue, url_prefix="/movie")
    app.register_blueprint(blueprint=pay_blue, url_prefix="/pay")
    app.register_blueprint(blueprint=mail_blue, url_prefix="/mail")

    # 添加 sqlite  数据库的配置信息
    # app.config.from_object(SqliteSettings)

    # 添加 mysql 数据库的配置信息
    app.config.from_object(MysqlSettings)

    # 邮件配置
    app.config.from_object(QQMailSettings)

    # redis 配置
    app.config.from_object(RedisSettings)

    # 将插件与app对象绑定
    init_extends(app)

    return app

```

##### 3.2.2.4 邮件发送测试

```python
import time

from flask import Blueprint, request, abort, current_app, url_for
from ..config import cache, mail, redis
from flask_mail import Message
from ..tools import get_random_int

mail_blue = Blueprint("mail", __name__)


@mail_blue.before_request
def verify_send_mail_counts():
    if request.path == url_for("{}.sendmail_for_register".format(mail_blue.name)):
        ip = request.remote_addr
        if cache.get(ip):
            abort(429)
        else:
            # 设置60s缓存,防止同一IP的用户重复发送邮件
            cache.set(ip, "None", timeout=60)


@mail_blue.route("/register", methods=["post"])
def register():
    # 一般需要校验用户名是否存在于数据库,为了简化处理流程,默认用户不存在
    username = request.form.get("username", None)
    password = request.form.get("password", None)
    email = request.form.get("email", None)
    submit_verify_code = request.form.get("verify_code", None)

    if all([username, password, email, submit_verify_code]):
        verify_code = redis.get(f"register_{username}")
        if verify_code.decode() == submit_verify_code:
            print("邮箱验证码校验成功")
            # ...  往数据库写入用户注册成功的信息

            # 注册成功后,删除保存在redis的验证码信息
            redis.delete(f"register_{username}")
            return "register success"
        else:
            print("邮箱验证失败,注册失败")
            return "register error"
    return "所有参数不能为空"


@mail_blue.route("/send_mail")
def sendmail_for_register():
    """
    构建邮件信息对象
    :param subject: 邮件主题
    :param email: 邮件接受者 ["t1@outlook.com","a2@gamil.com"]
    :param body: 邮件主题信息
    :param html: 邮件主体
    :return:
    """

    username = request.args.get("username")
    email = request.args.get("email")
    if all([username, email]):
        subject = "用户注册"
        recipients = [email]
        message = Message(subject=subject, recipients=recipients)
        message.body = "用户注册"

        register_verify_code = get_random_int()
        redis.set(f"register_{username}", register_verify_code, ex=600)
        message.html = f"你好:{username},你的注册验证码是:{register_verify_code}"

        with current_app.app_context():
            try:
                mail.send(message)
                return "邮件发送成功"
            except Exception as e:
                print(e)
                return "邮件发送失败"
    else:
        return "用户名,邮箱不能为空"


@mail_blue.route("/forget_password")
def sendmail_for_forget_password():
    # 1.检验用户名,邮箱是否匹配.. 省略...
    username = request.args.get("username")
    email = request.args.get("email")

    if all([username, email]):
        subject = "忘记密码"
        recipients = [email]
        message = Message(subject=subject, recipients=recipients)
        message.body = "忘记密码"

        # 生成随机token
        import hashlib
        md5_obj1 = hashlib.md5()
        md5_obj1.update(f"{username}".encode("utf-8"))
        md5_obj1.update(f"{str(time.time())}".encode("utf-8"))
        md5_obj1.update(f"chinaIsNo1".encode("utf-8"))
        token = md5_obj1.hexdigest()

        redis.set(f"forget_password_{username}", token, ex=600)

        url = f"http://192.168.10.100:8080/mail/update_password?username={username}&&token={token}"
        print(url)
        message.html = f"<a href='{url}'>点击此处,修改密码</a>"

        with current_app.app_context():
            try:
                mail.send(message)
                return "邮件发送成功"
            except Exception as e:
                print(e)
                return "邮件发送失败"


@mail_blue.route("/update_password", methods=["get"])
def update_password():
    username = request.args.get("username")
    submit_token = request.args.get("token")

    if all([username, submit_token]):
        token = redis.get(f"forget_password_{username}")
        if submit_token == token.decode():
            print("token校验成功,执行修改密码操作...")

            # 修改密码成功后,删除保存在redis的token信息
            redis.delete(f"forget_password_{username}")
            return "修改密码成功"
        else:
            print("token校验失败...")
            return "token校验失败"
    return "修改密码失败"

```







### 3.3 利用钩子函数反爬

```python
@movie_blue.before_request
def test_spider():
    """
    基于钩子函数实现简单的反爬效果
    """

    # 检测UA
    import re
    if re.search(r"python", request.user_agent.string, re.I):
        abort(401)

    ip = request.remote_addr
    if cache.get(ip):
        abort(429)
    else:
        # 设置缓存,同一IP 3 秒内访问缓存的数据
        cache.set(ip, "None", timeout=3)


@movie_blue.errorhandler(401)
def handler_error_page(error_info):
    # error_info 表示是错误信息,flask调用这个方法时会自动将错误信息对象 传递过来
    # abort(401) 的时候,下面的信息会直接返回给前端
    return f"你已经被禁止访问"


@movie_blue.route("/")
def movie_index():
    return "hello movie"
```



### 4. 支付宝支付demo

#### 4.1 申请沙箱环境测试账号

申请地址:https://open.alipay.com/develop/sandbox/app

![image-20240430191236469](./assets/image-20240430191236469.png)

![image-20240430191416583](./assets/image-20240430191416583.png)

![image-20240430191502620](./assets/image-20240430191502620.png)



#### 4.2 支付类编写

`pip install python-alipay-sdk`

```python
from alipay import AliPay


class APay(object):
    def __init__(self):
        self.app_id = '9021000136666182'
        self.pid = '2088721034695771'
        self.public_key = open('./App/config/keys/alipay_public_key.pem').read()
        self.private_key = open('./App/config/keys/app_private_key.pem').read()

        self.pay = AliPay(
            appid=self.app_id,
            # 支付宝回调我们自己系统的url 等到部署到服务器配置就好了
            app_notify_url="https://frp.mshare.top/pay/result",
            # 私钥
            app_private_key_string=self.private_key,
            # 公钥
            alipay_public_key_string=self.public_key,
            # 开启debug(如果是部署好的项目要关掉)
            debug=True,
        )

    def create_order(self, amount: str, order_on: str):
        # 订单支付的信息(自己系统的订单信息)
        # 订单金额
        amount = amount
        # 支付页面展示的标题
        subject = f"商城订单{order_on}支付"

        url = self.pay.api_alipay_trade_page_pay(
            # 支付页面的标题
            subject=subject,
            # 商户生成的订单号(自己系统中的订单号)
            out_trade_no=order_on,
            # 订单支付的金额
            total_amount=amount,
            # 部署好项目以后在配置
            return_url="https://frp.mshare.top/pay/index",  # 支付完后后用户需要返回的页面
            notify_url="https://frp.mshare.top/pay/result",  # 支付结果的通知请求的地址
        )

        pay_url = 'https://openapi-sandbox.dl.alipaydev.com/gateway.do?' + url
        return pay_url


pay_obj = APay()

```



#### 4.3 支付测试

```python
import time

from flask import Blueprint, redirect, url_for, request
from ..tools import pay_obj

pay_blue = Blueprint("pay", __name__)


@pay_blue.route("/")
def to_pay():
    # 跳转到支付界面
    url = url_for("{}.test_pay".format(pay_blue.name))
    print(url)
    return redirect(url)


@pay_blue.route("/test")
def test_pay():
    order_on = str(int(time.time()))
    print("生成的订单编号是:" + order_on)
    url = pay_obj.create_order("0.33", order_on)
    return "<a href='{}'>支付链接</a>".format(url)


@pay_blue.route("/result", methods=["post"])
def pay_result():
    """
    这个视图函数专门用于支付完成后,对支付结果的处理
    方便商户更新数据库的订单状态
    :return:
    """
    # 获取支付完成后回调的结果
    result_dict = request.form.to_dict()

    # 获取支付结果的令牌
    sign = result_dict.get("sign", None)
    # 需要去除字典里面 sign字段才能检验
    result_dict.pop("sign")

    # 获取本次支付的订单号(商家生成的订单号,不是支付订单号)
    # 根据支付完成的状态,完成后续对订单数据库的处理
    out_trade_no = result_dict.get("out_trade_no", None)

    # pay_obj.pay.verify(...) 这个方法专门用于检验订单状态
    pay_status = pay_obj.pay.verify(result_dict, sign)
    if pay_status:
        # 支付成功,下面可以根据需求更新数据库订单信息
        print(f"success:{out_trade_no}")
        return "pay success"
    else:
        # 支付失败,下面可以根据需求更新数据库订单信息
        print(f"error:{out_trade_no}")
        return "pay error"


@pay_blue.route("/index", methods=["get"])
def index():
    """
    这个函数用于接收支付完成后,通知用户跳转到哪个页面的
    根据支付的状态,可以对用户完成不同页面的跳转
    :return:
    """
    param_dict = request.args.to_dict()
    sign = param_dict.get("sign", None)
    print(sign)
    if sign:
        # 需要去除字典里面 sign 字段才能检验订单状态
        param_dict.pop("sign")
        status = pay_obj.pay.verify(param_dict, sign)
        if status:
            # 需要返回一个数据给发送方(支付宝),通知它 我们已经处理了这个订单的状态
            # 如果没有及时回复,或者服务器这个时候宕机,导致我们没有回复支付宝这条通知信息,
            # 24小时内,支付宝会有间隔的持续发送这条订单状态信息给这个视图函数
            return "支付成功"
        else:
            return "支付失败"
    return "支付错误"

```

