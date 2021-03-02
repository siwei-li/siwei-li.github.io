---
layout: post
title: 当 SqlAlchemy 没有了外键
date: 2016-12-27 04:00
tags: [Database, ORM, SQLAlachemy]
---

目前团队习惯于不在数据库中使用外键（ForeignKey），大概是出于性能以及对脏数据的容忍度的考虑吧。这本来没什么，但是由于我们使用 SqlAlchemy 作为 ORM，就带来了一个问题：如何在不使用外键的情况下使用 SqlAlchemy 中的各种高级特性呢（如：relationship，单表继承，join表继承等）？

首先需要明确的是，上面提到的高级特性其实对外键并没有什么直接的依赖关系的，只不过如果存在外键定义的话，很多特性参数可以根据外键的定义自动配置好。

仔细查阅了 SqlAlchemy 的文档，终于整理好了各种场景下的使用方法。

后续的示例代码假设都已经执行了如下片段:

```python
import sqlalchemy as sa
import sqlalchemy.orm as orm
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()
```

## One-to-Many Relationship
首先看标准的带外键的表达如下:

```python
class Parent(Base):
    __tablename__ = 'parent'
    id = sa.Column(sa.Integer, primary_key=True)
    children = orm.relationship('Child', backref='parent')

class Child(Base):
    __tablename__ = 'child'
    id = sa.Column(sa.Integer, primary_key=True)
    parent_id = sa.Column(sa.Integer, sa.ForeignKey('parent.id'))
```

我们只需要标注出关联的目标，具体如何关联等信息 ORM 会根据外键帮我们自动配置好。

而如果不用外键的话，所有的关联关系都需要明确指定，代码就会复杂一些了：

```python
class Parent(Base):
    __tablename__ = 'parent'
    id = sa.Column(sa.Integer, primary_key=True)
    children = orm.relationship(
        'Child',
        primaryjoin='Parent.id == Child.parent_id',
        foreign_keys='Child.parent_id',
        backref='parent')

class Child(Base):
    __tablename__ = 'child'
    id = sa.Column(sa.Integer, primary_key=True)
    parent_id = sa.Column(sa.Integer, sa.ForeignKey('parent.id'))
```

如上，需要明确指定 `primary join` 和 `foriegn_keys` 这两个参数。

## One-to-One Relationship
各个只是在一对多关联的基础上添加一个 `uselist=False` 的参数，没啥好说的。

```python
class Parent(Base):
    __tablename__ = 'parent'
    id = sa.Column(sa.Integer, primary_key=True)
    child = orm.relationship(
        'Child',
        primaryjoin='Parent.id == Child.parent_id',
        foreign_keys='Child.parent_id',
        uselist=False,
        backref='parent')

class Child(Base):
    __tablename__ = 'child'
    id = sa.Column(sa.Integer, primary_key=True)
    parent_id = sa.Column(sa.Integer, sa.ForeignKey('parent.id'))
```

## Many-to-Many Relationship
同样，先看使用 ForeignKey 的版本：

```python
association_table = sa.Table('association', Base.metadata,
    sa.Column('left_id', sa.Integer, sa.ForeignKey('left.id')),
    sa.Column('right_id', sa.Integer, sa.ForeignKey('right.id'))
)

class Parent(Base):
    __tablename__ = 'left'
    id = sa.Column(sa.Integer, primary_key=True)
    children = orm.relationship(
        "Child",
        secondary=association_table,
        backref='parents')

class Child(Base):
    __tablename__ = 'right'
    id = Column(Integer, primary_key=True)
```

不用 ForeignKey 的版本:

```python
association_table = sa.Table('association', Base.metadata,
    sa.Column('left_id', sa.Integer, sa.ForeignKey('left.id')),
    sa.Column('right_id', sa.Integer, sa.ForeignKey('right.id'))
)

class Parent(Base):
    __tablename__ = 'left'
    id = sa.Column(sa.Integer, primary_key=True)
    children = orm.relationship(
        "Child",
        secondary=association_table,
        primaryjoin='left.id == association.left_id',
        secondaryjoin='right.id == association.right_id',
        backref='parents')

class Child(Base):
    __tablename__ = 'right'
    id = Column(Integer, primary_key=True)
```

## Single Table Inheritance
单表继承不需要用到外键，所以可以直接使用：

```python
class Person(Base):
    __tablename__ = 'people'
    id = sa.Column(sa.Integer, primary_key=True)
    discriminator = sa.Column('type', sa.String(50))
    __mapper_args__ = {'polymorphic_on': discriminator}

class Engineer(Person):
    __mapper_args__ = {'polymorphic_identity': 'engineer'}
    primary_language = sa.Column(sa.String(50))
```


## Joined Table Inheritance
外键版本如下：

```python
class Person(Base):
    __tablename__ = 'people'
    id = sa.Column(sa.Integer, primary_key=True)
    discriminator = sa.Column('type', sa.String(50))
    __mapper_args__ = {'polymorphic_on': discriminator}

class Engineer(Person):
    __tablename__ = 'engineers'
    __mapper_args__ = {'polymorphic_identity': 'engineer'}
    id = sa.Column(Integer, sa.ForeignKey('people.id'), primary_key=True)
    primary_language = sa.Column(sa.String(50))
```

去外键版本如下：

```python
class Person(Base):
    __tablename__ = 'people'
    id = sa.Column(sa.Integer, primary_key=True)
    discriminator = sa.Column('type', sa.String(50))
    __mapper_args__ = {'polymorphic_on': discriminator}

class Engineer(Person):
    __tablename__ = 'engineers'

    _id = sa.Column(Integer, primary_key=True)
    __mapper_args__ = {
        'polymorphic_identity': 'engineer',
        'inherit_condition': (_id == Person.id),
        'inherit_foreign_keys': _id,
    }
    id = orm.column_property(_id, Person.id)
    del _id

    primary_language = sa.Column(sa.String(50))
```

这里有两个重点，分别是如何表达继承关联，以及如何将两个表的 id 列映射到一个字段上。

在 `__mapper_args__` 中我们手动指定了 `inherit_condition` 和 `inherit_foreign_keys` 这两个参数来表达和父表的关系。

用 `orm.column_property` 可以将多个 column 映射到同一个字段上，这里将子表的 id 和父表的 id 都映射到了 id 这个字段上了，并在结束时，清理掉了多余的 `_id` 这个引用。

其实这里为什么需要将两个 id 列映射到用一个字段上我也不是很清楚，这么做只是为了完整的还原使用外键版本的结果。

## 小结

所以说，为啥不用外键呢？
