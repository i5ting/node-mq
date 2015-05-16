# node-mq

优化系统性能最管用的2招

1. 增加cache（memcache，redis）
2. 使用MQ（RabbitMQ，ActiveMQ，ZeroMQ）

既然有朋友问到这个问题，那我们就一起聊聊

## 什么是MQ

MQ即message queue,消息队列

消息队列（MQ）是一种应用程序对应用程序的通信方法。

应用程序通过读写出入队列的消息（针对应用程序的数据）来通信，而无需专用连接来链接它们。

消 息传递指的是程序之间通过在消息中发送数据进行通信，而不是通过直接调用彼此来通信，直接调用通常是用于诸如远程过程调用（rmi）的技术。

排队指的是应用程序通过队列来通信。

队列的使用除去了接收和发送应用程序同时执行的要求。

MQ是消费-生产者模型的一个典型的代表，一端往消息队列中不断写入消息，而另一端则可以读取或者订阅队列中的消息。

其中较为成熟的MQ产品有

- IBM WEBSPHERE MQ（老牌企业用的mq）
- RabbitMQ（互联网）
- ActiveMQ（互联网 + 企业）
- ZeroMQ（互联网）
- kafka（新贵，用于日志处理的分布式消息队列）

## 适用场景举例

很多项目中都有消息分发或者事件通知机制，尤其是模块化程度高的项目。

在项目中，将一些无需即时返回且耗时的操作提取出来，进行了异步处理，而这种异步处理的方式大大的节省了服务器的请求响应时间，从而提高了系统的吞吐量。


### 场景0

在你的系统中，很多模块都对 **新建用户** 感兴趣。权限模块希望给新用户设置默认权限，报表模块希望重新生成当月的报表，邮件系统希望给用户发送激活邮件...诸如此类的代码都写到新建用户的业务逻辑后面，会加大耦合度，降低可维护性，并且对于每个模块都是一个独立系统的情况，这种方式更是不可取。

### 场景1

长耗时的报表，这个在程序中经常遇见，处理海量数据时，可能生成一个报表需要5分中或是更长的时间，客户不能在线实时等待，报表处理比较耗费资源，不能同时处理很多请求，甚至同时只允许处理一个，这时就可以使用MQ。客户端将报表请求和一些必要的报表条件放到Queue中，报表由另一个服务一个一个的处理，处理好后再给用户发一个消息（WebSocket消息，或mail等）用户再在浏览器或其他报表浏览器中查看报表。

### 场景2

在线商店，在客户下订单的过程后，系统只需做减库存、记录收货人信息和必要的日志，其他的必须配送处理、交易统计等其他处理可以不同时完成，这时就可以将后续处理消息放入Queue中，让另一台（组）服务器去处理，这样可以加快下订单的过程，提高客户的体验

对于简单的情形，观察者模式 `The Observer Pattern` 就足够了。如果系统中有很多地方都需要收发消息，那么它就不适用了。否则会造成类数量的膨胀，增加类的复杂性，这时候就需要一种更集中的机制来处理这些业务逻辑。

## 什么是发布/订阅模式(PUB-SUB)

把MQ精简一下，我们只说发布 / 订阅模式

现实中，并不是所有请求都期待答复，而不期待答复，自然就没有了状态。广播听过吧？收音机用过吧？就这个意思。

发布/订阅模式定义了一种一对多的依赖关系，让多个订阅者对象同时监听某一个主题对象。这个主题对象在自身状态变化时，会通知所有订阅者对象，使它们能够自动更新自己的状态。

## 特点

- 一个订阅者可以订阅多个发布者
- 消息是会到达所有订阅者处，订阅者根据 filter 丢掉自己不需要的消息(filter 是在订阅端起作用的)
- 每个订阅者都会接收到每条消息的一个副本
- 基于推送 push，其中消息自动地向订阅者广播，它们无须请求或轮询主题来获得新消息

发布/订阅模式内部，有多种不同类型的订阅者。

- 非持久订阅者是临时订阅类型，它们只是在主动侦听主题时才接收消息。
- 持久订阅者将接收到发布的每条消息的一个副本，即便在发布消息，它们处于"离线"状态时也是如此。
- 另外还有动态持久订阅者和受管的持久订阅者等类型。

## 优势

降低了模块间的耦合度：发布者与订阅者松散地耦合，并且不需要知道对方的存在。相关操作都集中在 `Publisher` 中。

可扩展性强：系统复杂后，可以把消息订阅和分发机制单独作为一个模块来实现，增加新特性以满足需求

## 缺陷

与其说缺陷，不如说它设计本身就有如下特点。但不管怎么说，这种模式在逻辑上不可靠的。主要体现在：

- 发布者不知道订阅者是否收到发布的消息
- 订阅者不知道自己是否收到了发布者发出的所有消息
- 发送者不能获知订阅者的执行情况
- 没人知道订阅者何时开始收到消息

## Nodejs MQ实现

推荐

- https://github.com/faye/faye
- https://github.com/Automattic/kue


其它可参考选型

- https://github.com/SOHU-Co/kafka-node
- https://github.com/JustinTulloss/zeromq.node
- https://github.com/technoweenie/coffee-resque


## 为什么推荐这2个？

faye最早是ruby写的，后来性能问题考虑，出了nodejs版本，它支持redis作为存储。(目前rubychina的回复提醒就是faye做的，而cnode的回复提醒是极光推送做的，哎)

Kue是后端由redis支持带有优先级的作业队列。如果你的业务系统需要带有优先级，推荐kue

它们的共同点

- redis快速存储特性
- redis队列数据结构，支持pub/sub
- redis性能很好

redis 3.0支持cluster之后就更爽了，而且还有[codis](https://github.com/wandoulabs/codis)这样的神器。

## 实例: fayejs

### 安装

在终端里输入

	git clone https://github.com/i5ting/fayeserver.git
	npm install
	npm start
	
默认端口是`4567`，如果需要，请修改`index.js`文件。


	npm run publish	
	
### 用法

在网页里

```
	<script type="text/javascript" 
	        src="http://127.0.0.1:4567/faye/client.js">
	        </script>
	<script type="text/javascript">
	  var client = new Faye.Client('http://127.0.0.1:4567/faye', {
	  	timeout : 120,
			retry		: 5
		});
		
		Logger = {
		  incoming: function(message, callback) {
		    console.log('incoming', message);
		    callback(message);
		  },
		  outgoing: function(message, callback) {
		    console.log('outgoing', message);
		    callback(message);
		  }
		};
	
		client.addExtension(Logger);
		
		client.on('transport:down', function() {
		  // the client is offline
		});
	
		client.on('transport:up', function() {
		  // the client is online
		});
		
		
		var subscription = client.subscribe('/foo', function(message) {
		  // handle message
			console.log(message);
		});
		
		setTimeout(function(){
			var publication = client.publish('/foo', {text: 'Hi there, foo test'});
	
			publication.then(function() {
			  alert('Message received by server!');
			}, function(error) {
			  alert('There was a problem: ' + error.message);
			});
			
		},2000);
	</script>
```

说明

- `/foo`是需要订阅的主题
- subscribe是订阅（浏览器端订阅主题，当其他地方发布一样的主题的时候，回调里的message会收到信息的）
- publish是发布（浏览器发布主题，服务器端其实也可以发布的，暂未提供，见todo）


### publish http api

发布主题的http api 说明

- POST
- url = /pub
- post data = key=foo&value=somevalue`

通过curl命令测试本地POST

	curl -d "key=foo&value=sss" http://127.0.0.1:4567/pub

使用node发送post请求(java可以使用httpclient)

```
var request = require('request');

request.post({
	url:'http://127.0.0.1:4567/pub', 
	form: {
		key:'foo',
		value:'' + str
	}
}, function(err,httpResponse,body){ 
	/* ... */ 
	if(err)
		console.log(err);
})	
```

检测步骤

- 打开client/index.html
- 在终端里执行curl命令
- 看浏览器，是否会弹出`sss`

更多见代码：https://github.com/i5ting/fayeserver.git


欢迎关注我的公众号【node全栈】  ![](node全栈-公众号.png)