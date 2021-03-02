---
layout: post
title: 纠结的Many-Many关联
date: 2014-01-23 21:00
tags: [sqlalchemy, python, postgresql]
---

使用了`SQLAlchemy`挺长时间的了，之前在使用的过程中一直有一件事挺纠结的，SA到底会不会在commit的使用自动对比多对多关系的变更生成合适的SQL去更新数据库呢？

现决定做具体实验，搞清楚。

## 数据定义

数据定义如下:

    :::python
    class Right(Base):
        """ 权限表 """
        __tablename__ = "right"
        name = Column(String(16), primary_key=True)

        def __repr__(self):
            return "<Right(name=%s)>" % self.name


    user_right = Table(
        "user_right", Base.metadata,
        Column("user_id", Integer, ForeignKey("user.id", ondelete="CASCADE")),
        Column("right", String(16), ForeignKey("right.name", ondelete="CASCADE"))
    )


    class User(Base):
        __tablename__ = "user"

        id = Column(Integer, primary_key=True)
        name = Column(String(100), unique=True)
        age = Column(Integer, nullable=False)
        rights = relationship(Right, secondary=user_right)

        def __repr__(self):
            return "<User(name=%s)>" % self.name

## 添加关联

当得到了`user`实例后，如果新增一个关联的`right`:

    :::python
    user.rights.append(newright)
    session.commit()

正常识别出了新增的关联并且生成了合适的*SQL*。

## 减少关联

如果减少一个关联:

    :::python
    user.rights.remove(someright)
    session.commit()

正常识别出了新增的关联并且生成了合适的*SQL*。

## 重置关联

这种方式是最值得怀疑的，不再是修改关联，而是重置关联：

    :::python
    user.rights = [someright1, someright2]
    session.commit()

结果是，一切正常运行。看来*SA*的确做的很到位了。

## 结论

以后可以不用纠结，直接把对比更新的复杂度丢给*SA*去处理就好了。
