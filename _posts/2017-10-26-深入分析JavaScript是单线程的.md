---
layout:     post
title:      深入分析JavaScript是单线程的
subtitle:   用2个小demo彻底弄懂JS的单线程机制
date:       2017-10-26
author:     Gage
header-img: img/post-bg-js-version.jpg
catalog: true
tags:
    - JavaScript
    - 单线程
    - 基础
---

>为什么JavaScript是单线程的却能让AJAX异步发送和回调请求，为什么setTimeout也看起来像是多线程的？

## 首先来看如下demo

```javascript
function fn() {
    console.log("first");
    setTimeout(function() {
        console.log("second");
    }, 5);
}
for (var i = 0; i < 10000; i++) {
    fn();
}
```
先思考下预计输出的结果，然后可以在把代码拷贝到控制台运行试试。

OK.执行结果如下：
![](https://ws1.sinaimg.cn/large/d8cefbbagy1fr1wrl7lyxj20fe02sq2y.jpg)

首先全部输出first，然后全部输出second；尽管中间的执行会超过5ms。

为什么呢？![](https://ws1.sinaimg.cn/large/d8cefbbagy1fr1x4h3iscj2019019a9w.jpg)

# JavaScript是单线程的

因为`JS运行在浏览器中，是单线程的，每个window一个JS线程`，既然是单线程的，在某个特定的时刻只有特定的代码能够被执行，并阻塞其它的代码。而浏览器是**事件驱动的（Event driven）**，浏览器中很多行为是**异步（Asynchronized）的**，会`创建事件并放入执行队列中`。JavaScript引擎是单线程处理它的任务队列，你可以理解成就是普通函数和回调函数构成的队列。当异步事件发生时，如mouse click, a timer firing, or an XMLHttpRequest completing（鼠标点击事件发生、定时器触发事件发生、XMLHttpRequest完成回调触发等），`将他们放入执行队列，等待当前代码执行完成`。

# 异步事件驱动

前面已经提到浏览器是**事件驱动的（Event driven）**，浏览器中很多行为是**异步（Asynchronized）的**，例如：鼠标点击事件、窗口大小拖拉事件、定时器触发事件、XMLHttpRequest完成回调等。当一个异步事件发生的时候，它就进入事件队列。**浏览器有一个内部大消息循环，Event Loop（事件循环），会[轮询](https://baike.baidu.com/item/%E8%BD%AE%E8%AF%A2/6078469?fr=aladdin)大的事件队列并处理事件**。例如，浏览器当前正在忙于处理onclick事件，这时另外一个事件发生了（如：window onSize），这个异步事件就被放入事件队列等待处理，只有前面的处理完毕了，空闲了才会执行这个事件。setTimeout也是一样，当调用的时候，js引擎会启动定时器timer，大约xxms以后执行xxx，当定时器时间到，就把该事件放到主事件队列等待处理（浏览器不忙的时候才会真正执行）。

每个浏览器具体实现主事件队列不尽相同，这里不做讨论。

# <span id ="jump">浏览器不是单线程的</span>

虽然**JS运行在浏览器中，是单线程的，每个window一个JS线程**，但浏览器不是单线程的，例如Web kit或是Gecko引擎，都可能有如下线程：

 * JavaScript引擎线程
 * 界面渲染线程
 * 浏览器事件触发线程
 * HTTP请求线程

如果js是单线程的，那么谁去轮询大的Event loop事件队列？答案是`浏览器会有单独的线程去处理这个队列`。

# Ajax异步请求是否真的异步?

既然说JavaScript是单线程运行的，那么XMLHttpRequest在连接后是否真的异步? 

其实请求确实是异步的，这请求是由`浏览器新开一个线程请求`（见前面的[浏览器多线程](#jump)）。当请求的状态变更时，如果先前已设置回调，这异步线程就产生状态变更事件放到 JavaScript引擎的事件处理队列中等待处理。当浏览器空闲的时候出队列任务被处理，JavaScript引擎始终是单线程运行回调函数。javascript引擎确实是单线程处理它的任务队列，能理解成就是普通函数和回调函数构成的队列。

总结一下，`Ajax请求确实是异步的，这请求是由浏览器新开一个线程请求，事件回调的时候是放入Event loop单线程事件队列等候处理。`

# setTimeout(func, 0)为什么有时候有用？

当写js多了之后可能发现，有时候加一个setTimeout(func, 0)非常有用，为什么？难道是模拟多线程吗？错！前面已经说过了，`JavaScript是JS运行在浏览器中，是单线程的，每个window一个JS线程`，既然是单线程的，setTimeout(func, 0)神奇在哪儿？那就是告诉js引擎，在0ms以后把func放到主事件队列中，等待当前的代码执行完毕再执行，注意：重点是改变了代码流程，把func的执行放到了等待当前的代码执行完毕再执行。这就是它的神奇之处了。它的用处有三个：

* 让浏览器渲染当前的变化（很多浏览器UI render和js执行是放在一个线程中，线程阻塞会导致界面无法更新渲染）
* 重新评估”script is running too long”警告
* 改变执行顺序

例如：下面的demo：

点击按钮就会显示"calculating...."，然后过了几秒种后，显示"calclation done"。

如果删除setTimeout，执行long（），就只会在卡顿几秒钟后显示"calclation done",不会显示"calculating...."。因为reDraw事件被进入事件队列到长时间操作的最后才能被执行，所以无法刷新。

```javascript
<button id='do'> Do long calc!</button>
<div id='status'></div>

$('#do').on('click', function(){

$('#status').text('calculating....'); //此处会触发redraw事件的fired，但会放到队列里执行，直到long()执行完。

// 1. without set timeout, user will never see "calculating...."
//long();//执行长时间任务，阻塞

// 2. with set timeout, works as expected
setTimeout(long,50);//用定时器，大约50ms以后执行长时间任务，放入执行队列，但在redraw之后了，根据先进先出原则
})

function long(){
    var result = 0
    for (var i = 0; i<1000; i++){
      for (var j = 0; j<1000; j++){
          for (var k = 0; k<1000; k++){
              result = result + i+j+k
          }
      } 
  }
  $('#status').text('calclation done') // has to be in here for this example. or else it will ALWAYS run instantly. This is the same as passing it a callback 
}
```

# 最后附上几个知识点概念

#### 进程 
狭义定义：进程是正在运行的程序的实例（an instance of a computer program that is being executed）。进程是一个“执行中的程序”。程序是一个没有生命的实体，只有处理器赋予程序生命时（操作系统执行之），它才能成为一个活动的实体，我们称其为进程。

广义定义：进程是一个具有一定独立功能的程序关于某个数据集合的一次运行活动。它是操作系统动态执行的基本单元，在传统的操作系统中，进程既是基本的分配单元，也是基本的执行单元。

#### 线程 
线程是程序中一个单一的顺序控制流程。进程内一个相对独立的、可调度的执行单元，是系统独立调度和分派CPU的基本单位指运行中的程序的调度单位。

#### 单线程
单线程在程序执行时，所走的程序路径按照连续顺序排下来，前面的必须处理好，后面的才会执行。单线程就是进程里只有一个线程。

#### 多线程
在单个程序中同时运行多个线程完成不同的工作，称为多线程。

#### 同步
同步就是指一个进程在执行某个请求的时候，若该请求需要一段时间才能返回信息，那么这个进程将会一直等待下去，直到收到返回信息才继续执行下去

#### 异步
异步是指进程不需要一直等下去，而是继续执行下面的操作，不管其他进程的状态。当有消息返回时系统会通知进程进行处理，这样可以提高执行的效率。

本文转载自[Javascript是单线程的深入分析](http://www.cnblogs.com/Mainz/p/3552717.html)
