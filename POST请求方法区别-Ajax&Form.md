---
title: "POST请求方法区别-Ajax&Form"
date: "2017-08-22"
tags: ["Web", "Ajax"]
categories: ["学习笔记", "Web开发"]
---
在交互设计中我们经常会用到POST请求来传输数据，或者获取数据。POST请求有两种实现方式，分别是Ajax和Form表单。我们需要直到两种方式的区别和适用情况。

- Ajax方式发送POST请求  

```javascript
<script type="text/javascript">
function loadXMLDoc()
{
var xmlhttp;
if (window.XMLHttpRequest)
  {// code for IE7+, Firefox, Chrome, Opera, Safari
  xmlhttp=new XMLHttpRequest();
  }
else
  {// code for IE6, IE5
  xmlhttp=new ActiveXObject("Microsoft.XMLHTTP");
  }
xmlhttp.onreadystatechange=function()
  {
  if (xmlhttp.readyState==4 && xmlhttp.status==200)
    {
    document.getElementById("myDiv").innerHTML=xmlhttp.responseText;
    }
  }
xmlhttp.open("POST","/ajax/demo_post2.asp",true);
xmlhttp.setRequestHeader("Content-type","application/x-www-form-urlencoded");
xmlhttp.send("fname=Bill&lname=Gates");
}
</script>
```

- Form表单方式发送POST请求

```javascript
	var tempform = document.createElement("form");  
    tempform.action = url;  
    tempform.method = "post";  
    tempform.style.display="none"  
 
    for (var x in params) {  
        var opt = document.createElement("input");  
        opt.name = x;  
        opt.value = params[x];  
        tempform.appendChild(opt);  
    }  
  
    document.body.appendChild(tempform);  
    tempform.submit();  
    document.body.removeChild(tempform); 
```
<br ><br>
Ajax方式发送POST请求主要用于请求数据，可以获得被请求页面所返回的数据信息和头部，但这种方式不能响应服务器下发的302跳转请求。  
Form表单方式发送POST请求主要用于提交数据，将数据提交到被请求页面后，执行被请求页面代码，可以响应服务器下发的302请求。