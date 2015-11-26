---
layout: post
category : technology
tags : [openresty , redis , websocket ]
title: 基于openresty,redis sub/pub实现websocket推送
date : 2015-11-24
---


> 背景

---------------

  * 公司需要实现WEB端消息推送，了解到websocket长连接方式。

  * 为尽量减少tomcat连接数，基于原有openresty的经验，采用redis pub/sub 方式实现推送。


> 实现思路

  * 采用openresty的官方组件[lua-resty-websocket](https://github.com/openresty/lua-resty-websocket),建立websocket连接时，根据参数订阅redis频道。

  * 在任意位置连接redis,发布消息，实现连接客户端的信息推送。

<!--more-->

> nginx lua 代码

-----------------

  {% blockquote %}

    local redis = require "resty.redis"
    local cjson = require "cjson"
    local red = redis:new()
    local bcpUtil= require("bcp.bcpUtil")
    local channel = cjson.decode(ngx.req.get_uri_args(0).channel)

    local server = require "resty.websocket.server"
    local wb, err = server:new{
      timeout = 5000,
      max_payload_len = 65535
    }
    if not wb then
      ngx.log(ngx.ERR, "failed to new websocket: ", err)
      return ngx.exit(444)
    end

    local function subscribe (ws)
      local sub = bcpUtil.getRedisCon()
      local res, err = sub:subscribe("chat:"..channel)
      if not res then
        ngx.say("1: failed to subscribe: ", err)
        return
      end

      while true do
        local bytes, err = sub:read_reply()
        if bytes then
          ws:send_text(bytes[3]..channel)
        elseif err ~= "timeout" then
          ngx.log(ngx.ERR, err)
        end
      end
      
      --sub.close()
    end

    ngx.thread.spawn(subscribe, wb)

    while true do 
      local data, typ, err = wb:recv_frame()
      if wb.fatal then
        return ngx.exit(444)
      end
      if not data then
        local bytes, err = wb:send_ping()
        if not bytes then
          return ngx.exit(444)
        end
      elseif typ == "close" then 
        break
      elseif typ == "ping" then
        local bytes, err = wb:send_pong()
        if not bytes then
          return ngx.exit(444)
        end
      elseif typ == "pong" then
        ngx.log(ngx.INFO, "client ponged")
      elseif typ == "text" then
        local pub = bcpUtil.getRedisCon()
        pub:publish("chat:"..channel, data)
        pub:close()
      end 
    end
    wb:send_close()

  {% endblockquote %}

> html测试代码

---------------
  
{% blockquote  %}
  var ws = null;
  function connect() {
    if (ws !== null) return log('already connected');
    ws = new WebSocket('ws://192.168.0.50/webws?channel="zhang"');
    ws.onopen = function () {
      log('connected');
    };
    ws.onerror = function (error) {
      log(error);
    };
    ws.onmessage = function (e) {
      log('recv: ' + e.data);
    };
    ws.onclose = function () {
      log('disconnected');
      ws = null;
    };
    return false;
  }
  function disconnect() {
    if (ws === null) return log('already disconnected');
    ws.close();
    return false;
  }
  function send() {
    if (ws === null) return log('please connect first');
    var text = document.getElementById('text').value;
    document.getElementById('text').value = "";
    log('send: ' + text);
    ws.send(text);
    return false;
  }
  function log(text) {
    var li = document.createElement('li');
    li.appendChild(document.createTextNode(text));
    document.getElementById('log').appendChild(li);
    return false;
  }    
{% endblockquote  %}

> 压力测试

--------------

    {% img '/hello.png' %}


> 一些问题

-----------------
  
  * 需要为每个连接建立redis 频道，redis 对小于10000的频道数支持较好。数量多了涉及redis的集群部署

  * 连接断开是会自动退出redis连接，sub 通道断开。



>参考文章

----------

[1][WebSockets with OpenResty](https://medium.com/technology-and-programming/websockets-with-openresty-1778601c9e05)