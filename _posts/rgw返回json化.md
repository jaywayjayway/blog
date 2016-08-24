title: rgw返回json化
categories: ceph
tags: [ceph,rgw]
date: 2016-08-22 20:10:04
---

默认的s3返回的格式是xml,本例本通过在**ngx_lua**做控制，把返回的xml格式转成`json` 

- nginx需要lua模块，推荐使用集成的 [`openresty`](https://openresty.org/en/)
- ngx_lua 需要 `LuaXml`模块，[仓库地址](https://github.com/LuaDist/luaxml)
- 本例中通过子请求获取fastcgi的请求结果


<!--more-->
```
init_by_lua_block { require("LuaXml") }

server {
    listen       80;
    server_name  node1.s3.com;

    access_log  logs/rgw.log   main;

    location / {
    if ($request_method = PUT) {
        rewrite ^ /PUT$request_uri;
    }

    content_by_lua_block {
        local json = require "cjson"
        local uri = ngx.var.request_uri
        res = ngx.location.capture("/custom"..uri)
        if res.status >  400  and  res.status < 500 then
            --return ngx.exit(401) -- can't find/authenticate user, refuse request
            --ngx.say("#####",type(res.body))
            --ngx.say(json.encode(parse_xml(res.body)))
            local datastr = xml.eval(res.body)

            if  res.body   then
                local table_need = {}
                for k,v in pairs(datastr) do
                    if k > 0 then
                          if type(v) == 'table' then
                              -- ngx.say(v[0],' =',v[1])
                               table_need[v[0]] = v[1]
                          end
                    end
                end
                ngx.header.content_type = "application/json"
                return ngx.say(json.encode(table_need))
                -- return ngx.exit(200)
            end
            return ngx.exit(403) -- can't find/authenticate user, refuse request

        end
           ngx.say(res.body)

    }

    }

    location   /custom {
    set $request_url $request_uri;
    internal;
    if ( $request_uri ~ ^/custom(.*)$ )
    { set $request_url  $1; }

    #set $request_uri  $request_url;
    fastcgi_pass_header Authorization;
    fastcgi_pass_request_headers on;
     fastcgi_param  REQUEST_URI $request_url;
    fastcgi_param QUERY_STRING  $query_string;
    fastcgi_param REQUEST_METHOD $request_method;
    fastcgi_param CONTENT_LENGTH $content_length;
    fastcgi_param  CONTENT_TYPE $content_type;

    include fastcgi_params;
    fastcgi_pass unix:/var/run/ceph/ceph-client.rgw.asok;
    }

    location /PUT/ {
    internal;
    fastcgi_pass_header Authorization;
    fastcgi_pass_request_headers on;

    include fastcgi_params;
    fastcgi_param QUERY_STRING  $query_string;
    fastcgi_param REQUEST_METHOD $request_method;
    fastcgi_param CONTENT_LENGTH $content_length;
    fastcgi_param  CONTENT_TYPE $content_type;
    fastcgi_pass unix:/var/run/ceph/ceph-client.rgw.asok;
    }

}
```
