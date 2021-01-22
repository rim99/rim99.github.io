---
layout: post
title: ThreadPoolExecutor源码阅读心得 
date: 2018-06-05 14:44:55
categories: 原创
---


ThreadPoolExecutor是JDK内置的线程池实现类，最初随JDK1.5发布。最近花了点时间看了下ThreadPoolExecutor的源码，JDK版本是JDK1.8.0_71。

# 整体结构

<div class="plantuml-diagram"><!--?xml version="1.0" encoding="UTF-8" standalone="no"?--><svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" contentscripttype="application/ecmascript" contentstyletype="text/css" height="624px" preserveAspectRatio="none" style="width:673px;height:624px;" version="1.1" viewBox="0 0 673 624" width="673px" zoomAndPan="magnify"><defs><filter height="300%" id="f187kkkrhf1647" width="300%" x="-1" y="-1"><feGaussianBlur result="blurOut" stdDeviation="2.0"></feGaussianBlur><feColorMatrix in="blurOut" result="blurOut2" type="matrix" values="0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 .4 0"></feColorMatrix><feOffset dx="4.0" dy="4.0" in="blurOut2" result="blurOut3"></feOffset><feBlend in="SourceGraphic" in2="blurOut3" mode="normal"></feBlend></filter></defs><g><!--cluster Executors--><polygon fill="#FFFFFF" filter="url(#f187kkkrhf1647)" points="463,513,538,513,545,535.4883,651,535.4883,651,612,463,612,463,513" style="stroke: #000000; stroke-width: 1.5;"></polygon><line style="stroke: #000000; stroke-width: 1.5;" x1="463" x2="545" y1="535.4883" y2="535.4883"></line><text fill="#000000" font-family="sans-serif" font-size="14" font-weight="bold" lengthAdjust="spacingAndGlyphs" textLength="69" x="467" y="528.5352">Executors</text><!--class Executors.DefaultThreadFactory--><rect fill="#FEFECE" filter="url(#f187kkkrhf1647)" height="48" id="Executors.DefaultThreadFactory" style="stroke: #A80036; stroke-width: 1.5;" width="156" x="479" y="548"></rect><ellipse cx="494" cy="564" fill="#ADD1B2" rx="11" ry="11" style="stroke: #A80036; stroke-width: 1.0;"></ellipse><path d="M496.9731,569.6431 Q496.3921,569.9419 495.7529,570.0913 Q495.1138,570.2407 494.4082,570.2407 Q491.9014,570.2407 490.5815,568.5889 Q489.2617,566.937 489.2617,563.8159 Q489.2617,560.6865 490.5815,559.0347 Q491.9014,557.3828 494.4082,557.3828 Q495.1138,557.3828 495.7612,557.5322 Q496.4087,557.6816 496.9731,557.9805 L496.9731,560.7031 Q496.3423,560.1221 495.7488,559.8523 Q495.1553,559.5825 494.5244,559.5825 Q493.1797,559.5825 492.4949,560.6492 Q491.8101,561.7158 491.8101,563.8159 Q491.8101,565.9077 492.4949,566.9744 Q493.1797,568.041 494.5244,568.041 Q495.1553,568.041 495.7488,567.7712 Q496.3423,567.5015 496.9731,566.9204 Z "></path><text fill="#000000" font-family="sans-serif" font-size="12" lengthAdjust="spacingAndGlyphs" textLength="124" x="508" y="568.5352">DefaultThreadFactory</text><line style="stroke: #A80036; stroke-width: 1.5;" x1="480" x2="634" y1="580" y2="580"></line><line style="stroke: #A80036; stroke-width: 1.5;" x1="480" x2="634" y1="588" y2="588"></line><!--class LinkedBlockingQueue--><rect fill="#FEFECE" filter="url(#f187kkkrhf1647)" height="48" id="LinkedBlockingQueue" style="stroke: #A80036; stroke-width: 1.5;" width="154" x="115" y="548"></rect><ellipse cx="130" cy="564" fill="#ADD1B2" rx="11" ry="11" style="stroke: #A80036; stroke-width: 1.0;"></ellipse><path d="M132.9731,569.6431 Q132.3921,569.9419 131.7529,570.0913 Q131.1138,570.2407 130.4082,570.2407 Q127.9014,570.2407 126.5815,568.5889 Q125.2617,566.937 125.2617,563.8159 Q125.2617,560.6865 126.5815,559.0347 Q127.9014,557.3828 130.4082,557.3828 Q131.1138,557.3828 131.7612,557.5322 Q132.4087,557.6816 132.9731,557.9805 L132.9731,560.7031 Q132.3423,560.1221 131.7488,559.8523 Q131.1553,559.5825 130.5244,559.5825 Q129.1797,559.5825 128.4949,560.6492 Q127.8101,561.7158 127.8101,563.8159 Q127.8101,565.9077 128.4949,566.9744 Q129.1797,568.041 130.5244,568.041 Q131.1553,568.041 131.7488,567.7712 Q132.3423,567.5015 132.9731,566.9204 Z "></path><text fill="#000000" font-family="sans-serif" font-size="12" lengthAdjust="spacingAndGlyphs" textLength="122" x="144" y="568.5352">LinkedBlockingQueue</text><line style="stroke: #A80036; stroke-width: 1.5;" x1="116" x2="268" y1="580" y2="580"></line><line style="stroke: #A80036; stroke-width: 1.5;" x1="116" x2="268" y1="588" y2="588"></line><!--class SynchronousQueue--><rect fill="#FEFECE" filter="url(#f187kkkrhf1647)" height="48" id="SynchronousQueue" style="stroke: #A80036; stroke-width: 1.5;" width="140" x="304" y="548"></rect><ellipse cx="319" cy="564" fill="#ADD1B2" rx="11" ry="11" style="stroke: #A80036; stroke-width: 1.0;"></ellipse><path d="M321.9731,569.6431 Q321.3921,569.9419 320.7529,570.0913 Q320.1138,570.2407 319.4082,570.2407 Q316.9014,570.2407 315.5815,568.5889 Q314.2617,566.937 314.2617,563.8159 Q314.2617,560.6865 315.5815,559.0347 Q316.9014,557.3828 319.4082,557.3828 Q320.1138,557.3828 320.7612,557.5322 Q321.4087,557.6816 321.9731,557.9805 L321.9731,560.7031 Q321.3423,560.1221 320.7488,559.8523 Q320.1553,559.5825 319.5244,559.5825 Q318.1797,559.5825 317.4949,560.6492 Q316.8101,561.7158 316.8101,563.8159 Q316.8101,565.9077 317.4949,566.9744 Q318.1797,568.041 319.5244,568.041 Q320.1553,568.041 320.7488,567.7712 Q321.3423,567.5015 321.9731,566.9204 Z "></path><text fill="#000000" font-family="sans-serif" font-size="12" lengthAdjust="spacingAndGlyphs" textLength="108" x="333" y="568.5352">SynchronousQueue</text><line style="stroke: #A80036; stroke-width: 1.5;" x1="305" x2="443" y1="580" y2="580"></line><line style="stroke: #A80036; stroke-width: 1.5;" x1="305" x2="443" y1="588" y2="588"></line><!--class Thread--><rect fill="#FEFECE" filter="url(#f187kkkrhf1647)" height="48" id="Thread" style="stroke: #A80036; stroke-width: 1.5;" width="74" x="6" y="548"></rect><ellipse cx="21" cy="564" fill="#ADD1B2" rx="11" ry="11" style="stroke: #A80036; stroke-width: 1.0;"></ellipse><path d="M23.9731,569.6431 Q23.3921,569.9419 22.7529,570.0913 Q22.1138,570.2407 21.4082,570.2407 Q18.9014,570.2407 17.5815,568.5889 Q16.2617,566.937 16.2617,563.8159 Q16.2617,560.6865 17.5815,559.0347 Q18.9014,557.3828 21.4082,557.3828 Q22.1138,557.3828 22.7612,557.5322 Q23.4087,557.6816 23.9731,557.9805 L23.9731,560.7031 Q23.3423,560.1221 22.7488,559.8523 Q22.1553,559.5825 21.5244,559.5825 Q20.1797,559.5825 19.4949,560.6492 Q18.8101,561.7158 18.8101,563.8159 Q18.8101,565.9077 19.4949,566.9744 Q20.1797,568.041 21.5244,568.041 Q22.1553,568.041 22.7488,567.7712 Q23.3423,567.5015 23.9731,566.9204 Z "></path><text fill="#000000" font-family="sans-serif" font-size="12" lengthAdjust="spacingAndGlyphs" textLength="42" x="35" y="568.5352">Thread</text><line style="stroke: #A80036; stroke-width: 1.5;" x1="7" x2="79" y1="580" y2="580"></line><line style="stroke: #A80036; stroke-width: 1.5;" x1="7" x2="79" y1="588" y2="588"></line><!--class Worker--><rect fill="#FEFECE" filter="url(#f187kkkrhf1647)" height="48" id="Worker" style="stroke: #A80036; stroke-width: 1.5;" width="73" x="69.5" y="440"></rect><ellipse cx="84.5" cy="456" fill="#ADD1B2" rx="11" ry="11" style="stroke: #A80036; stroke-width: 1.0;"></ellipse><path d="M87.4731,461.6431 Q86.8921,461.9419 86.2529,462.0913 Q85.6138,462.2407 84.9082,462.2407 Q82.4014,462.2407 81.0815,460.5889 Q79.7617,458.937 79.7617,455.8159 Q79.7617,452.6865 81.0815,451.0347 Q82.4014,449.3828 84.9082,449.3828 Q85.6138,449.3828 86.2612,449.5322 Q86.9087,449.6816 87.4731,449.9805 L87.4731,452.7031 Q86.8423,452.1221 86.2488,451.8523 Q85.6553,451.5825 85.0244,451.5825 Q83.6797,451.5825 82.9949,452.6492 Q82.3101,453.7158 82.3101,455.8159 Q82.3101,457.9077 82.9949,458.9744 Q83.6797,460.041 85.0244,460.041 Q85.6553,460.041 86.2488,459.7712 Q86.8423,459.5015 87.4731,458.9204 Z "></path><text fill="#000000" font-family="sans-serif" font-size="12" lengthAdjust="spacingAndGlyphs" textLength="41" x="98.5" y="460.5352">Worker</text><line style="stroke: #A80036; stroke-width: 1.5;" x1="70.5" x2="141.5" y1="472" y2="472"></line><line style="stroke: #A80036; stroke-width: 1.5;" x1="70.5" x2="141.5" y1="480" y2="480"></line><!--class BlockingQueue--><rect fill="#FEFECE" filter="url(#f187kkkrhf1647)" height="48" id="BlockingQueue" style="stroke: #A80036; stroke-width: 1.5;" width="116" x="187" y="440"></rect><ellipse cx="202" cy="456" fill="#B4A7E5" rx="11" ry="11" style="stroke: #A80036; stroke-width: 1.0;"></ellipse><path d="M197.9277,451.7651 L197.9277,449.6069 L205.3071,449.6069 L205.3071,451.7651 L202.8418,451.7651 L202.8418,459.8418 L205.3071,459.8418 L205.3071,462 L197.9277,462 L197.9277,459.8418 L200.3931,459.8418 L200.3931,451.7651 Z "></path><text fill="#000000" font-family="sans-serif" font-size="12" font-style="italic" lengthAdjust="spacingAndGlyphs" textLength="84" x="216" y="460.5352">BlockingQueue</text><line style="stroke: #A80036; stroke-width: 1.5;" x1="188" x2="302" y1="472" y2="472"></line><line style="stroke: #A80036; stroke-width: 1.5;" x1="188" x2="302" y1="480" y2="480"></line><!--class ThreadFactory--><rect fill="#FEFECE" filter="url(#f187kkkrhf1647)" height="48" id="ThreadFactory" style="stroke: #A80036; stroke-width: 1.5;" width="115" x="445.5" y="440"></rect><ellipse cx="460.5" cy="456" fill="#B4A7E5" rx="11" ry="11" style="stroke: #A80036; stroke-width: 1.0;"></ellipse><path d="M456.4277,451.7651 L456.4277,449.6069 L463.8071,449.6069 L463.8071,451.7651 L461.3418,451.7651 L461.3418,459.8418 L463.8071,459.8418 L463.8071,462 L456.4277,462 L456.4277,459.8418 L458.8931,459.8418 L458.8931,451.7651 Z "></path><text fill="#000000" font-family="sans-serif" font-size="12" font-style="italic" lengthAdjust="spacingAndGlyphs" textLength="83" x="474.5" y="460.5352">ThreadFactory</text><line style="stroke: #A80036; stroke-width: 1.5;" x1="446.5" x2="559.5" y1="472" y2="472"></line><line style="stroke: #A80036; stroke-width: 1.5;" x1="446.5" x2="559.5" y1="480" y2="480"></line><!--class ThreadPoolExecutor--><rect fill="#FEFECE" filter="url(#f187kkkrhf1647)" height="48" id="ThreadPoolExecutor" style="stroke: #A80036; stroke-width: 1.5;" width="148" x="171" y="332"></rect><ellipse cx="186" cy="348" fill="#ADD1B2" rx="11" ry="11" style="stroke: #A80036; stroke-width: 1.0;"></ellipse><path d="M188.9731,353.6431 Q188.3921,353.9419 187.7529,354.0913 Q187.1138,354.2407 186.4082,354.2407 Q183.9014,354.2407 182.5815,352.5889 Q181.2617,350.937 181.2617,347.8159 Q181.2617,344.6865 182.5815,343.0347 Q183.9014,341.3828 186.4082,341.3828 Q187.1138,341.3828 187.7612,341.5322 Q188.4087,341.6816 188.9731,341.9805 L188.9731,344.7031 Q188.3423,344.1221 187.7488,343.8523 Q187.1553,343.5825 186.5244,343.5825 Q185.1797,343.5825 184.4949,344.6492 Q183.8101,345.7158 183.8101,347.8159 Q183.8101,349.9077 184.4949,350.9744 Q185.1797,352.041 186.5244,352.041 Q187.1553,352.041 187.7488,351.7712 Q188.3423,351.5015 188.9731,350.9204 Z "></path><text fill="#000000" font-family="sans-serif" font-size="12" lengthAdjust="spacingAndGlyphs" textLength="116" x="200" y="352.5352">ThreadPoolExecutor</text><line style="stroke: #A80036; stroke-width: 1.5;" x1="172" x2="318" y1="364" y2="364"></line><line style="stroke: #A80036; stroke-width: 1.5;" x1="172" x2="318" y1="372" y2="372"></line><!--class AbstractExecutorService--><rect fill="#FEFECE" filter="url(#f187kkkrhf1647)" height="48" id="AbstractExecutorService" style="stroke: #A80036; stroke-width: 1.5;" width="170" x="160" y="224"></rect><ellipse cx="175" cy="240" fill="#A9DCDF" rx="11" ry="11" style="stroke: #A80036; stroke-width: 1.0;"></ellipse><path d="M175.1133,235.3481 L173.9595,240.4199 L176.2754,240.4199 Z M173.6191,233.1069 L176.6157,233.1069 L179.9609,245.5 L177.5122,245.5 L176.7485,242.437 L173.4697,242.437 L172.7227,245.5 L170.2739,245.5 Z "></path><text fill="#000000" font-family="sans-serif" font-size="12" font-style="italic" lengthAdjust="spacingAndGlyphs" textLength="138" x="189" y="244.5352">AbstractExecutorService</text><line style="stroke: #A80036; stroke-width: 1.5;" x1="161" x2="329" y1="256" y2="256"></line><line style="stroke: #A80036; stroke-width: 1.5;" x1="161" x2="329" y1="264" y2="264"></line><!--class ExecutorService--><rect fill="#FEFECE" filter="url(#f187kkkrhf1647)" height="48" id="ExecutorService" style="stroke: #A80036; stroke-width: 1.5;" width="122" x="184" y="116"></rect><ellipse cx="199" cy="132" fill="#B4A7E5" rx="11" ry="11" style="stroke: #A80036; stroke-width: 1.0;"></ellipse><path d="M194.9277,127.7651 L194.9277,125.6069 L202.3071,125.6069 L202.3071,127.7651 L199.8418,127.7651 L199.8418,135.8418 L202.3071,135.8418 L202.3071,138 L194.9277,138 L194.9277,135.8418 L197.3931,135.8418 L197.3931,127.7651 Z "></path><text fill="#000000" font-family="sans-serif" font-size="12" font-style="italic" lengthAdjust="spacingAndGlyphs" textLength="90" x="213" y="136.5352">ExecutorService</text><line style="stroke: #A80036; stroke-width: 1.5;" x1="185" x2="305" y1="148" y2="148"></line><line style="stroke: #A80036; stroke-width: 1.5;" x1="185" x2="305" y1="156" y2="156"></line><!--class Executor--><rect fill="#FEFECE" filter="url(#f187kkkrhf1647)" height="48" id="Executor" style="stroke: #A80036; stroke-width: 1.5;" width="82" x="204" y="8"></rect><ellipse cx="219" cy="24" fill="#B4A7E5" rx="11" ry="11" style="stroke: #A80036; stroke-width: 1.0;"></ellipse><path d="M214.9277,19.7651 L214.9277,17.6069 L222.3071,17.6069 L222.3071,19.7651 L219.8418,19.7651 L219.8418,27.8418 L222.3071,27.8418 L222.3071,30 L214.9277,30 L214.9277,27.8418 L217.3931,27.8418 L217.3931,19.7651 Z "></path><text fill="#000000" font-family="sans-serif" font-size="12" font-style="italic" lengthAdjust="spacingAndGlyphs" textLength="50" x="233" y="28.5352">Executor</text><line style="stroke: #A80036; stroke-width: 1.5;" x1="205" x2="285" y1="40" y2="40"></line><line style="stroke: #A80036; stroke-width: 1.5;" x1="205" x2="285" y1="48" y2="48"></line><!--link Executor to ExecutorService--><path d="M245,76.024 C245,89.579 245,104.038 245,115.678 " fill="none" id="Executor-ExecutorService" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="none" points="238,76,245,56,252,76,238,76" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--link ExecutorService to AbstractExecutorService--><path d="M245,184.024 C245,197.579 245,212.038 245,223.678 " fill="none" id="ExecutorService-AbstractExecutorService" style="stroke: #A80036; stroke-width: 1.0; stroke-dasharray: 7.0,7.0;"></path><polygon fill="none" points="238,184,245,164,252,184,238,184" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--link AbstractExecutorService to ThreadPoolExecutor--><path d="M245,292.024 C245,305.579 245,320.038 245,331.678 " fill="none" id="AbstractExecutorService-ThreadPoolExecutor" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="none" points="238,292,245,272,252,292,238,292" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--link ThreadPoolExecutor to ThreadFactory--><path d="M313.244,385.038 C354.872,402.141 407.438,423.738 446.682,439.862 " fill="none" id="ThreadPoolExecutor-ThreadFactory" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#FFFFFF" points="300.982,380,305.0118,385.98,312.0818,384.5602,308.0519,378.5802,300.982,380" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--link ThreadPoolExecutor to BlockingQueue--><path d="M245,393.338 C245,408.681 245,426.098 245,439.678 " fill="none" id="ThreadPoolExecutor-BlockingQueue" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#FFFFFF" points="245,380,241,386,245,392,249,386,245,380" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--link ThreadPoolExecutor to Worker--><path d="M204.307,388.032 C182.659,404.541 156.48,424.504 136.582,439.678 " fill="none" id="ThreadPoolExecutor-Worker" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#FFFFFF" points="214.839,380,207.6424,380.4571,205.2964,387.2759,212.493,386.8188,214.839,380" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--link BlockingQueue to SynchronousQueue--><path d="M288.652,500.869 C307.462,516.325 328.922,533.9595 345.618,547.6784 " fill="none" id="BlockingQueue-SynchronousQueue" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="none" points="283.999,506.106,272.991,488,292.888,495.289,283.999,506.106" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--link BlockingQueue to LinkedBlockingQueue--><path d="M224.453,506.094 C217.402,520.196 209.758,535.4838 203.661,547.6784 " fill="none" id="BlockingQueue-LinkedBlockingQueue" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="none" points="218.294,502.758,233.5,488,230.816,509.019,218.294,502.758" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--link Worker to Thread--><path d="M85.5129,499.47 C76.1326,515.253 65.2617,533.5437 56.8609,547.6784 " fill="none" id="Worker-Thread" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="#FFFFFF" points="92.3299,488,85.826,491.1143,86.1992,498.3157,92.7031,495.2014,92.3299,488" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--link ThreadFactory to Executors.DefaultThreadFactory--><path d="M523.934,506.094 C531.119,520.196 538.907,535.4838 545.119,547.6784 " fill="none" id="ThreadFactory-Executors.DefaultThreadFactory" style="stroke: #A80036; stroke-width: 1.0;"></path><polygon fill="none" points="517.558,508.999,514.717,488,530.033,502.644,517.558,508.999" style="stroke: #A80036; stroke-width: 1.0;"></polygon><!--
@startuml

class LinkedBlockingQueue
class SynchronousQueue
class Thread
class Worker
interface BlockingQueue
interface ThreadFactory
class ThreadPoolExecutor 
abstract class AbstractExecutorService
interface ExecutorService 
interface Executor 
 
Executor <|- - ExecutorService
ExecutorService <|.. AbstractExecutorService
AbstractExecutorService <|- - ThreadPoolExecutor 
ThreadPoolExecutor o- - ThreadFactory  
ThreadPoolExecutor o- - BlockingQueue  
ThreadPoolExecutor o- - Worker   
BlockingQueue <|- - SynchronousQueue 
BlockingQueue <|- - LinkedBlockingQueue 
Worker o- - Thread  
ThreadFactory <|- - Executors.DefaultThreadFactory  

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

上面这张图展示了ThreadPoolExecutor的一个大致的类结构。ThreadPoolExecutor当然依赖很多类，但最主要的还是三个大类ThreadFactory、BlockingQueue、Thread：

* ThreadFactory。主要定义了创建线程的方式。这一块，用户可以自己定义线程的创建行为，例如调整线程的栈大小等。这应该是设计模式里的模版模式。
* BlockingQueue。用来存放用户提交的、尚未开始执行的Runnable任务。当然BlockingQueue也未必真的有存储容量，例如SynchronousQueue。
* Thread。线程池里当然有线程，这是句废话。不过ThreadPoolExecutor把Thread封装在一个私有类Worker里面。ThreadPoolExecutor和Worker既保持了良好的封装性，又是紧耦合。这一点不确定好还是不好，应该再体会体会。

# 创建方式

JDK在Executors类里面提供了若干工厂方法，可以方便地创建线程池。其中和ThreadPoolExecutor关联的工厂方法有：

* newSingleThreadExecutor：创建单线程的线程池。主要用来实现异步调用。
* newFixedThreadPool：创建固定线程数量的线程池。主要用于处理计算密集型任务。
* newCachedThreadPool：这个线程池，能够按需创建线程。闲置的线程可以复用，避免创建开销。闲置超过60s的线程会被回收。主要用于处理能够快速返回的IO密集型任务。

ScheduledThreadPoolExecutor是ThreadPoolExecutor的子类，可以实现周期性的执行任务，这里暂不涉及。

这些工厂方法虽然方便，但是在业界被吐槽。《阿里Java编码规范》指出这些工厂方法的没什么可变参数，没什么调优的余地，而且自身的参数也不直观。试想一下，如果CachedThreadPool遇到密集的IO请求任务时，线程不断创建，最后会不会内存溢出呢？对于FixedThreadPool，如果任务量并不大，时刻保持固定数量的线程是不是有点浪费资源呢？所以说，创建线程池的最好的方法还是直接用ThreadPoolExecutor的构造器方法。

ThreadPoolExecutor的构造器有这些参数：

* corePoolSize：线程池核心线程数量。
* maximumPoolSize：线程池最大线程数量。
* keepAliveTime：线程的闲置时间。
* unit：线程限制时间单位。
* workQueue：线程工作队列。
* threadFactory：线程工厂。
* RejectedExecutionHandler：RejectedExecution的捕获器。

# 运行时行为

ThreadPoolExecutor把所管理的线程分为两类。一类是核心线程，一类是非核心线程。

当一个ThreadPoolExecutor刚刚创建出来的时候，里面没有任何正在运行的线程。当开始提交任务的时候，线程池会首先创建核心线程。即是当前有闲置线程，如果线程总数低于corePoolSize，那么依然会创建新的线程来运行任务。这个阶段可以视为线程池处于“热身”阶段。ThreadPoolExecutor也提供了prestartCoreThread、ensurePrestart、prestartAllCoreThreads三个方法来手动实现“无任务热身”。

当线程总数达到corePoolSize同时所有线程都在运行时，继续提交的任务，就会交给workQueue来处理。其实调用的就是workQueue的offer方法。通常是将任务留在队列里面，开始排队。

随着任务持续提交，当队列中任务达到最大值的时候，线程池开始创建新的线程，来处理队列中的任务。随着线程不增加，当线程总数达到maximumPoolSize时，继续提交的任务会被线程池拒绝，抛出RejectedExecutionException运行时异常。默认情况下，异常交由任务提交方来处理。但是，用户也可以自定义RejectedExecutionHandler来统一处理该异常。

当任务处理完毕后，线程就会闲置起来。一直闲置着当然不是个事儿。线程会闲置keepAliveTime时间，随后销毁回收。

默认情况下，核心线程不会因为闲置超时而被回收。但这可以通过allowCoreThreadTimeOut方法来修改设置。

总的来讲，ThreadPoolExecutor会**积极**创建**核心线程**，对于**非核心线程**则是**消极**创建。

保持足够数量的核心线程是出于及时响应任务提交的需要。这个很好理解。

但是，消极创建非核心线程就不好理解了。我一直觉得，池内线程的数量应该与队列里的任务数量保持一种台阶式的准线性关系比较好。随着队列任务的增加，逐渐增加线程数量，这样系统能够保持在一个稳定的状态，比来回变化的线程数量和任务数量，应当更有利于GC的表现。当然了，这一块一直没有真正的去测试一下，就留作以后的一个课题吧。

# 源码阅读心得

这里说下我觉得值得注意的几点。

## 线程池状态管理

ThreadPoolExecutor对象有五个运行状态，使用int来表示：

状态                 |   二进制数值                                                  |   意义
  :--:                 |    :--:                                                             |   :--:
RUNNING        |   11100000000000000000000000000000  |  正常运行
SHUTDOWN    |   00000000000000000000000000000000  |  不再接受新任务，但仍然在处理队列中的任务
STOP              |   00100000000000000000000000000000  |  不接受新任务，不处理队列任务，终止正在处理的任务
TIDYING          |   01000000000000000000000000000000  |  所有任务终止，线程数量归零
TERMINATED  |   01100000000000000000000000000000  |  terminated()方法执行完毕

1. RUNNING状态可以到达SHUTDOWN状态，也可以直接到达STOP状态。
2. 当然SHUTDOWN状态，也可以到达STOP状态。
3. SHUTDOWN状态和STOP状态都可以到达TIDYING状态。
4. TIDYING状态只能到达TERMINATED状态。

Java中整形变量最大是32个bit。ThreadPoolExecutor使用了最高的3个bit来表示当前线程池的状态，低位的29个bit用来表示当前池内的线程数。所以说，ThreadPoolExecutor支持的最大线程数为2^29=536_870_911。5亿多，想来应该是够用了。

ThreadPoolExecutor使用一个AtomicInteger变量来维护线程池状态。使用原子类型可以保证该变量能够高效的并发修改（相比于锁对象）并及时发布（这个有点类似于volatile的作用）。

ThreadPoolExecutor通过位的求或、求与、求反来实现对线程池状态和线程数量的查询。位运算，相比于算术运算，有更好的执行效率。这个可以好好研究下。

## 无阻塞算法的使用

ThreadPoolExecutor对象调用私有的addWorker方法来创建线程。

这个方法的内部最先是一个两层嵌套的for循环（labeled for-loop）。

```Java
private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }
    ...
}
```

外部循环带有一个`retry`标签，而内部循环可以是正常退出，也可以是在外层循环框架下执行break、continue动作。这个语法很少用到，但在某些场景下会很有用，值得记一笔。

其实观察一下内层循环里，要想能够打破循环，继续执行下面的内容，必须成功的完成函数compareAndIncrementWorkerCount调用。这个方法是对线程池状态变量的原子化替换。多个外部线程同时调用该函数的时候，有且只会有一个线程成功执行，然后继续增加内部线程。成功完成原子量变更的前提是，从读取初始值、完成新值计算到准备变更之间，该原子量不能有变化，否则就会重现执行外层循环。

这实际上是一种乐观的无锁算法。

乐观的意思是，函数的执行倾向于认为变量的并发修改频率不大，自己总能足够快的完成函数执行。

无锁，就很好理解了。各个线程不需要阻塞在锁对象上。如果一次执行失败，那就再来一次好了。少量for循环的开销比若干线程的阻塞和唤醒所带来的开销划算多了。

其实注意观察一下，这个方法和ReentrantLock类的tryLock方法都是乐观算法。两者共同的套路是，**无限循环中包含一个试错条件**，也就是使用一个原子变量的CAS(Coompare & Swap)操作作为成功与否的标记。如果成功，就会退出循环；如果失败，就会再循环一次。

在并发量不够高的场景下，无阻塞算法可以显著降低同步开销，与使用锁的算法相比，有更高的吞吐量。但是，如果并发量极高的时候，有锁反而更好。因为在后者场景下，无阻塞算法会有很多轮无效for循环，反而不如有序地获取锁来的高效。其实，这个可以用道路交通来类比：在车流量不大的路口，环岛的通过率更高；但当车流量很大的时候，红绿灯的效率更高。

## 线程池对象的全局锁对象

ThreadPoolExecutor对象内部有一把全局锁。这把锁会在，调用内部线程增加或者回收的时候调用，也会在读取线程池状态的时候调用。因此要注意这之间内部存在的阻塞关系。

此外，设想一种场景。所有线程全都在运行，队列满负荷运行，外部还有任务持续提交。如果此时线程运行中大量的抛异常退出，那么线程池会不停的创建线程、处理线程退出，这个行为就会被全局锁阻塞，导致线程池的低效运转。

要小心这种情况的发生。


## 队列任务的获取

ThreadPoolExecutor对象通过BlockingQueue获取任务。正如其名，BlockingQueue是一种阻塞式的队列。ArrayBlockingQueue、LinkedBlockQueue、PriorityBlockingQueue使用显式的锁对象，保护每一次队列任务的读取和放置；而SynchronousQueue则是将访问线程排队park起来，然后依次唤醒（unpark）提交或者获取任务。

频繁的读取BlockingQueue里的任务，会造成池内线程消耗大量时间在队列内部的同步机制上。因此，为了降低同步开销的比例，队列内部的任务不易快速返回。而出于吞吐量的考虑，队列内部的任务又不易消耗过长时间。这两者需要小心的平衡。

JDK1.7引入了新的LinkedTransferQueue据说有更好的并发效率。这个随后得花时间看看。

## 不要中途修改基础参数

尽管ThreadPoolExecutor对象可以在对象运行过程中，实时地修改corePoolSize、keepAliveTime等参数。但是修改这些参数，会在内部interupt所有线程，包括正在运行的线程。通常情况下，一般的业务逻辑不大会特别的处理InterruptedException。所以应该尽量避免在运行时修改线程池设置参数。

