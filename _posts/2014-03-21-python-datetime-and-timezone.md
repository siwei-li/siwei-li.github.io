---
layout: post
title: Python datetime and timezone
date: 2014-03-21 15:17
tags: [python, datetime, timezone]
---

悲剧的发现之前一直都理解有误啊，仔细读了一遍文档，这里做个记录。

## 基本类型

`datetime`库里面主要有这么五种数据：

* date
* datetime
* time
* timedelta
* tzinfo

### date

`date`表示纯粹的日期数据，即“2020-02-21”这种，不含时区概念。

文档里面有这么一句：

> January 1 of year 1 is called day number 1, January 2 of year 1 is called day number 2, and so on.

在加上`date.fromordinal`和`date.toorinal`这两个方法，我觉得其内部可能就是用`day number`这个整型数来表示的。不过这个无所谓啦。

### time

`time`表示时间数据，即”12:22:34”这种，但是它是**可以**有时区概念`time.tzinfo`在里面的。

关于Time Zone的东西下面会详细说，这里先不说了。

### datetime

`datetime`是`date`和`time`的结合体，表示完整的时间概念，即”2014-02-21 12:23:23”这种。也是**可以**有时区概念的。

另外，`datetime`是`date`的子类。

### timedelta

`timedelta`表示时间长度（时间差），很好理解，没啥好说的。

### tzinfo

即Time Zone Info，时区信息。这个原始的`tzinfo`是一个抽象基类，不应该直接实例化，应该子类化并实现一些必要函数再使用。

需要实现的函数如下：

* tzinfo.utcoffset(self, dt)
* tzinfo.dst(self, dt)
* tzinfo.tzname(self, dt)

待实现的这些函数主要都是用于被`datetime`或者`time`实例调用。逻辑会在下面说明。

## 关于时区

上面说到`datetime`和`time`是**可以**有时区概念的，为什么是**可以**有呢？

先从文档里面直接拿出两个名词：

> There are two kinds of date and time objects: “naive” and “aware”.

“Naive” Object表示单纯的时间数据，不包含时区信息，所以无法直接进行时区转换等操作。

“Aware” Object表示包含了时区信息的时间数据，这其实才是真实世界中可以在时间轴上找到自己位置的对象。由于带有了时区信息，所以也就原生支持了各时区转换等操作。

同样是一个`datetime`(或者`time`)对象，当`datetime.tzinfo is None`时它就是”naive”的，否则`datetime.tzinfo`存放了时区信息，它就是”aware”的。

让一个naive对象变成aware对象最直接的方法就是给它设置一个`tzinfo`对象：

    n = datetime.now()
    # 注意，replace方法只会直接修改属性值，不会自动根据时区做时间调整
    n.replace(tzinfo=sometz)

如果对一个naive对象调用一些跟时区相关的方法(如: `datetime.astimezone`)，就会报错或无效，因为这些方案的实现中通常需要使用`datetime.tzinfo`中实现的方法。

`tzinfo`只是一个抽象基类，定义了一些子类化时候需要实现的方案，在python的标准库里面没有任何实现。

可以为什么呢？标准库里面不应该有一些基本的通用的实现么。猜测估计是由于当前世界上各时区的混乱（还有夏令时这种奇葩的东西），以及各平台下时区信息的存储方式的差异，导致不适合在标准库里面实现吧。

实现一个最简单的UTC Time Zone如下：

    from datetime import tzinfo, timedelta

    ZERO = timedelta(0)

    class UTC(tzinfo):
        # 本时区与UTC时间的差值
        def utcoffset(self, dt):
            return ZERO

        # 时区名称
        def tzname(self, dt):
            return “UTC”

        # 夏令时
        def dst(self, dt):
            return ZERO
            

当然也没有必要真的就自己手动实现，使用`pytz`或者`dateutils`这些第三方库就好了。

## 关于时间戳(TimeStamp)

时间戳的含义就是UTC时间1970-01-01 00:00:00到当前时间的秒数，这本质上是一个时间差，所以没有时区的概念。

对于时间戳的操作，主要涉及的是另一个标准库`time`，注意这里的`time`并不是`datetime.time`。

获取当前时间戳:

    import time
    current_timestamp = time.time()

## 使用Naive对象

标准库里面没有实现`tzinfo`，所以如果我们没有使用第三方库，那么我们一定都是在使用*Naive*对象了。

但是，实际情况是即时我们使用的都是*Naive*对象，还是会有Local时间和UTC时间的区分！这就是我之前一直很混乱的地方。

其实，如果只需要做本地时间和UTC时间的转换是用不着`tzinfo`的。

### datetime.now([tzinfo])

`datetime.now`是一个类函数，返回表示当前时间的`datetime`对象。它有一个可选参数，可传入一个具体的时区信息来返回指定时区的当前时间。如果不带参数调用，那么它会返回本地时间的*naive*对象。

比如，我现在是北京时间`2014-03-21 14:11:00`:

    >>> from datetime import datetime
    >>> now = datetime.now()
    >>> print now
    2014-03-21 14:11:23.414917

虽然返回的是本地时间（北京时间），但是它是*naive*对象，不带时区信息！

    >>> print now.tzinfo
    None

如果想要得到UTC时间:

    >>> utcnow = datetime.utcnow()
    >>> print utcnow
    2014-03-21 06:16:10.025043

### datetime <=> timestamp

本地时间和时间戳之间转换:

    >>> now = datetime.now()
    >>> print now
    2014-03-21 14:22:11
    >>> ts = time.mktime(now.timetuple())
    >>> print ts
    1395382931.0
    >>> _now = datetime.fromtimestamp(ts)
    >>> print _now
    2014-03-21 14:22:11

如果想把时间戳转成UTC时间:

    >>> _utcnow = datetime.utcfromtimestamp(ts)
    >>> print _utcnow
    2014-03-21 06:22:11

如果想把UTC时间转成时间戳:

    >>> ts = time.mktimefromutc(datetime.utcnow().timetuple())
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
    AttributeError: 'module' object has no attribute 'mktimefromutc'

好吧，总觉得`time`里面缺少这这么个函数，不过我们可以这么来：

    utcnow = datetime.utcnow()
    ts_from_utcdt = time.mktime(utcnow.timetuple()) - time.timezone

或者:

    import calendar
    ts_from_utcdt = calendar.timegm(utcnow.timetuple())

又用到了`calendar`这个库，真够乱的。
    
### Local DateTime <=> UTC DateTime

对于naive对象，貌似没有方法能够直接支持本地时间和UTC时间转换的，所以只能从TimeStamp中绕一圈了。

Local DateTime => UTC DateTime

    >>> print now
    2014-03-21 14:41:42.529159
    >>> ts = time.mktime(now.timetuple())
    >>> utcnow = datetime.utcfromtimestamp(ts)
    >>> print utcnow
    2014-03-21 06:41:42

UTC DateTime => Local DateTime

    >>> utc = datetime.utcnow()
    >>> print utc
    2014-03-21 06:44:15.069810
    >>> ts = time.mktime(utc.timetuple()) - time.timezone
    >>> local = datetime.fromtimestamp(ts)
    >>> print local
    2014-03-21 14:44:15

## 使用Aware对象

如果使用`Aware`对象，一些貌似都清晰了不少。

### datetime.now([tzinfo])

我们可以返回一个指定时区的`datetime`

    >>> from datetime import datetime
    >>> import pytz
    >>> tz_cn = pytz.timezone("Asia/Shanghai")
    >>> tz_us = pytz.timezone('America/Los_Angeles')
    >>> print datetime.now(tz_cn)
    2014-03-21 14:58:38.059945+08:00
    >>> print datetime.now(tz_us)
    2014-03-20 23:58:45.427634-07:00
    >>> print datetime.now(pytz.UTC)
    2014-03-21 06:58:56.507517+00:00

并且它是`aware`的

    >>> dt_cn = datetime.now(tz_cn)
    >>> dt_cn.tzinfo
    <DstTzInfo 'Asia/Shanghai' CST+8:00:00 STD>

### 时区转换

对于`aware`对象之间的转换很简单:

    >>> dt_cn = datetime.now(tz_cn)
    >>> print dt_cn
    2014-03-21 15:04:21.700744+08:00
    >>> dt_us = dt_cn.astimezone(tz_us)
    >>> print dt_us
    2014-03-21 00:04:21.700744-07:00

对于非`aware`的对象，可以先将其转换成`aware`的再操作：

    >>> dt_naive = datetime.utcnow()
    >>> print dt_naive
    2014-03-21 07:05:58.194690
    >>> dt_utc = dt_naive.replace(tzinfo=pytz.UTC)
    >>> print dt_utc
    2014-03-21 07:05:58.194690+00:00
    >>> dt_cn = dt_utc.astimezone(tz_cn)
    >>> print dt_cn
    2014-03-21 15:05:58.194690+08:00

对于`aware`对象和时间戳的转换，我觉得还是全都转成UTC时间再操作吧。
