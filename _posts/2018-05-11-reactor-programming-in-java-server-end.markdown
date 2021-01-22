---
layout: post
title: Java的http服务端响应式编程
date: 2018-05-11 14:33:12
categories: 原创
---


**为什么要响应式编程？**

# 传统的Servlet模型走到了尽头

传统的Java服务器编程遵循的是J2EE的Servlet规范，是一种基于线程的模型：每一次http请求都由一个线程来处理。

<div class="plantuml-diagram"><!--?xml version="1.0" encoding="UTF-8" standalone="no"?--><svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" contentscripttype="application/ecmascript" contentstyletype="text/css" height="340px" preserveAspectRatio="none" style="width:185px;height:340px;" version="1.1" viewBox="0 0 185 340" width="185px" zoomAndPan="magnify"><defs><filter height="300%" id="f1o4ml790i3vp4" width="300%" x="-1" y="-1"><feGaussianBlur result="blurOut" stdDeviation="2.0"></feGaussianBlur><feColorMatrix in="blurOut" result="blurOut2" type="matrix" values="0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 .4 0"></feColorMatrix><feOffset dx="4.0" dy="4.0" in="blurOut2" result="blurOut3"></feOffset><feBlend in="SourceGraphic" in2="blurOut3" mode="normal"></feBlend></filter></defs><g><line style="stroke: #A80036; stroke-width: 1.0;" x1="38" x2="38" y1="38.4883" y2="77.7988"></line><line style="stroke: #A80036; stroke-width: 1.0; stroke-dasharray: 1.0,4.0;" x1="38" x2="38" y1="77.7988" y2="105.7988"></line><line style="stroke: #A80036; stroke-width: 1.0;" x1="38" x2="38" y1="105.7988" y2="232.7305"></line><line style="stroke: #A80036; stroke-width: 1.0; stroke-dasharray: 1.0,4.0;" x1="38" x2="38" y1="232.7305" y2="260.7305"></line><line style="stroke: #A80036; stroke-width: 1.0;" x1="38" x2="38" y1="260.7305" y2="300.041"></line><line style="stroke: #A80036; stroke-width: 1.0;" x1="114" x2="114" y1="38.4883" y2="77.7988"></line><line style="stroke: #A80036; stroke-width: 1.0; stroke-dasharray: 1.0,4.0;" x1="114" x2="114" y1="77.7988" y2="105.7988"></line><line style="stroke: #A80036; stroke-width: 1.0;" x1="114" x2="114" y1="105.7988" y2="232.7305"></line><line style="stroke: #A80036; stroke-width: 1.0; stroke-dasharray: 1.0,4.0;" x1="114" x2="114" y1="232.7305" y2="260.7305"></line><line style="stroke: #A80036; stroke-width: 1.0;" x1="114" x2="114" y1="260.7305" y2="300.041"></line><rect fill="#FEFECE" filter="url(#f1o4ml790i3vp4)" height="30.4883" style="stroke: #A80036; stroke-width: 1.5;" width="56" x="8" y="3"></rect><text fill="#000000" font-family="sans-serif" font-size="14" lengthAdjust="spacingAndGlyphs" textLength="42" x="15" y="23.5352">客户端</text><rect fill="#FEFECE" filter="url(#f1o4ml790i3vp4)" height="30.4883" style="stroke: #A80036; stroke-width: 1.5;" width="56" x="8" y="299.041"></rect><text fill="#000000" font-family="sans-serif" font-size="14" lengthAdjust="spacingAndGlyphs" textLength="42" x="15" y="319.5762">客户端</text><rect fill="#FEFECE" filter="url(#f1o4ml790i3vp4)" height="30.4883" style="stroke: #A80036; stroke-width: 1.5;" width="56" x="84" y="3"></rect><text fill="#000000" font-family="sans-serif" font-size="14" lengthAdjust="spacingAndGlyphs" textLength="42" x="91" y="23.5352">服务器</text><rect fill="#FEFECE" filter="url(#f1o4ml790i3vp4)" height="30.4883" style="stroke: #A80036; stroke-width: 1.5;" width="56" x="84" y="299.041"></rect><text fill="#000000" font-family="sans-serif" font-size="14" lengthAdjust="spacingAndGlyphs" textLength="42" x="91" y="319.5762">服务器</text><polygon fill="#A80036" points="102,65.7988,112,69.7988,102,73.7988,106,69.7988" style="stroke: #A80036; stroke-width: 1.0;"></polygon><line style="stroke: #A80036; stroke-width: 1.0;" x1="38" x2="108" y1="69.7988" y2="69.7988"></line><text fill="#000000" font-family="sans-serif" font-size="13" lengthAdjust="spacingAndGlyphs" textLength="52" x="45" y="65.0566">发送请求</text><line style="stroke: #A80036; stroke-width: 1.0; stroke-dasharray: 2.0,2.0;" x1="114" x2="156" y1="127.1094" y2="127.1094"></line><line style="stroke: #A80036; stroke-width: 1.0; stroke-dasharray: 2.0,2.0;" x1="156" x2="156" y1="127.1094" y2="140.1094"></line><line style="stroke: #A80036; stroke-width: 1.0; stroke-dasharray: 2.0,2.0;" x1="115" x2="156" y1="140.1094" y2="140.1094"></line><polygon fill="#A80036" points="125,136.1094,115,140.1094,125,144.1094,121,140.1094" style="stroke: #A80036; stroke-width: 1.0;"></polygon><text fill="#000000" font-family="sans-serif" font-size="13" lengthAdjust="spacingAndGlyphs" textLength="26" x="121" y="122.3672">解码</text><line style="stroke: #A80036; stroke-width: 1.0;" x1="114" x2="156" y1="169.4199" y2="169.4199"></line><line style="stroke: #A80036; stroke-width: 1.0;" x1="156" x2="156" y1="169.4199" y2="182.4199"></line><line style="stroke: #A80036; stroke-width: 1.0;" x1="115" x2="156" y1="182.4199" y2="182.4199"></line><polygon fill="#A80036" points="125,178.4199,115,182.4199,125,186.4199,121,182.4199" style="stroke: #A80036; stroke-width: 1.0;"></polygon><text fill="#000000" font-family="sans-serif" font-size="13" lengthAdjust="spacingAndGlyphs" textLength="52" x="121" y="164.6777">处理请求</text><line style="stroke: #A80036; stroke-width: 1.0; stroke-dasharray: 2.0,2.0;" x1="114" x2="156" y1="211.7305" y2="211.7305"></line><line style="stroke: #A80036; stroke-width: 1.0; stroke-dasharray: 2.0,2.0;" x1="156" x2="156" y1="211.7305" y2="224.7305"></line><line style="stroke: #A80036; stroke-width: 1.0; stroke-dasharray: 2.0,2.0;" x1="115" x2="156" y1="224.7305" y2="224.7305"></line><polygon fill="#A80036" points="125,220.7305,115,224.7305,125,228.7305,121,224.7305" style="stroke: #A80036; stroke-width: 1.0;"></polygon><text fill="#000000" font-family="sans-serif" font-size="13" lengthAdjust="spacingAndGlyphs" textLength="26" x="121" y="206.9883">编码</text><polygon fill="#A80036" points="49,278.041,39,282.041,49,286.041,45,282.041" style="stroke: #A80036; stroke-width: 1.0;"></polygon><line style="stroke: #A80036; stroke-width: 1.0;" x1="43" x2="113" y1="282.041" y2="282.041"></line><text fill="#000000" font-family="sans-serif" font-size="13" lengthAdjust="spacingAndGlyphs" textLength="52" x="55" y="277.2988">返回结果</text><!--
@startuml

客户端 -> 服务器: 发送请求
...
服务器 - -> 服务器: 解码
服务器 -> 服务器: 处理请求
服务器 - -> 服务器: 编码
...
服务器 -> 客户端: 返回结果

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

线程模型的缺陷在于，每一条线程都要自行处理套接字的读写操作。对于大部分请求来讲，本地处理请求的速度很快，请求的读取和返回是最耗时间的。也就是说大量的线程浪费在了远程连接上，而没有发挥出计算能力。但是需要注意一点，线程的创建是有开销的，每一条线程都需要独立的内存资源。JVM里的-Xss参数就是用来调整线程堆栈大小的。而JVM堆的总大小局限在了-Xmx参数上，因此一个正在运行的JVM服务器能够同时运行的线程数是固定的。

即便通过调整JVM参数，使其能够运行更多线程。但是JVM的线程会映射成为操作系统的用户线程，而操作系统依然只能调度有限数量的线程。例如，Linux系统可以参考这里的讨论：[Maximum number of threads per process in Linux?](https://stackoverflow.com/questions/344203/maximum-number-of-threads-per-process-in-linux)。

此外，大量线程在切换的时候，也会产生上下文加载卸载的开销，同样会降低系统的性能。

# 可伸缩 IO

Doug Lea大神有一篇很经典的PPT[Scalable IO in Java](http://gee.cs.oswego.edu/dl/cpjslides/nio.pdf)，讲述了一个更为优秀的服务器模型。

>一个可伸缩的网络服务系统应当满足以下条件：
>
>1. 能够随着计算资源（CPU、内存、磁盘容量、网络带宽等）的增加提高负载能力。
>2. 当网络负载增加超过能力的时候，能够优雅降级，避免直接崩溃。例如，拒绝为超过能力范围的请求提供服务，但对于能力范围内的请求，依然提供服务。当流量洪峰过去之后，依然能够正常运行。
>3. 当然高可用、高性能依然是必须的：例如低响应延迟、随负载变化请求或释放计算资源等。

作者给出的解决方案就是Reactor模式。

Reactor模式将耗时的IO资源封装为handle对象。handle对象注册在操作系统的内核里，当对象满足一定的条件时（可读或者可写），才会处理handle对象。在Reactor模式中，同步多路复用器负责处理handle对象的状态变更，当满足条件时，会调用handle对象注册时提供的回调函数。

同步多路复用器在一个单独的线程里专门处理IO链接。当请求读取完毕之后，任务提交至工作线程池完成请求的解码、处理、编码等工作，最后将由多路复用器负责将结果返回给客户端，而池内线程继续处理下一个任务。相比JDK1.5之前的对每一次请求新建一个线程的方式，线程池能够实现线程复用，降低创建回收线程的开销，在应对密集计算负载的时候有更好的表现。同时，在多个线程上分别部署一个同步多路复用器，也可以更好地利用多核CPU的处理能力。

<div class="plantuml-diagram"><!--?xml version="1.0" encoding="UTF-8" standalone="no"?--><svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" contentscripttype="application/ecmascript" contentstyletype="text/css" height="1323px" preserveAspectRatio="none" style="width:268px;height:1323px;" version="1.1" viewBox="0 0 268 1323" width="268px" zoomAndPan="magnify"><defs><filter height="300%" id="f182r26ex8sn5k" width="300%" x="-1" y="-1"><feGaussianBlur result="blurOut" stdDeviation="2.0"></feGaussianBlur><feColorMatrix in="blurOut" result="blurOut2" type="matrix" values="0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 .4 0"></feColorMatrix><feOffset dx="4.0" dy="4.0" in="blurOut2" result="blurOut3"></feOffset><feBlend in="SourceGraphic" in2="blurOut3" mode="normal"></feBlend></filter></defs><g><!--cluster 请求多路复用器组--><polygon fill="#FFFFE0" filter="url(#f182r26ex8sn5k)" points="60,72.84,178,72.84,185,95.3283,212,95.3283,212,369.56,60,369.56,60,72.84" style="stroke: #000000; stroke-width: 1.5;"></polygon><line style="stroke: #000000; stroke-width: 1.5;" x1="60" x2="185" y1="95.3283" y2="95.3283"></line><text fill="#000000" font-family="sans-serif" font-size="14" font-weight="bold" lengthAdjust="spacingAndGlyphs" textLength="112" x="64" y="88.3752">请求多路复用器组</text><!--cluster 线程池--><polygon fill="#ADD8E6" filter="url(#f182r26ex8sn5k)" points="30,418.56,78,418.56,85,441.0483,246,441.0483,246,901.28,30,901.28,30,418.56" style="stroke: #000000; stroke-width: 1.5;"></polygon><line style="stroke: #000000; stroke-width: 1.5;" x1="30" x2="85" y1="441.0483" y2="441.0483"></line><text fill="#000000" font-family="sans-serif" font-size="14" font-weight="bold" lengthAdjust="spacingAndGlyphs" textLength="42" x="34" y="434.0952">线程池</text><!--cluster 响应多路复用器组--><polygon fill="#90EE90" filter="url(#f182r26ex8sn5k)" points="86,950.28,204,950.28,211,972.7683,238,972.7683,238,1247,86,1247,86,950.28" style="stroke: #000000; stroke-width: 1.5;"></polygon><line style="stroke: #000000; stroke-width: 1.5;" x1="86" x2="211" y1="972.7683" y2="972.7683"></line><text fill="#000000" font-family="sans-serif" font-size="14" font-weight="bold" lengthAdjust="spacingAndGlyphs" textLength="112" x="90" y="965.8152">响应多路复用器组</text><rect fill="#FEFECE" filter="url(#f182r26ex8sn5k)" height="34.1328" rx="12.5" ry="12.5" style="stroke: #A80036; stroke-width: 1.5;" width="104" x="84" y="145.56"></rect><text fill="#000000" font-family="sans-serif" font-size="12" lengthAdjust="spacingAndGlyphs" textLength="84" x="94" y="167.1616">请求多路复用器</text><polygon fill="#FEFECE" filter="url(#f182r26ex8sn5k)" points="129,233.56,141,245.56,129,257.56,117,245.56,129,233.56" style="stroke: #A80036; stroke-width: 1.5;"></polygon><rect fill="#FEFECE" filter="url(#f182r26ex8sn5k)" height="34.1328" rx="12.5" ry="12.5" style="stroke: #A80036; stroke-width: 1.5;" width="68" x="95" y="311.56"></rect><text fill="#000000" font-family="sans-serif" font-size="12" lengthAdjust="spacingAndGlyphs" textLength="48" x="105" y="333.1616">提交请求</text><polygon fill="#FEFECE" filter="url(#f182r26ex8sn5k)" points="129,491.28,141,503.28,129,515.28,117,503.28,129,491.28" style="stroke: #A80036; stroke-width: 1.5;"></polygon><rect fill="#000000" filter="url(#f182r26ex8sn5k)" height="8" style="stroke: none; stroke-width: 1.0;" width="80" x="54" y="582.28"></rect><rect fill="#FEFECE" filter="url(#f182r26ex8sn5k)" height="34.1328" rx="12.5" ry="12.5" style="stroke: #A80036; stroke-width: 1.5;" width="68" x="154" y="569.28"></rect><text fill="#000000" font-family="sans-serif" font-size="12" lengthAdjust="spacingAndGlyphs" textLength="48" x="164" y="590.8816">拒绝服务</text><rect fill="#FEFECE" filter="url(#f182r26ex8sn5k)" height="34.1328" rx="12.5" ry="12.5" style="stroke: #A80036; stroke-width: 1.5;" width="52" x="54" y="644.28"></rect><text fill="#000000" font-family="sans-serif" font-size="12" lengthAdjust="spacingAndGlyphs" textLength="32" x="64" y="665.8816">解码1</text><rect fill="#FEFECE" filter="url(#f182r26ex8sn5k)" height="34.1328" rx="12.5" ry="12.5" style="stroke: #A80036; stroke-width: 1.5;" width="52" x="54" y="719.28"></rect><text fill="#000000" font-family="sans-serif" font-size="12" lengthAdjust="spacingAndGlyphs" textLength="32" x="64" y="740.8816">处理1</text><rect fill="#FEFECE" filter="url(#f182r26ex8sn5k)" height="34.1328" rx="12.5" ry="12.5" style="stroke: #A80036; stroke-width: 1.5;" width="52" x="54" y="794.28"></rect><text fill="#000000" font-family="sans-serif" font-size="12" lengthAdjust="spacingAndGlyphs" textLength="32" x="64" y="815.8816">编码1</text><rect fill="#000000" filter="url(#f182r26ex8sn5k)" height="8" style="stroke: none; stroke-width: 1.0;" width="80" x="98" y="869.28"></rect><rect fill="#FEFECE" filter="url(#f182r26ex8sn5k)" height="34.1328" rx="12.5" ry="12.5" style="stroke: #A80036; stroke-width: 1.5;" width="52" x="126" y="644.28"></rect><text fill="#000000" font-family="sans-serif" font-size="12" lengthAdjust="spacingAndGlyphs" textLength="32" x="136" y="665.8816">解码2</text><rect fill="#FEFECE" filter="url(#f182r26ex8sn5k)" height="34.1328" rx="12.5" ry="12.5" style="stroke: #A80036; stroke-width: 1.5;" width="52" x="126" y="719.28"></rect><text fill="#000000" font-family="sans-serif" font-size="12" lengthAdjust="spacingAndGlyphs" textLength="32" x="136" y="740.8816">处理2</text><rect fill="#FEFECE" filter="url(#f182r26ex8sn5k)" height="34.1328" rx="12.5" ry="12.5" style="stroke: #A80036; stroke-width: 1.5;" width="52" x="126" y="794.28"></rect><text fill="#000000" font-family="sans-serif" font-size="12" lengthAdjust="spacingAndGlyphs" textLength="32" x="136" y="815.8816">编码2</text><rect fill="#FEFECE" filter="url(#f182r26ex8sn5k)" height="34.1328" rx="12.5" ry="12.5" style="stroke: #A80036; stroke-width: 1.5;" width="104" x="110" y="1023"></rect><text fill="#000000" font-family="sans-serif" font-size="12" lengthAdjust="spacingAndGlyphs" textLength="84" x="120" y="1044.6016">响应多路复用器</text><polygon fill="#FEFECE" filter="url(#f182r26ex8sn5k)" points="155,1111,167,1123,155,1135,143,1123,155,1111" style="stroke: #A80036; stroke-width: 1.5;"></polygon><rect fill="#FEFECE" filter="url(#f182r26ex8sn5k)" height="34.1328" rx="12.5" ry="12.5" style="stroke: #A80036; stroke-width: 1.5;" width="68" x="121" y="1189"></rect><text fill="#000000" font-family="sans-serif" font-size="12" lengthAdjust="spacingAndGlyphs" textLength="48" x="131" y="1210.6016">关闭连接</text><ellipse cx="136" cy="17.84" fill="#000000" filter="url(#f182r26ex8sn5k)" rx="10" ry="10" style="stroke: none; stroke-width: 1.0;"></ellipse><ellipse cx="155" cy="1302" fill="none" filter="url(#f182r26ex8sn5k)" rx="10" ry="10" style="stroke: #000000; stroke-width: 1.0;"></ellipse><ellipse cx="155.5" cy="1302.5" fill="#000000" rx="6" ry="6" style="stroke: none; stroke-width: 1.0;"></ellipse><!--link start to 请求多路复用器组--><path d="M136,28.06 C136,38.285 136,55.295 136,70.1187 C136,70.582 136,71.0431 136,71.5018 C136,71.7311 136,71.9599 136,72.188 C136,72.3021 136,72.416 136,72.5297 " fill="none" id="start-请求多路复用器组" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="136,72.5297,140,63.5297,136,67.5297,132,63.5297,136,72.5297" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--link 请求多路复用器组 to 请求多路复用器--><path d="M136,100.6 C136,101.7 136,123.05 136,140.18 " fill="none" id="请求多路复用器组-请求多路复用器" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="136,145.33,140,136.33,136,140.33,132,136.33,136,145.33" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--link 请求多路复用器 to #7--><path d="M126.44,179.86 C123.437,185.85 120.512,192.82 119,199.56 C116.518,210.63 119.85,223.24 123.319,232.29 " fill="none" id="请求多路复用器-#7" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="125.225,236.91,125.4785,227.0644,123.3126,232.2902,118.0868,230.1243,125.225,236.91" style="stroke: #A80036; stroke-width: 1.0;"></polygon><text fill="#000000" font-family="sans-serif" font-size="11" lengthAdjust="spacingAndGlyphs" textLength="79" x="30.2668" y="230.3187">socket读取完成</text><!--link #7 to 请求多路复用器--><path d="M129.892,234.24 C130.964,221.83 132.792,200.68 134.169,184.75 " fill="none" id="#7-请求多路复用器" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="134.617,179.56,129.8357,188.1704,134.1742,184.5404,137.8043,188.8789,134.617,179.56" style="stroke: #A80036; stroke-width: 1.0;"></polygon><text fill="#000000" font-family="sans-serif" font-size="11" lengthAdjust="spacingAndGlyphs" textLength="25" x="133" y="211.1948">false</text><!--link #7 to 提交请求--><path d="M129,257.82 C129,270.32 129,290.77 129,306.3 " fill="none" id="#7-提交请求" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="129,311.36,133,302.36,129,306.36,125,302.36,129,311.36" style="stroke: #A80036; stroke-width: 1.0;"></polygon><text fill="#000000" font-family="sans-serif" font-size="11" lengthAdjust="spacingAndGlyphs" textLength="22" x="130" y="289.1948">true</text><!--link 提交请求 to 线程池--><path d="M129,345.6 C129,361.2015 129,385.5055 129,406.166 C129,408.7486 129,411.2742 129,413.7188 C129,414.9411 129,416.1431 129,417.3219 C129,417.6166 129,417.9098 129,418.2015 " fill="none" id="提交请求-线程池" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="129,418.2015,133,409.2015,129,413.2015,125,409.2015,129,418.2015" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--link 线程池 to #15--><path d="M129,446.441 C129,448.516 129,470.252 129,485.982 " fill="none" id="线程池-#15" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="129,491.049,133,482.049,129,486.049,125,482.049,129,491.049" style="stroke: #A80036; stroke-width: 1.0;"></polygon><text fill="#000000" font-family="sans-serif" font-size="11" lengthAdjust="spacingAndGlyphs" textLength="55" x="60.5938" y="481.029">线程池可用</text><!--link #15 to B1--><path d="M125.644,512.047 C119,527.423 104.2322,561.6 97.4847,577.215 " fill="none" id="#15-B1" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="95.3916,582.059,102.6346,575.3852,97.3757,577.4695,95.2914,572.2107,95.3916,582.059" style="stroke: #A80036; stroke-width: 1.0;"></polygon><text fill="#000000" font-family="sans-serif" font-size="11" lengthAdjust="spacingAndGlyphs" textLength="22" x="116" y="546.9148">true</text><!--link #15 to 拒绝服务--><path d="M133.828,510.909 C142.302,522.542 160.176,547.08 173.079,564.795 " fill="none" id="#15-拒绝服务" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="176.243,569.139,174.1765,559.5094,173.2988,565.0977,167.7105,564.2201,176.243,569.139" style="stroke: #A80036; stroke-width: 1.0;"></polygon><text fill="#000000" font-family="sans-serif" font-size="11" lengthAdjust="spacingAndGlyphs" textLength="25" x="163" y="546.9148">false</text><!--link B1 to 解码1--><path d="M93.3984,590.417 C91.8142,598.677 87.4204,621.588 84.0926,638.94 " fill="none" id="B1-解码1" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="83.0984,644.124,88.7214,636.0381,84.0398,639.2134,80.8645,634.5318,83.0984,644.124" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--link 解码1 to 处理1--><path d="M80,678.481 C80,688.972 80,702.743 80,714.098 " fill="none" id="解码1-处理1" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="80,719.124,84,710.124,80,714.124,76,710.124,80,719.124" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--link 处理1 to 编码1--><path d="M80,753.481 C80,763.972 80,777.743 80,789.098 " fill="none" id="处理1-编码1" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="80,794.124,84,785.124,80,789.124,76,785.124,80,794.124" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--link 编码1 to B2--><path d="M95.5429,828.359 C107.078,840.291 122.262,855.999 130.996,865.034 " fill="none" id="编码1-B2" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="134.721,868.888,131.3419,859.6369,131.246,865.293,125.5899,865.197,134.721,868.888" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--link B1 to 解码2--><path d="M96.4922,590.417 C103.1684,598.82 121.89,622.382 135.754,639.832 " fill="none" id="B1-解码2" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="139.164,644.124,136.6988,634.5887,136.0543,640.2087,130.4343,639.5642,139.164,644.124" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--link 解码2 to 处理2--><path d="M152,678.481 C152,688.972 152,702.743 152,714.098 " fill="none" id="解码2-处理2" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="152,719.124,156,710.124,152,714.124,148,710.124,152,719.124" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--link 处理2 to 编码2--><path d="M152,753.481 C152,763.972 152,777.743 152,789.098 " fill="none" id="处理2-编码2" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="152,794.124,156,785.124,152,789.124,148,785.124,152,794.124" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--link 编码2 to B2--><path d="M148.248,828.359 C145.591,839.749 142.13,854.579 139.988,863.762 " fill="none" id="编码2-B2" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="138.792,868.888,144.732,861.032,139.9279,864.0187,136.9412,859.2146,138.792,868.888" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--link B2 to 响应多路复用器组--><path d="M138.799,877.686 C140.8485,886.4275 146.501,910.5355 151.7133,932.766 C153.0163,938.3236 154.2919,943.7639 155.4767,948.8174 C155.5508,949.1333 155.6245,949.4476 155.6978,949.7603 C155.7345,949.9167 155.771,950.0727 155.8075,950.2283 " fill="none" id="B2-响应多路复用器组" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="155.8075,950.2283,157.6475,940.5528,154.6662,945.3603,149.8587,942.379,155.8075,950.2283" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--link 拒绝服务 to 响应多路复用器组--><path d="M195.463,603.424 C201.499,617.969 209,640.103 209,660.28 C209,660.28 209,660.28 209,874.28 C209,898.5275 197.973,923.3008 186.6014,942.4015 C185.1799,944.7891 183.7531,947.0881 182.3417,949.2863 C182.1653,949.5611 181.9891,949.8343 181.8132,950.1059 " fill="none" id="拒绝服务-响应多路复用器组" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="181.8132,950.1059,190.0629,944.726,184.5311,945.9091,183.348,940.3773,181.8132,950.1059" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--link 响应多路复用器组 to 响应多路复用器--><path d="M162,978.044 C162,979.143 162,1000.495 162,1017.621 " fill="none" id="响应多路复用器组-响应多路复用器" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="162,1022.773,166,1013.773,162,1017.773,158,1013.773,162,1022.773" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--link 响应多路复用器 to #42--><path d="M152.44,1057.299 C149.437,1063.287 146.512,1070.256 145,1077 C142.518,1088.069 145.85,1100.677 149.319,1109.726 " fill="none" id="响应多路复用器-#42" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="151.225,1114.354,151.4993,1104.509,149.3223,1109.7302,144.1011,1107.5532,151.225,1114.354" style="stroke: #A80036; stroke-width: 1.0;"></polygon><text fill="#000000" font-family="sans-serif" font-size="11" lengthAdjust="spacingAndGlyphs" textLength="79" x="56.2668" y="1107.7641">socket写入完成</text><!--link #42 to 响应多路复用器--><path d="M155.892,1111.68 C156.964,1099.27 158.792,1078.124 160.169,1062.188 " fill="none" id="#42-响应多路复用器" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="160.617,1057.003,155.8578,1065.6257,160.187,1061.9845,163.8282,1066.3137,160.617,1057.003" style="stroke: #A80036; stroke-width: 1.0;"></polygon><text fill="#000000" font-family="sans-serif" font-size="11" lengthAdjust="spacingAndGlyphs" textLength="25" x="159" y="1088.6348">false</text><!--link #42 to 关闭连接--><path d="M155,1135.263 C155,1147.76 155,1168.211 155,1183.743 " fill="none" id="#42-关闭连接" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="155,1188.803,159,1179.803,155,1183.803,151,1179.803,155,1188.803" style="stroke: #A80036; stroke-width: 1.0;"></polygon><text fill="#000000" font-family="sans-serif" font-size="11" lengthAdjust="spacingAndGlyphs" textLength="22" x="156" y="1166.6348">true</text><!--link 关闭连接 to end--><path d="M155,1223.3736 C155,1241.2373 155,1269.556 155,1286.6007 " fill="none" id="关闭连接-end" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#A80036" points="155,1291.9175,159,1282.9175,155,1286.9175,151,1282.9175,155,1291.9175" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--
@startuml

(*) - -> "请求多路复用器组"

partition 请求多路复用器组 #LightYellow {

- -> "请求多路复用器"
if "socket读取完成" then
- ->[true] "提交请求"
else
- ->[false] "请求多路复用器"
endif

"提交请求" - -> "线程池"
}

partition 线程池 #LightBlue {
if "线程池可用" then
- ->[true] ===B1===
else
- ->[false] "拒绝服务"
endif


===B1=== - -> "解码1"  
- -> "处理1" 
- -> "编码1" 
- -> ===B2===

===B1=== - -> "解码2"  
- -> "处理2" 
- -> "编码2" 
- -> ===B2===

===B2=== - -> "响应多路复用器组"
"拒绝服务" - -> "响应多路复用器组"
}

partition 响应多路复用器组 #LightGreen {
- -> "响应多路复用器"

if "socket写入完成" then
- ->[true] "关闭连接"
else
- ->[false] "响应多路复用器"
endif
}

"关闭连接" - -> (*)

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

这样，线程的任务分工就很明确，分别专门处理IO密集任务和专门处理CPU密集任务。

# NIO普及艰难

从最早的select到后来Linux的epoll和BSD的Kqueue，操作系统的多路复用性能一直在不断增强。

JDK 1.4引入了NIO模块，屏蔽了操作系统层面的细节，将各个系统的多路复用API做了统一封装。JDK的NIO有以下几个核心组件：

* Buffer，一种容量在创建时被固定的数据容器
* Charset，负责数据的编解码工作
* Channel，对远程连接的抽象
* Selector，多路复用选择器

网络连接封装在Channels对象里面。Channels在Selector上注册感兴趣的SelectionKey事件：可读OP_READ、可写OP_WRITE、可连接OP_CONNECT还有服务器端套接字才有的可接入OP_ACCEPT。多路复用选择器调用阻塞式select方法的时，在等待某一事件可用，然后就通知Channels执行相应的handler。Buffer是Channels实现读写操作的缓冲区。Charset用于对Buffer的内容进行编解码。在NIO模式下，Selector能够管理多个套接字的网络读写，避免了过多计算线程阻塞在读写请求上。

在JVM之外的世界里，多路复用通过Nginx、基于V8引擎的Node.js早就大放异彩。但是Java NIO在生产环境里的发展却很慢。例如，Tomcat直到2016年发布8.5版本的时候，才彻底移除BIO连接器，完全拥抱NIO。

JDK NIO主要有这样几个问题比较麻烦：

1. 首先是NIO为了提高数据收发性能，可以创建DirectBuffer对象。该对象的内存开辟在JVM堆之外，无法通过正常的GC收集器来回收，只能在JVM的老年代触发全量GC的时候回收。而全量GC往往导致系统卡顿，降低响应效率。如果被动等待老年代区域自行触发全量GC，又有可能造成堆外内存溢出。两者之间的矛盾需要在开发的时候小心的平衡。
2. 其次就是，JDK1.8依然存在的epoll bug：若Selector的轮询结果为空，也没有wakeup或新消息处理，则发生空轮询，CPU使用率100%。

# Netty才是NIO该有的水准

作为一个第三方框架，Netty做到了JDK本应做到的事情。

*Netty的数据容器ByteBuf更为优秀*。

ByteBuf同时维护两个索引：读索引和写索引。从而保证容器对象能够同时适配读写同时进行的场景。而NIO的Buffer却需要执行一次flip操作来适应读写场景的切换。同时ByteBuf容器使用引用计数来手工管理，可以在引用计数归零时通过反射调用jdk.internal.ref.Cleaner来回收内存，避免泄露。在GC低效的时候，选择使用手工方式来管理内存，完全没问题。

*Netty的API封装度更高*。

观察一下Netty官网Tutorial给出的[demo](http://netty.io/wiki/user-guide-for-4.x.html#writing-a-discard-server)，只要几十行代码就完成了一个具备Reactor模式的服务器。ServerBootstrap的group方法定义了主套接字和子套接字的处理方式，例中使用的NioEventLoopGroup类为Java NIO + 多线程的实现方式。对于NIO的epoll bug，NioEventLoopGroup的解决方案是rebuildSelectors对象方法。这个方法允许在selector失效时重建新的selector，将旧的释放掉。此外，Netty还通过JNI实现了自己的EpollEventLoopGroup，规避了NIO版本的bug。

Netty使用责任链模式实现了对server进出站消息的处理，使得server的代码能够更好的扩展和维护。

Netty在生产领域得到大量应用，Hadoop Avro、Dubbo、RocketMQ、Undertow等广泛应用于生产领域的产品的通信组件都选择了Netty作为基础，并经受住了考验。

Netty是一个优秀的异步通信框架，但是主要应用在基础组件中。因为Netty向开发者暴露出大量的细节，对于业务系统的开发仍然形成了困扰，所以没法得到进一步的普及。

举个例子。Netty使用ChannelFuture来接收传入的请求。相比于JDK的Future实现，ChannelFuture可以添加一组GenericFutureListener来管理对象状态，避免了反复对Future对象状态的询问或阻塞获取。这是个进步。但是，这些Listener都带来了另一个问题——Callback hell。而嵌套的回调代码往往难以维护。

# 对于Callback hell，我们可以做什么

Netty做一个优秀的基础组件就很好了。业务层面的问题就让我们用业务层面的API来解决。

## Java API的适应性不佳

### JDK7以前的异步代码难以组织

在JDK7以及之前，Java多线程的编程工具主要就是Thread、ExecutorService、Future以及相关的同步工具，实现出来的代码较为繁琐、且性能不高。

#### Thread

举个例子A，考虑一个场景有X、P、Q三个逻辑需要执行，其中X的执行需要在P、Q一起完成之后才启动执行。

如果使用Thread，那么代码会是这个样子：

```Java
/* 创建线程 */
Thread a = new Thread(new Runnable() {
    @Override
    public void run() {
        /* P逻辑 */
    }
});

Thread b = new Thread(new Runnable() {
    @Override
    public void run() {
        /* Q逻辑 */
    }
});

/* 启动线程 */
a.start();
b.start();

/* 等候a、b线程执行结束 */
try {
    a.join();
    b.join();
} catch (InterruptedException e) {
    e.printStackTrace();
}

/* 启动X逻辑的执行 */
Thread c = new Thread(new Runnable() {
    @Override
    public void run() {
        /* X逻辑 */
    }
});
c.start();

...

```

上面这个代码，先不论线程创建的开销，单从形式上看，线程内部的执行逻辑、线程本身的调度逻辑，还有必须捕获的InterruptedException的异常处理逻辑混杂在一起，整体很混乱。假想一下，当业务逻辑填充在其中的时候，代码更难维护。

#### ThreadPoolExecutor、Future

ThreadPoolExecutor和Future有助于实现线程复用，但对于代码逻辑的规范没什么帮助。

```Java
ExecutorService pool = Executors.newCachedThreadPool();
Future<?> a = pool.submit(new Runnable() {
    @Override
    public void run() {
        /* P逻辑 */
    }
});
Future<?> b = pool.submit(new Runnable() {
    @Override
    public void run() {
        /* Q逻辑 */
    }
});

/* 获取线程执行结果
 * 依然要捕获异常，处理逻辑
 */
try {
    a.get();
    b.get();
    Future<?> c = pool.submit(new Runnable() {
        @Override
        public void run() {
            /* X逻辑 */
        }
    });
} catch (InterruptedException e) {
    e.printStackTrace();
} catch (ExecutionException e) {
    e.printStackTrace();
}

```

### JDK8代码可读性有了显著提高

JDK8借鉴了相当多的函数式编程的特点，提供了几样很称手的工具。

#### CompleteableFuture和ForkJoinPool

如果要用CompleteableFuture实现上一个例子，可以这样写。

```Java
CompletableFuture<?> a = CompletableFuture.runAsync(() -> {
    /* P逻辑 */
}).exceptionally(ex -> {
    /* 异常处理逻辑 */
    return ...;
});
CompletableFuture<?> b = CompletableFuture.runAsync(() -> {
    /* Q逻辑 */
});
CompletableFuture<?> c = CompletableFuture.allOf(a, b).thenRun(() -> {
    /* X逻辑 */
});
```

有了lambda表达式的加持，例中的代码整体以线程内部逻辑为主，调度逻辑通过allOf()、thenRun()等方法名直观地展示出来。特别是可选的异常捕获逻辑，更是使得代码可读性得到了极大的提高。

需要注意的是，CompletableFuture是可以使用指定ExecutorService来执行的。如果像上例那样没有指定ExecutorService对象，那么会默认使用ForkJoinPool里的静态对象commonPool来执行。而ForkJoinPool.commonPool作为一个JVM实例中唯一的对象，也是Stream并发流的执行器，因此应当尽量保证CompletableFuture里的逻辑不会阻塞线程。如果无法规避，可以使用ManagedBlocker来降低影响。


ForkJoinPool是JDK1.7提供的并发线程池，可以很好地应对计算密集型并发任务，特别适用于可以“分-治”的任务。传统的ThreadPoolExecutor需要指定线程池里的线程数量，而ForkJoinPool使用了一个相似但更有弹性的概念——“并发度”。并发度指的是池内的活跃线程数。对于可能的阻塞任务，ForkJoinPool设计了一个ManagedBlocker接口。当池内线程执行到`ForkJoinPool.managedBlock(ForkJoinPool.ManagedBlocker blocker)`方法时，线程池会新建一个线程去执行队列里的其他任务，并轮询该对象的isReleasable方法，决定是否恢复线程继续运行。JDK1.7里的Phaser类源码用到了这个方法。

关于CompleteableFuture的用法，推荐看看这篇博客:[理解CompletableFuture](http://kriszhang.com/CompletableFuture/)，总结的很好。
而对于ForkJoinPool，可以看看这篇博客:[Java 并发编程笔记：如何使用 ForkJoinPool 以及原理](http://blog.dyngr.com/blog/2016/09/15/java-forkjoinpool-internals/)。

#### Stream

Stream流也是JDK8引入的一个很好的编程工具。

Stream对象通常通过Iterator、Collection来构造。也可以用StreamSupport的stream静态方法来创建自定义行为的实例。

Stream流对象采用链式编程风格，可以制定一系列对流的定制行为，例如过滤、排序、转化、迭代，最后产生结果。看个例子。

```Java
List<Integer> intList = List.of(1, 2, 3);

List<String> strList = intList.stream()
        .filter(k -> k>1)
        .map(String::valueOf)
        .collect(Collectors.toList());
```

上面这段代码中，intList通过stream方法获取到流对象，然后筛选出大于1的元素，并通过String的valueOf静态方法生成String对象，最后将各个String对象收集为一个列表strList。就像CompletableFuture的方法名一样，Stream的方法名都是自描述的，使得代码可读性极佳。

除此之外，Stream流的计算还是惰性的。Stream流对象的方法大致分为两种：

* 中间方法，例如filter、map等对流的改变
* 终结方法，例如collect、forEach等可以结束流

只有在执行终结方法的时候，流的计算才会真正执行。之前的中间方法，都作为步骤记录下来，但没有实时地执行修改操作。

如果将例子里的stream方法修改为parallelStream，那么得到的流对象就是一个并发流，而且总在ForkJoinPool.commonPool中执行。

关于Stream，极力推荐Brian Goetz大神的系列文章[Java Streams](https://www.ibm.com/developerworks/cn/java/j-java-streams-1-brian-goetz/index.html)。

#### 还有一点问题

ForkJoinPool是一款强大的线程池组件，只要使用的得当，线程池总会保持一个合理的并发度，充分利用计算资源。

但是，CompleteableFuture也好，Stream也好，他们都存在一个相同的问题：**无法通过后端线程池的负载变化，来调整前端的调用压力**。打比方说，当后端的ForkJoinPool.commonPool在全力运算而且队列里有大量的任务排队时，新提交的任务很可能会有很高的响应延迟，但是前端的CompleteableFuture或者Stream没有途径去获取这样一个状态，来延缓任务的提交。这种情况就违背了“响应式系统”的“灵敏性”要求。


## 来自第三方API的福音

### Reactive Streams

[Reactive Streams](https://github.com/reactive-streams/reactive-streams-jvm)是一套标准，定义了一个运行于JVM平台上的响应式编程框架实现所应该具备的行
为。

Reactive Streams规范衍生自“观察者模式”，将前后依赖的逻辑流，拆解为事件和订阅者。只有当事件发生变更时，感兴趣的观察者才随之执行随后的逻辑。Reactive Stream和JDK的Stream的理念有点接近，两者都是注重对数据流的控制。紧耦合的逻辑流拆分为“订阅-发布”方式其实是一大进步。代码变得维护性更强，而且很容易随着业务的需要按照消息驱动模式拆解。

Reactive Streams规范定义了四种接口：

* Publisher，负责生产数据流，每一个订阅者都会调用subscribe方法来订阅消息。
* Subscriber，就是订阅者。
* Subscription，其实就是一个订单选项，相当于饭馆里的菜单，由发布者传递给订阅者。
* Processor，处于数据流的中间位置，即是订阅者，也是新数据流的生产者。

当Subscriber调用Publisher.subscribe方法订阅消息时，Publisher就会调用Subscriber的onSubscribe方法，回传一个Subscription菜单。

Subscription菜单包含两个选择：

1. 一个是request方法，对数据流的请求，参数为所请求的数据流的数量，最大为Long.MAX_VALUE；
2. 另一个是cancel方法，对数据流订阅的取消，需要注意的是数据流或许会继续发送一段时间，以满足之前的请求调用。

一个Subscription对象只能由同一个Subscriber调用，所以不存在对象共享的问题。因此即便Subscription对象有状态，也不会危及逻辑链路的线程安全。

订阅者Subscriber还需要定义三种行为：

1. onNext，接受到数据流之后的执行逻辑；
2. onError，当发布出现错误的时候如何应对；
3. onComplete，当订阅的数据流发送完毕之后的行为。

相比于Future、Thread那样将业务逻辑和异常处理逻辑混杂在一起，Subscriber将其分别定义在三个方法里，代码显得更为清晰。java.util.Observer（在JDK9中开始废弃）只定义了update方法，相当于这里的onNext方法，相比之下Subscriber增加了对流整体的管理和对异常的处理。异常如果随着调用链传递出去，调试定位会非常麻烦。因此要重视onError方法，尽可能在订阅者内部就处理这个异常。

尽管Reactive Streams规范和Stream都关注数据流，但两者有一个显著的区别。那就是Stream是基于生产一方的，生产者有多大能力，Stream就制造多少数据流。而Reactive Streams规范是基于消费者的。逻辑链下游可以通过对request方法参数的变更，通知上游调整生产数据流的速度。从而实现了“响应式系统”的“灵敏性”要求。这在响应式编程中，用术语“背压”来描述。

Reactive Streams规范仅仅是一个标准，其实现又依赖其他组织的成果。其意义在于各家实现能够通过这样一个统一的接口，相互调用，有助于响应式框架生态的良性发展。Reactive Streams规范虽然是Netflix、Pivatol、RedHat等第三方大厂合作推出的，但已经随着JDK9的发布收编为官方API，位于java.util.concurrent.Flow之内。JDK8也可以在项目中直接集成相应的模块调用。

顺便吐槽一下，JDK9官方文档给出的demo里的数据流居然从Subscription里生产出来，吓得我反复确认了一下Reactive Streams官方规范。

### RxJava2

RxJava由Netfilx维护，实现了ReactiveX API规范。该规范有[很多语言实现](http://reactivex.io/languages.html)，生态很丰富。

Rx范式最先是微软在.NET平台上实现的。2014年11月，Netfilx将Rx移植到JVM平台，发布了1.0稳定版本。而Reactive Streams规范是在2015年首次发布，2017年才形成稳定版本。所以RxJava 1.x和Reactive Streams规范有很大出入。1.x版本迭代至2018年3月的1.3.8版本时，宣布停止维护。

Netflix在2016年11月发布2.0.1稳定版本，实现了和Reactive Streams规范的兼容。2.x如今是官方的推荐版本。

RxJava框架里主要有这些概念：

* Observable与Observer。RxJava直接复用了“观察者模式”里的概念，有助于更快地被开发社区接受。Observeble和Publisher有一点差异：前者有“冷热”的区分，“冷”表示只有订阅的时候才发布消息流，“热”表示消息流的发布与时候有对象订阅无关。Publisher更像是“冷”的Observeble。
* Operators，也就是操作符。RxJava和JDK Stream类似，但设计了更多的自描述的函数方法，并同样实现了链式编程。这些方法包括但不限于转换、过滤、结合等等。
* Single，是一种特殊的Observable。一般的Observable能够产生数据流，而Single只能产生一个数据。所以Single不需要onNext、onComplete方法，而是用一个onSuccess取而代之。
* Subject，注意这个不是事件，而是介于Observable与Observer之间的中介对象，类似于Reactive Streams规范里的Processor。
* Scheduler，是一类线程池，用于处理并发任务。RxJava默认执行在主线程上，可以通过observeOn/subscribeOn方法来异步调用阻塞式任务。

RxJava 2.x在Zuul 2、Hystrix、Jersey等项目都有使用，在生产领域已经得到了应用。

### Reactor3

Reactor3有Pivotal来开发维护，也就是Spring的同门师弟。

整体上，Reactor3框架里的概念和RxJava都是类似的。Mono和Flux都等同于RxJava的Single和Observable。Reactor3也使用自描述的操作符函数实现链式编程。

RxJava 2.x支持JVM 6+平台，对老旧项目很友好；而Reactor3要求必须是JVM8+。所以说，如果是新项目，使用Reactor3更好，因为它使用了很多新的API，支撑很多函数式接口，代码可读性维护性都更好。

背靠Spring大树，Reactor3的设计目标是服务器端的Java项目。Reactor社区针对服务器端，不断推出新产品，例如Reactor Netty、Reactor Kafka等等。但如果是Android项目，RxJava2更为合适（来自Reactor3官方文档的建议）。

老实讲，Reactor3的文档内容更丰富。

# 什么是响应式系统

[响应式宣言](https://www.reactivemanifesto.org/zh-CN)里面说的很清楚，一个响应式系统应当是：

* 灵敏的：能够及时响应
* 有回复性的：即使遇到故障，也能够自行恢复、并产生回复
* 可伸缩的：能够随着工作负载的变化，自行调用或释放计算资源；也能够随着计算资源的变化，相应的调整工作负载能力
* 消息驱动的：显式的消息传递能够实现系统各组件解耦，各类组件自行管理资源调度。

# 构建响应式Web系统

## Vert.X

Vert.X目前由Eclipse基金会维护，打造了一整套响应式Web系统开发环境，包括数据库管理、消息队列、微服务、权限认证、集群管理器、Devops等等，生态很丰富。

Vert.X Core框架基于Netty开发，是一种事件驱动框架：每当事件可行时都会调用其对应的handler。在Vert.X里，有专门的线程负责调用handler，被称作eventloop。每一个Vert.X实例都维护了多个eventloop。

Vert.X Core框架有两个重要的概念：Verticle和Event Bus。

### Verticle

Verticle类似于Actor模型的Actor角色。

>[Actor](https://en.wikipedia.org/wiki/Actor_model)是什么？
>
>这里泛泛的说一下吧。
>
>Actor模型主要针对于分布式计算系统。Actor是其中最基本的计算单元。每一个Actor都有一个私有的消息队列。Actor之间的通信依靠发送消息。每一个Actor都可以并发地做到：
>
>1. 向其他Actor发送消息
>2. 创建新的Actor
>3. 指定当接收到下一个消息时的行为

Verticle之间的消息传递依赖于下面要说的Event Bus。

Vert.X为Verticle的部署提供了高可用特性：在Vert.X集群中，如果一个节点的上运行的Veticle实例失效，其他节点就会重新部署一份新的Verticle实例。

Verticle只是Vert.X提供的一种方案，并非强制使用。

### Event Bus

Event Bus是Vert.X框架的中枢系统，能够实现系统中各组件的消息传递和handler的注册与注销。其消息传递既支持“订阅-发布”模式，也支持“请求-响应”模式。

当多个Vert.X实例组成集群的时候，各系统的Event Bus能够组成**一个统一的分布式Event Bus**。各Event Bus节点相互之间通过TCP协议通信，没有依赖Cluster Manager。这是一种可以实现节点发现，提供了分布式基础组件（锁、计数器、map）等的组件。


## Spring WebFlux

Spring5的亮点之一就是响应式框架Spring WebFlux，使用自家的Reactor3开发，但同样支持RxJava。

Spring WebFlux的默认服务端容器是Reactor Netty，也可以使用Undertow或者Tomcat、Jetty的实现了Servlet 3.1 非阻塞API接口的版本。Spring WebFlux分别为这些容器实现了与Reactive Streams规范实现框架交互的适配器（Adapter），没有向用户层暴露Servlet API。

Spring WebFlux的注解方式和Spring MVC很像。这有助于开发团队快速适应新框架。而且Spring WebFlux兼容Tomcat、Jetty，有助于项目运维工作的稳定性。

但如果是新的项目、新的团队，给我大概会选Vert.X，因为Event Bus确实很吸引人。






# 参考资料

* [深入理解Java虚拟机 第二版](https://book.douban.com/subject/24722612/)
* [Netty的高性能及NIO的epoll空轮询bug](https://blog.csdn.net/a925907195/article/details/73853771)
* [JAVA NIO存在的问题](https://my.oschina.net/cloudcoder/blog/669753)
* [Reactor模式详解](http://www.blogjava.net/DLevin/archive/2015/09/02/427045.html)
* [Netty实战](https://book.douban.com/subject/27038538/)
* [Guide to the Fork/Join Framework in Java](http://www.baeldung.com/java-fork-join)
* [Java's Fork/Join vs ExecutorService - when to use which?](https://stackoverflow.com/questions/21156599/javas-fork-join-vs-executorservice-when-to-use-which)
* [ReactiveX](http://reactivex.io)
* [RxJava Essentials 中文翻译版](https://rxjava.yuxingxin.com)
* [Reactor 3 Reference Guide](http://projectreactor.io/docs/core/release/reference/#processors)
* [使用 Reactor 进行反应式编程](https://www.ibm.com/developerworks/cn/java/j-cn-with-reactor-response-encode/index.html)
* [Vert.x Core Manual](https://vertx.io/docs/vertx-core/java/)
* [Spring 5 Documentation - Web on Reactive Stack](https://docs.spring.io/spring/docs/5.0.5.BUILD-SNAPSHOT/spring-framework-reference/web-reactive.html)

## 延伸阅读

* [Five ways to maximize Java NIO and NIO.2](https://www.javaworld.com/article/2078654/core-java/java-se-five-ways-to-maximize-java-nio-and-nio-2.html)
* [ForkJoinPool的commonPool相关参数配置](https://segmentfault.com/a/1190000008470012)
* [Is there anything wrong with using I/O + ManagedBlocker in Java8 parallelStream()?](https://stackoverflow.com/questions/37512662/is-there-anything-wrong-with-using-i-o-managedblocker-in-java8-parallelstream)
* [Can I use the work-stealing behaviour of ForkJoinPool to avoid a thread starvation deadlock?](https://stackoverflow.com/questions/26576260/can-i-use-the-work-stealing-behaviour-of-forkjoinpool-to-avoid-a-thread-starvati)
