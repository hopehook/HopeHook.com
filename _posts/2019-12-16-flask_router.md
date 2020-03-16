---
date: 2019-12-16
layout: post
title: Flask 路由原理简介
thread: 2019-12-16-flask_router.md
categories: web
tags: 框架
---


#### 框架简介

Flask 是由 python 实现的一个 web 微框架, 有对应的 python3 及 python2 版本.
非常适用于小型网站, 非常适用于开发 web 服务的 API, 
开发大型网站无压力, 但代码架构需要自己设计, 开发成本取决于开发者的能力和经验.
Flask 自由、灵活, 可扩展性强, 第三方库的选择面广, 开发时可以结合自己最喜欢用的轮子, 也能结合最流行最强大的 Python 库.
个人也将 Flask 作为 web 开发的首选框架.


- hello world

这是一个 Flask 的 hello world 例子, 访问浏览器主页将输出 "hello world".

```
from flask import Flask

app = Flask(__name__)


@app.route('/', methods=['GET'])
def get_hello_world():
    return "hello world"


app.run(port=8080)
```

Flask 的 handler 以函数为单位进行组织, 通过 `@app.route` 装饰器把 handler 注册到路由表, `app.run(port=8080)` 监听本地 8080,
一个简单的 api 服务就搭建完成了.

如果要满足商业应用的开发, 可以根据实际需要, 选择 Flask 的已有组件, 或者其他丰富的 Python 库, 搭积木式开发模式让开发者得心应手.


#### 路由注册过程

- @app.route('/', methods=['GET'])

装饰器的定义:

```
    def route(self, rule, **options):
        def decorator(f):
            endpoint = options.pop('endpoint', None)
            self.add_url_rule(rule, endpoint, f, **options)
            return f
        return decorator
```

- add_url_rule

路由的注册是通过 add_url_rule 函数实现的, 最终注册在 app.view_functions 字典中. 
key 是 endpoint (默认为函数名), value 是 view_func (handler 函数引用).


```
    def add_url_rule(self, rule, endpoint=None, view_func=None, **options):
        if endpoint is None:
            endpoint = _endpoint_from_view_func(view_func)
        options['endpoint'] = endpoint
        methods = options.pop('methods', None)

        # if the methods are not given and the view_func object knows its
        # methods we can use that instead.  If neither exists, we go with
        # a tuple of only ``GET`` as default.
        if methods is None:
            methods = getattr(view_func, 'methods', None) or ('GET',)
        if isinstance(methods, string_types):
            raise TypeError('Allowed methods have to be iterables of strings, '
                            'for example: @app.route(..., methods=["POST"])')
        methods = set(item.upper() for item in methods)

        # Methods that should always be added
        required_methods = set(getattr(view_func, 'required_methods', ()))

        # Add the required methods now.
        methods |= required_methods
        rule = self.url_rule_class(rule, methods=methods, **options)

        self.url_map.add(rule)
        if view_func is not None:
            old_func = self.view_functions.get(endpoint)
            if old_func is not None and old_func != view_func:
                raise AssertionError('View function mapping is overwriting an '
                                     'existing endpoint function: %s' % endpoint)
            self.view_functions[endpoint] = view_func
```


- app.url_map.add(rule)

值得注意的是 url 与 view_func 并非是直接映射的关系, 中介桥梁是 endpoint, url -> endpoint -> view_func.
`rule = self.url_rule_class(rule, methods=methods, **options)` 将 rule 和 endpoint 绑定.

我们看看 `self.url_map.add(rule)` 的细节.


```
class Rule(RuleFactory):

    def __init__(self, string, defaults=None, subdomain=None, methods=None,
                 build_only=False, endpoint=None, strict_slashes=None,
                 redirect_to=None, alias=False, host=None):
        # url path
        self.rule = string
        
        # app.view_functions key
        self.endpoint = endpoint
        
        # methods: "GET", "POST" ...
        if methods is None: 
            self.methods = None
        else:
            if isinstance(methods, str):
                raise TypeError('param `methods` should be `Iterable[str]`, not `str`')
            self.methods = set([x.upper() for x in methods])
            if 'HEAD' not in self.methods and 'GET' in self.methods:
                self.methods.add('HEAD')
``` 

```
class Map(object):

    def __init__(self):
        self._rules = []
        self._rules_by_endpoint = {}
        self._remap = True

    def add(self, rulefactory):
        for rule in rulefactory.get_rules(self):
            rule.bind(self)
            self._rules.append(rule)
            self._rules_by_endpoint.setdefault(rule.endpoint, []).append(rule)
        self._remap = True
```

上述代码在 app.url_map._rules 保存了 url_rule_class (url -> endpoint 的绑定关系)


- 路由查找过程

这是简化的 Flask 响应处理过程伪代码, 路由的查找过程发生在 dispatch_request():match_request() 函数中.

```
def wsgi_app(environ, start_response):
  with RequestContext(environ):
    rv = dispatch_request()
    response = make_response(rv)
    response = process_response(response)
    return response(environ, start_response)    
```

```
    def dispatch_request(self):
         url_rule, view_args = self.match_request()
         return self.view_functions[url_rule.endpoint](**view_args)
```


具体的路由查找匹配过程代码如下:

(1) 从 app.url_map._rules 中遍历出 rule (url endpoint 绑定的类)
(2) 通过 url 的路径 path 正则匹配查找
(3) 返回匹配的 rule

```
class MapAdapter(object):

    def match(self, path_info=None, method=None, query_args=None):
        have_match_for = set()

        for rule in self.map._rules:
            rv = rule.match(path)
            if rv is None:
                continue
            if rule.methods is not None and method not in rule.methods:
                have_match_for.update(rule.methods)
                continue

            return rule, rv

        if have_match_for:
            raise MethodNotAllowed(valid_methods=list(have_match_for))

        raise NotFound()
```

```
class Rule(RuleFactory):

    def match(self, path, method=None):
        m = self._regex.search(path)
        if m is not None:
            return {}
        return None
```


#### 结论

Flask 的路由有两部分组成, 一个是 view_functions 字典, 存储着函数名 endpoint 和函数引用 view_func 一一对应关系, 
另一个部分是 url_map, 存储的 rules 数组, 数组成员是 Rule 对象, 记录着 url 和 endpoint 的一一对应关系.
通过遍历数组 app.map._rules, 正则匹配查找 url, 最终调用 view_func.
