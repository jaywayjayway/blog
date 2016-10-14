

title: lua实现验证码验证
date: 2016-10-04 09:44:10
tags:
- lua

categories: devops
---

配合API提供的验证，实现认证



``` lua 
    location = /code {
        proxy_read_timeout 1500;
        internal; proxy_pass http://api.renzheng.com:8333/identifycode?code=$arg_code&name=$arg_name;
    }

    location / {
        access_by_lua '
        local json = require("cjson")
        local whiteiplist = {"1.1.1.1"}     --加白
        local headers = ngx.req.get_headers()
        local request_uri = ngx.var.request_uri
        ngx.header["Access-Control-Allow-Headers"]="X-Auth-Token,X-Authorization,Content-type"
        if ngx.req.get_method() == "POST" then
            if request_uri == "/v2.0/tokens" then
                if headers["X-Auth-Token"] == nil then
                    local code = headers["X-Authorization"]
                    ngx.req.read_body()
                    local raw_json = ngx.req.get_body_data()
                    local success, body = pcall(json.decode,raw_json)
                    if not success then
                        return
                    end
                    if body.auth ~= nil then
                        if body.auth.passwordCredentials ~= nil then
                            if body.auth.passwordCredentials.username ~= nil then
                                local ip = ngx.req.get_headers()["X-Real-IP"]
                                if ip == nil then
                                    ip = ngx.var.remote_addr
                                end
                                if ip ~= nil then
                                    for _,client in pairs(whiteiplist) do
                                        if client == ip then
                                            return
                                        end
                                    end
                                end
                                local name = body.auth.passwordCredentials.username
                                local res = ngx.location.capture("/code",{args={code=code,name=name}})
                                if string.len(res.body) == 0 then
                                    ngx.exit(405)
                                end
                            end
                        end
                    end
                end
            end
        end
        ngx.req.clear_header("X-Authorization")
        ';
        try_files $uri @backend;
    }

```
