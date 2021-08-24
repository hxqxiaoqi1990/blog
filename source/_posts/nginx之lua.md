---
title: nginx之lua
tags: [nginx]
categories: linux基础
date: 2020-12-03 10:10:43
---

```bash
worker_processes  1;

events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    server {
        listen       80;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;
		
		set $name $args;

        location / {
            root   html;
            index  index.html index.htm;
        }
        location /luatest
        {       
                default_type text/html;
                #这里的lua文件的路径为绝对路径，请根据自己安装的实际路径填写
                #记得斜杠是/这个。
                content_by_lua_file /usr/local/openresty/lualib/testcode/testlua.lua;
        } 
        location /whoami {
                default_type application/json;
                content_by_lua_block {
						local did, err = ngx.re.match(ngx.req.get_body_data(),"[0-9]+","isjo")
                        if not did then
                                ngx.status = 400
                                ngx.print("{\"get_did\":\"false\"}")
                                ngx.exit(ngx.HTTP_OK)
                        end
                        local cjson = require "cjson"
                        local r = {}
                        r["forword-for"] = ngx.var.http_x_forwarded_for
                        r["remote_addr"] = ngx.var.remote_addr
                        r["request"] = ngx.var.request
                        r["request_time"] = ngx.var.request_time
						r["http_user_agent"] = ngx.var.http_user_agent
						r["arg"] = ngx.var.name
						r["did"] = did
						
                        local s = cjson.encode(r)
                        ngx.say(s)
                }
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
}

```

