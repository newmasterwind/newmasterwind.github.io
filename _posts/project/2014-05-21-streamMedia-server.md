---
layout: post
title : 	如何快速搭建流媒体服务器
description : 接上文的流媒体分析，本文带来一个简单的基于flv格式视频的流媒体服务器搭建教程。
category : project
---

> 在之前的一文中介绍了关于流媒体及flv的相关知识：[流媒体及FLV定点播放浅析](http://itweige.com/flv-stream-media/)，本文将手把手带来如何实现一个基于flv格式视频的流媒体服务器。

##FLV视频发布方式  
<hr/>
###HTTP和RTMP方式简介
　　现在网络上的视频网站youtube，6rooms和56，发现他们用的播放协议也都是Http(**这个可以用chrome插件或者firebug观察到**)。按说FMS/Red5作为流媒体服务器，是专门做过优化的。但为何这些网站都没采用RTMP 的协议呢。RTMP 协议和Http比有哪些优势呢，或者说：我们为什么要使用FMS/Red5呢？  
　　两种协议HTTP和RTMP ，有一些不同：

- **HTTP方式**：先通过网络将FLV下载到本地缓存（例如youku是默认存放缓存在：C:\Users\xxx用户名\Documents\Youku Files\transcode中，其中的.dat文件就是加密后的视频缓存文件），然后再通过NetConnection的本地连接来播放这个FLV，这种方法是播放本地的视频，并不是播放服务器的视频，因此在本地缓存里可以找到这个FLV。其优点就是服务器下载完这个FLV，服务器就没有消耗了，节省服务器消耗。其缺点就是FLV会缓存在客户端，对FLV的保 密性不好（利用dat格式文件可以有效缓解这种问题）。HTTP的视频协议，主要是在互联网普及之后。在互联网上看视频的需求下形成的。  
- **RTMP方式**：通过NetConnection连接到FMS/Red5服务器，并实时播放服务器的FLV文件，这种方式可以任意选择视频播放点（SEEK()），并不象HTTP方式需要缓存完整个FLV文件到本地才可以任意选择播放点。其优点就是FLV不会缓存在客户端，FLV的保密性好，其缺点就是消耗服务器资源，连接始终是实时的。  
　一句话，一个是本地播放，一个是服务器实时播放，因需而定。
    
###HTTP方式变更史  


- **最初的HTTP视频协议，没有任何特别之处，就是通用的HTTP文件渐进式下载**。本质就是下载视频文件，而利用视频文件本身的特点，就是存在头部信息，和部分视频帧数据，就完全可以解码播放了。显然这种方式需要将视频文件的头部信息放在文件的前面。有些例如faststart工具，就是专门做这个功能的。但是最为原始的状态下，视频无法进行快进或者跳转播放到文件尚未被下载到的部分。这个时候对HTTP协议提出了range-request的要求。这个目前几乎所有HTTP的服务器都支持了。range-request，是请求文件的部分数据，指定偏移字节数。在视频客户端解析出视频文件的头部后，就可以判断后续视频相应的帧的位置了。或者根据码率等信息，计算相应的为位置。  
- **支持HTTP range-request的服务器**，目前就可以支持我们一般网络看视频的要求了。但是，这种方式，无法支持实时视频流，或者准实时视频流。因为视频流，是不存在一个视频文件的概念的。HTTP协议播放视频的好处，就是简单。采取通用的HTTP服务器，就可以提供服务，不需要专门类型的服务器，也不需要特别的网络端口，穿过路由器防火墙一点都没有问题。而且还可以利用通用的CDN来简化视频的部署分发的工作，减少带宽的使用。**这个是目前用于PC端或者网页端，视频点播业务，最常见的解决方案**。客户端的实现，一般采用flash完成，flash本是的videoplayer或者videodisplay控件就可以完成。资源一般采用flv格式，也可以使用mp4格式。在此基础上，多家公司推出了自己的解决方法。

###HTTP方式和RTMP方式比较  

- rtmp从协议实现来讲需要一定的客户端上行流量，而http则小得多，因此在网络环境不佳的条件下其性能不如http；
- rtmp虽然有服务器端开源实现，但是其性能和强度终究不如FMS，例如用于非关键帧seek的Enhanced Seeking和多码流带宽自适应特性等功能由服务器端实现，其协议细节并未公开；
- 另外rtmp方案也无法覆盖iOS/Andriod等移动设备。


	综合部署成本，复杂度和普适性考虑，对于常规直播点播应用，HTTP方案优于RTMP方案。

##开始搭建流媒体服务器
---
　　了解了上述知识背景，下面开始进行服务器的搭建，本服务器的架构将是：**Jwplayer+Red5+Nginx+RTMP**,其中前端播放器采用了Adobe 开源播放器Jwplayer，流媒体服务器采用Red5，传输协议RTMP。（其实Nginx服务器也是一款不错的选择，其既可以进行流媒体服务，还可以提供负载均衡/反向代理/缓存等多种服务，可惜本文只提供流媒体服务，故采用Red5这款开源服务器。）  
　　利用red5，自动为其目录下的每个flv文件生成filename.meta的xml信息文件（文件第一次播放时自动产生，所以第一次播放会有一些延迟时间用来产生meta文件，是不是可以实现meta文件提前generate呢？比如在文件放入目录后，服务器端有一个进程轮询查询，查询到新的文件进入，提前先产生meta文件），其中包含了flv视频的关键帧，用于seek操作。
###前端播放器配置
jwplayer的用法很简单，自行百度，此处给出一个配置样例：  

    var videoStreamer='rtmp://172.16.90.238:1935/vod/';//流媒体服务地址
    var player = jwplayer('videoDiv').setup( {
      width : "100%",
      height : 280,
      flashplayer : 'jwplayer/pl.swf',
      image : 'images/roadPreview.jpg',
      controlbar : 'bottom',
      file : "red5",//媒体文件根目录
      provider : 'rtmp',//传输协议
      streamer : videoStreamer//配置服务地址
     });

    player.load( [ {
       file : videoUrl//可以通过这种方式更换视频地址，注意不是服务地址
      } ]);
    player.play();//开始播放
    player.seek(preVideoTime);//根据时间，进行seek地位播放

###Red5服务器配置
　　利用red5，自动为其目录下的每个flv文件生成filename.meta的xml信息文件（文件第一次播放时自动产生，所以第一次播放会有一些延迟时间用来产生meta文件，是不是可以实现meta文件提前generate呢？比如在文件放入目录后，服务器端有一个进程轮询查询，查询到新的文件进入，提前先产生meta文件），其中包含了flv视频的关键帧，用于seek操作。

参考链接：

- [Red5下载](http://red5.org/downloads/red5/1_0/)
- [安装说明文档](http://www.red5.org/downloads/docs/)  


在开启Red5前，需首先查看1935端口是否被占用，Windows下查看当前服务：  

    #查看当前系统被占用的端口  
    netstat –ano –p tcp
如果Red5的默认端口1935被占用了，可以为其配置新的端口号：  

    #设置为一个未被其他进程占用的端口，今后流媒体client都通过此端口请求流媒体数据
    rtmp.port=1936
在使用Red5搭建流媒体服务器时，需要配置其对外提供流媒体的IP（文档位置Red5/conf/ red5.properties）：　　

    #本地通过localhost:port方式访问时，使用此配置。（与下面的配置二选一）
    rtmp.host=127.0.0.1
    #远程/实际应用场景中通过 IP:port方式访问时，使用此配置。（将下一行#去掉，即生效）
    #rtmp.host=10.210.46.11
[NOTE]：**如果上述rtmp.host配置的IP地址，与服务器上实际获取的IP不同，则Red5服务无法正常开启。**  

Red5提供的功能很强大，在提供流媒体服务的同时，还开启了自带的Tomcat，同样需要注意将Tomcat的端口配置为一个未占用的端口。（文档位置Red5/conf/ red5.properties）　　

    #配置Red5自带的Tomcat服务。
	http.port=8888

启动Red5流媒体服务器：
直接运行Red5安装路径下的red5.bat文件，当出现如下信息时，表示已经成功启动Red5流媒体服务器：  

	[INFO] [Launcher:/installer] org.red5.server.service.Installer - Installer service created

Red5提供流媒体服务，即，可以将其上存储的视频数据，以可控的数据流形式，向外提供服务。通常将需要对外提供的原始视频数据，存放于如下位置：  

	Red5/webapps/vod/streams/
    
**注：本节内容从ningg大神那里学到的，这里只是引用，班门弄斧了！**

搭建完成，此时就实现了一个基于RTMP协议和FLV格式视频文件的流媒体服务器搭建。  
上一个我们开发的项目视频播放展示图:  
![展示图](/images/projectImage/stream-server-show.png)  
更好的优化方案，有待继续深入研究～。～

最后附RTMP协议详解：
[实时消息传输协议 RTMP(Real Time Messaging Protocol)](http://blog.csdn.net/defonds/article/details/17403225) 
