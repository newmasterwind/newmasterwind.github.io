---
layout: post
title: 服务器Push技术--总结
description: 针对网络上存在的一些push技术，进行了一些综合分析。
category: project
---

#1.网络上的讨论：
##1.1 浏览器端：##
###1.1.1.轮询：
    浏览器做不到开端口监听，所以一般是轮询。
###1.1.2.用firebug调试一下weibo.com的网络请求可以发现
    微博用的是轮询来实现消息提醒的，应该是用set timer隔个30秒或一分钟去服务器进行查询。和即时通信的web应用不同，微博提醒实时性要求不高，所以用轮询方式比较合理，没有必要用长连接。没错，不是非要使用长连接的。长连接对象保存在服务器，要占用一定的内存。降低了服务器的并发。除非实时性要求非常高的应用。例如WEB IM，否则不需要长连接。
    开新浪微薄其实你就会发现，其实他们是会定时去请求  
    http://rm.api.weibo.com/remind/unread_count.json  
    这个地址的，大概每隔30秒左右一次，如下是我打开微薄后的请求地址：  
    http://rm.api.weibo.com/remind/unread_count.json?source=3818214747&target=api&user_id=1700602585&_pid=10001&count=1&callback=STK_132196097145395  
    在服务器端，他们应该是把用户的通知放到了缓存里，用户请求时，只要返回对应的通知信息就行，只要有负载均衡的服务器，压力应该不成问题，另外这些通知的维护，就像@范铭川说的那样，会一系列的策略以及程序去维护。

###1.1.3.gtalk和facebook用的是comet：  
    http://www.ibm.com/developerworks/cn/web/wa-lo-comet/

###1.1.4.使用 HTML5 WebSocket 构建实时 Web 应用
    不过要求服务器端要起一个socket端口，使用wsc:// 协议连接。这种是比较即时的。

###1.1.5.服务器端与浏览器端保持长连接实现推送的实现：
    一种机制是由网景于1995年引入，基于一种特定的名叫“multipart/x-mixed-replace”的MIME类型[1]：http://www.cnblogs.com/syhan/archive/2006/12/31/609281.html  
    第二种方式： Web服务器通过CGI提供这种功能（如Apache上不处理请求头的脚本）。
###1.1.6.基于 Java 的成熟的服务器推送框架有 DWR  / Pushlet。
    **DW**是一个开放源码的使用 Apache许可协议的解决方案，它包含服务器端 Java库、一个 DWR servlet以及 JavaScript库。虽然 DWR不是 Java平台上唯一可用的 Ajax-RPC 工具包，但是它是最成熟的，而且提供了许多有用的功能。DW[1]R 从 2.0 开始增加了 push 功能 , 也就是在异步传输的情况下可以从 Web-Server 端发送数据到 Browser。
      
    **Pushlet** 是一个开源的 Comet 框架，在设计上有很多值得借鉴的地方，对于开发轻量级的 Comet 应用很有参考价值。
    观察者模型
    Pushlet 使用了观察者模型：客户端发送请求，订阅感兴趣的事件；服务器端为每个客户端分配一个会话 ID 作为标记，事件源会把新产生的事件以多播的方式发送到订阅者的事件队列里。
    客户端 JavaScript 库
    pushlet 提供了基于 AJAX 的 JavaScript 库文件用于实现长轮询方式的“服务器推”；还提供了基于 iframe 的 JavaScript 库文件用于实现流方式的“服务器推”。

##1.2 客户端：


###1.2.1 普通请求
    常用的方法有2种。一种是定时去服务器上查询数据，也叫Polling，还有一种手机跟服务器之间维护一个 TCP 长连接，当服务器有数据时，实时推送到客户端，也就是我们说的 Push。从耗费的电量、流量和数据送达的及时性来说，Push 都会有明显的优势，但 Push 的实现和维护成本相对较高。
###1.2.2 push服务商提供服务
    如：OpenMarket’s Push Notifications Service 
    It lets you send messages to people who have installed your application on Apple, Android, and BlackBerry devices：
    http://www.openmarket.com/messaging/push-notifications/  
  ***
#2.浏览器端总结：
***
##2.1 推方式介绍
    其中最常用的就是轮询 (Polling) 和 Comet 技术，而 Comet 技术实际上是轮询技术的改进，又可细分为两种实现方式，一种是长轮询机制，一种称为流技术。

轮询：
        
    这是最早的一种实现实时 Web 应用的方案。客户端以一定的时间间隔向服务端发出请求，以频繁请求的方式来保持客户端和服务器端的同步。这种同步方案的最大问题是，当客户端以固定频率向服务器发起请求的时候，服务器端的数据可能并没有更新，这样会带来很多无谓的网络传输，所以这是一种非常低效的实时方案。
 长轮询：

        长轮询是对定时轮询的改进和提高，目地是为了降低无效的网络传输。当服务器端没有数据更新的时候，连接会保持一段时间周期直到数据或状态改变或者时间过期，通过这种机制来减少无效的客户端和服务器间的交互。当然，如果服务端的数据变更非常频繁的话，这种机制和定时轮询比较起来没有本质上的性能的提高。

 流：

        流技术方案通常就是在客户端的页面使用一个隐藏的窗口向服务端发出一个长连接的请求。服务器端接到这个请求后作出回应并不断更新连接状态以保证客户端和服务器端的连接不过期。通过这种机制可以将服务器端的信息源源不断地推向客户端。这种机制在用户体验上有一点问题，需要针对不同的浏览器设计不同的方案来改进用户体验，同时这种机制在并发比较大的情况下，对服务器端的资源是一个极大的考验。
        

> 综合这几种方案，您会发现这些目前我们所使用的所谓的实时技术并不是真正的实时技术，它们只是在用 Ajax
> 方式来模拟实时的效果，在每次客户端和服务器端交互的时候都是一次 HTTP 的请求和应答的过程，而每一次的 HTTP 请求和应答都带有完整的HTTP头信息，这就增加了每次传输的数据量，而且这些方案中客户端和服务器端的编程实现都比较复杂，在实际的应用中，为了模拟比较真实的实时效果，开发人员往往需要构造两个
> HTTP连接来模拟客户端和服务器之间的双向通讯，一个连接用来处理客户端到服务器端的数据传输，一个连接用来处理服务器端到客户端的数据传输，这不可避免地增加了编程实现的复杂度，也增加了服务器端的负载，制约了应用系统的扩展性。

##2.2 Comet
###2.2.1 comet介绍：
        参考链接：http://blog.csdn.net/ocean20/article/details/3420693
**comet定义**：简单说还是利用Ajax与服务器建立http长连接查询是否有数据更新，服务器收到一个连接如果没有数据更新就阻塞这个连接不要返回给客户端，直到有新数据再返回给客户端。Web客户端，发起的连接一旦被返回，或者超时就再次建立http长连接。这样就能保证数据的即时更新，以及尽量减少服务器的计算工作。
        

> 下面将介绍两种 Comet 应用的实现模型。

 - 基于 AJAX 的长轮询（long-polling）方式

AJAX 的出现使得 JavaScript 可以调用 XMLHttpRequest 对象发出 HTTP 请求，JavaScript 响应处理函数根据服务器返回的信息对 HTML 页面的显示进行更新。使用 AJAX 实现“服务器推”与传统的 AJAX 应用不同之处在于：
    1. 服务器端会阻塞请求直到有数据传递或超时才返回。
    2. 客户端 JavaScript 响应处理函数会在处理完服务器返回的信息后，再次发出请求，重新建立连接。
    3. 当客户端处理接收的数据、重新建立连接时，服务器端可能有新的数据到达；这些信息会被服务器端保存直到客户端重新建立连接，客户端会一次把当前服务器端所有的信息取回。

> iframe流

iframe流方式是在页面中插入一个隐藏的iframe，利用其src属性在服务器和客户端之间创建一条长链接，服务器向iframe传输数据（通常是HTML，内有负责插入信息的javascript），来实时更新页面。iframe流方式的优点是浏览器兼容好，Google公司在一些产品中使用了iframe流，如Google Talk。iframe 是很早就存在的一种 HTML 标记， 通过在 HTML 页面里嵌入一个隐蔵帧，然后将这个隐蔵帧的src属性设为对一个长连接的请求，服务器端就能源源不断地往客户端输入数据。

在 iframe 方案的客户端，iframe 服务器端并不返回直接显示在页面的数据，而是返回对客户端 Javascript 函数的调用，如“<script type="text/javascript">js_func(“data from server ”)</script>”。服务器端将返回的数据作为客户端 JavaScript 函数的参数传递；客户端浏览器的 Javascript 引擎在收到服务器返回的 JavaScript 调用时就会去执行代码。

每次数据传送不会关闭连接，连接只会在通信出现错误时，或是连接重建时关闭（一些防火墙常被设置为丢弃过长的连接， 服务器端可以设置一个超时时间， 超时后通知客户端重新建立连接，并关闭原来的连接）。

使用 iframe 请求一个长连接有一个很明显的不足之处：IE、Morzilla Firefox 下端的进度栏都会显示加载没有完成，而且 IE 上方的图标会不停的转动，表示加载正在进行。Google 的天才们使用一个称为“htmlfile”的 ActiveX 解决了在 IE 中的加载显示问题，并将这种方法用到了 gmail+gtalk 产品中。Alex Russell 在 “What else is burried down in the depth's of Google's amazing JavaScript?”文章中介绍了这种方法。Zeitoun 网站提供的 comet-iframe.tar.gz，封装了一个基于 iframe 和 htmlfile 的 JavaScript comet 对象，支持 IE、Mozilla Firefox 浏览器，可以作为参考。

##2.3 如何有效的释放和利用资源：在客户和服务器之间保持“心跳”信息
在浏览器与服务器之间维持一个长连接会为通信带来一些不确定性：因为数据传输是随机的，客户端不知道何时服务器才有数据传送。服务器端需要确保当客户端不再工作时，释放为这个客户端分配的资源，防止内存泄漏。因此需要一种机制使双方知道大家都在正常运行。在实现上：
服务器端在阻塞读时会设置一个时限，超时后阻塞读调用会返回，同时发给客户端没有新数据到达的心跳信息。此时如果客户端已经关闭，服务器往通道写数据会出现异常，服务器端就会及时释放为这个客户端分配的资源。
如果客户端使用的是基于 AJAX 的长轮询方式；服务器端返回数据、关闭连接后，经过某个时限没有收到客户端的再次请求，会认为客户端不能正常工作，会释放为这个客户端分配、维护的资源。
当服务器处理信息出现异常情况，需要发送错误信息通知客户端，同时释放资源、关闭连接。
##2.4 comet应用 
        目前Comet主要应用在一些股票web客户端，以及一些基于web的即时聊天系统中。比较成熟的框架有Dojo ，Dwr 等一些Ajax框架中实现了该功能。 

 - **Comet  优、 缺点**

**缺点**  
长期占用连接，丧失了无状态高并发的特点。
server push不会是一个没有副作用的解决方案，是否适合还要仔细权衡。
**优点**  
实时性好（消息延时小）
性能好（能支持大量用户）

##2.5 Pushlet - 开源 Comet 框架

> Pushlet 是一个开源的 Comet 框架，在设计上有很多值得借鉴的地方，对于开发轻量级的 Comet 应用很有参考价值。 观察者模型
> Pushlet 使用了观察者模型：客户端发送请求，订阅感兴趣的事件；服务器端为每个客户端分配一个会话 ID
> 作为标记，事件源会把新产生的事件以多播的方式发送到订阅者的事件队列里。 客户端 JavaScript 库 pushlet 提供了基于
> AJAX 的 JavaScript 库文件用于实现长轮询方式的“服务器推”；还提供了基于 iframe 的 JavaScript
> 库文件用于实现流方式的“服务器推”。

JavaScript 库做了很多封装工作：
    1. 定义客户端的通信状态：STATE_ERROR、STATE_ABORT、STATE_NULL、STATE_READY、STATE_JOINED、STATE_LISTENING；
    2. 保存服务器分配的会话 ID，在建立连接之后的每次请求中会附上会话 ID 表明身份；
    3. 提供了 join()、leave()、subscribe()、 unsubsribe()、listen() 等 API 供页面调用；
    4. 提供了处理响应的 JavaScript 函数接口 onData()、onEvent()…
网页可以很方便地使用这两个 JavaScript 库文件封装的 API 与服务器进行通信。
客户端与服务器端通信信息格式
pushlet 定义了一套客户与服务器通信的信息格式，使用 XML 格式。定义了客户端发送请求的类型：join、leave、subscribe、unsubscribe、listen、refresh；以及响应的事件类型：data、join_ack、listen_ack、refresh、heartbeat、error、abort、subscribe_ack、unsubscribe_ack。
服务器端事件队列管理
pushlet 在服务器端使用 Java Servlet 实现，其数据结构的设计框架仍可适用于 PHP、C 编写的后台客户端。
Pushlet 支持客户端自己选择使用流、拉（长轮询）、轮询方式。服务器端根据客户选择的方式在读取事件队列（fetchEvents）时进行不同的处理。“轮询”模式下 fetchEvents() 会马上返回。”流“和”拉“模式使用阻塞的方式读事件，如果超时，会发给客户端发送一个没有新信息收到的“heartbeat“事件，如果是“拉”模式，会把“heartbeat”与“refresh”事件一起传给客户端，通知客户端重新发出请求、建立连接。
客户服务器之间的会话管理
服务端在客户端发送 join 请求时，会为客户端分配一个会话 ID， 并传给客户端，然后客户端就通过此会话 ID 标明身份发出subscribe 和 listen 请求。服务器端会为每个会话维护一个订阅的主题集合、事件队列。
服务器端的事件源会把新产生的事件以多播的方式发送到每个会话（即订阅者）的事件队列里。

##2.6 其他服务器推技术

> Comet 只是众多服务器推技术中的一种，目前市面上还有许多其他流行服务器推技术。

###2.6.1.WebSocket（最新的技术）
websocket参考：
http://www.ibm.com/developerworks/cn/web/1112_huangxa_websocket/
http://www.websocket.org/index.html

        在HTML5标准中，定义了客户端和服务器通讯的WebSocket方式，在得到浏览器支持以后，WebSocket将会取代Comet成为服务器推送的方法，目前chrome、Firefox、Opera、Safari等主流版本均支持，Internet Explorer从10开始支持。不过要求服务器端要起一个socket端口，使用wsc:// 协议连接。这种是比较即时的。
        有“Web 的 TCP ”之称的 WebSocket 格外吸引开发人员的注意。WebSocket 的出现使得浏览器提供对 Socket 的支持成为可能，从而在浏览器和服务器之间提供了一个基于 TCP 连接的双向通道。
        HTML5 WebSocket 设计出来的目的就是要取代轮询和 Comet 技术，使客户端浏览器具备像 C/S 架构下桌面系统的实时通讯能力。 浏览器通过 JavaScript 向服务器发出建立 WebSocket 连接的请求，连接建立以后，客户端和服务器端就可以通过 TCP 连接直接交换数据。因为 WebSocket 连接本质上就是一个 TCP 连接，所以在数据传输的稳定性和数据传输量的大小方面，和轮询以及 Comet 技术比较，具有很大的性能优势。
        WebSocket 协议本质上是一个基于 TCP 的协议。为了建立一个 WebSocket 连接，客户端浏览器首先要向服务器发起一个 HTTP 请求，这个请求和通常的 HTTP 请求不同，包含了一些附加头信息，其中附加头信息”Upgrade: WebSocket”表明这是一个申请协议升级的 HTTP 请求，服务器端解析这些附加的头信息然后产生应答信息返回给客户端，客户端和服务器端的 WebSocket 连接就建立起来了，双方就可以通过这个连接通道自由的传递信息，并且这个连接会持续存在直到客户端或者服务器端的某一方主动的关闭连接。
一个典型的 WebSocket 发起请求和得到响应的例子看起来如下：

    清单 1. WebSocket 握手协议
    
    客户端到服务端： 
    GET /demo HTTP/1.1 
    Host: example.com 
    Connection: Upgrade 
    Sec-WebSocket-Key2: 12998 5 Y3 1 .P00 
    Upgrade: WebSocket 
    Sec-WebSocket-Key1: 4@1 46546xW%0l 1 5 
    Origin: http://example.com 
    [8-byte security key] 

服务端到客户端：

    HTTP/1.1 101 WebSocket Protocol Handshake 
    Upgrade: WebSocket 
    Connection: Upgrade 
    WebSocket-Origin: http://example.com 
    WebSocket-Location: ws://example.com/demo 
    [16-byte hash response]

###2.6.1.2.Flash XMLSocket   
这种方案实现的基础是：

> 1. Flash 提供了 XMLSocket 类。
> 2. JavaScript 和 Flash 的紧密结合：在 JavaScript 可以直接调用 Flash 程序提供的接口。

但此方案的缺点在于：

> 1. 因为 XMLSocket 没有 HTTP 隧道功能，XMLSocket 类不能自动穿过防火墙；
> 2. 因为是使用套接口，需要设置一个通信端口，防火墙、代理服务器也可能对非 HTTP 通道端口进行限制； 不过这种方案在一些网络聊天室，网络互动游戏中已得到广泛使 用 。

###2.6.1.3.Java Applet 套接口   

> 在客户端使用 Java Applet，通过  java.net.Socket  或  java.net.DatagramSocket  或 
> java.net.MulticastSocket  建立与服务器端的套接口连接，从而实现“服务器推”。 这种方案最大的不足在于 Java
> applet  需要客户端安装JAVA虚拟机 。

#3.客户端总结：


----------


> 参考链接：http://blog.csdn.net/shanpengfei77/article/details/8138108
> &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;http://blog.csdn.net/sunchaoenter/article/details/7972829
> 关于Andriod推送一个不错的博客：http://www.androidpush.cn/

##3.1 概况：
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Push能够有效地激活用户更多地使用 App，目前大多数应用都有Push功能。Push功能依赖于Push服务，现在主流的智能手机操作系统都集成了免费的Push服务，如IPhone的APNS(Apple Push Notification service)、windowsphone的MPNS(Microsoft Push Notification service，Android的GCM (GoogleCloud Messaging)。但是由于Android操作系统是开源的，很多手机厂商都没有预装Google服务，并且Google服务在国内不稳定，所以国内无法使用GCM实现Push功能，这种情况下我们只能使用第三方的Push服务或自己开发Push服务。
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;国内应用开发商大多自己部署一个简单的Http服务，客户端每隔一段时间到服务器查询，这种方式的缺点也是显而易见的：
> Ø  HTTP请求的数据包较大，会增加数据流量 Ø  不够及时，如果轮询的时间间隔太短就会很耗电、耗流量

##3.2 第三方Push服务介绍
由于上述一些原因，很多的第三方Push服务应运而生，服务提供商大多数是国外的.
**1、 Urban Airship**
Urban Airship是业界最知名的一个提供推送服务的平台，每月的推送数量达到5.2亿次，平均每分钟的信息发送量约为1.3万次。
除了基本推送服务外，UrbanAirship还提供Rich Push：让Push信息可以带HTML、视频、音频等多媒体信息。此外，UrbanAirship还为iOS和Android提供In-App Purchase(IAP)服务，帮助开发者处理内容存放和安全支付等问题。Urban Airship提供了一个管理后台。开发者在这里不仅能用信息编辑界面来发送Push，还可以监测Push消息的传达情况，观察用户是否产生了交互等统计信息。
推送服务支持以下三个平台：
Ø  IOS
Ø  Android
Ø  BlackBerry
**2、 push.io**
push.io也是一个很出名的推送服务平台，已经成功推送了60亿次通知，当然和Urban Airship比还差一大截。
 
支持以下平台：
Ø  iOS
Ø  Android
Ø  Windows Phone
Ø  Nokia Ovi and S40
**3、 极光推送**
极光推送是国产的一个免费的推送服务，架构非常类似Urban Airship，很多后台网页和操作也基本上和Urban Airship一样。
支持的平台：
Ø  Android
Ø  IOS
收费标准：
***免费***
##3.3. 自主开发push服务
Push服务是一个很复杂的系统，要考虑到性能、并发量、安全性等因素，这些都是技术上的难题，我们需要慢慢摸索、积累。我们可以参考上述几个服务平台，看看他们是怎么做的。

设计角度：
Ø  支持多应用，每个应用分配唯一appKey
Ø  支持多平台
Ø  支持发送到所有设备
Ø  支持发送到指定设备
Ø  支持按时间段发送
Ø  及时推送
Ø  统计推送规律
 
技术角度：
Ø  使用socket长连接，消息能够及时达到
Ø  优化协议，减少数据包大小
Ø  一般使用json数据格式
 
流程：
Ø  生成设备唯一ID
Ø  注册到pushserver
Ø  等待消息
Ø  收到消息
Ø  弹出通知
 
##3.4 个人对服务器架构的想法：
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;首先服务器应该是分布式的、可扩展的，其中一个服务作为总服务，然后架设多个子服务器，终端设备需要先连接到总服务拿到需要连接的子服务器地址，然后再连接到具体的子服务器，总服务用于接收各app运营后台提交的push请求，然后通过子服务器分发到每个终端设备。流程：

 - 服务器之间的交互：

> 1.架设总服务
> 2.架设子服务
> 3.子服务连接并登录到总服务
> 4.定时向总服务报告连接情况

 - 客户端登录过程：

> 1.连接到总服务并获取子服务器地址
> 2.连接并登录到子服务
> 3.到数据库查询有没有发往此客户端的push
> 4.如果有就发送，发送成功后删除

 - 运营后台发送push过程：

> 1.运营后台向总服务发送push数据
> 2.总服务把数据保存到数据库
> 3.然后把此push下发到子服务
> 4.子服务在连接池里查找有没有发送对象
> 5.发送到对应的客户端并删除数据库里对应的数据

