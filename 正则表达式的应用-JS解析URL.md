---
title: "正则表达式的应用-JS解析URL"
date: "2017-08-22"
tags: ["JS","正则表达式"]
categories: ["学习笔记", "零散知识"]
---
在页面交互设计中，经常需要对URL进行解析操作，这里记录下一个网络上比较流行的解析函数，能对URL进行较好的解析。

```javascript
<script>  
/** 
*@param {string} url 完整的URL地址 
*@returns {object} 自定义的对象 
*@description 用法示例：var myURL = parseURL('http://abc.com:8080/dir/index.html?id=255&m=hello#top');
myURL.file='index.html' 

myURL.hash= 'top' 

myURL.host= 'abc.com' 

myURL.query= '?id=255&m=hello' 

myURL.params= Object = { id: 255, m: hello } 

myURL.path= '/dir/index.html' 

myURL.segments= Array = ['dir', 'index.html'] 

myURL.port= '8080' 

myURL.protocol= 'http' 

myURL.source= 'http://abc.com:8080/dir/index.html?id=255&m=hello#top' 

*/  
function parseURL(url) {  
 var a =  document.createElement('a');  
 a.href = url;  
 return {  
 source: url,  
 protocol: a.protocol.replace(':',''),  
 host: a.hostname,  
 port: a.port,  
 query: a.search,  
 params: (function(){  
     var ret = {},  
         seg = a.search.replace(/^\?/,'').split('&'),   
         len = seg.length, i = 0, s;  
     for (;i<len;i++) {  
         if (!seg[i]) { continue; }  
         s = seg[i].split('=');  
         ret[s[0]] = s[1];  
     }  
     return ret;  
 })(),  
 file: (a.pathname.match(/\/([^\/?#]+)$/i) || [,''])[1],  
 hash: a.hash.replace('#',''),  
 path: a.pathname.replace(/^([^\/])/,'/$1'),  
 relative: (a.href.match(/tps?:\/\/[^\/]+(.+)/) || [,''])[1],  
 segments: a.pathname.replace(/^\//,'').split('/')  
 };  
}    

//var myURL = parseURL('http://abc.com:8080/dir/index.html?id=255&m=hello#top');  
var myURL = parseURL('http://localhost:8080/test/mytest/toLogina.ction?m=123&pid=abc');  
alert(myURL.path);  
alert(myURL.params.m);  
alert(myURL.params.pid);  
</script>
```

URL : 统一资源定位符 (Uniform Resource Locator, URL)  
完整的URL由这几个部分构成：scheme://host:port/path?query#fragment  
对于这样一个URL  
http://www.jb51.net:80/seo/?ver=1.0&id=6#imhere    
我们可以用javascript获得其中的各个部分   

1. window.location.href  
   整个URl字符串(在浏览器中就是完整的地址栏)   
2. window.location.protocol  
   URL 的协议部分  
   本例返回值:http:  
3. window.location.host  
   URL 的主机部分  
   本例返回值:www.jb51.net  
4. window.location.port  
   URL 的端口部分  
   如果采用默认的80端口(update:即使添加了:80)，那么返回值并不是默认的80而是空字符  
   本例返回值:”"  
5. window.location.pathname  
   URL 的路径部分(就是文件地址)  
   本例返回值:/seo/  
6. window.location.search  
   查询(参数)部分  
   除了给动态语言赋值以外，我们同样可以给静态页面,并使用ja vascript来获得相信应的参数值  
   本例返回值:?ver=1.0&id=6  
7. window.location.hash  
   锚点   
   本例返回值:#imhere  