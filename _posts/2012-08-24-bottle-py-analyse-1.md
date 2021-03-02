---
layout: post
title: Bottle.py 分析 1
date: 2012-08-24 03:52
tags: [bottle, python-challenge]
---

## 前言

想下一份工作尝试一下python的web开发，正好趁着这个找工作的间隙时间自己充充电，打算用python的一个web开发框架做一个自己的博客。

选定了使用一个精简的web框架bottle来做开发。其他的一个更成熟的框架比如django等以前也大概了解过，为什么选择这个精简框架呢？主要是想重点学习一下web开发的原理和相关基本知识。这个框架只有3000+行python代码，是一个单独文件，且对除了python标准库之外没有其他依赖。这样，里面的各个关键点应该就可以没有过多的封装很清晰的展现出来。即使把这个框架整个读一篇应该也不是很难吧。

## 整体概览

直接来分析代码不合适，还是从基本使用开始吧，就以官方的第一个例子来做例子。

    from bottle import route, run

    @route('/hello/:name')
    def index(name='World'):
        return 'Hello %s!' % name

    run(host='localhost', port=8080)

上面是官方的实例代码，也就是使用bottle来做web开发的一个最简单的demo。代码很简单，第一行从bottle模块中导入`route`和`run`两个量。`route`是一个python的装饰器（也就是一个callable对象），`run`也是一个callable对象。

代码的意思也不难理解，将`index`这个函数绑定为`/hello/`这个路径（url）的处理函数，即`http://localhost/hello/carl`这个地址的处理函数。函数将路径的最后一个量作为name变量传递给`index`处理函数，该函数将返回处理得到的结果，也就是一段html。这样，这段html就会在浏览器中显示出来。

在这里我们会提出以下几个问题。
1. 这里的index函数式怎么绑定到这个路径的呢？
2. 代码里面我们只看到定义该函数时使用了route这个装饰器，route里面到底做了什么呢？
3. run这个函数是怎么回事，它的参数里面只有监听的地址和端口，但是它到底运行了什么呢，它和route装饰器是怎么联系到一起的呢？

## run和route怎么联系的

先看这个最奇怪的问题是怎么回事。在bottle的文档里面还给了这个demo的另一种写法，如下：

    from bottle import Bottle, run

    app = Bottle()

    @app.route('/hello/:name')
    def index(name='World'):
        return 'Hello %s!' % name

    run(app=app, host='localhost', port=8080)

哈，这样就看出来联系了吧。其实第一个demo里面的route是一个默认app的route，run里面运行的app就是这个默认app。那么他们到底是怎么实现的呢。看bottle.py代码

    def run(app=None, server='wsgiref', host='127.0.0.1', port=8080,
            interval=1, reloader=False, quiet=False, plugins=None,
            debug=False, **kargs):

        ...此处省略...

        try:
            _debug(debug)
            app = app or default_app()
            if isinstance(app, basestring):
                app = load_app(app)
            if not callable(app):
                raise ValueError("Application is not callable: %r" % app)

            ...此处省略...

            if reloader:
                lockfile = os.environ.get('BOTTLE_LOCKFILE')
                bgcheck = FileCheckerThread(lockfile, interval)
                with bgcheck:
                    server.run(app)
                if bgcheck.status == 'reload':
                    sys.exit(3)
            else:
                server.run(app)
        ...此处省略...

看到这个`app = app or default_app()`了吗，看来当run的参数没有给定app的时候就是运行的`default_app()`的返回值。再看这个`default_app`是啥。

    class AppStack(list):
        """ A stack-like list. Calling it returns the head of the stack. """

        def __call__(self):
            """ Return the current default application. """
            return self[-1]

        def push(self, value=None):
            """ Add a new :class:`Bottle` instance to the stack """
            if not isinstance(value, Bottle):
                value = Bottle()
            self.append(value)
            return value
    ...此处省略...
    # Initialize app stack (create first empty Bottle app)
    # BC: 0.6.4 and needed for run()
    app = default_app = AppStack()
    app.push()

`AppStack`是一个存放Bottle实例的list，同时它的实例又是一个callable对象，每次调用都会返回list最后一个Bottle的实例。这样，我们就可以知道，当前状态下`default_app()`返回的就是执行`app.push()`时创建的Bottle实例。

好了，`run`运行的Bottle实例我们清楚了，可是它是怎么和`route`联系在一起的呢？再看`route`代码

    def make_default_app_wrapper(name):
        ''' Return a callable that relays calls to the current default app. '''
        @functools.wraps(getattr(Bottle, name))
        def wrapper(*a, **ka):
            return getattr(app(), name)(*a, **ka)
        return wrapper

    route     = make_default_app_wrapper('route')
    get       = make_default_app_wrapper('get')
    post      = make_default_app_wrapper('post')
    put       = make_default_app_wrapper('put')
    delete    = make_default_app_wrapper('delete')
    error     = make_default_app_wrapper('error')
    mount     = make_default_app_wrapper('mount')
    hook      = make_default_app_wrapper('hook')
    install   = make_default_app_wrapper('install')
    uninstall = make_default_app_wrapper('uninstall')
    url       = make_default_app_wrapper('get_url')

即使不知道`functools.wraps(getattr(Bottle, name))`是干什么的，我们也能够看出来，这个`route`其实就是把`app().route`包装了一下而已，`app()`也就是`default_app()`了。

这样我们应该就很清楚了，其他第一种写法的demo和第二种写法的demo其实没啥区别，只不过第一种写法的demo用了default_app，这样就可以直接使用从bottle模块导出的route，可以少些几行代码而已。

那么到底`functools.wraps`是干啥的呢，可以参见[这篇文章](http://www.cnblogs.com/twelfthing/articles/2145656.html)。至少可以确定和我们理解的当前内容不相关。
route做了什么

猜测一下，既然`route`就是`app.route`（这里app指的是Bottle的一个实例），那么`route`这个装饰器应该是把这些被装饰的函数对象记录到`app`中的某个地方了，用来以后处理请求的时候查找出来调用。是不是呢，看代码

    class Bottle(object):
        ...此处省略...
        def route(self, xxxx):
            <missing content>
        
两个参数可以是多个元素的列表。重点是用callback（也就是`index`函数）创建了`Route`实例，然后加入到`self.add_route(route)`里面了。这个函数就暂时不跟进了，看字面就知道意思了。

所以，这个`route`装饰器的作用和我们预想的差不多了。

...未完待续
