---
layout: post
title: Tomcat是怎样使用NIO的
date: 2018-07-01 14:55:56
categories: 原创
---

# Tomcat的结构

一个Tomcat实例为一个`Server`。

`Server`中，包含有很多个`Service`实例。一个`Service`可以理解为Tomcat目录下的一个运行中的war包。所以嵌入式启动的Tomcat只有一个`Service`。

每一个`Service`包含若干`Connector`实例和一个唯一的`Container`实例。其中`Connector`负责网络通信，`Container`负责对通信报文的处理。

Tomcat的`Connector`标准实现是`Coyote`，而`Container`的标准实现是`Catalina`。

`Coyote`的构造器参数只有一个，就是协议的类名，例如`org.apache.coyote.http11.Http11Nio2Protocol`。每一个协议类同时指定了IO机制（NIO，NIO2，APR）和协议类型（HTTP/1.1，AJP）。

`Catalina`里面有多个容器类型：`Engine`、`Host`、`Context`等。三者为嵌套关系。

1. `Enigne`负责处理`Service`的所有请求，每个`Service`里面只能有一个`Enigne`。
2. `Host`代表虚拟主机，关联于一个特定的域名。一个`Engine`里面可以有多个`Host`，每一个`Engine`里必须有且只有一个`Host`对应于`defaultHost`。
3. `Context`容器关联于一个`servlet context`。

这次的主角是`Coyote`。

# Coyote的协议类

如前所述，每一个协议类同时指定了IO机制（NIO，NIO2，APR）和协议类型（HTTP/1.1，AJP）。

`Coyote`的三种IO机制分别是指:

1. NIO，指的是使用随JDK1.4发布的`java.nio.*`包下的组件开发出的Reactor多路复用通信模式。
2. NIO2，指的是使用随JDK1.7发布的`java.nio.*`包下的组件开发出的Proactor异步通信模式。
3. APR，是指使用[Apache Portable Runtime](https://apr.apache.org)组件开发出的JNI组件实现的非阻塞通信模式。

其中APR模式使用的是C/C++实现的套接字，NIO和NIO2使用的是Java实现的套接字。所以要想使用APR模式，必须首先在环境中安装APR组件。

`Coyote`支持两种末端协议：HTTP/1.1，AJP。AJP是一种二进制协议，它将HTTP协议中常见的字段，例如`Content-type`，使用数字来代替，从而实现高效的通信。但是对于不常见的字段，还是使用字符串，所以AJP是有可能退化成为HTTP协议的。该不该使用AJP协议应当根据业务场景来测试后决定。

`Coyote`也支持HTTP/2，WebSocket协议，但这两者需要首先使用上面两种末端协议建立连接后，根据特定条件转换为目标协议。HTTP/2协议的连接建立可以参考[这篇博文](http://www.blogjava.net/yongboy/archive/2015/03/18/423570.html)。

# Coyote NIO的调用链

<div class="plantuml-diagram"><!--?xml version="1.0" encoding="UTF-8" standalone="no"?--><svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" contentscripttype="application/ecmascript" contentstyletype="text/css" height="507px" preserveAspectRatio="none" style="width:1120px;height:507px;" version="1.1" viewBox="0 0 1120 507" width="1120px" zoomAndPan="magnify"><defs><filter height="300%" id="faxpwwtlk7lox" width="300%" x="-1" y="-1"><feGaussianBlur result="blurOut" stdDeviation="2.0"></feGaussianBlur><feColorMatrix in="blurOut" result="blurOut2" type="matrix" values="0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 .4 0"></feColorMatrix><feOffset dx="4.0" dy="4.0" in="blurOut2" result="blurOut3"></feOffset><feBlend in="SourceGraphic" in2="blurOut3" mode="normal"></feBlend></filter></defs><g><rect fill="#ADD8E6" height="492.4609" style="stroke: #A80036; stroke-width: 1.0;" width="314" x="613" y="4"></rect><text fill="#000000" font-family="sans-serif" font-size="13" font-weight="bold" lengthAdjust="spacingAndGlyphs" textLength="112" x="714" y="16.5684">Http11Processor</text><rect fill="#FFFFFF" filter="url(#faxpwwtlk7lox)" height="71.6211" style="stroke: #A80036; stroke-width: 1.0;" width="10" x="988.5" y="264.3516"></rect><line style="stroke: #A80036; stroke-width: 1.0; stroke-dasharray: 5.0,5.0;" x1="33" x2="33" y1="86.4883" y2="409.9727"></line><line style="stroke: #A80036; stroke-width: 1.0; stroke-dasharray: 5.0,5.0;" x1="121.5" x2="121.5" y1="86.4883" y2="409.9727"></line><line style="stroke: #A80036; stroke-width: 1.0; stroke-dasharray: 5.0,5.0;" x1="239.5" x2="239.5" y1="86.4883" y2="409.9727"></line><line style="stroke: #A80036; stroke-width: 1.0; stroke-dasharray: 5.0,5.0;" x1="384.5" x2="384.5" y1="86.4883" y2="409.9727"></line><line style="stroke: #A80036; stroke-width: 1.0; stroke-dasharray: 5.0,5.0;" x1="564" x2="564" y1="86.4883" y2="409.9727"></line><line style="stroke: #A80036; stroke-width: 1.0; stroke-dasharray: 5.0,5.0;" x1="688" x2="688" y1="86.4883" y2="409.9727"></line><line style="stroke: #A80036; stroke-width: 1.0; stroke-dasharray: 5.0,5.0;" x1="846" x2="846" y1="86.4883" y2="409.9727"></line><line style="stroke: #A80036; stroke-width: 1.0; stroke-dasharray: 5.0,5.0;" x1="993" x2="993" y1="86.4883" y2="409.9727"></line><text fill="#000000" font-family="sans-serif" font-size="14" lengthAdjust="spacingAndGlyphs" textLength="44" x="8" y="83.5352">socket</text><ellipse cx="33" cy="13" fill="#FF0000" filter="url(#faxpwwtlk7lox)" rx="8" ry="8" style="stroke: #A80036; stroke-width: 2.0;"></ellipse><path d="M33,21 L33,48 M20,29 L46,29 M33,48 L20,63 M33,48 L46,63 " fill="none" filter="url(#faxpwwtlk7lox)" style="stroke: #A80036; stroke-width: 2.0;"></path><text fill="#000000" font-family="sans-serif" font-size="14" lengthAdjust="spacingAndGlyphs" textLength="44" x="8" y="422.5078">socket</text><ellipse cx="33" cy="435.4609" fill="#FF0000" filter="url(#faxpwwtlk7lox)" rx="8" ry="8" style="stroke: #A80036; stroke-width: 2.0;"></ellipse><path d="M33,443.4609 L33,470.4609 M20,451.4609 L46,451.4609 M33,470.4609 L20,485.4609 M33,470.4609 L46,485.4609 " fill="none" filter="url(#faxpwwtlk7lox)" style="stroke: #A80036; stroke-width: 2.0;"></path><rect fill="#FEFECE" filter="url(#faxpwwtlk7lox)" height="30.4883" style="stroke: #A80036; stroke-width: 1.5;" width="75" x="82.5" y="51"></rect><text fill="#000000" font-family="sans-serif" font-size="14" lengthAdjust="spacingAndGlyphs" textLength="61" x="89.5" y="71.5352">Acceptor</text><rect fill="#FEFECE" filter="url(#faxpwwtlk7lox)" height="30.4883" style="stroke: #A80036; stroke-width: 1.5;" width="75" x="82.5" y="408.9727"></rect><text fill="#000000" font-family="sans-serif" font-size="14" lengthAdjust="spacingAndGlyphs" textLength="61" x="89.5" y="429.5078">Acceptor</text><rect fill="#FEFECE" filter="url(#faxpwwtlk7lox)" height="30.4883" style="stroke: #A80036; stroke-width: 1.5;" width="133" x="171.5" y="51"></rect><text fill="#000000" font-family="sans-serif" font-size="14" lengthAdjust="spacingAndGlyphs" textLength="119" x="178.5" y="71.5352">AbstractEndpoint</text><rect fill="#FEFECE" filter="url(#faxpwwtlk7lox)" height="30.4883" style="stroke: #A80036; stroke-width: 1.5;" width="133" x="171.5" y="408.9727"></rect><text fill="#000000" font-family="sans-serif" font-size="14" lengthAdjust="spacingAndGlyphs" textLength="119" x="178.5" y="429.5078">AbstractEndpoint</text><rect fill="#FEFECE" filter="url(#faxpwwtlk7lox)" height="30.4883" style="stroke: #A80036; stroke-width: 1.5;" width="128" x="318.5" y="51"></rect><text fill="#000000" font-family="sans-serif" font-size="14" lengthAdjust="spacingAndGlyphs" textLength="114" x="325.5" y="71.5352">AbstractProtocol</text><rect fill="#FEFECE" filter="url(#faxpwwtlk7lox)" height="30.4883" style="stroke: #A80036; stroke-width: 1.5;" width="128" x="318.5" y="408.9727"></rect><text fill="#000000" font-family="sans-serif" font-size="14" lengthAdjust="spacingAndGlyphs" textLength="114" x="325.5" y="429.5078">AbstractProtocol</text><rect fill="#FEFECE" filter="url(#faxpwwtlk7lox)" height="30.4883" style="stroke: #A80036; stroke-width: 1.5;" width="81" x="522" y="51"></rect><text fill="#000000" font-family="sans-serif" font-size="14" lengthAdjust="spacingAndGlyphs" textLength="67" x="529" y="71.5352">Processor</text><rect fill="#FEFECE" filter="url(#faxpwwtlk7lox)" height="30.4883" style="stroke: #A80036; stroke-width: 1.5;" width="81" x="522" y="408.9727"></rect><text fill="#000000" font-family="sans-serif" font-size="14" lengthAdjust="spacingAndGlyphs" textLength="67" x="529" y="429.5078">Processor</text><rect fill="#FEFECE" filter="url(#faxpwwtlk7lox)" height="30.4883" style="stroke: #A80036; stroke-width: 1.5;" width="138" x="617" y="51"></rect><text fill="#000000" font-family="sans-serif" font-size="14" lengthAdjust="spacingAndGlyphs" textLength="124" x="624" y="71.5352">Http11InputBuffer</text><rect fill="#FEFECE" filter="url(#faxpwwtlk7lox)" height="30.4883" style="stroke: #A80036; stroke-width: 1.5;" width="138" x="617" y="408.9727"></rect><text fill="#000000" font-family="sans-serif" font-size="14" lengthAdjust="spacingAndGlyphs" textLength="124" x="624" y="429.5078">Http11InputBuffer</text><rect fill="#FEFECE" filter="url(#faxpwwtlk7lox)" height="30.4883" style="stroke: #A80036; stroke-width: 1.5;" width="150" x="769" y="51"></rect><text fill="#000000" font-family="sans-serif" font-size="14" lengthAdjust="spacingAndGlyphs" textLength="136" x="776" y="71.5352">Http11OutputBuffer</text><rect fill="#FEFECE" filter="url(#faxpwwtlk7lox)" height="30.4883" style="stroke: #A80036; stroke-width: 1.5;" width="150" x="769" y="408.9727"></rect><text fill="#000000" font-family="sans-serif" font-size="14" lengthAdjust="spacingAndGlyphs" textLength="136" x="776" y="429.5078">Http11OutputBuffer</text><rect fill="#FEFECE" filter="url(#faxpwwtlk7lox)" height="30.4883" style="stroke: #A80036; stroke-width: 1.5;" width="117" x="933" y="51"></rect><text fill="#000000" font-family="sans-serif" font-size="14" lengthAdjust="spacingAndGlyphs" textLength="103" x="940" y="71.5352">CoyoteAdapter</text><rect fill="#FEFECE" filter="url(#faxpwwtlk7lox)" height="30.4883" style="stroke: #A80036; stroke-width: 1.5;" width="117" x="933" y="408.9727"></rect><text fill="#000000" font-family="sans-serif" font-size="14" lengthAdjust="spacingAndGlyphs" textLength="103" x="940" y="429.5078">CoyoteAdapter</text><rect fill="#FFFFFF" filter="url(#faxpwwtlk7lox)" height="71.6211" style="stroke: #A80036; stroke-width: 1.0;" width="10" x="988.5" y="264.3516"></rect><polygon fill="#A80036" points="110,113.7988,120,117.7988,110,121.7988,114,117.7988" style="stroke: #A80036; stroke-width: 1.0;"></polygon><line style="stroke: #A80036; stroke-width: 1.0;" x1="33" x2="116" y1="117.7988" y2="117.7988"></line><text fill="#000000" font-family="sans-serif" font-size="13" lengthAdjust="spacingAndGlyphs" textLength="65" x="40" y="113.0566">客户端请求</text><polygon fill="#A80036" points="228,143.1094,238,147.1094,228,151.1094,232,147.1094" style="stroke: #A80036; stroke-width: 1.0;"></polygon><line style="stroke: #A80036; stroke-width: 1.0;" x1="122" x2="234" y1="147.1094" y2="147.1094"></line><text fill="#000000" font-family="sans-serif" font-size="13" lengthAdjust="spacingAndGlyphs" textLength="52" x="129" y="142.3672">接受请求</text><polygon fill="#A80036" points="372.5,172.4199,382.5,176.4199,372.5,180.4199,376.5,176.4199" style="stroke: #A80036; stroke-width: 1.0;"></polygon><line style="stroke: #A80036; stroke-width: 1.0;" x1="240" x2="378.5" y1="176.4199" y2="176.4199"></line><text fill="#000000" font-family="sans-serif" font-size="13" lengthAdjust="spacingAndGlyphs" textLength="65" x="247" y="171.6777">套接字可读</text><polygon fill="#A80036" points="552.5,217.041,562.5,221.041,552.5,225.041,556.5,221.041" style="stroke: #A80036; stroke-width: 1.0;"></polygon><line style="stroke: #A80036; stroke-width: 1.0;" x1="384.5" x2="558.5" y1="221.041" y2="221.041"></line><text fill="#000000" font-family="sans-serif" font-size="13" lengthAdjust="spacingAndGlyphs" textLength="156" x="391.5" y="200.9883">确定应用层协议后，将报文</text><text fill="#000000" font-family="sans-serif" font-size="13" lengthAdjust="spacingAndGlyphs" textLength="130" x="391.5" y="216.2988">交给合适的协议处理器</text><polygon fill="#A80036" points="676,231.041,686,235.041,676,239.041,680,235.041" style="stroke: #A80036; stroke-width: 1.0;"></polygon><line style="stroke: #A80036; stroke-width: 1.0;" x1="564.5" x2="682" y1="235.041" y2="235.041"></line><polygon fill="#A80036" points="976.5,260.3516,986.5,264.3516,976.5,268.3516,980.5,264.3516" style="stroke: #A80036; stroke-width: 1.0;"></polygon><line style="stroke: #A80036; stroke-width: 1.0;" x1="688" x2="982.5" y1="264.3516" y2="264.3516"></line><text fill="#000000" font-family="sans-serif" font-size="13" lengthAdjust="spacingAndGlyphs" textLength="78" x="695" y="259.6094">完成报文解析</text><line style="stroke: #A80036; stroke-width: 1.0;" x1="998.5" x2="1040.5" y1="293.6621" y2="293.6621"></line><line style="stroke: #A80036; stroke-width: 1.0;" x1="1040.5" x2="1040.5" y1="293.6621" y2="306.6621"></line><line style="stroke: #A80036; stroke-width: 1.0;" x1="999.5" x2="1040.5" y1="306.6621" y2="306.6621"></line><polygon fill="#A80036" points="1009.5,302.6621,999.5,306.6621,1009.5,310.6621,1005.5,306.6621" style="stroke: #A80036; stroke-width: 1.0;"></polygon><text fill="#000000" font-family="sans-serif" font-size="13" lengthAdjust="spacingAndGlyphs" textLength="103" x="1005.5" y="288.9199">Catalina处理请求</text><polygon fill="#A80036" points="857,331.9727,847,335.9727,857,339.9727,853,335.9727" style="stroke: #A80036; stroke-width: 1.0;"></polygon><line style="stroke: #A80036; stroke-width: 1.0;" x1="851" x2="992.5" y1="335.9727" y2="335.9727"></line><text fill="#000000" font-family="sans-serif" font-size="13" lengthAdjust="spacingAndGlyphs" textLength="52" x="863" y="331.2305">回传响应</text><polygon fill="#A80036" points="575.5,345.9727,565.5,349.9727,575.5,353.9727,571.5,349.9727" style="stroke: #A80036; stroke-width: 1.0;"></polygon><line style="stroke: #A80036; stroke-width: 1.0;" x1="569.5" x2="845" y1="349.9727" y2="349.9727"></line><polygon fill="#A80036" points="395.5,359.9727,385.5,363.9727,395.5,367.9727,391.5,363.9727" style="stroke: #A80036; stroke-width: 1.0;"></polygon><line style="stroke: #A80036; stroke-width: 1.0;" x1="389.5" x2="563.5" y1="363.9727" y2="363.9727"></line><polygon fill="#A80036" points="251,373.9727,241,377.9727,251,381.9727,247,377.9727" style="stroke: #A80036; stroke-width: 1.0;"></polygon><line style="stroke: #A80036; stroke-width: 1.0;" x1="245" x2="383.5" y1="377.9727" y2="377.9727"></line><polygon fill="#A80036" points="44,387.9727,34,391.9727,44,395.9727,40,391.9727" style="stroke: #A80036; stroke-width: 1.0;"></polygon><line style="stroke: #A80036; stroke-width: 1.0;" x1="38" x2="239" y1="391.9727" y2="391.9727"></line><!--
@startuml
actor socket #red
socket -> Acceptor : 客户端请求
Acceptor -> AbstractEndpoint : 接受请求
AbstractEndpoint -> AbstractProtocol : 套接字可读
AbstractProtocol -> Processor : 确定应用层协议后，将报文\n交给合适的协议处理器
box "Http11Processor" #LightBlue 
participant Http11InputBuffer
participant Http11OutputBuffer
end box
Processor -> Http11InputBuffer
Http11InputBuffer -> CoyoteAdapter : 完成报文解析

activate CoyoteAdapter 
CoyoteAdapter -> CoyoteAdapter : Catalina处理请求
CoyoteAdapter -> Http11OutputBuffer : 回传响应
deactivate CoyoteAdapter

Http11OutputBuffer -> Processor 
Processor -> AbstractProtocol 
AbstractProtocol -> AbstractEndpoint
AbstractEndpoint -> socket 
@enduml

PlantUML version 1.2018.03(Fri Apr 06 00:59:15 CST 2018)
(GPL source distribution)
Java Runtime: Java(TM) SE Runtime Environment
JVM: Java HotSpot(TM) 64-Bit Server VM
Java Version: 10.0.1+10
Operating System: Mac OS X
OS Version: 10.13.2
Default Encoding: UTF-8
Language: zh
Country: CN
--></g></svg></div>


上图是HTTP/1.1协议的NIO/NIO2的整体调用链示意图。

可以看出Coyote NIO大量使用模版模式，整体框架使用接口、抽象类组织。

* `Acceptor`是一个线程实例，只用来接受套接字。如果已经建立的套接字达到了连接最大限制，就会等待。
* `AbstractEndpoint`是底层通信框架的抽象类，NIO和NIO2的区别就在这个类，两者分别对应于`NioEndpoint`和`NioEndpoint`。
* `AbstractProtocol`就是前面说的协议类的抽象类，其中`AbstractEndpoint`和`Processor`均为其实例属性。参考下方`Http11Nio2Protocol`类图。
* `Processor`是所有协议处理器的统一抽象，而HTTP/1.1协议和AJP协议的处理器均直接继承了`AbstractProcessor`抽象类。
* `Processor`完成协议处理后，就将报文提交给了`Container`，交互接口是`Adapter`。也就是说用户如过不想用Tomcat的`Catalina`容器，可以自行实现Adapter接口。不过Tomcat的`Connector`类默认的`Adapter`实现就是`CoyoteAdapter`，而且是硬编码，所以还得实现`Connector`子类覆盖相关方法。

<div class="plantuml-diagram"><!--?xml version="1.0" encoding="UTF-8" standalone="no"?--><svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" contentscripttype="application/ecmascript" contentstyletype="text/css" height="391px" preserveAspectRatio="none" style="width:327px;height:391px;" version="1.1" viewBox="0 0 327 391" width="327px" zoomAndPan="magnify"><defs><filter height="300%" id="fpnmat672o1jj" width="300%" x="-1" y="-1"><feGaussianBlur result="blurOut" stdDeviation="2.0"></feGaussianBlur><feColorMatrix in="blurOut" result="blurOut2" type="matrix" values="0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 .4 0"></feColorMatrix><feOffset dx="4.0" dy="4.0" in="blurOut2" result="blurOut3"></feOffset><feBlend in="SourceGraphic" in2="blurOut3" mode="normal"></feBlend></filter></defs><g><!--class AbstractProtocol--><rect fill="#FEFECE" filter="url(#fpnmat672o1jj)" height="48" id="AbstractProtocol" style="stroke: #A80036; stroke-width: 1.5;" width="126" x="102" y="8"></rect><ellipse cx="117" cy="24" fill="#A9DCDF" rx="11" ry="11" style="stroke: #A80036; stroke-width: 1.0;"></ellipse><path d="M117.1133,19.3481 L115.9595,24.4199 L118.2754,24.4199 Z M115.6191,17.1069 L118.6157,17.1069 L121.9609,29.5 L119.5122,29.5 L118.7485,26.437 L115.4697,26.437 L114.7227,29.5 L112.2739,29.5 Z "></path><text fill="#000000" font-family="sans-serif" font-size="12" font-style="italic" lengthAdjust="spacingAndGlyphs" textLength="94" x="131" y="28.5352">AbstractProtocol</text><line style="stroke: #A80036; stroke-width: 1.5;" x1="103" x2="227" y1="40" y2="40"></line><line style="stroke: #A80036; stroke-width: 1.5;" x1="103" x2="227" y1="48" y2="48"></line><!--class AbstractHttp11Protocol--><rect fill="#FEFECE" filter="url(#fpnmat672o1jj)" height="48" id="AbstractHttp11Protocol" style="stroke: #A80036; stroke-width: 1.5;" width="167" x="81.5" y="116"></rect><ellipse cx="96.5" cy="132" fill="#A9DCDF" rx="11" ry="11" style="stroke: #A80036; stroke-width: 1.0;"></ellipse><path d="M96.6133,127.3481 L95.4595,132.4199 L97.7754,132.4199 Z M95.1191,125.1069 L98.1157,125.1069 L101.4609,137.5 L99.0122,137.5 L98.2485,134.437 L94.9697,134.437 L94.2227,137.5 L91.7739,137.5 Z "></path><text fill="#000000" font-family="sans-serif" font-size="12" font-style="italic" lengthAdjust="spacingAndGlyphs" textLength="135" x="110.5" y="136.5352">AbstractHttp11Protocol</text><line style="stroke: #A80036; stroke-width: 1.5;" x1="82.5" x2="247.5" y1="148" y2="148"></line><line style="stroke: #A80036; stroke-width: 1.5;" x1="82.5" x2="247.5" y1="156" y2="156"></line><!--class Http11Nio2Protocol--><rect fill="#FEFECE" filter="url(#fpnmat672o1jj)" height="48" id="Http11Nio2Protocol" style="stroke: #A80036; stroke-width: 1.5;" width="146" x="6" y="224"></rect><ellipse cx="21" cy="240" fill="#ADD1B2" rx="11" ry="11" style="stroke: #A80036; stroke-width: 1.0;"></ellipse><path d="M23.9731,245.6431 Q23.3921,245.9419 22.7529,246.0913 Q22.1138,246.2407 21.4082,246.2407 Q18.9014,246.2407 17.5815,244.5889 Q16.2617,242.937 16.2617,239.8159 Q16.2617,236.6865 17.5815,235.0347 Q18.9014,233.3828 21.4082,233.3828 Q22.1138,233.3828 22.7612,233.5322 Q23.4087,233.6816 23.9731,233.9805 L23.9731,236.7031 Q23.3423,236.1221 22.7488,235.8523 Q22.1553,235.5825 21.5244,235.5825 Q20.1797,235.5825 19.4949,236.6492 Q18.8101,237.7158 18.8101,239.8159 Q18.8101,241.9077 19.4949,242.9744 Q20.1797,244.041 21.5244,244.041 Q22.1553,244.041 22.7488,243.7712 Q23.3423,243.5015 23.9731,242.9204 Z "></path><text fill="#000000" font-family="sans-serif" font-size="12" lengthAdjust="spacingAndGlyphs" textLength="114" x="35" y="244.5352">Http11Nio2Protocol</text><line style="stroke: #A80036; stroke-width: 1.5;" x1="7" x2="151" y1="256" y2="256"></line><line style="stroke: #A80036; stroke-width: 1.5;" x1="7" x2="151" y1="264" y2="264"></line><!--class Http11Processor--><rect fill="#FEFECE" filter="url(#fpnmat672o1jj)" height="48" id="Http11Processor" style="stroke: #A80036; stroke-width: 1.5;" width="129" x="187.5" y="224"></rect><ellipse cx="202.5" cy="240" fill="#ADD1B2" rx="11" ry="11" style="stroke: #A80036; stroke-width: 1.0;"></ellipse><path d="M205.4731,245.6431 Q204.8921,245.9419 204.2529,246.0913 Q203.6138,246.2407 202.9082,246.2407 Q200.4014,246.2407 199.0815,244.5889 Q197.7617,242.937 197.7617,239.8159 Q197.7617,236.6865 199.0815,235.0347 Q200.4014,233.3828 202.9082,233.3828 Q203.6138,233.3828 204.2612,233.5322 Q204.9087,233.6816 205.4731,233.9805 L205.4731,236.7031 Q204.8423,236.1221 204.2488,235.8523 Q203.6553,235.5825 203.0244,235.5825 Q201.6797,235.5825 200.9949,236.6492 Q200.3101,237.7158 200.3101,239.8159 Q200.3101,241.9077 200.9949,242.9744 Q201.6797,244.041 203.0244,244.041 Q203.6553,244.041 204.2488,243.7712 Q204.8423,243.5015 205.4731,242.9204 Z "></path><text fill="#000000" font-family="sans-serif" font-size="12" lengthAdjust="spacingAndGlyphs" textLength="97" x="216.5" y="244.5352">Http11Processor</text><line style="stroke: #A80036; stroke-width: 1.5;" x1="188.5" x2="315.5" y1="256" y2="256"></line><line style="stroke: #A80036; stroke-width: 1.5;" x1="188.5" x2="315.5" y1="264" y2="264"></line><!--class Nio2Endpoint--><rect fill="#FEFECE" filter="url(#fpnmat672o1jj)" height="48" id="Nio2Endpoint" style="stroke: #A80036; stroke-width: 1.5;" width="110" x="24" y="332"></rect><ellipse cx="39" cy="348" fill="#ADD1B2" rx="11" ry="11" style="stroke: #A80036; stroke-width: 1.0;"></ellipse><path d="M41.9731,353.6431 Q41.3921,353.9419 40.7529,354.0913 Q40.1138,354.2407 39.4082,354.2407 Q36.9014,354.2407 35.5815,352.5889 Q34.2617,350.937 34.2617,347.8159 Q34.2617,344.6865 35.5815,343.0347 Q36.9014,341.3828 39.4082,341.3828 Q40.1138,341.3828 40.7612,341.5322 Q41.4087,341.6816 41.9731,341.9805 L41.9731,344.7031 Q41.3423,344.1221 40.7488,343.8523 Q40.1553,343.5825 39.5244,343.5825 Q38.1797,343.5825 37.4949,344.6492 Q36.8101,345.7158 36.8101,347.8159 Q36.8101,349.9077 37.4949,350.9744 Q38.1797,352.041 39.5244,352.041 Q40.1553,352.041 40.7488,351.7712 Q41.3423,351.5015 41.9731,350.9204 Z "></path><text fill="#000000" font-family="sans-serif" font-size="12" lengthAdjust="spacingAndGlyphs" textLength="78" x="53" y="352.5352">Nio2Endpoint</text><line style="stroke: #A80036; stroke-width: 1.5;" x1="25" x2="133" y1="364" y2="364"></line><line style="stroke: #A80036; stroke-width: 1.5;" x1="25" x2="133" y1="372" y2="372"></line><!--link AbstractProtocol to AbstractHttp11Protocol--><path d="M165,76.024 C165,89.579 165,104.038 165,115.678 " fill="none" id="AbstractProtocol-AbstractHttp11Protocol" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="none" points="158,76,165,56,172,76,158,76" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--link AbstractHttp11Protocol to Http11Processor--><path d="M192.154,174.084 C205.359,190.174 220.916,209.128 232.859,223.678 " fill="none" id="AbstractHttp11Protocol-Http11Processor" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="183.878,164,184.5925,171.1756,191.491,173.2759,190.7765,166.1003,183.878,164" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--link AbstractHttp11Protocol to Http11Nio2Protocol--><path d="M133.6,179.702 C121.636,194.448 108.377,210.791 97.9213,223.678 " fill="none" id="AbstractHttp11Protocol-Http11Nio2Protocol" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="none" points="128.302,175.121,146.339,164,139.174,183.942,128.302,175.121" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--link Http11Nio2Protocol to Nio2Endpoint--><path d="M79,285.3383 C79,300.6813 79,318.098 79,331.6784 " fill="none" id="Http11Nio2Protocol-Nio2Endpoint" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="79,272,75,278,79,284,83,278,79,272" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--
@startuml
abstract class AbstractProtocol 
abstract class AbstractHttp11Protocol 
class Http11Nio2Protocol
class Http11Processor
class Nio2Endpoint
AbstractProtocol <|- - AbstractHttp11Protocol
AbstractHttp11Protocol *- - Http11Processor 
AbstractHttp11Protocol <|- - Http11Nio2Protocol
Http11Nio2Protocol *- - Nio2Endpoint 
@enduml

PlantUML version 1.2018.03(Fri Apr 06 00:59:15 CST 2018)
(GPL source distribution)
Java Runtime: Java(TM) SE Runtime Environment
JVM: Java HotSpot(TM) 64-Bit Server VM
Java Version: 10.0.1+10
Operating System: Mac OS X
OS Version: 10.13.2
Default Encoding: UTF-8
Language: zh
Country: CN
--></g></svg></div>

# NIO vs NIO2

## `NIOEndpoint`的实现

`NIOEndpoint`使用Java NIO来实现，是一种Reactor模式。

<div class="plantuml-diagram"><!--?xml version="1.0" encoding="UTF-8" standalone="no"?--><svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" contentscripttype="application/ecmascript" contentstyletype="text/css" height="602px" preserveAspectRatio="none" style="width:448px;height:602px;" version="1.1" viewBox="0 0 448 602" width="448px" zoomAndPan="magnify"><defs><filter height="300%" id="f3mn5jd9aoltn" width="300%" x="-1" y="-1"><feGaussianBlur result="blurOut" stdDeviation="2.0"></feGaussianBlur><feColorMatrix in="blurOut" result="blurOut2" type="matrix" values="0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 .4 0"></feColorMatrix><feOffset dx="4.0" dy="4.0" in="blurOut2" result="blurOut3"></feOffset><feBlend in="SourceGraphic" in2="blurOut3" mode="normal"></feBlend></filter></defs><g><!--class NioEndpoint--><rect fill="#FEFECE" filter="url(#f3mn5jd9aoltn)" height="48" id="NioEndpoint" style="stroke: #A80036; stroke-width: 1.5;" width="102" x="184.5" y="8"></rect><ellipse cx="199.5" cy="24" fill="#ADD1B2" rx="11" ry="11" style="stroke: #A80036; stroke-width: 1.0;"></ellipse><path d="M202.4731,29.6431 Q201.8921,29.9419 201.2529,30.0913 Q200.6138,30.2407 199.9082,30.2407 Q197.4014,30.2407 196.0815,28.5889 Q194.7617,26.937 194.7617,23.8159 Q194.7617,20.6865 196.0815,19.0347 Q197.4014,17.3828 199.9082,17.3828 Q200.6138,17.3828 201.2612,17.5322 Q201.9087,17.6816 202.4731,17.9805 L202.4731,20.7031 Q201.8423,20.1221 201.2488,19.8523 Q200.6553,19.5825 200.0244,19.5825 Q198.6797,19.5825 197.9949,20.6492 Q197.3101,21.7158 197.3101,23.8159 Q197.3101,25.9077 197.9949,26.9744 Q198.6797,28.041 200.0244,28.041 Q200.6553,28.041 201.2488,27.7712 Q201.8423,27.5015 202.4731,26.9204 Z "></path><text fill="#000000" font-family="sans-serif" font-size="12" lengthAdjust="spacingAndGlyphs" textLength="70" x="213.5" y="28.5352">NioEndpoint</text><line style="stroke: #A80036; stroke-width: 1.5;" x1="185.5" x2="285.5" y1="40" y2="40"></line><line style="stroke: #A80036; stroke-width: 1.5;" x1="185.5" x2="285.5" y1="48" y2="48"></line><!--class NioSelectorPool--><rect fill="#FEFECE" filter="url(#f3mn5jd9aoltn)" height="73.9102" id="NioSelectorPool" style="stroke: #A80036; stroke-width: 1.5;" width="120" x="175.5" y="518"></rect><ellipse cx="190.5" cy="534" fill="#ADD1B2" rx="11" ry="11" style="stroke: #A80036; stroke-width: 1.0;"></ellipse><path d="M193.4731,539.6431 Q192.8921,539.9419 192.2529,540.0913 Q191.6138,540.2407 190.9082,540.2407 Q188.4014,540.2407 187.0815,538.5889 Q185.7617,536.937 185.7617,533.8159 Q185.7617,530.6865 187.0815,529.0347 Q188.4014,527.3828 190.9082,527.3828 Q191.6138,527.3828 192.2612,527.5322 Q192.9087,527.6816 193.4731,527.9805 L193.4731,530.7031 Q192.8423,530.1221 192.2488,529.8523 Q191.6553,529.5825 191.0244,529.5825 Q189.6797,529.5825 188.9949,530.6492 Q188.3101,531.7158 188.3101,533.8159 Q188.3101,535.9077 188.9949,536.9744 Q189.6797,538.041 191.0244,538.041 Q191.6553,538.041 192.2488,537.7712 Q192.8423,537.5015 193.4731,536.9204 Z "></path><text fill="#000000" font-family="sans-serif" font-size="12" lengthAdjust="spacingAndGlyphs" textLength="88" x="204.5" y="538.5352">NioSelectorPool</text><line style="stroke: #A80036; stroke-width: 1.5;" x1="176.5" x2="294.5" y1="550" y2="550"></line><line style="stroke: #A80036; stroke-width: 1.5;" x1="176.5" x2="294.5" y1="558" y2="558"></line><text fill="#000000" font-family="sans-serif" font-size="11" lengthAdjust="spacingAndGlyphs" textLength="43" x="181.5" y="572.6348">write(...)</text><text fill="#000000" font-family="sans-serif" font-size="11" lengthAdjust="spacingAndGlyphs" textLength="41" x="181.5" y="585.5898">read(...)</text><!--class Poller--><rect fill="#FEFECE" filter="url(#f3mn5jd9aoltn)" height="86.8652" id="Poller" style="stroke: #A80036; stroke-width: 1.5;" width="162" x="154.5" y="116"></rect><ellipse cx="215.25" cy="132" fill="#ADD1B2" rx="11" ry="11" style="stroke: #A80036; stroke-width: 1.0;"></ellipse><path d="M218.2231,137.6431 Q217.6421,137.9419 217.0029,138.0913 Q216.3638,138.2407 215.6582,138.2407 Q213.1514,138.2407 211.8315,136.5889 Q210.5117,134.937 210.5117,131.8159 Q210.5117,128.6865 211.8315,127.0347 Q213.1514,125.3828 215.6582,125.3828 Q216.3638,125.3828 217.0112,125.5322 Q217.6587,125.6816 218.2231,125.9805 L218.2231,128.7031 Q217.5923,128.1221 216.9988,127.8523 Q216.4053,127.5825 215.7744,127.5825 Q214.4297,127.5825 213.7449,128.6492 Q213.0601,129.7158 213.0601,131.8159 Q213.0601,133.9077 213.7449,134.9744 Q214.4297,136.041 215.7744,136.041 Q216.4053,136.041 216.9988,135.7712 Q217.5923,135.5015 218.2231,134.9204 Z "></path><text fill="#000000" font-family="sans-serif" font-size="12" lengthAdjust="spacingAndGlyphs" textLength="32" x="235.75" y="136.5352">Poller</text><line style="stroke: #A80036; stroke-width: 1.5;" x1="155.5" x2="315.5" y1="148" y2="148"></line><line style="stroke: #A80036; stroke-width: 1.5;" x1="155.5" x2="315.5" y1="156" y2="156"></line><text fill="#000000" font-family="sans-serif" font-size="11" lengthAdjust="spacingAndGlyphs" textLength="27" x="160.5" y="170.6348">run()</text><text fill="#000000" font-family="sans-serif" font-size="11" lengthAdjust="spacingAndGlyphs" textLength="150" x="160.5" y="183.5898">register(NioChannel socket)</text><text fill="#000000" font-family="sans-serif" font-size="11" lengthAdjust="spacingAndGlyphs" textLength="79" x="160.5" y="196.5449">processKey(...)</text><!--class PollerEvent--><rect fill="#FEFECE" filter="url(#f3mn5jd9aoltn)" height="60.9551" id="PollerEvent" style="stroke: #A80036; stroke-width: 1.5;" width="95" x="188" y="263"></rect><ellipse cx="203" cy="279" fill="#ADD1B2" rx="11" ry="11" style="stroke: #A80036; stroke-width: 1.0;"></ellipse><path d="M205.9731,284.6431 Q205.3921,284.9419 204.7529,285.0913 Q204.1138,285.2407 203.4082,285.2407 Q200.9014,285.2407 199.5815,283.5889 Q198.2617,281.937 198.2617,278.8159 Q198.2617,275.6865 199.5815,274.0347 Q200.9014,272.3828 203.4082,272.3828 Q204.1138,272.3828 204.7612,272.5322 Q205.4087,272.6816 205.9731,272.9805 L205.9731,275.7031 Q205.3423,275.1221 204.7488,274.8523 Q204.1553,274.5825 203.5244,274.5825 Q202.1797,274.5825 201.4949,275.6492 Q200.8101,276.7158 200.8101,278.8159 Q200.8101,280.9077 201.4949,281.9744 Q202.1797,283.041 203.5244,283.041 Q204.1553,283.041 204.7488,282.7712 Q205.3423,282.5015 205.9731,281.9204 Z "></path><text fill="#000000" font-family="sans-serif" font-size="12" lengthAdjust="spacingAndGlyphs" textLength="63" x="217" y="283.5352">PollerEvent</text><line style="stroke: #A80036; stroke-width: 1.5;" x1="189" x2="282" y1="295" y2="295"></line><line style="stroke: #A80036; stroke-width: 1.5;" x1="189" x2="282" y1="303" y2="303"></line><text fill="#000000" font-family="sans-serif" font-size="11" lengthAdjust="spacingAndGlyphs" textLength="27" x="194" y="317.6348">run()</text><!--class SocketProcessor--><rect fill="#FEFECE" filter="url(#f3mn5jd9aoltn)" height="60.9551" id="SocketProcessor" style="stroke: #A80036; stroke-width: 1.5;" width="125" x="6" y="390.5"></rect><ellipse cx="21" cy="406.5" fill="#ADD1B2" rx="11" ry="11" style="stroke: #A80036; stroke-width: 1.0;"></ellipse><path d="M23.9731,412.1431 Q23.3921,412.4419 22.7529,412.5913 Q22.1138,412.7407 21.4082,412.7407 Q18.9014,412.7407 17.5815,411.0889 Q16.2617,409.437 16.2617,406.3159 Q16.2617,403.1865 17.5815,401.5347 Q18.9014,399.8828 21.4082,399.8828 Q22.1138,399.8828 22.7612,400.0322 Q23.4087,400.1816 23.9731,400.4805 L23.9731,403.2031 Q23.3423,402.6221 22.7488,402.3523 Q22.1553,402.0825 21.5244,402.0825 Q20.1797,402.0825 19.4949,403.1492 Q18.8101,404.2158 18.8101,406.3159 Q18.8101,408.4077 19.4949,409.4744 Q20.1797,410.541 21.5244,410.541 Q22.1553,410.541 22.7488,410.2712 Q23.3423,410.0015 23.9731,409.4204 Z "></path><text fill="#000000" font-family="sans-serif" font-size="12" lengthAdjust="spacingAndGlyphs" textLength="93" x="35" y="411.0352">SocketProcessor</text><line style="stroke: #A80036; stroke-width: 1.5;" x1="7" x2="130" y1="422.5" y2="422.5"></line><line style="stroke: #A80036; stroke-width: 1.5;" x1="7" x2="130" y1="430.5" y2="430.5"></line><text fill="#000000" font-family="sans-serif" font-size="11" lengthAdjust="spacingAndGlyphs" textLength="43" x="12" y="445.1348">doRun()</text><!--class NioSocketWrapper--><rect fill="#FEFECE" filter="url(#f3mn5jd9aoltn)" height="73.9102" id="NioSocketWrapper" style="stroke: #A80036; stroke-width: 1.5;" width="138" x="166.5" y="384"></rect><ellipse cx="181.5" cy="400" fill="#ADD1B2" rx="11" ry="11" style="stroke: #A80036; stroke-width: 1.0;"></ellipse><path d="M184.4731,405.6431 Q183.8921,405.9419 183.2529,406.0913 Q182.6138,406.2407 181.9082,406.2407 Q179.4014,406.2407 178.0815,404.5889 Q176.7617,402.937 176.7617,399.8159 Q176.7617,396.6865 178.0815,395.0347 Q179.4014,393.3828 181.9082,393.3828 Q182.6138,393.3828 183.2612,393.5322 Q183.9087,393.6816 184.4731,393.9805 L184.4731,396.7031 Q183.8423,396.1221 183.2488,395.8523 Q182.6553,395.5825 182.0244,395.5825 Q180.6797,395.5825 179.9949,396.6492 Q179.3101,397.7158 179.3101,399.8159 Q179.3101,401.9077 179.9949,402.9744 Q180.6797,404.041 182.0244,404.041 Q182.6553,404.041 183.2488,403.7712 Q183.8423,403.5015 184.4731,402.9204 Z "></path><text fill="#000000" font-family="sans-serif" font-size="12" lengthAdjust="spacingAndGlyphs" textLength="106" x="195.5" y="404.5352">NioSocketWrapper</text><line style="stroke: #A80036; stroke-width: 1.5;" x1="167.5" x2="303.5" y1="416" y2="416"></line><line style="stroke: #A80036; stroke-width: 1.5;" x1="167.5" x2="303.5" y1="424" y2="424"></line><text fill="#000000" font-family="sans-serif" font-size="11" lengthAdjust="spacingAndGlyphs" textLength="88" x="172.5" y="438.6348">fillReadBuffer(...)</text><text fill="#000000" font-family="sans-serif" font-size="11" lengthAdjust="spacingAndGlyphs" textLength="58" x="172.5" y="451.5898">doWrite(...)</text><!--class NioChannel--><rect fill="#FEFECE" filter="url(#f3mn5jd9aoltn)" height="48" id="NioChannel" style="stroke: #A80036; stroke-width: 1.5;" width="97" x="340" y="397"></rect><ellipse cx="355" cy="413" fill="#ADD1B2" rx="11" ry="11" style="stroke: #A80036; stroke-width: 1.0;"></ellipse><path d="M357.9731,418.6431 Q357.3921,418.9419 356.7529,419.0913 Q356.1138,419.2407 355.4082,419.2407 Q352.9014,419.2407 351.5815,417.5889 Q350.2617,415.937 350.2617,412.8159 Q350.2617,409.6865 351.5815,408.0347 Q352.9014,406.3828 355.4082,406.3828 Q356.1138,406.3828 356.7612,406.5322 Q357.4087,406.6816 357.9731,406.9805 L357.9731,409.7031 Q357.3423,409.1221 356.7488,408.8523 Q356.1553,408.5825 355.5244,408.5825 Q354.1797,408.5825 353.4949,409.6492 Q352.8101,410.7158 352.8101,412.8159 Q352.8101,414.9077 353.4949,415.9744 Q354.1797,417.041 355.5244,417.041 Q356.1553,417.041 356.7488,416.7712 Q357.3423,416.5015 357.9731,415.9204 Z "></path><text fill="#000000" font-family="sans-serif" font-size="12" lengthAdjust="spacingAndGlyphs" textLength="65" x="369" y="417.5352">NioChannel</text><line style="stroke: #A80036; stroke-width: 1.5;" x1="341" x2="436" y1="429" y2="429"></line><line style="stroke: #A80036; stroke-width: 1.5;" x1="341" x2="436" y1="437" y2="437"></line><!--link NioEndpoint to Poller--><path d="M235.5,69.141 C235.5,83.718 235.5,100.648 235.5,115.825 " fill="none" id="NioEndpoint-Poller" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="235.5,56.019,231.5,62.019,235.5,68.019,239.5,62.019,235.5,56.019" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--link Poller to PollerEvent--><path d="M235.5,216.458 C235.5,232.378 235.5,249.097 235.5,262.791 " fill="none" id="Poller-PollerEvent" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="235.5,203.357,231.5,209.357,235.5,215.357,239.5,209.357,235.5,203.357" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--link PollerEvent to NioChannel--><path d="M281.958,332.607 C307.451,353.518 338.236,378.77 360.145,396.741 " fill="none" id="PollerEvent-NioChannel" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="271.759,324.242,273.8608,331.14,281.0366,331.8529,278.9348,324.9549,271.759,324.242" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--link PollerEvent to NioSocketWrapper--><path d="M235.5,337.484 C235.5,352.546 235.5,369.3 235.5,383.755 " fill="none" id="PollerEvent-NioSocketWrapper" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="235.5,324.242,231.5,330.242,235.5,336.242,239.5,330.242,235.5,324.242" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--link PollerEvent to SocketProcessor--><path d="M185.249,332.263 C160.432,350.914 130.93,373.084 107.793,390.472 " fill="none" id="PollerEvent-SocketProcessor" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="195.923,324.242,188.7234,324.6488,186.3298,331.4511,193.5294,331.0443,195.923,324.242" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--link NioSocketWrapper to NioSelectorPool--><path d="M235.5,471.256 C235.5,486.755 235.5,503.5733 235.5,517.974 " fill="none" id="NioSocketWrapper-NioSelectorPool" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="235.5,458.07,231.5,464.07,235.5,470.07,239.5,464.07,235.5,458.07" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--
@startuml
class NioEndpoint
class NioSelectorPool
NioSelectorPool : write(...)
NioSelectorPool : read(...)
class Poller
Poller : run()
Poller : register(NioChannel socket)
Poller : processKey(...)
class PollerEvent
PollerEvent : run()
class SocketProcessor
SocketProcessor : doRun()
class NioSocketWrapper
NioSocketWrapper : fillReadBuffer(...)
NioSocketWrapper : doWrite(...)
class NioChannel
NioEndpoint *- - Poller
Poller *- - PollerEvent
PollerEvent *- - NioChannel 
PollerEvent *- - NioSocketWrapper 
PollerEvent *- - SocketProcessor
NioSocketWrapper *- - NioSelectorPool 
@enduml

PlantUML version 1.2018.03(Fri Apr 06 00:59:15 CST 2018)
(GPL source distribution)
Java Runtime: Java(TM) SE Runtime Environment
JVM: Java HotSpot(TM) 64-Bit Server VM
Java Version: 10.0.1+10
Operating System: Mac OS X
OS Version: 10.13.2
Default Encoding: UTF-8
Language: zh
Country: CN
--></g></svg></div>


Java NIO需要在用户态维护多路复用器`Selector`。业务线程在`Selector`上注册相应套接字上希望发生的事件（可读、可写等等）。`Selector`是对操作系统的多路复用器的封装，也就是说这个“兴趣”实际上注册到了内核态。`Selector`会阻塞式等候内核通知，一旦有事件可用，就会检查注册在自己上的所有套接字句柄，如果相应的注册事件发生了，就会通知注册方执行下一步动作。`Selector`的阻塞式等候导致其必须占用一个线程来运行。

在`NIOEndpoint`里使用`Poller`线程对象来实现`Selector`的阻塞式等候。一个`NIOEndpoint`对象中最多有2个`Poller`线程对象。当客户端的请求套接字完成接受后，`Poller`就会调用`register`方法，将`NioChannel`对象和对应的`NioSocketWrapper`对象绑定在同一个`PollerEvent`对象里。`NioChannel`对象就是对套接字的封装，`NioSocketWrapper`对象使用`NioSelectPool`实现了对报文的阻塞式接受和回写。当`Poller`发现感兴趣的事件发生时，会调用`processKey`方法，将`NioSocketWrapper`对象注入到`SocketProcessor`私有类中，在线程池中调用`AbstractProtocol`处理套接字。

## `NIO2Endpoint`的实现

`NIO2Endpoint`使用Java NIO2来实现，是一种Proactor模式。

<div class="plantuml-diagram"><!--?xml version="1.0" encoding="UTF-8" standalone="no"?--><svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" contentscripttype="application/ecmascript" contentstyletype="text/css" height="283px" preserveAspectRatio="none" style="width:323px;height:283px;" version="1.1" viewBox="0 0 323 283" width="323px" zoomAndPan="magnify"><defs><filter height="300%" id="frlea4a6fxcf4" width="300%" x="-1" y="-1"><feGaussianBlur result="blurOut" stdDeviation="2.0"></feGaussianBlur><feColorMatrix in="blurOut" result="blurOut2" type="matrix" values="0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 .4 0"></feColorMatrix><feOffset dx="4.0" dy="4.0" in="blurOut2" result="blurOut3"></feOffset><feBlend in="SourceGraphic" in2="blurOut3" mode="normal"></feBlend></filter></defs><g><!--class Nio2Endpoint--><rect fill="#FEFECE" filter="url(#frlea4a6fxcf4)" height="48" id="Nio2Endpoint" style="stroke: #A80036; stroke-width: 1.5;" width="110" x="109" y="8"></rect><ellipse cx="124" cy="24" fill="#ADD1B2" rx="11" ry="11" style="stroke: #A80036; stroke-width: 1.0;"></ellipse><path d="M126.9731,29.6431 Q126.3921,29.9419 125.7529,30.0913 Q125.1138,30.2407 124.4082,30.2407 Q121.9014,30.2407 120.5815,28.5889 Q119.2617,26.937 119.2617,23.8159 Q119.2617,20.6865 120.5815,19.0347 Q121.9014,17.3828 124.4082,17.3828 Q125.1138,17.3828 125.7612,17.5322 Q126.4087,17.6816 126.9731,17.9805 L126.9731,20.7031 Q126.3423,20.1221 125.7488,19.8523 Q125.1553,19.5825 124.5244,19.5825 Q123.1797,19.5825 122.4949,20.6492 Q121.8101,21.7158 121.8101,23.8159 Q121.8101,25.9077 122.4949,26.9744 Q123.1797,28.041 124.5244,28.041 Q125.1553,28.041 125.7488,27.7712 Q126.3423,27.5015 126.9731,26.9204 Z "></path><text fill="#000000" font-family="sans-serif" font-size="12" lengthAdjust="spacingAndGlyphs" textLength="78" x="138" y="28.5352">Nio2Endpoint</text><line style="stroke: #A80036; stroke-width: 1.5;" x1="110" x2="218" y1="40" y2="40"></line><line style="stroke: #A80036; stroke-width: 1.5;" x1="110" x2="218" y1="48" y2="48"></line><!--class Nio2SocketWrapper--><rect fill="#FEFECE" filter="url(#frlea4a6fxcf4)" height="48" id="Nio2SocketWrapper" style="stroke: #A80036; stroke-width: 1.5;" width="146" x="6" y="116"></rect><ellipse cx="21" cy="132" fill="#ADD1B2" rx="11" ry="11" style="stroke: #A80036; stroke-width: 1.0;"></ellipse><path d="M23.9731,137.6431 Q23.3921,137.9419 22.7529,138.0913 Q22.1138,138.2407 21.4082,138.2407 Q18.9014,138.2407 17.5815,136.5889 Q16.2617,134.937 16.2617,131.8159 Q16.2617,128.6865 17.5815,127.0347 Q18.9014,125.3828 21.4082,125.3828 Q22.1138,125.3828 22.7612,125.5322 Q23.4087,125.6816 23.9731,125.9805 L23.9731,128.7031 Q23.3423,128.1221 22.7488,127.8523 Q22.1553,127.5825 21.5244,127.5825 Q20.1797,127.5825 19.4949,128.6492 Q18.8101,129.7158 18.8101,131.8159 Q18.8101,133.9077 19.4949,134.9744 Q20.1797,136.041 21.5244,136.041 Q22.1553,136.041 22.7488,135.7712 Q23.3423,135.5015 23.9731,134.9204 Z "></path><text fill="#000000" font-family="sans-serif" font-size="12" lengthAdjust="spacingAndGlyphs" textLength="114" x="35" y="136.5352">Nio2SocketWrapper</text><line style="stroke: #A80036; stroke-width: 1.5;" x1="7" x2="151" y1="148" y2="148"></line><line style="stroke: #A80036; stroke-width: 1.5;" x1="7" x2="151" y1="156" y2="156"></line><!--class SocketProcessor--><rect fill="#FEFECE" filter="url(#frlea4a6fxcf4)" height="48" id="SocketProcessor" style="stroke: #A80036; stroke-width: 1.5;" width="125" x="187.5" y="116"></rect><ellipse cx="202.5" cy="132" fill="#ADD1B2" rx="11" ry="11" style="stroke: #A80036; stroke-width: 1.0;"></ellipse><path d="M205.4731,137.6431 Q204.8921,137.9419 204.2529,138.0913 Q203.6138,138.2407 202.9082,138.2407 Q200.4014,138.2407 199.0815,136.5889 Q197.7617,134.937 197.7617,131.8159 Q197.7617,128.6865 199.0815,127.0347 Q200.4014,125.3828 202.9082,125.3828 Q203.6138,125.3828 204.2612,125.5322 Q204.9087,125.6816 205.4731,125.9805 L205.4731,128.7031 Q204.8423,128.1221 204.2488,127.8523 Q203.6553,127.5825 203.0244,127.5825 Q201.6797,127.5825 200.9949,128.6492 Q200.3101,129.7158 200.3101,131.8159 Q200.3101,133.9077 200.9949,134.9744 Q201.6797,136.041 203.0244,136.041 Q203.6553,136.041 204.2488,135.7712 Q204.8423,135.5015 205.4731,134.9204 Z "></path><text fill="#000000" font-family="sans-serif" font-size="12" lengthAdjust="spacingAndGlyphs" textLength="93" x="216.5" y="136.5352">SocketProcessor</text><line style="stroke: #A80036; stroke-width: 1.5;" x1="188.5" x2="311.5" y1="148" y2="148"></line><line style="stroke: #A80036; stroke-width: 1.5;" x1="188.5" x2="311.5" y1="156" y2="156"></line><!--class Nio2Channel--><rect fill="#FEFECE" filter="url(#frlea4a6fxcf4)" height="48" id="Nio2Channel" style="stroke: #A80036; stroke-width: 1.5;" width="105" x="26.5" y="224"></rect><ellipse cx="41.5" cy="240" fill="#ADD1B2" rx="11" ry="11" style="stroke: #A80036; stroke-width: 1.0;"></ellipse><path d="M44.4731,245.6431 Q43.8921,245.9419 43.2529,246.0913 Q42.6138,246.2407 41.9082,246.2407 Q39.4014,246.2407 38.0815,244.5889 Q36.7617,242.937 36.7617,239.8159 Q36.7617,236.6865 38.0815,235.0347 Q39.4014,233.3828 41.9082,233.3828 Q42.6138,233.3828 43.2612,233.5322 Q43.9087,233.6816 44.4731,233.9805 L44.4731,236.7031 Q43.8423,236.1221 43.2488,235.8523 Q42.6553,235.5825 42.0244,235.5825 Q40.6797,235.5825 39.9949,236.6492 Q39.3101,237.7158 39.3101,239.8159 Q39.3101,241.9077 39.9949,242.9744 Q40.6797,244.041 42.0244,244.041 Q42.6553,244.041 43.2488,243.7712 Q43.8423,243.5015 44.4731,242.9204 Z "></path><text fill="#000000" font-family="sans-serif" font-size="12" lengthAdjust="spacingAndGlyphs" textLength="73" x="55.5" y="244.5352">Nio2Channel</text><line style="stroke: #A80036; stroke-width: 1.5;" x1="27.5" x2="130.5" y1="256" y2="256"></line><line style="stroke: #A80036; stroke-width: 1.5;" x1="27.5" x2="130.5" y1="264" y2="264"></line><!--link Nio2Endpoint to Nio2SocketWrapper--><path d="M137.101,66.545 C124.279,82.534 109.258,101.267 97.7013,115.678 " fill="none" id="Nio2Endpoint-Nio2SocketWrapper" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="145.556,56,138.6818,58.1783,138.0486,65.3615,144.9228,63.1832,145.556,56" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--link Nio2Endpoint to SocketProcessor--><path d="M191.216,66.545 C204.188,82.534 219.386,101.267 231.079,115.678 " fill="none" id="Nio2Endpoint-SocketProcessor" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="182.661,56,183.3346,63.1796,190.221,65.3191,189.5474,58.1396,182.661,56" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--link Nio2SocketWrapper to Nio2Channel--><path d="M79,177.3383 C79,192.6813 79,210.098 79,223.6784 " fill="none" id="Nio2SocketWrapper-Nio2Channel" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="79,164,75,170,79,176,83,170,79,164" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--
@startuml
class Nio2Endpoint
class Nio2SocketWrapper
class SocketProcessor
class Nio2Channel
Nio2Endpoint *- - Nio2SocketWrapper
Nio2Endpoint *- - SocketProcessor 
Nio2SocketWrapper *- - Nio2Channel 
@enduml

PlantUML version 1.2018.03(Fri Apr 06 00:59:15 CST 2018)
(GPL source distribution)
Java Runtime: Java(TM) SE Runtime Environment
JVM: Java HotSpot(TM) 64-Bit Server VM
Java Version: 10.0.1+10
Operating System: Mac OS X
OS Version: 10.13.2
Default Encoding: UTF-8
Language: zh
Country: CN
--></g></svg></div>

尽管NIO2的`AsynchronousServerSocketChannel`也支持使用`CompletionHandler`回调接口来异步实现接受套接字请求，但是Tomcat NIO2还是使用阻塞式的`Future`对象来接收请求。这估计是出于和NIO兼容复用代码的考虑。

Tomcat NIO2主要是用`AsynchronousChannelGroup`完成异步调用。`AsynchronousServerSocketChannel`首先阻塞式的接受客户端请求，得到`AsynchronousSocketChannel`套接字对象，然后注入到一个`Nio2Channel`对象里。这个`Nio2Channel`对象也被注入到`Nio2SocketWrapper`对象里面，再将`Nio2SocketWrapper`对象注入到`SocketProcessor`私有类对象里面，然后在线程池中调用`AbstractProtocol`处理套接字。

在Tomcat NIO2中，`Nio2SocketWrapper`中的报文接受与回写也是阻塞式的。

## 对比一下

NIO2明显比NIO的结构要简单。这其实得归功于Proactor模式。Proactor模式屏蔽了事件细节，用户只需要明确事件和回调函数就可以了。Proactor框架可以自动触发回调函数。

而NIO的Reactor模式，需要用户自行处理事件发生之后的所有事情（`Poller`线程）。

不过，NIO2虽然逻辑简单，但是其背后的Proactor的实现依赖于具体的操作系统环境。Windows内核提供了真正的Proactor API，而Linux的网络IO依然是Epoll。所以Linux环境下的NIO2实际上是在JDK内部对Epoll的封装。

因此说，NIO2和NIO的性能孰优孰劣，是需要测试才能明确的。

## 一点心得

Tomcat在处理请求的过程中，大量使用对象池缓存开销较大的对象，例如`ByteBuffer`、`byte[]`等。这个在实际运行中，可以感受到很明显的优化效果。我在Chrome的Postman插件中测试的时候发现，一般新启动的Tomcat在处理第一个请求的响应时间一般在200+ms，但是在处理第二个请求的时候，响应时间立刻缩短到了两位数。

这一点确实很值得学习。

# 参考资料

* [详解 Tomcat 配置文件 server.xml](https://mp.weixin.qq.com/s?src=11&timestamp=1530356294&ver=970&signature=x9*Td1lrIUqv-7bHVHtLzQS2EFQ9GaTXpnaVevKmSdsDVPMUOt-XTAkOHNmgmj9f*jtbyWWsuWWfCFN4mFRz24yDhZvGw-2OCIVaHyoeC*zgVRoq-pTnXDZhmMH49EA6&new=1)
* [Apache Tomcat 9 Configuration Reference](https://tomcat.apache.org/tomcat-9.0-doc/config/index.html)
* [异步通道 API](https://www.ibm.com/developerworks/cn/java/j-nio2-1/)
* [两种高性能I/O设计模式(Reactor/Proactor)的比较](http://blog.jobbole.com/59676/)
* [聊聊Tomcat中的NIO2通道](http://www.10tiao.com/html/308/201609/2650076338/1.html)

# 附：如何嵌入式启动Tomcat

一开始研究Tomcat，发现嵌入式启动搞不定。有可能是因为我的Tomcat 9版本太新的缘故，网上找了很多嵌入式启动的代码都是无效的。后来在Spring Boot找到了代码才解决了这个问题。原来这些代码都没有设置Connector。

下面是我使用的调试代码。

```Java
import org.apache.catalina.Context;
import org.apache.catalina.LifecycleException;
import org.apache.catalina.connector.Connector;
import org.apache.catalina.startup.Tomcat;
import org.apache.coyote.http2.Http2Protocol;
import org.apache.tomcat.util.net.Acceptor;
import org.apache.tomcat.util.net.Nio2Endpoint;
import org.apache.tomcat.util.net.NioEndpoint;

import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.io.Writer;

public class App {

    public static class HelloWorld extends HttpServlet {
        private static final long serialVersionUID = 1L;
        @Override
        public void doGet(HttpServletRequest req, HttpServletResponse res)
                throws IOException {
            Writer w = res.getWriter();
            w.write("Hello, Tomcat!");
            w.flush();
            w.close();
        }
    }
    
    public static void main(String[] args) throws LifecycleException {
        Tomcat tomcat = new Tomcat();
        String mWorkingDir = System.getProperty("java.io.tmpdir");
        tomcat.setBaseDir(mWorkingDir);
        Connector connector = new Connector("org.apache.coyote.http11.Http11Nio2Protocol");
        connector.setPort(8080);
        tomcat.setConnector(connector);
        Context ctx = tomcat.addContext("/1", null);
        Tomcat.addServlet(ctx, "hi", new HelloWorld());
        ctx.addServletMappingDecoded("/hi", "hi", false);
        tomcat.start();
        tomcat.getServer().await();
    }
}

```

访问`http://localhost:8080/1/hi`应该能够得到响应`Hello, Tomcat!`。



