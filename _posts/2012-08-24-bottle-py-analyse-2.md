---
layout: post
title: Bottle.py 分析 2
date: 2012-08-24 04:00
tags: [bottle, python-challenge]
---

接上篇，前面分析了Bottle里面的`route`和`run`是怎么工作的，以及它们之间的联系；接下来，我们从`bottle.run`入手，分析它是如何运行起来的。
##bottle.run

先把代码列出来：

    :::python
    def run(app=None, server='wsgiref', host='127.0.0.1', port=8080,
            interval=1, reloader=False, quiet=False, plugins=None,
            debug=False, **kargs):
        ...此处省略...
        if NORUN: return
        if reloader and not os.environ.get('BOTTLE_CHILD'):
            ...这里面是为了处理reload运行模式的，不用关心...
            ...此处省略...

        try:
            _debug(debug)
            app = app or default_app()
            if isinstance(app, basestring):
                app = load_app(app)
            if not callable(app):
                raise ValueError("Application is not callable: %r" % app)

            for plugin in plugins or []:
                app.install(plugin)

            if server in server_names:
                server = server_names.get(server)
            if isinstance(server, basestring):
                server = load(server)
            if isinstance(server, type):
                server = server(host=host, port=port, **kargs)
            if not isinstance(server, ServerAdapter):
                raise ValueError("Unknown or unsupported server: %r" % server)

            server.quiet = server.quiet or quiet
            if not server.quiet:
                _stderr("Bottle v%s server starting up (using %s)...\n" % (__version__, repr(server)))
                _stderr("Listening on http://%s:%d/\n" % (server.host, server.port))
                _stderr("Hit Ctrl-C to quit.\n\n")

            if reloader:
                lockfile = os.environ.get('BOTTLE_LOCKFILE')
                bgcheck = FileCheckerThread(lockfile, interval)
                with bgcheck:
                    server.run(app)
                if bgcheck.status == 'reload':
                    sys.exit(3)
            else:
                server.run(app)
        ...以下是异常处理，省略...

这里的代码主要就是这个try代码块，可以看出来分成下面这几个部分

* app预处理
  * plugin加载
    * 创建server
      * 运行server.run(app)

## app预处理

    :::python
    app = app or default_app()
    if isinstance(app, basestring):
        app = load_app(app)
    if not callable(app):
        raise ValueError("Application is not callable: %r" % app)

意思很简单

* 当用户没有在`bottle.run`中指定`app`参数时，就使用默认的Bottle实例；
  * 当用户指定`app`参数为一个实例时，就使用该实例；
    * 当用户指定`app`参数为一个字符串时，加载该字符串对应的模块并赋值给app；

最后再确认`app`是一个可执行对象。（这是使用`wsgi`接口所必须的）

## plugin加载

    :::python
    for plugin in plugins or []:
        app.install(plugin)

用户可以在执行`run`时指定需要加载的plugins，关于plugin的工作原理会在后面分析；

## 初始化server

    :::python
    if server in server_names:
        server = server_names.get(server)
    if isinstance(server, basestring):
        server = load(server)
    if isinstance(server, type):
        server = server(host=host, port=port, **kargs)
    if not isinstance(server, ServerAdapter):
        raise ValueError("Unknown or unsupported server: %r" % server)

同上，用户可以选择使用`默认值 or Class or 字符串`来指定使用的server。

假定用户使用默认值，则最终会创建`WSGIRefServer`的实例。所有可用server都是`ServerAdapter`的子类，我们也可以通过继承`ServerAdapter`创建自定义的`Server`。

    :::python
    class ServerAdapter(object):
        quiet = False
        def __init__(self, host='127.0.0.1', port=8080, **config):
            self.options = config
            self.host = host
            self.port = int(port)

        def run(self, handler): # pragma: no cover
            pass

        def __repr__(self):
            args = ', '.join(['%s=%s'%(k,repr(v)) for k, v in self.options.items()])
            return "%s(%s)" % (self.__class__.__name__, args)

这个`Server`父类中主要定义了通用的构造函数，和一个等待子类去实现的`run`方法，再看下`WSGIRefServer`的实现。

    :::python
    class WSGIRefServer(ServerAdapter):
        def run(self, handler): # pragma: no cover
            from wsgiref.simple_server import make_server, WSGIRequestHandler
            if self.quiet:
                class QuietHandler(WSGIRequestHandler):
                    def log_request(*args, **kw): pass
                self.options['handler_class'] = QuietHandler
            srv = make_server(self.host, self.port, handler, **self.options)
            srv.serve_forever()

`WSGIRefServer`实现了`run`函数。其主要就是从`wsgiref.simple_server`中导入`make_server`，执行并将`IP:PORT`信息和当前的`app`（即Bottle实例）传入，创建了“真正”的`Server`，然后执行`serve_forever`。

`wsgiref.simple_server`是Python标准库的一部分，看名字就知道，这应该是一个符合`WSGI`标准的Server。关于WSGI的介绍，放在后面的文章中。
执行server.run

最后的`server.run`没啥好说的了，程序会在`serve_forever`中工作起来。

小结，
看到这里，我们其实还是不知道程序是如何工作的，再往下看就不是Bottle的代码了，但还是得看，顺便介绍下WSGI，见下一篇。

PS. 这篇好水，请见谅 !!

......未完待续
