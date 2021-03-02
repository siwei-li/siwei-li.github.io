---
layout: post
title: Flask 如何在请求处理过程中创建并处理一个子请求
date: 2013-09-13 13:30
tags: [flask, python]
---

## 起因

之前已经用`flask`实现了`restful`的接口，接口主要是提供给移动平台的App使用的。

之前看过WunderList的Web的上收发请求使用了一种**批量**的请求来一次性处理多个请求的使用方法，又加上移动平台App的网络和性能的问题，觉得我也应该做这么一个**批量**接口，基本定义如下：

	method: post
	uri: /app/batch
	data:[json]
	    {
	        "on-fail": "ignore|stop",
	        "requests": [
	            {
	                "method": "get",
	                "uri": "/user/~me",
	                "args": {
	                    "arg1": "value1",
	                    "arg2": "value2"
	                }
	            },
	            {
	                "method": "put",
	                "uri": "/user/~me/oauth/QZONE",
	                "args": {
	                    "arg1": "value1",
	                    "arg2": "value2",
	                }
	            },
	            ....
	        ]
	    }
	
	return:[json]
	    {
	        "responses": [
	            {
	                "method": "get",
	                "uri": "/user/~me",
	                "status_code": "200",
	                "body": { ... }
	            },
	            {
	                "method": "put",
	                "uri": "/user/~me/oauth/QZONE",
	                "status_code": "200",
	                "body": { ... }
	            },
	            ....
	        ]
	    }

所以我需要在batch请求的处理过程中创建一系列的子请求并处理他们收集结果，但是不知道有没有一种比较好的实现方式。

## 实现

在StackOverFlow和Python-China上问了一圈没人回答之后，我觉得还是得靠自己。于是我把`Flask`里面的`ApplicationContext` `ReqeustContext` `werkzeug.test`等模块的相关实现都看了一遍，觉得可以这么做。

	:::python
	from flask import request, current_app
	from werkzeug.test import EnvironBuilder
	
	def handle_request(uri, method, data, **kwargs):
	    method = method.upper()
	    args = {
	        "path": uri,
	        "method": method,
	        "headers": {
	            "Authorization": request.headers["Authorization"],
	        }
	    }
	    if method in ("POST", "PUT", "PATCH"):
	        args["data"] = data
	    else:
	        args["query_string"] = data
	    builder = EnvironBuilder(**args)
	    environ = builder.get_environ()
	    with current_app.request_context(environ):
	        try:
	            resp = current_app.full_dispatch_request()
	        except Exception as e:
	            resp = current_app.make_response(
	                current_app.handle_exception(e)
	            )
	    return resp

这样在batch请求的处理函数里面调用该函数即可获取子请求的处理结果。

这样唯一觉得不爽的是使用了`werkzeug.test`模块，这个模块本来就应该是给测试环境使用的，我这么搞算不算滥用呢？
