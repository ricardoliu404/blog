---
title: HTTP
date: 2020-01-18 09:35:12
tags: HTTP
categories:
- 计算机网络
---

> “HTTP”这个名字取得很大很空，可能是我飘了。

# HTTP报文结构
![HTTP报文结构 图源《计算机网络》](http-message.PNG)

## request报文实例
调试这篇博客样式时，查看浏览器请求，有如下请求报文：
``` http
GET /2020/01/18/HTTP/ HTTP/1.1
Host: localhost:4000
Connection: keep-alive
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (iPhone; CPU iPhone OS 11_0 like Mac OS X) AppleWebKit/604.1.38 (KHTML, like Gecko) Version/11.0 Mobile/15A372 Safari/604.1
Sec-Fetch-User: ?1
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Referer: http://localhost:4000/
Accept-Encoding: gzip, deflate, br
Accept-Language: en,zh-CN;q=0.9,zh;q=0.8
```
- 首行（请求行）中`GET`指请求方法，`/2020/01/18/HTTP`指请求URL，`HTTP/1.1`指HTTP请求版本号为1.1，三部分由空格间隔，由`CRLF`换行。
- 2-12行（首部行）均为headers字段内容

## response报文实例
``` http
HTTP/1.1 200 OK
X-Powered-By: Hexo
Content-Type: text/html
Date: Sat, 18 Jan 2020 02:26:24 GMT
Connection: keep-alive
Transfer-Encoding: chunked

#响应内容
```
- 首行（状态行）中`200 OK`为响应状态码和短语，标记当前响应的状态。
- 2-6行（首部行）均为headers字段内容
- 响应内容与首部行之间有一个空行