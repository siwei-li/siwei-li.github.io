---
layout: post
title: SQLAlchemy 中的级联删除
date: 2014-03-24 16:13
tags: [sqlalchemy, python, database]
---

又被一个问题折腾了半天，找到原因，记录下来。

## 问题描述

使用`SQLAlchemy`建立一对多的`relationship`：

    class Parent(db.Model):
        __tablename__ = "parent"
        id = db.Column(db.Integer, primary_key=True)
        name = db.Column(db.Text)


    class Child(db.Model):
        __tablename__ = "child"
        id = db.Column(db.Integer, primary_key=True)

        parent_id = db.Column(db.Integer,
                              db.ForeignKey("parent.id", ondelete="CASCADE"),
                              nullable=False)
        parent = db.relationship("Parent", backref="children")

        name = db.Column(db.Text)

先增加一个`parent`和其下的两个`child`，然后删除这个`parent`

    #增加记录
    p = Parent(name="P1")
    p.children.append(Child(name="C1"))
    p.children.append(Child(name="C2"))
    db.session.add(p)
    db.session.commit()

    #删除记录
    db.session.delete(p)
    db.session.commit()

报错：

    BEGIN (implicit)
    SELECT parent.id AS parent_id, parent.name AS parent_name
    FROM parent
    WHERE parent.id = ?
    (1,)
    SELECT child.id AS child_id, child.parent_id AS child_parent_id, child.name AS child_name
    FROM child
    WHERE ? = child.parent_id
    (1,)
    UPDATE child SET parent_id=? WHERE child.id = ?
    (None, 1)
    ROLLBACK
    Traceback (most recent call last):
      File "main.py", line 44, in <module>
        test()
      File "main.py", line 41, in test
        db.session.commit()
      ……
      File "build/bdist.macosx-10.4-x86_64/egg/sqlalchemy/engine/default.py", line 388, in do_execute
    sqlalchemy.exc.IntegrityError: (IntegrityError) child.parent_id may not be NULL u'UPDATE child SET parent_id=? WHERE child.id = ?' (None, 1)


很明显错误原因是由于`sa`在处理删除`parent`的时候产生了

    'UPDATE child SET parent_id=? WHERE child.id = ?' (None, 1)

这样的SQL语句。而`child`的外键设置里面明确有`nullable=False`，所以提交失败。

## 问题解决

我之前一直以为当我在外键设置里面有`ondelete="CASCADE"`这一条之后，`sa`应该会自动产生级联的删除命令，或者只产生单独对`parent`的删除SQL，让数据库自己产生对`child`的删除。可事实是`sa`产生了`SET NULL`这样的SQL，为什么呢。

网上搜索未果，再从官方文档找答案，终于找到了。

### ForeignKey(..ondelete)

这里的`ondelete`控制的只是创建数据库Schema时候的ONDELETE规则，不会影响运行中`sa`如何产生SQL命令。

### relationship(…cascade)

这里参数参数才会影响`sa`如何级联产生SQL语句，默认值为`save-update, merge`，这就是罪魁祸首。

把它设置成`all, delete-orphan`之后，`sa`就会产生级联的删除命令，而不是`set null`命令了。

    parent = db.relationship("Parent", backref=db.backref("children",  cascade="all, delete-orphan"))

输出：

    BEGIN (implicit)
    SELECT parent.id AS parent_id, parent.name AS parent_name
    FROM parent
    WHERE parent.id = ?
    (1,)
    SELECT child.id AS child_id, child.parent_id AS child_parent_id, child.name AS child_name
    FROM child
    WHERE ? = child.parent_id
    (1,)
    DELETE FROM child WHERE child.id = ?
    ((1,), (2,))
    DELETE FROM parent WHERE parent.id = ?
    (1,)
    COMMIT


cascade还有很多其他选项，可具体查看文档。

### relationship(…passive_deletes)

这是另一种可选的解决方案。

如果想完全靠数据库的`ONDELETE`规则在自动删除数据，应该将该`passive_deletes`置为True，这样就不会在删除的时候产生多余的SQL命令了。

    parent = db.relationship("Parent", backref=db.backref("children",  passive_deletes=True))

输出:

    SELECT parent.id AS parent_id, parent.name AS parent_name
    FROM parent
    WHERE parent.id = ?
    (1,)
    DELETE FROM parent WHERE parent.id = ?
    (1,)
    COMMIT
