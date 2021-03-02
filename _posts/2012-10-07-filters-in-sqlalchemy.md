---
layout: post
title: 终于知道 SQLAlchemy 里面 filter 到底是怎么回事了
date: 2012-10-07 23:42
tags: [python, sqlalchemy, sql]
---

之前一直对这个很迷惑，为什么SQLAlchemy的代码可以这么写

    :::python
    reply_list = db.session.query(CommentBase.content, User.name) \
            .filter(CommentBase.user_id==User.id) \
            .filter(CommentBase.status.op('&')(1)==1) \
            .filter(CommentBase.id.in_(id_list)).all()

那个扎眼的"=="难道不会直接把CommentBase.user_id和User.id比较出结果再传入filter函数吗，filter是怎么知道这个比较逻辑并生成对应的SQL语句的呢。

在网上找了好久也没有找到答案，因为我一直以为在function的内部是可以有方法得到该方法被调用的时候传入的表达式（表达式字串）的。

直到后来看到了StackOverFlow上的这么一条：

  > What advantage is there to overriding the == operator in an ORM?
  >
  > Apparently alot of ORM's do something like this:
  >
  >     query.filter(username == "bob")
  >
  > to generate sql like
  >
  >     ... WHERE username = 'bob'
  >
  > Why override the == operator instead of something like:
  >
  >     query.filter(username.eq("bob"))

这是一个提出的问题，不过不用看回答就已经解决我的疑惑了，我居然把运算法override这个概念给忘了，真个悲剧。

    >>> CommentBase.user_id == User.id
    <sqlalchemy.sql.expression._BinaryExpression object at 0x2bdfd50>

一切都真相大白了。
