---
layout: post
title: 翻译：How the web works - cgi and http explained
date: 2012-09-01 17:16
tags: [web, http]
---

发现这是一篇1999年的文章，12年了都，觉得时间过的好快，那时的我应该根本就没见过电脑吧。[原文地址](http://www.garshol.priv.no/download/text/http-tut.html)

## 介绍
### 关于本文

本文的目的是就当前web技术的工作原理给出基本解释。很多社区论坛的文章中很明确的表明了我们应该懂这些知识，但是我没有在网络上找到一篇能够完整介绍相关技术的文章，所以我决定将这些相关文章的内容集合到本文中，完整的介绍Web技术的工作原理。希望这对你有用。

本文涵盖了http协议（用于传输和接受网页）以及一些服务端程序和脚本的相关技术。本文假定你已经知道如何如构建一个网页，对html有较好的理解，并且对url有基本了解（url其实就是一个地址，你想要获取的文档的地址）。

本文可能存在不完善的地方，欢迎反馈。如果你不是一名专业人员，发现对本文有任何不懂的地方，或者发现本文并没有回答你所有的疑问，欢迎联系我。同样欢迎对本文的修正和意见。

### 一些背景

当你在上网时，情况通常是这样的：你坐在你的电脑前，想要查看网络上的一篇文章，并且你知道这篇文章的地址(url)。

由于你想访问的这篇文章很可能是在离你非常遥远的地球的的某个角落的硬盘上，你需要一些条件才能来访问到它。首先是你的浏览器，你会打开你的浏览器，将url写入地址栏（或者其他方法告诉浏览器地址，比如点击一个超链接）。

当然，这还不够，你的浏览器不可能直接从遥远角落的另一块磁盘上读出数据。为了你能够访问到这篇文章，存放这篇文章的计算机上必须运行一个网络服务器(web server)。Web server是一个专门用于监听并处理从浏览器发来的请求的计算机程序。

接下来，浏览器连接server并请求这篇文章，server就会返回一个包含了这篇文章的回应（response），浏览器就可以很高兴的将文章显示给用户了。同时，server还会告诉浏览器传回来的数据到底是什么（可能是html页面，pdf文档，或者一个zip压缩包），浏览器会根据用户配置调用对应的处理程序将内容正确的显示出来。

浏览器会直接将html内容显示出来，但是如果html中引用了一些其他内容（比如图片，java applet，声音片段），浏览器会继续向这些内容所在的服务器请求数据（这通常是同一个服务器，但也有可能不是）。这里的重点是，这里很可能会产生一系列独立的请求，并且会增加server和网络的负荷。当用户访问另一个地址时，这一过程就会重演。

这些request和response是通过一种叫http的协议来分发和传输的，它是HyperText Transfer Protocol的缩写。本文主要就是在介绍该协议是如何工作的。还有一些协议如ftp和gopher的工作方式类似，但也有一些协议是按照一种完全不同的方式工作的。本文不对这些其他协议做任何介绍（文章的最后会给出一些对ftp协议的更详细的介绍的文章链接）。

需要注意的是，http协议仅仅是定义了浏览器和服务器交流的内容是什么，而不是他们是如何通信的。对于这些比特数据的传输工作实际上是由tcp/ip协议去完成的。同样，ftp和Gopher（以及大部分的互联网协议）都是使用了tcp/ip来作为底层传输协议。

在继续之前，你需要知道，任何行为和浏览器一样的程序都会在网络术语中被称作客户端（client），在互联网（web）术语中被称作用户代理（user agent）。通常我们说的服务器（server）很可能指的是服务器程序（server programe），而不是运行服务器程序的计算机（server machine）。

## 当我访问一个链接的时候发生了什么

### 第一步，解析url

浏览器要做的第一件事就是从文档对应的url中找出获取该文档的方法。大多数的url都有如下的格式`protocol://server/request-URI`。其中，`protocol`部分描述了浏览器如何告诉服务器它需要的文档及如何检索该文档；`server`部分告诉浏览器它应该与哪个服务器通讯；`request-URI`用户服务器去定位该文档。（我之所以使用request-URI这个术语是由于它是被HTTP标准所使用的，并且我找不到其他足够典型且不会引起误解的术语）

### 第二步，发送request

我们通常使用的是http协议。浏览器想要通过http协议来获取一个文档，它需要向服务器发送一个request，内容如下`GET/request-URI HTTP/version`。其中version告诉服务器应该使用哪个版本的http协议。（通常，这个request还会包含其他一些信息，细节在后面会讨论到）

这些request字符串是server唯一能够得到的东西，所以server根本不关心这个request到底是来自于一个浏览器，还是link checker，还是validator，还是搜索引擎机器人，还是你手动输入的。它仅仅是执行这个请求并返回结果。

### 第三步，服务器回应

当server收到这个http request后，它找到对应的文档并将其返回。一个http response会有如下的固定格式：

    HTTP/[VER] [CODE] [TEXT]
    Field1: Value1
    Field2: Value2

    ...Document content here...

第一行标识了正在使用的http的哪个版本，紧接着是状态码和对应简单解释。通常，状态码为200表示基本上一切正常。接下来的几行叫做header，描述了文档的一些信息。header以一个空行作为结束，再接下来就是文档内容了。它们看起来会像这样：

    HTTP/1.0 200 OK
    Server: Netscape-Communications/1.1
    Date: Tuesday, 25-Nov-97 01:22:04 GMT
    Last-modified: Thursday, 20-Nov-97 10:44:53 GMT
    Content-length: 6372
    Content-type: text/html

    <!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
    <HTML>
    ...followed by document content...

我们从第一行得知本次请求是成功的。第二行是可选的，它告诉我们server运行的是Netscape Communications服务器，版本1.1。接下来是服务认为的当前时间及文档的最后修改时间。再接下来是文档的大小（单位bytes）及最重要的字段**Content-type**。

Content-type字段告诉浏览器它接收到的文档是什么格式的，HTML会标记为`text/html`，纯文本会标记为`text/plain`，GIF图片会标记为`image/gif`等等。这样的好处是URL可以是任何形式的，同时浏览器总能知道正确的文档格式。

这里有一个重要的概念，对于浏览器来说，server就是一个黑匣子。浏览器请求一个文档，然后它要么得到了该文档，要么得到一条错误消息。server处理请求的过程对浏览器是不可见的。也就是说，server可以从一个文件中读出该文档，可以运行一个程序产生该文档，可以从命令行工具编译出一个文档，甚至通过语音识别系统记录管理员的口述来形成一篇文档（这夸张了点，但原理上是可行的）。这就给了server管理员极大地自由去尝试各种方法，因为用户（浏览器）不关心文档是如何产生的。

### server做了什么

通常server会将磁盘上的某个目录配置成其根目录（root directory），并且每个目录都会有一个默认的文件名（通常是"index.html"）。当我们向server请求"/"这个文档的时候（比如"http://www.domain.com/"），就会被server定位到根目录下的index.html文件。如果我们请求"/foo/bar.html"就会被定位到根目录下的foo目录下的bar.html文件。

但这不是一定的，server可以见"/foo/"映射到磁盘上的任意目录，甚至是使用服务端程序来处理所有该路径下的请求。server设置可以讲request映射成不按照路径的格式而是其他一些格式。

### HTTP协议版本

到目前为止，共有三个http版本（注：这是1999年的文章，所以...)。第一个是*HTTP/0.9*，这是一个极原始的版本，几乎没有规定任何标准。这些到*HTTP/1.0*被修正了，RFC1945文档规定了其标准。*HTTP/1.0*是当前最常用的版本，*HTTP/0.9*已经很少使用了。（还有一些简单的HTTP客户端使用0.9的版本，因为它们不需要最新的扩展功能）

RFC2068文档规定了*HTTP/1.1*的标准，它在很多方面扩展并提升了*HTTP/1.0*。目前很少有浏览器能支持它（据作者所知，只有IE4.0支持），但是服务端已经开始进入该标准了。

*HTTP/1.1*主要的更新在于对通过HTTP协议在线编写文档的支持，以及可以让客户端在一次请求完成后保持连接从而不需要再下次请求时重新建立连接。这可以在一定程度上减少延时和服务端负载。

本文章主要描述*HTTP/1.0*，但会涉及一些*HTTP/1.1*的扩展，这些会详细标明。

## 客户端发出的请求

### 请求是什么样的

基本上，所以的请求会看起来像这样：

    [METH] [REQUEST-URI] HTTP/[VER]
    [fieldname1]: [field-value1]
    [fieldname2]: [field-value2]

    [request body, if any]

METH描述了本次请求使用的method，method有很多种，每种都做各自不一样的事情。之前的例子使用了GET，下面还会介绍一些其他的。REQUEST-URI用于在server中定位文档，比如"/index.html"等等。VER是HTTP版本，和server response中的一样。header字段也和server response中的一样。

Reqeust body只有在需要向服务器传输数据的时候才会用到，比如POST和PUT（下面会介绍）。

### 获取一个文档

有多种请求种类，最常用的是GET。一个GET请求就表示“发送给我一个文档”，其内容"GET document\_path HTTP/version"。对于"http://www.yahoo.com/"文档路径是"/"，对于"http://www.w3.org/Talks/General.html"其文档路径是"/Talks/General.html"。

虽然这第一行非常重要，但通常客户端不会单单只发送第一行，它还会包含一些需要让服务器知道的其他信息在header字段中。这些字段以"fieldname: value"的格式接在第一行之后，每个字段独占一行。

一些常用的字段有如下这些：

*   User-Agent

    这是用户代理（user agent）的字符标识。运行在Windows NT下的英文版Netscape 4.03浏览器会以"Mozilla/4.03[en] (WinNT;I;Nav)"来标识。

*   Referer

    Referer字段告诉server用户是从哪来的（上一个访问的url是什么）。这在处理登陆及检测谁链接到本页面时非常有用。

*   If-Modified-Since

    当浏览器的缓存中已经包含了这个文档的一个之前的版本，它就可以添加该字段，附上上次获得该文档的时间。server可以检查自从这个时间之后文档是否被修改过，如果没有，服务器只需要同时浏览器文档没有变化而不用重新发送整个文档。这样可以节省一些延时和网络负荷。

*   Form

    该字段让一些垃圾邮件制造者美梦成真了，因为它被设计成存放用户代理的使用者的email地址。浏览器很少用它，部分原因就是因为这些垃圾邮件制造者。但是，网络机器人应该使用它，这样Web管理员在能在网络机器人做出不合适的举动的时候能够联系到它的主人。

*   Authorization

    该字段存放一些认证信息，有些文档需要认证用户身份之后才能有访问权限。

将这些信息放一起：下面是一个由我的浏览器（Opera）发出的典型的GET请求内容

    GET / HTTP/1.0
    User-Agent: Mozilla/3.0 (compatible; Opera/3.0; Windows 95/NT4)
    Accept: */*
    Host: birk105.studby.uio.no:81

### HEAD: 检查文档

有时候人们只想查看服务器返回的header信息，而不需要下载整个文档，这就是HEAD method的应用场景。HEAD的工作方式和GET十分相似，唯一的区别就是server只会返回header信息而不会返回文档内容。

这在一些场景下十分有用，比如link checkers一类的软件，比如人们只想查看response headers信息（来确认server运行的是什么服务器等等）。

### 扮演浏览器

你甚至可以通过直接编写request内容来扮演浏览器的行为。你只需要使用telnet链接服务器的80端口，并输入如下内容，最后再敲两次回车就可以：

    larsga - tyrfing>telnet www.w3.org 80
    Trying 18.23.0.23...
    Connected to www.w3.org.
    Escape character is '^]'.
    HEAD / HTTP/1.0

    HTTP/1.1 200 OK
    Date: Tue, 17 Feb 1998 22:24:53 GMT
    Server: Apache/1.2.5
    Last-Modified: Wed, 11 Feb 1998 18:22:22 GMT
    ETag: "2c3136-23c1-34e1ec5e"
    Content-Length: 9153
    Accept-Ranges: bytes
    Connection: close
    Content-Type: text/html; charset=ISO-8859-1

    Connection closed by foreign host.
    larsga - tyrfing>

这会在Unix下很好的工作，Windows Telnet不能很好的工作（且很难配置好）。作为替代，你也可以使用HTTPTest，这是一个CGI脚本，我把它列在reference中了。

## 服务端的Response

### 概述

服务端的返回内容由这些内容组成：一行状态码，接着是一些header字段，然后是一个空行，最后是文档内容。大概像这样：

    HTTP/1.0 code text
    Field1: Value1
    Field2: Value2

    ...Document content here...

### 状态码

状态码由三个十进制数字组成，根据地一个数字的不同可以分为5组。下面给出的状态码对应的短语仅仅是建议内容，server可以返回任意的短语。

*   **1xx: Infomational**

    1xx的状态码没有被直接定义，它仅用于实验目的。

*   **2xx: Successful**

    表示该请求已经被成功的执行了。

    *200 OK*  <br/>
    表示服务完全按要求完成了它的任务，一切都正常。  <br/>
    *Others*  <br/>
    其他的2xx状态一般表示脚本的执行，不常用。  <br/>

*   **3xx: Redirection**

    表示资源在其他地方，客户端需要重新尝试一个新的地址。

    *301 Moved permanently*  <br/>
    客户端请求的资源已经前移到新的地址了，客户端需要冲新区该地址请求资源。所有对该资源的引用都需要更新。  <br/>
    *302 Moved temporarily*  <br/>
    这个301表示相同的意思，只是对该资源的链接和引用不需要更新，因为资源可能在未来的什么时候会迁移回来。  <br/>
    *304 Not modified*  <br/>
    当客户端的request带有if-modified-since字段时，server可能会返回该状态码，表示在指定的时间内资源没有修改，所以客户端可以直接使用其缓存中的版本。

*   4xx: Client error

    这表示客户端出错了，通常指客户端请求了一个它不该请求的地址。

    *400: Bad request*  <br/>
    客户端发来的请求语法不对。  <br/>
    *401: Unauthorized*  <br/>
    这表示客户端没有权限访问该资源，它需要客户端重新发送带有authorization的header字段的request才能访问。  <br/>
    *403: Forbidden*  <br/>
    表示客户端没有权限访问该资源，即使带有authorization字段也不行。  <br/>
    *404: Not found*  <br/>
    表示服务端根本不知道该资源是什么，也没有进一步的线索。  <br/>
    换句话说就是**死链**。

*   **5xx: Server error**

    这表示服务端出错了，它不能完成客户端的请求。

    *500: Internal server error*  <br/>
    服务端内部出错。  <br/>
    *501: Not implemented*  <br/>
    表示客户端请求的方法目前并不支持。  <br/>
    *503: Service unavailable*  <br/>
    这常用在服务端负载过高，无法及时处理这次请求。通常客户端等一段时间后重新尝试就可能成功。

### Header字段

下面是一些常用的response的header字段

*   **Location**  <br/>
    该字段给出了资源重定向后的新地址。
*   **Server**  <br/>
    该字段告诉浏览器当前使用的server。几乎所有的Web服务器都会返回该字段，但有时它们只是将其留空。
*   **Content-length**  <br/>
    该字段给出资源（文档内容）的大小，单位是bytes。
*   **Content-encoding**  <br/>
    该字段说明了资源是以何种方式编码的，以便浏览器能够是用正确的方式对其解码。
*   **Expires**  <br/>
    该字段高速浏览器本次请求的资源将在多长时间之后过期，以便浏览器可以在这段时间之内使用缓存的数据。
*   **Last-modified**  <br/>
    该字段告诉浏览器该资源上次修改是在什么时候。可以用在镜像，及更新通知等场景。

## 缓存：在客户端和服务端之间的代理

### 浏览器缓存

你可能会注意到，当你返回一个之前访问过的网页的时候，它的加载速度是很快的。这是因为浏览器已经保存了一个网页的副本在缓存中了。通常浏览器会设置一个缓存的最大值及保存网页的最长失效时间。

也就是说，当你访问一个新的网页的时候，浏览器会把它保存在缓存中。当缓存满了之后，浏览器会选择最不常用的网页从缓存中删除来腾出空间，当该网页已经保存了8天而浏览器设置的最长失效时间是7天，那么该页面也是需要重新下载的。

准确来说，每个浏览器处理缓存的方式是不同的，但是它们的基本思路大致相同，并且都是为了减少演示并节约带宽。其实她还与一些HTTP协议细节相关，这会在稍后谈到。

### Proxy缓存

浏览器缓存是一个很好的方法，但是当很多用户都访问同一个地址，当它们的缓存失效后都需要一遍遍的访问服务器来刷新它们的缓存。显然，这不是最理想的。

解决方案就是让多个用户能够共享缓存，Proxy缓存就是这么来的。当用户浏览器需要访问一个新的网页的时候，它不是将请求直接发送给服务器，而是将请求发送给Proxy，如果Proxy的缓存中有对应的文档就会将该文档返回给浏览器，如果Proxy缓存中没有指定的文档，它会去向服务器发送请求来获取该文档，并将其保存在自己的缓存中，最后将文档返回给浏览器。

所以Proxy就是一个被多用户共享的缓存，能够大幅度减小网络负荷。当然，他也会影响基于日志的统计数据。

一种比单独的Proxy缓存更高级的方案是使用层级的缓存。想象一个大型的ISP会为每个国家分配一个区域Proxy缓存，然后每个区域Proxy缓存会向同一个全局Proxy缓存中请求数据。这样能够更大程度的减小网络负荷。更多的相关细节可以参考后面的Reference。

## 服务器端编程

### 是什么和为什么

服务器端脚本（或者叫程序）就是一些运行在服务器上的简单程序，用于回应客户端发来的请求。它们产生正常的HTML文档并将其返回给客户端，就好像客户端请求的就是一个普通的文档一样。实际上，客户端永远都不会知道它们得到的文档是不是程序产生的。

而像JavaScript,VBScript,Java applet等技术都是运行在客户端的，所以它们不是服务端脚本。客户端脚本和服务端脚本主要缺别的原因是由于通常客户端和服务端并不是同一台机器。所以需要处理大量服务端数据的任务通常都会使用服务端脚本而不是客户端脚本。当需要实现更多的人机交互功能的时候，客户端脚本就是更好的选择了，因为这样可以减少客户端向服务端的请求。

所以，总的来说，需要处理大量服务端数据且不需要频繁的人机交互功能的时候，通常是使用服务端脚本。相反，需要处理少量数据但是大量人机交互功能的时候，通常是使用客户端脚本。

使用服务端脚本的一个典型例子就是搜索引擎Altavista（看来那时候Google还没出来啊），使用客户端脚本将Altavista公司搜集的所有文档资料下载下来显然是不可行的。使用客户端脚本的一个典型例子就是一个简单的棋牌游戏，这不需要什么数据处理，如果将用户的每一部动作都返回给服务端也太麻烦了。

不过这里遗留了一个问题，如果一项任何同时需要数据处理和人机交互该怎么办呢？目前没有很好地解决方案，不过一想叫做XML的新技术可能可以解决这个问题。（可以在reference中找到更多详细信息）

### 它是如何工作的

服务端脚本的工作方式根据使用的大量不同的技术而不同，但是，有些东西是不变的。Web服务器接收到普通的request，但是这个URL不是映射到一个具体文件上，而是映射到一个脚本区域。

Web服务器会执行这个脚本，并将request的所有信息传递给它，脚本最后会返回HTTP headers和HTML内容，服务器将这些传递给客户端。
### CGI

CGI（全称是Common Gateway Interface）规定了一种Server与服务端脚本交互的方式。CGI是完全独立于编程语言、操作系统及服务器之外的。它是当前最流行的服务端脚本技术，现有的大部分server都能够支持它。并且，各种server几乎都以相同的方式来实现它。所以，你可以写一个CGI脚本，然后把它部署到各种server上都能正常运行。

就像我上面提到的，server需要一种机制来判断一个url是否是映射到了一个服务端脚本上还是映射到了一个html文件上。在CGI规定中，我们可以通过创建CGI目录来解决这个问题。在server的配置中，我们指定一个目录为CGI目录，这就表示该目录下都为CGI脚本，对该目录文件的请求都是通过执行对应脚本来获得回应（这个目录通常是*/cgi-bin/*，所以类似于*http://www.domain.com/cgi-bin/search*这个url就会被映射到一个CGI脚本上，注意其实该目录可以是任何名称）。我们也可以不使用CGI目录，但是需要保证所以的CGI程序文件都以.cgi结尾。

CGI脚本只是普通的可执行程序（或者是一些解释性语言的源文件，如Python/Perl，或者是任何操作系统知道怎么去执行的文件），所以你可以使用任意的编程语言。Server在执行cgi程序之前会将收到的request中的一些信息设置到环境变量，比如客户端ip、以及一些header字段等。并且，如果url中包含"?"这个字符，该字符后面的内容会被设置到环境变量中。

这就表示我们可以在url中存放一些额外的信息。例如，当多个用户都要访问一个计数器url，而我们又想知道每次都是谁访问了的时候，我们就可以让各个用户都去访问类似于*http://stats.vendor.com/cgi-bin/counter.pl?username*这样的url（要将username替换成用户姓名）。这样该脚本执行的时候就可以知道到底是谁访问了它。

CGI程序返回它的输出给server的方式十分简单，只需要将输出写入标准输出就可以了。也就是说，如果使用python/perl，我们只需要使用print语句，如果使用c/c++，我们可以用printf或者一些类似的，如果使用java，那就用System.out.println。

更多关于CGI的详细信息可以在reference中找到。

### 其他技术

服务端编程并不是只有CGI这一种方式，况且CGI的方式被认为是效率比较低的，一个很重要的原因是每次处理一个请求的时候都需要将CGI程序全部载入内存并且重头执行脚本。

一种效率更高的编程方式是直接使用server的api。也就是说，程序会变成server进程的一部分，并且使用server提供的api。这种方式的一个缺点就是，这样的程序显然就是依赖于不同的server的，并且，如果使用c/c++编程的话，可能程序的错误会直接导致整个server的崩溃。

使用server的api编程的最大好处是运行效率会比较高，应该当一个request到达server的时候，处理它所需要的程序及数据已经载入到内存中了。

一些server支持使用crash-proof型的编程语言，比如AOLserver可以使用tcl来编程。或者一些server例如Apache可以通过模块插件的形式来支持例如python/perl的语言。这样就可以很大程序上的减少因为程序的错误导致的整个server崩溃的可能性。

还有很多server支持其他各种各样专属的或者通用的编程语言，其中，最有名的例如ASP、MetaHTML以及PHP3等。

### 表单提交

客户端与服务端交流的最常用的方式是使用HTML表单。用户在表单中填写数据，然后点击表单的提交按钮，表单数据就会提交到server。如果表单的作者规定了使用GET方式提交数据，表单数据就会被编码进url中，使用上面提到的"?"语法。这种编码方式很简单，例如，如果表单中的name框和email框被填入*Joe*和*joe@hotmail.com*，那么被编码后的url就会是这样*http://www.domain.com/cgi-bin/script?name=joe&email=joe@hotmail.com*。

如果要提交的数据中包含了url中不能使用的字符，那么这些字符就会先进行一步转换然后再加入url中。例如"~"字符就会变成"%7E"，更多细节在RFC1738中会详细说明。

### POST：发送数据给服务器

GET不是向浏览器提交数据的唯一方式，还可以使用POST。在这种情况下，request就会包含header和body（就像server返回的response一样），body中就存放要提交的表单数据。

通常来说，POST使用在当request会改变server的状态（例如提交一条记录），GET用在不会改变server状态的场景（例如执行一次查询）。

当我们需要提交的数据较长时（大于256字符），使用GET方式就比较危险，因为GET方式会使用环境变量来传递，而一些操作系统会限制环境变量的大小为256字符，所以可能GET方式提交的长数据会变切断。这种情况可以使用POST方式提交数据来避免。

一些POST请求的处理脚本会在完成后通过302状态码将浏览器重定向到一个确认页面，就是为了防止浏览器多次向之前的页面提交POST数据。

## 更多细节

### 内容协商

考虑这样的场景，你刚刚听说PNG这种新的图片格式，然后打算将网站当前使用的所以GIF的图片都替换成这种格式。但是问题是，GIF图片是当前所有浏览器都支持的，但是PNG图片格式只有最流行的一些浏览器才能支持。所以如果你替换了你的图片格式，那么可能只有一部分用户能够看到你的图片了。

为了防止这种情况，浏览器在html发现需要请求新的图片时，它会在图片request中添加accept字段，大部分浏览器都会这么做。

通常accept字段的内容是这样的*"image/\*,image image/png"*，意思是，我希望得到PNG格式的图片，如果你不能给我，那就给我其他格式的吧。

这是在标准中描述的功能，但是我们生活在一个并不是那么标准的世界中，大部分浏览器会直接在accept字段中写入*"\*/\*"*，这就相当于啥也没说。这种情况可能会在未来改变，但目前为止，这个特性没有一点作用。

### Cookies

HTTP协议有一个很不方便的地方，每个request都是完全独立且无状态的。这对服务端程序来说，如果单纯的使用HTTP协议，我们就根本无法得知用户在发来这个request请求之前都做过些什么。（其实也有一些技巧可以使用，但是它们既丑陋又很低效）

举例来说，想象这样一种场景，server通过http协议向用户提供抽奖功能，我们当然不希望用户能够不断的刷新页面直到中奖。我们虽然可以通过限制同一个ip的用户的请求间隔时间来达到目的，但是这样不够精确，我们无法判断他们到底是不是同一个用户。

如果用户通过调制解调器连接Internet，他们会被分配一个唯一的ip地址，但是当一个用户下线后，这个同样的ip可能被分配给另一个刚刚上线的用户。还有一种情况，当多个用户都在使用终端登录一个大型的Unix系统的使用，这些用户都会使用同一个ip地址。

网景公司的解决办法是使用一种叫做**cookies**的魔法字符串（规格说明书上说cookie这个名称没有任何含义，其实这个名称还是有一段很长的历史的）。server返回一个"set-cookies"的header字段，存放了cookie名称、失效时间、以及一些其他信息。当浏览器访问同一个url（或者其他url，由server指定）的时候，它会在request中包含该cookie信息（如果没有失效的话）。

这样的话，我们就可以在用户访问上面说的抽奖url时设置一个cookie，这样，下载访问的时候server就可以知道该用户是不是已经访问过该地址了，然后继续处理该request或者通知用户过段时间之后再尝试。当前，前提是浏览器没有禁用cookies。

Cookies可以用来检测用户访问该Web站点的足迹，或者对每个用户提供用户自定义的专属页面。这是很有用的，但是可能会牵涉到一些隐私问题。

### 服务器日志

大部分server都会在它们正常工作的时候创建日志，也就是说，它们没处理一个request的时候都会在日志中添加一条记录。下面是摘录的一部分日志内容

    rip.axis.se - - [04/Jan/1998:21:24:46 +0100] "HEAD /ftp/pub/software/ HTTP/1.0" 200 6312 - "Mozilla/4.04 [en] (WinNT; I)"
    tide14.microsoft.com - - [04/Jan/1998:21:30:32 +0100] "GET /robots.txt HTTP/1.0" 304 158 - "Mozilla/4.0 (compatible; MSIE 4.0; MSIECrawler; Windows 95)"
    microsnot.HIP.Berkeley.EDU - - [04/Jan/1998:22:28:21 +0100] "GET /cgi-bin/wwwbrowser.pl HTTP/1.0" 200 1445 "http://www.ifi.uio.no/~larsga/download/stats/" "Mozilla/4.03 [en] (Win95; U)"
    isdn69.ppp.uib.no - - [05/Jan/1998:00:13:53 +0100] "GET /download/RFCsearch.html HTTP/1.0" 200 2399 "http://www.kvarteret.uib.no/~pas/" "Mozilla/4.04 [en] (Win95; I)"
    isdn69.ppp.uib.no - - [05/Jan/1998:00:13:53 +0100] "GET /standard.css HTTP/1.0" 200 1064 - "Mozilla/4.04 [en] (Win95; I)"

该日志使用的是扩展的通用日志格式，大部分的server都支持这种格式。第一行显示该request是从Netscape 4.04浏览器发来的，第二行的request是由使用了MSIE的机器人发来的，接下来三行又是Netscape 4.04浏览器。

这些日志对于调试服务端程序及配置server时是非常有用的。我们也可以通过日志分析器来分析这些日志而得出一些报告（由于cache的存在，这些保存不一定准确）。

### 一个HTTP客户端的简单例子

为了举例，下面给出一个使用Python写的一个简单的http客户端的例子。它接受hostname和path作为参数，产生一个request请求并将收到的返回值打印出来。（我们可以使用python的url库使代码变得更少，不过这样的话，例子就没用了）

    :::python
    # Simple Python function that issues an HTTP request

    from socket import *

    def http_req(server, path):

        # Creating a socket to connect and read from
        s=socket(AF_INET,SOCK_STREAM)

        # Finding server address (assuming port 80)
        adr=(gethostbyname(server),80)

        # Connecting to server
        s.connect(adr)

        # Sending request
        s.send("GET "+path+" HTTP/1.0\n\n")

        # Printing response
        resp=s.recv(1024)
        while resp!="":
            print resp
            resp=s.recv(1024)

我们这样执行

    :::python
    http_req("birk105.studby.uio.no","/")

服务端产生这样的日志

    birk105.studby.uio.no - - [26/Jan/1998:12:01:51 +0100] "GET / HTTP/1.0" 200 2272 - -

注意日志最后的"- -"，这是因为我们发出的request中没有包含相关的信息。

### 身份验证

我们可以将Server配置成只有通过了身份验证的用户才能访问某个url，通常在配置Server的时候就设定好username/password，当然也还有其他方法。

当用户访问该url的时候，server返回"401 Not authorized"状态码。然后浏览器通常会弹出一个窗口让用户填写用户名密码，完成后浏览器会将这些信息放入Authorization字段中并重新尝试请求该url。

如果server验证通过了，就会像普通request一样去处理，如果没有通过，它会再次返回401状态码。

### 服务端HTML扩展

一些server，例如Roxen、MetaHTML允许用户在他们的HTML文件中嵌入一些非HTML的命令。这些命令会在server处理request的时候执行并最终产生普通的html，然后再发送给客户端。这可以用于裁减一些Html内容或者访问数据库插入一些内容。

这种语言的两个关键点，一个是与具体的server绑定，一个是会产生正常的header字段和html内容作为输出。

这样的好处是对客户端是透明的，任何浏览器都可以使用。

### 写作与维护（HTTP扩展）

HTTP/1.0只定义了GET、HEAD、POST方法，对于普通的浏览来说，这些方法已经足够了。但是我们可能希望能够使用http协议去修改和维护server文件及目录，这样可以免去使用ftp连接服务器。这样HTTP/1.1就是为了这种需要新增了一些方法。

*   **PUT**

    PUT上传一个新的资源（文件）到server的当前url路径下。HTTP/1.1没有规定server应该具体做什么，但是像Netscape composer这样的写作程序使用PUT将文件上传并保存到server上。PUT请求不应该被缓存。

*   **DELETE**

    不用说也应该能知道，DELETE方法请求server删除该url指定的文件。在Unix系统上，这个方法可能会失败，因为文件系统可能不允许server删除文件。

### META HTTP-EQUIV

如果你希望对web服务器上的某些页面设置特定的header字段，例如Expires等，可能在一些server上这是一件很麻烦的事情。有一种方法可以解决这个问题，在http中可以插入META元素来帮助设置HTTP的header字段。它希望server去解析这个元素并把对应的header字段插入到response中，但是实际上很少有server实现了这个功能。不过，作为替代，一些浏览器实现了该特性。不是所有的浏览器都支持它，不过这依然能够提供一些帮助。

基本用法就是将下面一行插入html的head标签内

    <META HTTP-EQUIV="header field name" CONTENT="field value">

### 主机header字段

一些Web服务商可能会使用同一台物理机器来提供多个不同的Web服务。比如，*http://www.foo.com/*、*http://www.bar.com/*和*http://www.baz.com/*可能都是同一台物理机器来提供服务，但是用户却能够通过这三个url会得到三种不同的服务。为了实现这个，需要http协议的一个扩展功能来告诉server用户需要访问的什么服务。

解决方案是"HOST"字段。当用户访问*http://www.bar.com*的时候，会将"www.bar.com"插入到HOST字段中，这样server就能够知道用户希望访问什么服务。它也常被用来为每个虚拟服务产生独立的日志。

## 一些常见问题的答案

略......

## Appendices

### Explanations of some technical terms

*   API

    An Application Programming Interface is an interface exposed by a program, part of an operating system or programming language to other programs, so that the programs that use the API can exploit the features of the program that exposes the API. One example of this would be the AWT windowing library of Java that exposes an API that can be used to write programs with graphical user interfaces. APIs are only used by programs, they are not user interfaces.

*   TCP/IP

    The IP protocol (IP is short for Internet Protocol) is the backbone of the internet, the foundation on which all else is built. To be a part of the internet a computer must support IP, which is what is used for all data transfer on the internet. TCP is another protocl (Transport Control Protocol) that extends IP with features useful for most higher-level protocols such as HTTP. (Lots of other protocols also use TCP: FTP, Gopher, SMTP, POP, IMAP, NNTP etc.) Some protocols use UDP instead of TCP.

### Acknowledgements

In closing I'd like to thank the following people who have helped me with this tutorial:

*   Jelks Cabaniss, who provided early feedback on form and contents as well as encouragement.
*   Alan J. Flavell, for detailed and highly useful criticism.
*   Jukka Korpela, for help with the markup in this document and extensive criticism on form and content, as well as a couple of useful links.
*   Christian Nybø, for suggesting that I mention telnetting to servers and making HTTPTest a little more prominent.
*   Harald Joerg, who found a number of typos, and suggested some improvements in the organization.
*   Bjørn Borud, for some hints on organization as well as a typo fix.
*   Ingrid Melve for some useful links.
*   Darren Moore for suggesting that I define some technical terms.
*   Felipe Wersen for pointing out a typo.

### References

Important specifications and official pages

*   [The W3C pages on HTTP.](http://www.w3.org/Protocols/)
*   [RFC 1945](http://www.cis.ohio-state.edu/htbin/rfc/rfc1945.html), the specification of HTTP 1.0.
*   [RFC 2068](http://www.cis.ohio-state.edu/htbin/rfc/rfc2068.html), the specification of HTTP 1.1.
*   [RFC 1738](http://www.cis.ohio-state.edu/htbin/rfc/rfc1738.html), which describes URLs.
*   [The magic cookie specification](http://home.netscape.com/newsref/std/cookie_spec.html), from Netscape.

Proxy caches


*   [Web caching architecture](http://www.uninett.no/prosjekt/desire/arneberg/), a guide for system administrators who want to set up proxy caches.
*   [A Distributed Testbed for National Information Provisioning](http://www.nlanr.net/Cache/), a project to set up a national US-wide cache system.

Various


*   [The Mozilla Museum](http://www.snafu.de/~tilman/mozilla/index.html)
*   [The registered MIME types](ftp://ftp.isi.edu/in-notes/iana/assignments/media-types/media-types), from IANA.
*   [HTTPTest](http://www.garshol.priv.no/download/HTTPTest.html). Try sending HTTP requests to various servers and see the responses.
*   [An overview of most web servers available.](http://webcompare.internet.com/)
*   [The POST redirect problem.](http://www.crl.com/%7Esubir/lynx/why.html#post-redirect)
*   [About the use of the word 'cookie' in computing.](http://www.wins.uva.nl/~mes/jargon/c/cookie.html)
*   [More information about XML.](http://www.garshol.priv.no/linker/WWWlinks.html#XML)
*   [About FTP URLs.](http://www.hut.fi/u/jkorpela/ftpurl.html)
*   [A short Norwegian intro to HTTP.](http://www.uninett.no/UNINyTT/2-96.http.html)
