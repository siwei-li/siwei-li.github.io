---
layout: post
title: SqlAlchemy 标记字段变更
date: 2014-06-16 19:59
tags: [python, sqlalchemy, database]
---

为了在数据库里面存储一些比较随意的结构化数据，我参照(抄袭)`sqlalchemy`的文档实现了一个自定义类型：

    class JSON(TypeDecorator):
        """ Json String Field """
        impl = Text

        def process_bind_param(self, value, dialect):
            if value is None:
                return value
            elif isinstance(value, (dict, list)):
                return json.dumps(value)
            else:
                raise ValueError("unsupported value type")

        def process_result_value(self, value, dialect):
            if value is None:
                return value
            else:
                return json.loads(value)


    class MyModel(Base):
        ...
        json = Column(JSON, default=[])

        def add_item(self, item):
            self.json.append(item)
        

其实这本质上就是把python的`dict`或`list`直接dumps成json内容存入数据库，从数据库读出之后再自动`loads`成`dict`或者`list`来使用。

但是再使用的过程中发现了一个问题。对这个`MyModel`的实例无论怎么修改其json字段都无法正常保存到数据库中，目测应该是SA没有检测到这个字段的变化而导致的。重新翻看文档，发现文档里面已经有说明了。

> Note that the ORM by default will not detect “mutability” on such a type - meaning, in-place changes to values will not be detected and will not be flushed. Without further steps, you instead would need to replace the existing value with a new one on each parent object to detect changes. Note that there’s nothing wrong with this, as many applications may not require that the values are ever mutated once created. For those which do have this requirement, support for mutability is best applied using the sqlalchemy.ext.mutable extension - see the example in Mutation Tracking.

再看看`sqlalchemy.ext.mutable`模块的文档，发现使用起来也没有那么简练，而我只想找个方法来主动标记字段变更就行了。于是翻看`mutable`模块的代码，找到了其最重要的一句：

    from sqlalchemy.orm.attributes import flag_modified

    class Mutable(MutableBase):

        def changed(self):
            """Subclasses should call this method whenever change events occur."""

            for parent, key in self._parents.items():
                flag_modified(parent, key)

其实就是这个`flag_modified`标记了修改状态，所以只需要把我自己的代码改一句就行了。

    from sqlalchemy.orm.attributes import flag_modified

    class MyModel(Base):
        ...
        json = Column(JSON, default=[])

        def add_item(self, item):
            self.json.append(item)
            flag_modified(self, "json")

打完收工。
