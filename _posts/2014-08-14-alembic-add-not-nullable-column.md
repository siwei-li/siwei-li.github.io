---
layout: post
title: Alembic 添加不能为空的列
date: 2014-08-14 18:01
tags: [alembic, sqlalchemy, python, 数据库]
---

当我们对一个已存在数据的Table中添加一个`nullable=False`的列时，通常`alembic`会直接报错，因为它不知道要给这个列赋予什么值，所以会默认给个`NULL`的值，但是这个值与`nullable=False`的规则冲突了。

通常这种不能为空的列都是应该有一个默认值的，所以可以直接在alembic中指定这个列的默认值：

    :::python
    op.add_column('callroom', sa.Column('consulvalid', sa.Integer(), nullable=True, server_default=text("0")))

但也有可能这个默认值只想在Python的`orm`环境中定义，不想定义到数据库中，比如

    :::python
    class MyClass(Base):
        name = Column(Unicode(40), default=u'hello')

这种情况下，可以这么修改alembic迁移脚本：

    :::python
    from sqlalchemy.sql import table, column
    op.add_column('callroom', sa.Column('consulvalid', sa.Boolean(), nullable=True))
    callroom = table("callroom", column("consulvalid", sa.Boolean()))
    op.execute(callroom.update().values(consulvalid=False))
    op.alter_column("callroom", "consulvalid", nullable=False)

即先增加了一个`nullable=True`的列，然后将这个列设置默认值后，再改成`nullable=False`。
