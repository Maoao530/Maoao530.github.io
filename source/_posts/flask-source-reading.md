---
title: flask源码阅读
date: 2016-10-24 20:26:04
categories:
- Python进阶
tags:
- Flask
---


Flask是一个使用 Python 编写的轻量级 Web 应用框架。其 WSGI 工具箱采用 Werkzeug ，模板引擎则使用 Jinja2 。

<!-- more -->

我们先从flask 0.1版本阅读起。

# 1 安装Flask 0.1
因为flask 0.1暂时不支持python3，所以我们使用python2.7版本：

```python
py -2 -m pip install flask==0.1
```

# 2 Hello World

这是一个很简单的示例，编写hello.py，参考官方网站的示例：

```python
from flask import Flask
app = Flask(__name__)
@app.route("/")
def hello():    
    return "Hello World!"
 
if __name__ == "__main__":
    app.run()
```
运行hello.py后，打开浏览器访问 localhost:5000可以看到浏览器输出Hello World!。

# 3 知识准备

在对flask有了一个比较简单的认识之后，我们知道，flask是基于Jinja2和Werkzeug的一个框架，其WSGI工具箱采用Werkzeug，模板引擎则使用 Jinja2。

# 3.1 WSGI

WSGI是Web Server Gateway Interface的缩写，是为Python语言定义的Web服务器和Web应用程序或框架之间的一种简单而通用的接口。引用一张图说明WSGI的位置：

![]()

WSGI接口定义非常简单，它只要求Web开发者实现一个函数，就可以响应HTTP请求。我们来看一个hello world：

```python
# args:
#  - environ：一个包含所有HTTP请求信息的dict对象；
#  - start_response：一个发送HTTP响应的函数。
#
def application(environ, start_response):
    start_response('200 OK', [('Content-Type', 'text/html')])
    return [b'<h1>Hello, web!</h1>']
```

1. 上面的application()函数就是符合WSGI标准的一个HTTP处理函数，函数必须由WSGI服务器来调用。
2. environ是一个字典，该字典可以包含了客户端请求的信息以及其他信息，可以认为是请求上下文，一般叫做environment（编码中多简写为environ、env）
3. start_response返回了http响应的header，Header只能发送一次，也就是这个函数只能调用一次。
有很多符合WSGI规范的服务器，我们可以挑选一个来用。比如：

```python
from eventlet import wsgi
import eventlet
def hello_world(environ, start_response):
    start_response('200 OK', [('Content-Type', 'text/plain')])
    return ['Hello, World!\r\n']
wsgi.server(eventlet.listen(('', 8090)), hello_world)
```

3.2 Jinja2

Jinja2是一个模板引擎，jinja2内部使用Unicode，一个比较简单的示例如下：

```python
from jinja2 import Template
template = Template('Hello {{ name }}!')
s = template.render(name='Shun')
print(s)  # Hello Shun!
```

或者

```python
from jinja2 import Environment, PackageLoader
env = Environment(loader = PackageLoader('module_name', 'templates_dir'))
template2 = env.get_template('my_template.html') # 'Hello {{ name }}!'
s = template2.render(name = 'shun')  # Hello shun!
```

更高级的用法可以参考官方文档 : [http://docs.jinkan.org/docs/jinja2/](http://docs.jinkan.org/docs/jinja2/)

# 3.3 Werkzeug

Werkzeug是一个WSGI工具包，官网将其描述为：The Python WSGI Utility Library。
一个简单的例子实现一个小的 Hello World 应用。显示用户输入的名字:

```python
from werkzeug.utils import escape
from werkzeug.wrappers import Request, Response
@Request.application
def hello_world(request):
    result = ['<title>Greeter</title>']
    if request.method == 'POST':
        result.append('<h1>Hello %s!</h1>' % escape(request.form['name']))
    result.append('''
        <form action="" method="post">
            <p>Name: <input type="text" name="name" size="20">
            <input type="submit" value="Greet me">
        </form>
    ''')
    return Response(''.join(result), mimet ype='text/html')
if __name__ == '__main__':
    from werkzeug.serving import run_simple
    run_simple('localhost', 8080, hello_world)
```

另外不用 request 和 response 对象也可以实现这个功能，那就是借助 werkzeug 提供的 解析函数:

```python
from werkzeug.formparser import parse_form_data
from werkzeug.utils import escape
def hello_world(environ, start_response):
    result = ['<title>Greeter</title>']
    if environ['REQUEST_METHOD'] == 'POST':
        form = parse_form_data(environ)[1]
        result.append('<h1>Hello %s!</h1>' % escape(form['name']))
    result.append('''
        <form action="" method="post">
            <p>Name: <input type="text" name="name" size="20">
            <input type="submit" value="Greet me">
        </form>
    ''')
    start_response('200 OK', [('Content-Type', 'text/html; charset=utf-8')])
    return [''.join(result)]
```

通常我们更倾向于使用高级的 API(request 和 response 对象)。但是也有些情况你可能更 想使用低级功能。

# 4 Flask-0.1源码分析

有了上面的基础之后，我们可以开始分析Flask的源码了。完整的源码可以参考这里：Flask.py
首先我们再来回顾一下Hello World是怎么样的：

```python
from flask import Flask
app = Flask(__name__)  
@app.route("/")
def hello():    
    return "Hello World!"

if __name__ == "__main__":
    app.run()
```

先初始化一个Flask的类，然后路由hello方法，最后再run起来。

## 4.1 Flask init()

我们来分析一下Flask类的初始化函数：

```python
class Flask(object):
    request_class = Request
    response_class = Response
    static_path = '/static'
    secret_key = None
    session_cookie_name = 'session'
    # 模板参数
    jinja_options = dict(    
        autoescape=True,
        extensions=['jinja2.ext.autoescape', 'jinja2.ext.with_']
    )
    def __init__(self, package_name):
        self.debug = False                  # 如果设置为True，则改完code后，服务器会自动部署（reload）  
        self.package_name = package_name    # __main__        
        self.root_path = _get_package_path(self.package_name)      
        self.view_functions = {}            # 保存url视图函数名到函数地址的映射    
        self.error_handlers = {}            # 保存错误函数，@app.errorhandler(404)...     
        self.before_request_funcs = []      # 保存request前执行的一些函数，比如打开数据库等     
        self.after_request_funcs = []       # 保存request后执行的一些函数
        
        self.template_context_processors = [_default_template_ctx_processor]
        
        self.url_map = Map()                
        
        # ???
        if self.static_path is not None:
            self.url_map.add(Rule(self.static_path + '/<filename>',
                                  build_only=True, endpoint='static'))
            if pkg_resources is not None:
                target = (self.package_name, 'static')
            else:
                target = os.path.join(self.root_path, 'static')
            self.wsgi_app = SharedDataMiddleware(self.wsgi_app, {
                self.static_path: target
            })    
        self.jinja_env = Environment(loader=self.create_jinja_loader(),
                                     **self.jinja_options)
        self.jinja_env.globals.update(
            url_for=url_for,
            get_flashed_messages=get_flashed_messages
        )
```

template_context_processors的内容涉及context，这个后面再讲。

self.url_map这个函数保存了URI（访问的后缀URL,比如/，/index/等）到视图函数字典的映射,我们可以看到路由这个函数是如何实现的：

```python
# rule : URI比如 '\'
# options : 这里是空的字典 {}，用于后面保存URI对应的访问规则，比如GET or POST，函数名等。
def route(self, rule, **options):
    # f指向实际执行的函数地址
    def decorator(f):
        self.add_url_rule(rule, f.__name__, **options)
        self.view_functions[f.__name__] = f
        return f
    return decorator
# rule : URI
# endpoint : 函数名字符串
def add_url_rule(self, rule, endpoint, **options):       
    options['endpoint'] = endpoint
    options.setdefault('methods', ('GET',))
    self.url_map.add(Rule(rule, **options))
```

假如我们写了：

```
app.route('/')
def hello():
    print('hello shun.')
```

相当于执行了:

```python
hello = app.route('/')(hello)
```

对装饰器不太熟悉的同学，可以参考我的另外一篇文章:[Python 装饰器decorator](http://maoao530.github.io/2016/05/17/python-decorator/)

## 4.2 wsgi_app

当我们执行app.run函数的时候,最终会执行到wsgi_app函数，这个函数是Flask的入口核心函数：

```python
app.run()
    run_simple(host, port, self, **options)
        __call__(self, environ, start_response)
            wsgi_app(self, environ, start_response)
```

最终会执行wsgi_app，这个函数是flask的入口，也是核心：

```python
def wsgi_app(self, environ, start_response):
    with self.request_context(environ):         # 创建request context
        rv = self.preprocess_request()          # 先调用预处理函数
        if rv is None:        
            rv = self.dispatch_request()        # 分发请求
        response = self.make_response(rv)       # 
        response = self.process_response(response)
        return response(environ, start_response)
```

### 4.2.1 生成request_context

我们一个一个来分析，先看是如何创建request context的：

```python
# Flask类函数
def request_context(self, environ):
        return _RequestContext(self, environ)
# _RequestContext类
class _RequestContext(object):
    def __init__(self, app, environ):
        self.app = app                                           # Flask app
        self.url_adapter = app.url_map.bind_to_environ(environ)  # 将ENV绑定到URL Adappter，可以参考werkzeug相关文档说明
        self.request = app.request_class(environ)                # app.request = Request类，其实就是把environ字典的一些信息封装为Request对象
        self.session = app.open_session(self.request)            # 从cookie中拿到sessionID，然后读取用户的session
        self.g = _RequestGlobals()                               
        self.flashes = None
    def __enter__(self):
        _request_ctx_stack.push(self)
    def __exit__(self, exc_type, exc_value, tb):
        if tb is None or not self.app.debug:
            _request_ctx_stack.pop()
```

URL Adappter更多，可以点击这里 : [Werkzeug 文档说明](http://werkzeug-docs-cn.readthedocs.io/zh_CN/latest/tutorial.html#step-4)

关于Session和Cookie，我这里多补充几句：

1. Session是在服务端保存的一个数据结构，用来对用户会话进行跟踪的一个机制，根据不同的Session ID来标识不同的用户，这个数据可以保存在集群、数据库、文件中。常见的使用场景，比如购物车等。
2. Cookie是客户端保存用户信息的一种机制，用来记录用户的一些信息。每次HTTP请求的时候，客户端都会发送相应的Cookie信息到服务端。
3. Session ID一般是存在Cookie中，所以如果浏览器禁用了Cookie，同时Session 也会失效（但是可以通过其它方式实现，比如在 url 中传递 session_id）

### 4.2.2 预处理preprocess_request

我们再来看看先调用预处理函数preprocess_request：

```python
def preprocess_request(self): 
    for func in self.before_request_funcs:  
        rv = func()
        if rv is not None:  
            # 如果这些函数中有返回值，则被视为来自试图的返回值，并停止其他函数处理
            # 预处理函数应当不需要有返回值的，这点需注意
            return rv
```

会遍历before_request_funcs这个列表的函数并且执行，这个列表函数通过装饰器@app.before_request来赋值：

```python
def before_request(self, f):
    """Registers a function to run before each request."""
    self.before_request_funcs.append(f)
    return f
```

### 4.2.3 分发请求dispatch_request()

简单一句话，这个函数的作用就是匹配到Request对应的URL和视图函数，并执行这个函数：

```python
def match_request(self):
    rv = _request_ctx_stack.top.url_adapter.match()  # self.url_adapter 上文有说明
    # endpoint 是指函数名
    # view_args 这里是一个dict，视图参数？？
    request.endpoint, request.view_args = rv         
    return rv
def dispatch_request(self):
    try:
        endpoint, values = self.match_request()         # 先匹配函数名和参数
        return self.view_functions[endpoint](**values)  # 匹配完后直接执行函数
    
    except HTTPException, e:                            # 匹配不到视图函数的处理
        handler = self.error_handlers.get(e.code)
        if handler is None:
            return e
        return handler(e)
    except Exception, e:
        handler = self.error_handlers.get(500)
        if self.debug or handler is None:
            raise
        return handler(e)
```

### 4.2.4 生成response对象

生成一个response对象，并返回：

```python
# ResponseBase由Werkzeug提供,
class Response(ResponseBase):
    default_mimetype = 'text/html'
def make_response(self, rv):       
    if isinstance(rv, self.response_class):
        return rv
    if isinstance(rv, basestring):
        return self.response_class(rv)
    if isinstance(rv, tuple):
        return self.response_class(*rv)
    return self.response_class.force_type(rv, request.environ)
```

如果不是response对象，则转化response对象，response对象继承自ResponseBase对象，其实就是Response对象。
若需要进一步了解，可以访问：http://werkzeug-docs-cn.readthedocs.io/zh_CN/latest/quickstart.html#response

### 4.2.5 self.process_response(response)

生成response对象后，保存session并执行一些request后的函数，最后再返回response对象：

```python
def process_response(self, response):   
    session = _request_ctx_stack.top.session  # 从RequestContext中拿到用户的session
    if session is not None:
        self.save_session(session, response)  # 保存session
    for handler in self.after_request_funcs:  
        response = handler(response)          # 执行装饰器 @app.after_request 标记的函数
    return response                           # 最后返回response对象
```

在所有都处理完成后，把需要返回的内容返回给客户端：

```python
return response(environ, start_response)
```

至此，flask 0.1主框架大致分析完成了。

# 5 Flask Context机制

细心的同学在上面应该注意到了，我们还有一个点没有讲，那就是Flask的Context机制。

Flask 中有分为请求上下文和应用上下文：

|对象         | Context类型     |  说明  |
| ------      | ------          | ------ |
|current_app  | AppContext      | 当前应用的对象|
|g            | AppContext      | 处理请求时用作临时存储的对象|
|request      | RequestContext  |  请求request对象|
|session      |RequestContext   |  请求的session对象|

拉到Flask.py最后面，我们可以看到有以下几行：

```python
_request_ctx_stack = LocalStack()
current_app = LocalProxy(lambda: _request_ctx_stack.top.app)
request = LocalProxy(lambda: _request_ctx_stack.top.request)
session = LocalProxy(lambda: _request_ctx_stack.top.session)
g = LocalProxy(lambda: _request_ctx_stack.top.g)
```

所有的对象都交由request_ctx_stack这个堆栈来管理了。

## 5.1 LocakStack()

LocalStack()会返回一个栈。栈肯定会有push 、pop和top函数，如下所示:

```python
class LocalStack(object):
    """
        >>> ls = LocalStack()
        >>> ls.push(42)
        >>> ls.top
        42
        >>> ls.push(23)
        >>> ls.top
        23
        >>> ls.pop()
        23
        >>> ls.top
        42
    """
    def __init__(self):
        self._local = Local()
    def push(self, obj):
        rv = getattr(self._local, 'stack', None)
        if rv is None:
            self._local.stack = rv = []
        rv.append(obj)
        return rv
    def pop(self):
        stack = getattr(self._local, 'stack', None)
        if stack is None:
            return None
        elif len(stack) == 1:
            release_local(self._local)
            return stack[-1]
        else:
            return stack.pop()
    @property
    def top(self):
        try:
            return self._local.stack[-1]
        except (AttributeError, IndexError):
            return None
```

按照我们的理解，要实现一个栈，那么LocalStack类应该有一个成员变量，是一个list，然后通过 这个list来保存栈的元素。然而，LocalStack并没有一个类型是list的成员变量， LocalStack仅有一个成员变量self._local = Local()。

我们再看push，pop，top函数可以知道，具体是通过self._local.stack这个来实现栈的操作。

## 5.2 Local()

当我们操作self._local.stack时，会调用Local()的getattr和setattr方法。

```python
class Local(object):
    __slots__ = ('__storage__', '__ident_func__')
    def __init__(self):
        object.__setattr__(self, '__storage__', {})
        object.__setattr__(self, '__ident_func__', get_ident)
    def __getattr__(self, name):
        try:
            return self.__storage__[self.__ident_func__()][name]
        except KeyError:
            raise AttributeError(name)
    def __setattr__(self, name, value):
        ident = self.__ident_func__()
        storage = self.__storage__
        try:
            storage[ident][name] = value
        except KeyError:
            storage[ident] = {name: value}
```

Local类有两个成员变量，分别是storage和ident_func，其中，前者 是一个字典，后者是一个函数。
这个函数的含义是，获取当前线程的id。

例如，当我们执行：

```python
request_ctx_stack = LocalStack()
_request_ctx_stack.push(RequestContext)
```

注意，这里赋值的时候，最终会调用：

```python
# name => stack
self.__storage__[self.__ident_func__()][name] = RequestContext
```

所以最终看起来会是这样一个数据结构：

```
{
    'thread_id1':{'stack':[_RequestContext()]},
    'thread_id2':{'stack':[_RequestContext()]}
}
```

至此，Flask0.1版本的源码已经大致分析完成，其实如果继续下去的话，还有很多值得深究的地方，待后续有时间继续深入分析。

