---
layout: post
category : technology
tags : [openresty , WEB推送 , redis ]
title: openresty 实现WEB端推送
date : 2016-09-27 15:54:20
---


## 背景
-------------

微信公众号**架构师之路**推送一篇文章[http如何像tcp一样实时的收消息？](http://mp.weixin.qq.com/s?__biz=MjM5ODYxMDA5OQ==&mid=2651959605&idx=1&sn=21f25087bef3c3a966ef03b824365621&mpshare=1&scene=1&srcid=0926GHRm3xAgIFrWRUEq6s0y#rd)。
感觉特别有趣，便基于openresty 结合redis 实现了一下。

<!--more-->
## openresty lua 脚本
--------------
{% blockquote %}

local json = require("cjson")
local bcpUtil= require("vis.visUtil")
local instance = bcpUtil.getRedisCon()
local key = json.decode(ngx.req.get_uri_args().key)
local ret = {["key"]=key,["code"]=110} 
local message = instance:brpop(key,90) --监听redis 阻塞list
if message ~= ngx.null and message ~= nil then --[ngx.null](http://www.pureage.info/2013/09/02/125.html)
  ret["code"] = 200
  ret["message"] = message[2]
end
bcpUtil.close(instance)
ngx.say(json.encode(ret))

{% endblockquote %}

## 前端html 
---------------
{% blockquote %}

 function push() {
    $.getJSON("http://192.168.0.53/webpush",{key:JSON.stringify("zhang")}).done(function(ret) {
      console.log("success", ret);
      push();
    }).fail(function() {
      console.log("error");
      //push();
    });
  }

  $(document).ready(function() {
    push()
  });

{% endblockquote %}

## 测试
------------------
选择任意喜欢的redis client 
> lpush zhang helloworld

## 问题

* 刷新网页原有的连接未断开，重新建立了连接，原有redis 的连接也不会断开，此时push data，旧有连接会接收该数据。
  - 未断开连接会在90s后失效。
  - 该问题暂时未想通如何解决。

## 注意

* redis.set_timeout 设置的客户端超时时间要长，不然brpop设置服务器超时时间大于客户端的，会在客户端超时时结束连接。[](https://groups.google.com/forum/#!topic/openresty/HP74ZfLZ6zA)

## 参考

* [redis.set_timeout会导致后来的blpop失效？](https://groups.google.com/forum/#!topic/openresty/HP74ZfLZ6zA)
* [http如何像tcp一样实时的收消息？](http://mp.weixin.qq.com/s?__biz=MjM5ODYxMDA5OQ==&mid=2651959605&idx=1&sn=21f25087bef3c3a966ef03b824365621&mpshare=1&scene=1&srcid=0926GHRm3xAgIFrWRUEq6s0y#rd)
* [nil、null与ngx.null](http://www.pureage.info/2013/09/02/125.html)