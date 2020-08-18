---
title: "CORS介绍与应用分析"
date: "2017-08-22"
tags: ["Web", "浏览器","CORS"]
toc: "true"
categories: ["学习笔记", "Web开发"]
---
# 需求背景  
问题产生背景：VPN合作方案，客户需要在内网资源中通过Ajax请求调用接口。  

客户在进入"a.com"资源页面后，由JS自动触发一个Ajax请求以POST方法调用建立在JAVA语言平台上的"b.com"的API，由其返回一个JSON串。  
但这个请求失败了
# 技术方案
## 技术简述
由上面的问题背景，我们可以引入问题: “为什么Ajax请求失败了？”  
首先认识[Ajax](http://www.w3school.com.cn/ajax/index.asp)：

>AJAX = Asynchronous JavaScript and XML（异步的 JavaScript 和 XML）。  
>简单来讲，AJAX 是在不重新加载整个页面的情况下,与服务器交换数据并更新部分网页的技术。

- POST方法下传输数据的Ajax示例


```javascript
xmlhttp.open("POST","ajax_test.php",true);
xmlhttp.setRequestHeader("Content-type","application/x-www-form-urlencoded");
xmlhttp.send("fname=Bill&lname=Gates");
```


初步了解Ajax后，我们就要知道为什么它会失败，很容易猜想到Ajax技术（或者说Web技术）本身有一定的限制。  
出于安全考虑，在Web应用中，引入了“[同源策略](https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy)”，它禁止向不同源的地址发起HTTP请求，或者说禁止跨域请求，Ajax自然也受限其中。  
那么何为跨域？  

>只要协议、域名、端口有任何一个不同，都被当作是不同的域。

很明显，在"a.com"上调用"b.com"提供的接口，已经是一个Ajax跨域请求。  

**如何解决？**
业界之前曾广泛使用过JSONP技术来实现跨域请求，但JSONP只能实现GET请求,因此在我们的Web资源中，我们引入更为先进的CORS技术。  

>CORS（跨域资源共享，Cross-Origin Resource Sharing）是一种网络浏览器的技术规范，它为Web服务器定义了一种方式，允许网页从不同的域访问其资源。而这种访问是被同源策略所禁止的。CORS系统定义了一种浏览器和服务器交互的方式来确定是否允许跨域请求。  

简单来讲，跨域资源共享标准新增了一组 HTTP 首部字段，允许服务器声明哪些源站有权限访问哪些资源。这样通过客户端和服务器端进行协商后，就可以实现资源的跨域访问。  
如图：
![](../../img/cors/简单请求.png)

<p align="center">CORS简单请求流程</p>&nbsp;

图中`Origin: Server-b.com`为客户端增加的请求头，表明请求来源。  
而`Access-Control-Allow-Origin: *`为服务端增加的请求头，表明允许所有来源的请求  

CORS与JSONP相比，无疑更为先进、方便和可靠。  

 > JSONP只能实现GET请求，而CORS支持所有类型的HTTP请求。  
 > 使用CORS，开发者可以使用普通的XMLHttpRequest发起请求和获得数据，比起JSONP有更好的错误处理。  
 > JSONP主要被老的浏览器支持，它们往往不支持CORS，而绝大多数现代浏览器都已经支持了CORS。  


![](../../img/cors/浏览器.png)


<p align="center">支持CORS的浏览器</p>&nbsp;  




## 技术详解

>同样，CORS也有其自身的限制，它可以有两种模型来实现，规范要求对那些可能对服务器数据产生副作用的 HTTP 请求方法（特别是 GET 以外的 HTTP 请求，或者搭配某些 MIME 类型的 POST 请求），浏览器必须首先使用 OPTIONS 方法发起一个预检请求（preflight request），从而获知服务端是否允许该跨域请求。服务器确认允许之后，才发起实际的 HTTP 请求。在预检请求的返回中，服务器端也可以通知客户端，是否需要携带身份凭证（包括 Cookies 和 HTTP 认证相关数据）。


###  首先我们介绍简单请求模型

某些请求不会触发 CORS 预检请求。称这样的请求为“简单请求”，请注意，该术语并不属于 Fetch （其中定义了 CORS）规范。若请求满足所有下述条件，则该请求可视为“简单请求”：

> **使用下列方法之一：**
>
> - GET
> - HEAD
> - POST
>
> **Content-Type ：**
>
> //注:仅当POST方法的Content-Type值等于下列之一才算作简单请求
>
> - text/plain
> - multipart/form-data
> - application/x-www-form-urlencoded
>
> **Fetch 规范定义了对 CORS 安全的首部字段集合，不得人为设置该集合之外的其他首部字段。该集合为：**
>
> - Accept
> - Accept-Language
> - Content-Language
> - Content-Type （需要注意额外的限制）
> - DPR
> - Downlink
> - Save-Data
> - Viewport-Width
> - Width

满足以上条件的请求将以“简单请求”的形式发送到服务器。
即如图中所示
![](../../img/cors/简单请求.png)

<p align="center">CORS简单请求流程</p>&nbsp;

``` http
GET /resources/public-data/ HTTP/1.1
Host: bar.other
User-Agent: Mozilla/5.0 (Macintosh; U; Intel Mac OS X 10.5; en-US; rv:1.9.1b3pre) Gecko/20081130 Minefield/3.1b3pre
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-us,en;q=0.5
Accept-Encoding: gzip,deflate
Accept-Charset: ISO-8859-1,utf-8;q=0.7,*;q=0.7
Connection: keep-alive
Referer: http://foo.example/examples/access-control/simpleXSInvocation.html
Origin: http://foo.example


HTTP/1.1 200 OK
Date: Mon, 01 Dec 2008 00:23:53 GMT
Server: Apache/2.0.61 
Access-Control-Allow-Origin: *
Keep-Alive: timeout=2, max=100
Connection: Keep-Alive
Transfer-Encoding: chunked
Content-Type: application/xml

[XML Data]
```

第 1~10 行是请求首部。第10行 的请求首部字段 Origin 表明该请求来源于 http://foo.exmaple。  

第 13~22 行是来自于 http://bar.other 的服务端响应。响应中携带了响应首部字段 `Access-Control-Allow-Origin`（第 16 行）。使用` Origin` 和 `Access-Control-Allow-Origin` 就能完成最简单的访问控制。本例中，服务端返回的` Access-Control-Allow-Origin: *` 表明，该资源可以被任意外域访问。如果服务端仅允许来自 http://foo.example 的访问，该首部字段的内容如下：  

`Access-Control-Allow-Origin: http://foo.example`    

现在，除了 http://foo.example， 其它外域均不能访问该资源（该策略由请求首部中的 ORIGIN 字段定义，见第10行）。`Access-Control-Allow-Origin `应当为 * 或者包含由 Origin 首部字段所指明的域名。  

###  接下来我们介绍预检请求模型

与前述简单请求不同，“需预检的请求”要求必须首先使用 OPTIONS 方法发起一个预检请求到服务器，以获知服务器是否允许该实际请求。"预检请求“的使用，可以避免跨域请求对服务器的用户数据产生未预期的影响。
因为我们的Web资源服务是应用于VPN服务的，在页面访问过程中，必须对相关链接进行改写，并附带身份信息进行访问。因此在经过我们改写后的Ajax请求使用的是第二种模式，即预检请求。
当请求满足下述任一条件时，即应首先发送预检请求：

> **使用了下面任一 HTTP 方法：**
>
> - PUT
> - DELETE
> - CONNECT
> - OPTIONS
> - TRACE
> - PATCH
>
> **人为设置了对 CORS 安全的首部字段集合之外的其他首部字段。该集合为：**
>
> - Accept
> - Accept-Language
> - Content-Language
> - Content-Type (but note the additional requirements below)
> - DPR
> - Downlink
> - Save-Data
> - Viewport-Width
> - Width
>
> **Content-Type 的值不属于下列之一:**
>
> - application/x-www-form-urlencoded
> - multipart/form-data
> - text/plain

其流程如图：
![](../../img/cors/预检请求.png)

<p align="center">CORS预检请求流程</p>&nbsp;

``` http
OPTIONS /resources/post-here/ HTTP/1.1
Host: bar.other
User-Agent: Mozilla/5.0 (Macintosh; U; Intel Mac OS X 10.5; en-US; rv:1.9.1b3pre) Gecko/20081130 Minefield/3.1b3pre
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-us,en;q=0.5
Accept-Encoding: gzip,deflate
Accept-Charset: ISO-8859-1,utf-8;q=0.7,*;q=0.7
Connection: keep-alive
Origin: http://foo.example
Access-Control-Request-Method: POST
Access-Control-Request-Headers: X-PINGOTHER, Content-Type
```

浏览器检测到，从 JavaScript 中发起的请求需要被预检。上面的报文中发送了一个使用 OPTIONS 方法的“预检请求”。 OPTIONS 是 HTTP/1.1 协议中定义的方法，用以从服务器获取更多信息。该方法不会对服务器资源产生影响。 预检请求中同时携带了下面两个首部字段：  
`Access-Control-Request-Method: POST`    
`Access-Control-Request-Headers: X-PINGOTHER`    
首部字段 `Access-Control-Request-Method` 告知服务器，实际请求将使用 POST 方法。  
首部字段 `Access-Control-Request-Headers` 告知服务器，实际请求将携带两个自定义请求首部字段：X-PINGOTHER 与 Content-Type。    
服务器据此决定，该实际请求是否被允许。  

``` http
HTTP/1.1 200 OK
Date: Mon, 01 Dec 2008 01:15:39 GMT
Server: Apache/2.0.61 (Unix)
Access-Control-Allow-Origin: http://foo.example
Access-Control-Allow-Methods: POST, GET, OPTIONS
Access-Control-Allow-Headers: X-PINGOTHER, Content-Type
Access-Control-Max-Age: 86400
Vary: Accept-Encoding, Origin
Content-Encoding: gzip
Content-Length: 0
Keep-Alive: timeout=2, max=100
Connection: Keep-Alive
Content-Type: text/plain
```

该部分为预检请求的响应，表明服务器将接受后续的实际请求。  
（**此前我们所查出的BUG造成的原因便是OPTIONS请求因为没有携带身份信息被VPN拦截，返回的响应中并没有接受后续请求**）  
`Access-Control-Allow-Origin: http://foo.example`   
`Access-Control-Allow-Methods: POST, GET, OPTIONS`   
`Access-Control-Allow-Headers: X-PINGOTHER, Content-Type`  
`Access-Control-Max-Age: 86400`  

首部字段 `Access-Control-Allow-Methods` 表明服务器允许客户端使用 POST, GET 和 OPTIONS 方法发起请求。  
首部字段 `Access-Control-Allow-Headers` 表明服务器允许请求中携带字段 X-PINGOTHER 与 Content-Type。与 Access-Control-Allow-Methods 一样，Access-Control-Allow-Headers 的值为逗号分割的列表。   
最后，首部字段 `Access-Control-Max-Age` 表明该响应的有效时间为 86400 秒，也就是 24 小时。在有效时间内，浏览器无须为同一请求再次发起预检请求。请注意，浏览器自身维护了一个最大有效时间，如果该首部字段的值超过了最大有效时间，将不会生效。  

###  其他需要注意的点

- **附带身份凭证的请求**

> Fetch 与 CORS 的一个有趣的特性是，可以基于  HTTP cookies 和 HTTP 认证信息发送身份凭证。一般而言，对于跨域XMLHttpRequest 或 Fetch 请求，浏览器不会发送身份凭证信息。如果要发送凭证信息，需要设置 XMLHttpRequest 的某个特殊标志位。  
> 本例中，http://foo.example 的某脚本向 http://bar.other 发起一个GET 请求，并设置 Cookies：  
``` javascript
var invocation = new XMLHttpRequest();
var url = 'http://bar.other/resources/credentialed-content/';
function callOtherDomain(){
  if(invocation) {
    invocation.open('GET', url, true);
    invocation.withCredentials = true;
    invocation.onreadystatechange = handler;
    invocation.send();
  }
}
```
>第 7 行将 XMLHttpRequest 的 withCredentials 标志设置为 true，从而向服务器发送 Cookies。因为这是一个简单 GET 请求，所以浏览器不会对其发起“预检请求”。但是，如果服务器端的响应中未携带 Access-Control-Allow-Credentials: true ，浏览器将不会把响应内容返回给请求的发送者。
![](../../img/cors/cred-req.png)
>
><p align="center">携带身份信息的CORS请求流程</p>&nbsp; 
>
>客户端与服务器端交互示例如下：  
>
``` http
GET /resources/access-control-with-credentials/ HTTP/1.1
Host: bar.other
User-Agent: Mozilla/5.0 (Macintosh; U; Intel Mac OS X 10.5; en-US; rv:1.9.1b3pre) Gecko/20081130 Minefield/3.1b3pre
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-us,en;q=0.5
Accept-Encoding: gzip,deflate
Accept-Charset: ISO-8859-1,utf-8;q=0.7,*;q=0.7
Connection: keep-alive
Referer: http://foo.example/examples/credential.html
Origin: http://foo.example
Cookie: pageAccess=2
>
HTTP/1.1 200 OK
Date: Mon, 01 Dec 2008 01:34:52 GMT
Server: Apache/2.0.61 (Unix) PHP/4.4.7 mod_ssl/2.0.61 OpenSSL/0.9.7e mod_fastcgi/2.4.2 DAV/2 SVN/1.4.2
X-Powered-By: PHP/5.2.6
Access-Control-Allow-Origin: http://foo.example
Access-Control-Allow-Credentials: true
Cache-Control: no-cache
Pragma: no-cache
Set-Cookie: pageAccess=3; expires=Wed, 31-Dec-2008 01:34:53 GMT
Vary: Accept-Encoding, Origin
Content-Encoding: gzip
Content-Length: 106
Keep-Alive: timeout=2, max=100
Connection: Keep-Alive
Content-Type: text/plain
>
[text/plain payload]
```
>即使第 11 行指定了 Cookie 的相关信息，但是，如果 bar.other 的响应中缺失 Access-Control-Allow-Credentials: true（第 19 行），则响应内容不会返回给请求的发起者。
>
>**对于附带身份凭证的请求，服务器不得设置 Access-Control-Allow-Origin 的值为“\*”。**

>这是因为请求的首部中携带了 Cookie 信息，如果 Access-Control-Allow-Origin 的值为“*”，请求将会失败。而将 Access-Control-Allow-Origin 的值设置为 http://foo.example， 则请求将成功执行。  

- 服务器响应头的设置

>在上面我们说过，对于客户端发来的OPTIONS预检请求，服务器必须返回一个带有相关头部表明接受后续请求的响应。这个响应请求并不是由服务器上部署的代码来执行返回的，所以我们需要对服务器进行相应的设置，让其通过预检请求的申请。   
>1.对所有请求返回带有头部的响应  
>对于Apache服务器：Apache需要使用mod_headers模块来激活HTTP头的设置，它默认是激活的。你只需要在Apache配置文件的< Directory>, < Location>, < Files>或< VirtualHost>的配置里加入以下内容即可：  
>`Header set Access-Control-Allow-Origin *  `
>当然，如果你需要携带身份信息，那么这个头部需要设定为某一个特定域名。  
>其他服务器的设置可以[参考这里](http://enable-cors.org/)  
>2.针对发来的特定请求作出回复  
>可以将服务器中接收到的请求做单独的处理，譬如在接受到某一OPTIONS请求后，可以根据请求内容做相应处理，根据不同的请求返回带有不同头部的响应，在安全问题上更有保障。

- 注意协定的头部内容

>因为协议、域名、端口任意一个不同都会认为是跨域请求，所以在写代码的时候需要注意头部内容的协定  
>`Response for preflight is invalid (redirect)`
>此前曾经遇到浏览器返回了这样的错误信息。  
>原因如下：
>>1.代码里Ajax请求了一个HTTPS地址，但却写入了HTTP协议，而API访问的是HTTPS链接的地址，因此所有HTTP请求都返回了302重定向响应，造成预检失败。因此需要调整代码内容，写入正确的请求/响应头。  
>>2.另一种可能就是服务器对于没有身份信息做了重定向（与我们遇到的BUG非常相似），从浏览器发来的预检请求是没有身份信息token的，而服务器在检测到没有token之后将这个预检请求进行了重定向或者拦截，并没有返回真正需要的返回的内容，所以造成了预检请求的失败。因此需要更改服务器设置让服务器正确处理这些请求。
>
>除此之外，服务器上部署的代码，在处理指定来源的请求时，必须要确定好来源的协议、域名、端口并写入到响应头部中，否则请求也会因为没有得到正确的准许而被拒绝。