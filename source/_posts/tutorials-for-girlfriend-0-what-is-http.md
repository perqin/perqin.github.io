---
title: '写给女朋友的教程系列0 什么是HTTP'
date: 2016-08-21 16:07:50
tags:
---

这篇教程是这个系列的第零篇，之所以叫这个系列，并不是想说这个系列的内容很简单，而是真的是字面意思……所以这个系列也比较杂，内容也会比较罗嗦，不过我相信我聪明可爱机智过人的女票大人一定能秒懂的！（好了可以把刀拿下来了嘛女票大人……）

# 先说说网络传输

在讨论HTTP这个东西之前，我们先来尝试理解一下互联网是怎么回事。当我们给计算机插上网线或者连上WiFi之后，计算机就连入了互联网，网络实际上就是数据在计算机之间的传输，比如我们最常见的下载文件，事实上就是文件中的数据从一台计算机通过网络传输到了自己的计算机（即本地计算机）。

# 为什么要有协议

计算机之间传输数据的时候，是以二进制的形式传输的，但是一个显而易见的问题是：对方如何知道需要给你发送什么数据呢？举个例子，使用浏览器浏览网页可以理解为从网站服务器获取“网页”数据。可是为什么我打开www.google.com自动登录的帐号是我的帐号，你打开的www.google.com时候自动登录的是你的帐号？拓展出去，为什么对于同一台服务器，它可以分辨出我要下载的是`慢播播放器`，你要下载的是`装逼教程.avi`呢？显然，两台计算机之间传输的数据必须遵照事先商定的格式，使得两台计算机可以互相理解，这就是所谓的“协议”。

# HTTP协议

有了前面的内容，我们可以开始说说HTTP了。

我们先从浏览器浏览网页说起。我们可以假设域名www.perqin.com指向的是服务器A，A是一台计算机，里面的某个文件夹里有一个网页文件home.html。现在，我们假设A是一台网站服务器，用户可以通过浏览器访问这个域名而查看到home.html里面的内容。当用户输入了域名，按下回车之后，用户的需求以HTTP协议所要求的格式，形成一串数据，被发送到了计算机A，计算机A遵循HTTP协议解读了这堆数据，并因此读取了home.html文件里的内容，也按照HTTP协议的格式发送了数据返回给用户。用户的浏览器得到服务器返回的数据之后，按照HTTP协议的格式解析了数据并显示到了浏览器窗口中。

上面描述的过程就是一次`HTTP访问`。在这个过程中，客户端发出了一个`HTTP请求`到服务器，服务器返回了一个`HTTP响应`给客户端，然后对话结束。

现在，一个很重要的事情是，我们只需要了解HTTP的过程，而不需要知道这个协议下的数据包到底长什么样。因此，后面所出现的各种相关名词，我们不需要知道他们从何而来，只要知道它们是协议的一部分。用写信寄信类比的话就是，我们只要知道一封信有寄信人、收信人、收信地址等等，而不需要知道这封信寄出去的时候收信人被写在了哪里、收信地址用了几行书写等。

# HTTP里的一些名词

## 客户端Client
我们需要理解的是，网络的通信其实是两个进程的通信：客户端的进程（比如浏览器）和服务器端的进程（比如某个服务器程序）通过网络建立了一个连接并相互发送消息。因此，客户端可以理解为本地的电脑，也可以理解为本地的一个程序。一般而言，我们在开发应用的时候访问网络，客户端就是这个应用。

## 主机Host
HTTP的请求类似于一问一答式，客户端首先发起请求，服务器再响应，主机指的就是服务器端的地址，可以是IP或者域名。

## 方法Method
客户端在发送请求的时候，可以指定这个请求的姿势，最常用的是GET和POST，不同的姿势往往用来代表不同的含义，而且会对请求和响应的格式产生影响。

## 路径Path/URI
前面我们说到过，浏览器浏览网页也是HTTP的请求（GET请求），那么`http://www.google.com/a`和`http://www.google.com/b`有什么区别呢？他们的域名部分是一样的，说明它们指向同一个主机，而域名后面的部分就是路径，它往往可以用来区分请求的目的。比如，我们可以指定`/users`这个路径用来请求用户数据，`/shops`这个路径用来请求店铺数据。这样的设计可以很容易体现数据的内在关系，比如`/users/perqin/friends/tom/mobile`就可以理解为用户perqin的朋友tom的手机号码的路径，这样的设计可以让请求的目的很明确。

## 查询Query
在浏览器输入[http://cn.bing.com/search?q=hello](http://cn.bing.com/search?q=hello)和[http://cn.bing.com/search?q=hi](http://cn.bing.com/search?q=hi)有什么区别？显然，两个操作都是进行“搜索”，路径也都是`/search`，但是得到的网页却不同。Query用来描述这个请求行为的参数，是以键值对的形式追加在路径后的，比如比应搜索的q参数就是搜索内容。同样的道理，假如用户通过`/login`来登录的话，也可以使用类似`/login?username=perqin&password=1234567`的方式，多个参数用&隔开，Query和Path之间用?隔开。你一定发现了，Query和Path都是出现在浏览器地址栏中的，因此网站设计者不会把类似密码的敏感信息放在Path和Query中。

## 请求体Body
既然不能放在Path和Query，那放到哪里呢？Body是请求体。如果前面的名词类比信封上的信息的话，Body就是信封里面的内容。Body可以是任何的格式，文本或图片或二进制，等等，因为它们本质上将会被当作二进制数据传输。需要注意的是，协议规定GET请求不携带Body部分，因此在GET请求中的Body往往会被服务器忽略。

# 案例Sudoku
我写了一个数独游戏的服务器程序，它的文档在[这里](https://github.com/CTJQ/Sudoku-server/wiki)。假设这个服务器程序运行在s.perqin.com指向的主机上，那么我们通过其中的“获取题目”接口来详细描述一次HTTP请求。

通过向s.perqin.com这个主机的/problems路径发送数据，我们将可以得到某个题目的数据。我们需要使用的方法是GET，路径是`/problems`，Query需要参数difficulty，于是我们可以构造这样一个请求：
```
GET http://s.perqin.com/problems?difficulty=1
```
然后服务器会在返回的响应中的Body部分携带这个题目的数据，数据是一个这样的字符串：
```
{
 "msg": "Succeed",
 "data": {
   "pid": 2,
   "content": "123456789456789123789123000......"
 }
}
```
我们先不考虑这个字符串奇葩的格式，但我们通过解析这个字符串就可以得到我们需要的内容。

# 如何使用第三方库发起HTTP请求

通过前面的内容，我们可以知道，加入我们想要写一个数独游戏的应用，并且通过这个服务器获取题目，那么我们的这个应用就需要能够发起HTTP请求，并解析获得的响应。一般而言，每个平台都会有官方库和各种第三方库用来发送HTTP的请求，并且他们的过程都是大同小异的：
1. 构造一个HttpClient对象
2. 设置请求方法、主机地址、Path
3. 按需要设置Query、Header（头部：一种不在浏览器地址栏中直接体现的可以携带少量数据的键值对）、Body等其他请求信息
4. 发送请求，等待返回响应（阻塞）或设置回调函数（非阻塞）
5. 得到响应之后，往往第三方库会将响应封装为一个对象，通过它可以得到响应中的数据。

# 实战：JavaScript vs Cocos2d-x
接下来，我们看看JavaScript和Cocos2d-x分别如何进行HTTP请求，以映证前面所述的基本过程。

## JavaScript：XMLHttpRequest
在JavaScript中，XMLHttpRequest对象可以充当Client，进行HTTP请求。我们谷歌一下，很容易可以得到用于请求数独题目的代码：
```
// #1
var xhr = new XMLHttpRequest();
// #2 #3
xhr.open('GET', 'http://s.perqin.com/problems?difficulty=1');
// #4 #5
xhr.onload = function() {
  console.log('得到的响应中Body内容为：\n' + xhr.responseText);
};
xhr.send(null);
```
可以看到，这段代码构造了XMLHttpRequest对象，通过open方法指定了请求方法、主机、路径和Query，并设置了回调函数（其中输出了响应的内容），最后发送请求，null参数表示请求中不包含Body。

## Cocos2d-x：CCHttpClient
Cocos2d-x自带了CCHttpClient类用于发送HTTP请求（其实大部分开发平台的Http库都是HttpClient命名的）。参考[HOW TO USE HTTPCLIENT](http://www.cocos2d-x.org/wiki/How_to_use_CCHttpClient)，直接得出以下代码：
```
cocos2d::network::HttpRequest * request = new cocos2d::network::HttpRequest();
request->setUrl("http://s.perqin.com/problems?difficulty=1");
request->setRequestType(cocos2d::network::HttpRequest::Type::GET);
request->setResponseCallback( CC_CALLBACK_2(HttpClientTest::onHttpRequestCompleted, this));
cocos2d::network::HttpClient::getInstance()->send(request);
request->release();
```
Cocos2d-x和JavaScript的一个不同在于它将发出的请求也封装成一个类HttpRequest，对这个类的实例进行设置之后将它发送出去，即可实现发送Http请求。以上代码构造了一个Request对象，指定了主机、路径和Query，指定了请求方法GET，制定回调函数，最后获得一个HttpClient实例用以发送请求。

通过对比我们发现，尽管不同的语言在实现上有许多区别，比如有的需要构造Request和Response对象、有的需要释放Request对象等等，但是他们的流程都是一样的，都是按照HTTP的协议要求，指定请求，发送请求，获取响应。因此，只要理解了HTTP请求的整个流程，那么不管什么语言和平台，我们都可以直接搜索到对应的库，通过类似的步骤发送HTTP请求！而我们所需要的，不过是不同平台的库的文档而已。有兴趣的话，可以看看我随手搜索到的Java发送HTTP请求的一个网页：[html - How to send HTTP request in java? - StackOverflow](http://stackoverflow.com/questions/1359689/how-to-send-http-request-in-java)，相信你很快就能对最顶上的答案看懂个大概。

当然，HTTP请求由于其丰富的应用场景，会使得代码不只是上面所写的那么简单，比如发送文字的时候要考虑字符集、发送图片和大文件的时候要考虑分片、GET请求还可能有缓存控制等等，但万变不离其宗，基本的步骤都是围绕HTTP协议来的，甚至如果彻底理解了HTTP协议，你也可以自己实现一个HTTP请求库！

很惭愧，只做了一点点微小的工作，谢谢女票大人捧场！撒花～