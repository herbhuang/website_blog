---
title: 网站启用 HSTS
date: 2018-07-16 00:19:19
categories: website
---  
## 什么是 HSTS  
HSTS (**H**TTP **S**trict **T**ransport **S**ecurity) 是指 HTTP 严格传输安全，具体而言，是在服务器返回的 HTTP 响应头中添加「严格传输」字段，客户端发现该字段后，在此后的连接中便只能采用有证书加密的方式（即 HTTPS 方式）连接。  
也就是说：当我第一次和你用中文打电话时，我以英语回答并且告诉你以后我们都用英语通话。那么下一次我们通话时，就必须用英语，否则手机自动挂机。当然，实际情况是浏览器会默认使用 HTTPS，对应到这个比喻，应该是你的*智能*手机会同声翻译成英语，你别想说中文。  
这个比喻有点怪异，不过你需要知道的是：它把使用安全连接的动作交给了你的浏览器而不是我的服务器，这样既节省资源，又能确保安全。  
## 具体操作  
操作简单得很，只需要在 nginx 的 server 配置文件里添加一行代码：  
{% codeblock nginx.conf %}
add_header Strict-Transport-Security "max-age=63072000; includeSubdomains; preload";
{% endcodeblock %}  
为了避免点击劫持，还可以添加一段：  
{% codeblock nginx.conf %}
add_header X-Frame-Options "DENY";
{% endcodeblock %}  
同时为了「说一次英语」，还增加了一个 301 跳转，使得用户第一次访问 HTTP 版本时强制跳转成 HTTPS：  
{% codeblock %}
return 301 https://herbhuang.com$request_uri;
{% endcodeblock %}  
有个小插曲是我把 uri 拼成了 url，这两者属于包含关系，url（**l**ocator，统一资源定位符）是以定位方式确定的uri（**i**dentifier，统一资源标识符）。  
## 加入 HSTS Preload List  
这个列表相当于一份名单，分发给各大现代浏览器。名单里的网站就模式使用 HTTPS 模式访问。其要求有三个：  
1. 所有域名和子域名都支持 HTTPS 访问
2. HTTP 域名要通过 301 跳转到 HTTPS 页面
3. HTTPS 头部要有 ```Strict-Transport-Security``` 字段  
一经加入，难以退出。（笑  
>参考资料
>
>[维基百科：HTTPS严格传输安全(https://zh.wikipedia.org/wiki/HTTP%E4%B8%A5%E6%A0%BC%E4%BC%A0%E8%BE%93%E5%AE%89%E5%85%A8)
>
>[nginx启用HSTS以支持从http到https不通过服务端而自动跳转(https://blog.csdn.net/socho/article/details/72456008)
>
>[HSTS Preload List 网站(https://hstspreload.org/)