---
layout: post
category : technology
tags : [nginx , lua , redis , json ]
title: Nginx+Lua+Redis+Json 内存读取方案
date : 2014-08-10
---


1. 安装Nginx,采用集成组件的OpenResty [http://openresty.org](http://openresty.org)

  * 安装依赖包 

      {% blockquote %}
       yum install readline-devel pcre-devel openssl-devel
      {% endblockquote %}

  * 下载OpenResty
  
      {% blockquote %}
       http://openresty.org/download/ngx_openresty-1.5.12.1.tar.gz
      {% endblockquote %}

  * 安装OpenResty
 
      {% blockquote %}
      ./configure --prefix=/opt/openresty
      {% endblockquote %}

2. 安装Redis [http://redis.io](http://redis.io)

  * 下载编译

      {% blockquote %}
       wget http://download.redis.io/releases/redis-2.8.9.tar.gz
       tar xzf redis-2.8.9.tar.gz
       cd redis-2.8.9
       make
      {% endblockquote %}

3. 修改nginx.conf

  * 代码描述

      ```
      
      worker_processes 1;
      error_log logs/error.log debug;
      events {
        worker_connections 1024;
      }
      http {
        include mime.types;
        default_type text/html;
        sendfile on;
        keepalive_timeout 65;
        server {
          listen 80;
          server_name localhost;
          root html;
          index index.html index.htm;
          location /get {
            set_unescape_uri $key $arg_key;
            redis2_query get $key;
            redis2_pass 127.0.0.1:6379;
          }
          location /get_redis {
            #internal;--内部访问
            set_unescape_uri $key $arg_key;
            redis2_query get $key;
            redis2_pass 127.0.0.1:6379;
          }
          location /json {
            set_unescape_uri $keys $arg_keys;
            content_by_lua_file conf/test.lua;
          }
        }
      }

      ```

4. conf/test.lua

  * 代码描述

      ```

        local json = require("cjson")
        local parser = require("redis.parser")
        function test(a_key)
            local res = ngx.location.capture("/get_redis",{
              args = { key = a_key }
            })
            if res.status == 200 then
              return parser.parse_reply(res.body)
            end
        end
        local args =json.decode(ngx.req.get_uri_args(0).keys)
        local tab = {}
        for i=1,#args do
          local key_args = args[i]
          tab[key_args]=test(key_args)
        end
        ngx.say(json.encode(tab))

      ```

5. 测试

  * 详细描述

      ```

        http://192.168.0.36/get?key=foo

        return:$3 bar

        http://192.168.0.36/json?key=[“foo”,”songsong”]

        return:{"foo":"bar","songsong":"songsong"}

      ```