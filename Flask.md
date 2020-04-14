# Flask

小而精，三方组件全，内置seesion

稳定性相对较差



```python
from flask import Flask
app = Flask(__name__)
app.run()
```

```python
from flask import Flask
app = Flask(__name__)
@app.route("/")
def index():
    return "Hello World"
app.run()
```

## Response 三剑客

1. HttpResponse

```Python
# 返回字符串至客户端
return "Hello World" 
```

2. render_template

```Python
# 与Django中的render 使用一致  返回模板由浏览器渲染
from flask import render_template
render : return render_template('login.html')
```

3. redirect

```Python
# 跳转，重定向URL 
from flask import redirect
def index():
    return redirect('/login')  #302
```

## Flask 中小儿子

1. jsonify

```python
from flask import jsonify
def index():
    return jsonify({name:'li'})  # 返回json标准的字符串 Content-Type:application/json
```

2. send_file

```python
# 打开文件并返回文件内容(自动识别文件格式)
from flask import send_file
def index():
    return send_file(path)
```

## Flask 中的request

- get
- post
- delete
- put

```python
from flask import request

request.method #请求方式
request.form #存放FormData中的数据 to_dict 序列化成字典
request.args # 获取URL中的数据 to_dict 序列化成字典
request.url # 访问的完整路径
request.path # 路由地址
request.host # 主机地址
request.values #获取FormData and URL中的数据  不要用to_dict
request.json # 如果提交时请求头中的Content-Type:application/json 字典操作
request.data # 如果提交时请求头中的Content-Type 无法被识别 将请求体中的原始数据存放 bytes
request.cookies #获取Cookie中的数据
request.headers # 获取请求头
request.files # 序列化文件存储  save(path) file_name
```




## Jinja2

{{}} 引用变量 执行函数

{%%} 逻辑代码

| safe  Markup  安全标签字符串

@app.template_global()

@app.template_filter()

{% macro create_input(na,ty) %}

{{ na }} : <input type="{{ ty }}" name="{{ na }}">

{% endmacro %}

{{ create_input("username","text")}}

## Session

app.secret_key = "加密字符串"  #用于序列化和反序列化 session信息

由于Flask中默认Session存放位置---客户端的Cookies中

所以Session需要加密 用到secret_key

请求进入视图函数 带上cookie 将Session从cookie序列化出来通过secret_key 反序列化字典

```Python
session['user']=user

if session.get('user'):
    pass
```

## 装饰器

```python
def debug(func):
    @functools,wraps(func)
    def wrapper(something):  # 指定一毛一样的参数
        print "[DEBUG]: enter {}()".format(func.__name__)
        return func(something)
    return wrapper  # 返回包装过函数

@debug
def say(something):
    print "hello {}!".format(something)
```

## 路由

1. endpoint 反向生成url地址标志 默认视图函数名 url_for(函数名)
2. methods=["GET","POST"] 视图函数请求方式
3. defaults={'id':'123'}  默认参数
4. struct_slashes=True 严格遵守路由地址
5. redirect_to 永久重定向 301 (地址更换之后可考虑使用)
6. 动态路由参数

```Python
@app.route("/index/<page>")
def index(page):
    pass
```

## 实例化配置

1. 指定模板路径

```Python
app= Flask(__name__,
           template_folder="template", # 默认模板路径
           static_folder="statics", # 默认静态文件路径 statics
           static_url_path='/statics' #访问静态文件路由地址 默认是"/" + static_folder
          )  
```

2. static_host =None  指定静态文件服务器地址


3. host_matching = False,  # 如果不是特别需要的话,慎用,否则所有的route 都需要host=""的参数
4. subdomain_matching = False,  # 理论上来说是用来限制SERVER_NAME子域名的,但是目前还没有感觉出来区别在哪里
5. template_folder = 'templates'  # template模板目录, 默认当前项目中的 templates 目录
6. instance_path = None,  # 指向另一个Flask实例的路径
7. instance_relative_config = False  # 是否加载另一个实例的配置
8. root_path = None  # 主模块所在的目录的绝对路径,默认项目目录

## 对象配置

```python
# FlaskSetting.py
class FlaskDebug(object):
    DEBUG=True
    SECRET_KEY=""
    PERMANENT_SESSION_LIFETIME=7
    SESSION_COOKIE_NAME="debug_session"
    
class FlaskTesting(object):
    TESTING=True
    SECRET_KEY=""
    PERMANENT_SESSION_LIFETIME=14
    SESSION_COOKIE_NAME="testing_session"
    
# app.py
from FlaskSetting import FlaskDebug,FlaskTesting
app.config.from_object(FlaskDebug)
```


```Python
{
    'DEBUG': False,  # 是否开启Debug模式
    'TESTING': False,  # 是否开启测试模式
    'PROPAGATE_EXCEPTIONS': None,  # 异常传播(是否在控制台打印LOG) 当Debug或者testing开启后,自动为True
    'PRESERVE_CONTEXT_ON_EXCEPTION': None,  # 一两句话说不清楚,一般不用它
    'SECRET_KEY': None,  # 之前遇到过,在启用Session的时候,一定要有它
    'PERMANENT_SESSION_LIFETIME': 31,  # days , Session的生命周期(天)默认31天
    'USE_X_SENDFILE': False,  # 是否弃用 x_sendfile
    'LOGGER_NAME': None,  # 日志记录器的名称
    'LOGGER_HANDLER_POLICY': 'always',
    'SERVER_NAME': None,  # 服务访问域名
    'APPLICATION_ROOT': None,  # 项目的完整路径
    'SESSION_COOKIE_NAME': 'session',  # 在cookies中存放session加密字符串的名字
    'SESSION_COOKIE_DOMAIN': None,  # 在哪个域名下会产生session记录在cookies中
    'SESSION_COOKIE_PATH': None,  # cookies的路径
    'SESSION_COOKIE_HTTPONLY': True,  # 控制 cookie 是否应被设置 httponly 的标志，
    'SESSION_COOKIE_SECURE': False,  # 控制 cookie 是否应被设置安全标志
    'SESSION_REFRESH_EACH_REQUEST': True,  # 这个标志控制永久会话如何刷新
    'MAX_CONTENT_LENGTH': None,  # 如果设置为字节数， Flask 会拒绝内容长度大于此值的请求进入，并返回一个 413 状态码
    'SEND_FILE_MAX_AGE_DEFAULT': 12,  # hours 默认缓存控制的最大期限
    'TRAP_BAD_REQUEST_ERRORS': False,
    # 如果这个值被设置为 True ，Flask不会执行 HTTP 异常的错误处理，而是像对待其它异常一样，
    # 通过异常栈让它冒泡地抛出。这对于需要找出 HTTP 异常源头的可怕调试情形是有用的。
    'TRAP_HTTP_EXCEPTIONS': False,
    # Werkzeug 处理请求中的特定数据的内部数据结构会抛出同样也是“错误的请求”异常的特殊的 key errors 。
    # 同样地，为了保持一致，许多操作可以显式地抛出 BadRequest 异常。
    # 因为在调试中，你希望准确地找出异常的原因，这个设置用于在这些情形下调试。
    # 如果这个值被设置为 True ，你只会得到常规的回溯。
    'EXPLAIN_TEMPLATE_LOADING': False,
    'PREFERRED_URL_SCHEME': 'http',  # 生成URL的时候如果没有可用的 URL 模式话将使用这个值
    'JSON_AS_ASCII': True,
    # 默认情况下 Flask 使用 ascii 编码来序列化对象。如果这个值被设置为 False ，
    # Flask不会将其编码为 ASCII，并且按原样输出，返回它的 unicode 字符串。
    # 比如 jsonfiy 会自动地采用 utf-8 来编码它然后才进行传输。
    'JSON_SORT_KEYS': True,
    #默认情况下 Flask 按照 JSON 对象的键的顺序来序来序列化它。
    # 这样做是为了确保键的顺序不会受到字典的哈希种子的影响，从而返回的值每次都是一致的，不会造成无用的额外 HTTP 缓存。
    # 你可以通过修改这个配置的值来覆盖默认的操作。但这是不被推荐的做法因为这个默认的行为可能会给你在性能的代价上带来改善。
    'JSONIFY_PRETTYPRINT_REGULAR': True,
    'JSONIFY_MIMETYPE': 'application/json',
    'TEMPLATES_AUTO_RELOAD': None,
}
```

## flash

```python
# 存数据
flash("string",category="tag")
# 取数据
get_flashed_messages(category_filter="tag")
```

## Flask 蓝图(Blueprint)

Blueprint 当成一个不能被启动的app Flask实例

```Python
from flask import Blueprint,render_template
blue_app =Blueprint("buleapp",__name__,template_folder='apptemp',url_prefix='/blue')

@blue_app.route('/index')
def index():
    return render_template('blueapp.html')

app.register_blueprint(views.blue_app,url_prefix='/blue')
# url_prefix 为url前缀 优先register
```

## Flask特殊装饰器

```Python
@app.before_request #请求进入视图函数之前
@app.after_request #在相应客户端之前
# 正常情况下流程 ： be1 - be2 -be3 -af3 - af2 -af1
# 异常情况下流程：be1 - af3 - af2 - af1

@app.errorhandler(404)
def error404(error_info):
    return error_info
```

















