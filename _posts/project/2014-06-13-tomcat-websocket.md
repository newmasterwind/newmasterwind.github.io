---
layout: post
title : 初探WebSocket
description : Websocket初探。
category : project
---
 

##背景
>　　对于HTTP/HTTPS协议的应用，由于它们是非连接协议，所以通常只能由客户端主动向服务端发送请求才能获得服务端的响应并取得相关的数据。  
>　　而当前越来越多的应用希望能够及时获取服务端提供的数据，甚至希望能够达到接近实时的数据交换(例如很多网站提供的在线客户系统)。  
>　　为达到此Push目的，通常采用的技术主要有轮询、长轮询、流等，而伴随着HTML5的出现，相对更优异的**WebSocket方案**也应运而生。
<hr/>
##Push技术
###轮询  
　　轮询是由客户端定时向服务端发起查询数据的请求的一种实现方式。早期的轮询是通过不断自动刷新页面而实现的(在那个基本是IE统治浏览器的时代，那不断刷新页面产生的噪声就难以让人忍受)，后来随着技术的发展，特别是Ajax技术的出现，实现了无刷新更新数据。**但本质上这些方式均是客户端定时轮询服务端，这种方式的最显著的缺点是如果客户端数量庞大并且定时轮询间隔较短服务端将承受响应这些客户端海量请求的巨大的压力。**

###长轮询  
　　在数据更新不够频繁的情况下，使用轮询方法获取数据时客户端经常会得到没有数据的响应，显然这样的轮询是一个浪费网络资源的无效的轮询。长轮询则是针对普通轮询的这种缺陷的一种改进方案，其具体实现方式是如果当前请求没有数据可以返回，则继续保持当前请求的网络连接状态，直到服务端有数据可以返回或者连接超时。长轮询通过这种方式减少了客户端与服务端交互的次数，避免了一些无谓的网络连接。**但是如果数据变更较为频繁，则长轮询方式与普通轮询在性能上并无显著差异。同时，增加连接的等待时间，往往意味着并发性能的下降,服务器hold连接会消耗资源。**实现实例有：WebQQ、Hi网页版、Facebook IM等。

###流--长连接  
　　所谓流（也称长连接方式）是指客户端在页面之下向服务端发起一个长连接请求，服务端收到这个请求后响应它并不断更新连接状态，以确保这个连接在客户端与服务端之间一直有效。**服务端可以通过这个连接将数据主动推送到客户端。显然，这种方案实现起来相对比较麻烦，而且可能被防火墙阻断。**当前流行的comet等方案都是基于此提出的。  
　　参考链接：[http://www.dewen.org/q/665/](http://www.dewen.org/q/665/ "http://www.dewen.org/q/665/")
<hr/>

##WebSocket协议

　　WebSocket是为解决客户端与服务端实时通信而产生的技术。**其本质是先通过HTTP-/HTTPS协议进行握手后创建一个用于交换数据的TCP连接，此后服务端与客户端通过此TCP连接进行实时通信。**  
　　**Tomcat 7.0.27开始支持WebSocket服务**，在tomcat webapps/examples目录下有关于websocket的示例及源码，有兴趣的可以自行查看。  
　　WebSocket规范当前还没有正式版本，草案变化也较为迅速。Tomcat7当前支持 RFC6455 定义的WebSocket，而RFC 6455目前还未成型，将来可能会修复一些Bug，甚至协议本身也可能会产生一些变化。**RFC6455定义的WebSocket协议由握手和数据传输两个部分组成：**  

###握手信息格式
　　首先是通过握手信息建立TCP链接，为后续的信息传输做好准备。  

* 来自客户端的**握手信息**类似如下：  
<pre><code>GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Origin: http://example.com
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13
</code></pre>

* 服务端的**握手信息**类似如下：

<pre><code>HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: chat
</code></pre>

###传输信息格式
　　一旦客户端和服务端都发送了握手信息并且成功握手，则数据传输部分将开始。数据传输对客户端和服务端而言都是一个双工通信通道，客户端和服务端来回传递的数据称之为“消息”。  
　　客户端通过WebSocket URI发起WebSocket连接，WebSocket URIs模式定义如下:
<pre><code>ws-URI = "ws:" "//" host [ ":" port ] path [ "?" query ]
wss-URI = "wss:" "//" host [ ":" port ] path [ "?" query ]</code></pre>
　　ws是普通的WebSocket通信协议，而wss是安全的WebSocket通信协议(就像HTTP与HTTPS之间的差异一样)。在缺省情况下，ws的端口是80而wss的端口是443(与HTTP/HTTPS是相同的嘛！)。当然也可以修改它的端口号，若改为8000，则形式如：  

	ws://localhost:8000/examples/websocket/chat  
　　**建立连接后，随后通过socket.send(message);即可实现消息的发送和接收。**  
###优势所在
> a) 服务器与客户端之间交换的标头信息很小，大概只有2字节；   
> b) 客户端与服务器都可以主动传送数据给对方，真正的全双工；  
> c) 不用频率创建TCP请求及销毁请求，减少网络带宽资源的占用，同时也节省服务器资源；  


<hr/>

##WebSocket实例
　　Tomcat7提供的与WebSocket相关的类均位于包org.apache.catalina.websocket之中，Servlet处理类以org.apache.catalina.websocket.WebSocketServlet作为它的父类。  
　　WebSocketServlet：提供遵循RFC6455的WebSocket连接的Servlet基本实现。客户端使用WebSocket连接服务端时，需要将WebSocketServlet的子类作为连接入口。同时，该子类应当实现WebSocketServlet的抽象方法createWebSocketInbound，以便创建一个inbound实例(MessageInbound或StreamInbound)。  
　　一个标准的websocket servlet如下所示，**其核心逻辑是在收到客户端发来的消息后立即将其发回客户端**：  

	package websocket.chat;
	
	import java.io.IOException;
	import java.nio.ByteBuffer;
	import java.nio.CharBuffer;
	import java.util.Set;
	import java.util.concurrent.CopyOnWriteArraySet;
	import java.util.concurrent.atomic.AtomicInteger;
	
	import javax.servlet.annotation.WebServlet;
	import javax.servlet.http.HttpServletRequest;
	
	import org.apache.catalina.websocket.MessageInbound;
	import org.apache.catalina.websocket.StreamInbound;
	import org.apache.catalina.websocket.WebSocketServlet;
	import org.apache.catalina.websocket.WsOutbound;
	
	@WebServlet("/ChatWebSocketServlet")
	public class ChatWebSocketServlet extends WebSocketServlet {
	    private static final long serialVersionUID = 1L;
	    private static final String GUEST_PREFIX = "Guest";
	    //用于自动生成连接ID
	    private final AtomicInteger connectionIds = new AtomicInteger(0);
	    //用于保存所有连接Inbound的集合
	    private final Set<ChatMessageInbound> connections =
	            new CopyOnWriteArraySet<ChatMessageInbound>();
	    //生成新的连接Inbound，WebSocketServlet必须实现该方法
	    @Override
	    protected StreamInbound createWebSocketInbound(String subProtocol,
	            HttpServletRequest request) {
	        return new ChatMessageInbound(connectionIds.incrementAndGet());
	    }
	    //聊天消息处理类
	    private final class ChatMessageInbound extends MessageInbound {
	        private final String nickname;
	        private ChatMessageInbound(int id) {
	            this.nickname = GUEST_PREFIX + id;
	        }
	        //创建新的连接时触发
	        @Override
	        protected void onOpen(WsOutbound outbound) {
	            connections.add(this);
	            String message = String.format("* %s %s",
	                    nickname, "has joined.");
	            broadcast(message);
	        }
	        //关闭连接时触发
	        @Override
	        protected void onClose(int status) {
	            connections.remove(this);
	            String message = String.format("* %s %s",
	                    nickname, "has disconnected.");
	            broadcast(message);
	        }
	        //发送二进制消息时触发
	        @Override
	        protected void onBinaryMessage(ByteBuffer message) throws IOException {
	            throw new UnsupportedOperationException(
	                    "Binary message not supported.");
	        }
	        //发送文本消息时触发
	        @Override
	        protected void onTextMessage(CharBuffer message) throws IOException {
	            // Never trust the client
	            String filteredMessage = String.format("%s: %s",
	                    nickname, message.toString());
	            broadcast(filteredMessage);
	        }
	        //该方法向所有活动连接发送文本信息
	        private void broadcast(String message) {
	            for (ChatMessageInbound connection : connections) {
	                try {
	                    CharBuffer buffer = CharBuffer.wrap(message);
	                    connection.getWsOutbound().writeTextMessage(buffer);
	                } catch (IOException ignore) {
	                }
	            }
	        }
	    }
	}


　　客户端在html页面中加入js处理代码即可：  

	<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN" "http://www.w3.org/TR/html4/loose.dtd">
	<html>
	<head>
	<meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
	<title>Insert title here</title>
	    <style type="text/css">
	        input#chat {
	            width: 410px
	        }
	        #console-container {
	            width: 400px;
	        }
	        #console {
	            border: 1px solid #CCCCCC;
	            border-right-color: #999999;
	            border-bottom-color: #999999;
	            height: 170px;
	            overflow-y: scroll;
	            padding: 5px;
	            width: 100%;
	        }
	        #console p {
	            padding: 0;
	            margin: 0;
	        }
	    </style>
	    <script type="text/javascript">
	        var Chat = {};
	        Chat.socket = null;
	        Chat.connect = (function(host) {
	            if ('WebSocket' in window) {
	                Chat.socket = new WebSocket(host);
	            } else if ('MozWebSocket' in window) {
	                Chat.socket = new MozWebSocket(host);
	            } else {
	                Console.log('Error: WebSocket is not supported by this browser.');
	                return;
	            }
				//建立连接触发事件
	            Chat.socket.onopen = function () {
	                Console.log('Info: WebSocket connection opened.');
	                document.getElementById('chat').onkeydown = function(event) {
	                    if (event.keyCode == 13) {
	                        Chat.sendMessage();
	                    }
	                };
	            };
				//关闭连接触发事件
	            Chat.socket.onclose = function () {
	                document.getElementById('chat').onkeydown = null;
	                Console.log('Info: WebSocket closed.');
	            };
				//接收消息触发事件
	            Chat.socket.onmessage = function (message) {
	                Console.log(message.data);
	            };
	        });
			//初始化聊天对象方法，注意URL中的项目名称和Servlet名称
	        Chat.initialize = function() {
	            if (window.location.protocol == 'http:') {
	                Chat.connect('ws://' + window.location.host + '/webchat/ChatWebSocketServlet');
	            } else {
	                Chat.connect('wss://' + window.location.host + '/webchat/ChatWebSocketServlet');
	            }
	        };
			//发送聊天信息方法
	        Chat.sendMessage = (function() {
	            var message = document.getElementById('chat').value;
	            if (message != '') {
	                Chat.socket.send(message);
	                document.getElementById('chat').value = '';
	            }
	        });
	        var Console = {};
			//显示消息记录
	        Console.log = (function(message) {
	            var console = document.getElementById('console');
	            var p = document.createElement('p');
	            p.style.wordWrap = 'break-word';
	            p.innerHTML = message;
	            console.appendChild(p);
	            while (console.childNodes.length > 25) {
	                console.removeChild(console.firstChild);
	            }
	            console.scrollTop = console.scrollHeight;
	        });
	        Chat.initialize();
	    </script>
	</head>
	<body>
	<div>
	    <p>
	        <input type="text" placeholder="type and press enter to chat" id="chat">
	    </p>
	    <div id="console-container">
	        <div id="console"></div>
	    </div>
	</div>
	</body>
	</html>

代码实现相关参考链接：[http://c5ms.iteye.com/blog/1527256](http://c5ms.iteye.com/blog/1527256 "http://c5ms.iteye.com/blog/1527256")