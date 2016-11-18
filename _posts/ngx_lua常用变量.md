title: ngx_lua常用变量
date: 2016-11-14 09:44:10
tags:
- nginx
- lua
categories: devops
---

```powershell
ngx.arg[index]              #ngx指令参数，当这个变量在set_by_lua或者set_by_lua_file内使用的时候是只读的，指的是在配置指令输入的参数.  
ngx.var.varname             #读写NGINX变量的值,最好在lua脚本里缓存变量值，避免在当前请求的生命周期内内存的泄漏  
ngx.config.ngx_lua_version  #当前ngx_lua模块版本号  
ngx.config.nginx_version    #nginx版本  
ngx.worker.exiting          #当前worker进程是否正在关闭  
ngx.worker.pid              #当前worker进程的PID  
ngx.config.nginx_configure  #编译时的./configure命令选项  
ngx.config.prefix           #编译时的prefix选项  
  
core constans:              #ngx_lua 核心常量  
    ngx.OK (0)  
    ngx.ERROR (-1)  
    ngx.AGAIN (-2)  
    ngx.DONE (-4)  
    ngx.DECLINED (-5)  
    ngx.nil  
http method constans:       #经常在ngx.location.catpure和ngx.location.capture_multi方法中被调用.  
    ngx.HTTP_GET  
    ngx.HTTP_HEAD  
    ngx.HTTP_PUT  
    ngx.HTTP_POST  
    ngx.HTTP_DELETE  
    ngx.HTTP_OPTIONS    
    ngx.HTTP_MKCOL      
    ngx.HTTP_COPY        
    ngx.HTTP_MOVE       
    ngx.HTTP_PROPFIND   
    ngx.HTTP_PROPPATCH   
    ngx.HTTP_LOCK   
    ngx.HTTP_UNLOCK      
    ngx.HTTP_PATCH     
    ngx.HTTP_TRACE    
http status constans:       #http请求状态常量   
    ngx.HTTP_OK (200)  
    ngx.HTTP_CREATED (201)  
    ngx.HTTP_SPECIAL_RESPONSE (300)  
    ngx.HTTP_MOVED_PERMANENTLY (301)  
    ngx.HTTP_MOVED_TEMPORARILY (302)  
    ngx.HTTP_SEE_OTHER (303)  
    ngx.HTTP_NOT_MODIFIED (304)  
    ngx.HTTP_BAD_REQUEST (400)  
    ngx.HTTP_UNAUTHORIZED (401)  
    ngx.HTTP_FORBIDDEN (403)  
    ngx.HTTP_NOT_FOUND (404)  
    ngx.HTTP_NOT_ALLOWED (405)  
    ngx.HTTP_GONE (410)  
    ngx.HTTP_INTERNAL_SERVER_ERROR (500)  
    ngx.HTTP_METHOD_NOT_IMPLEMENTED (501)  
    ngx.HTTP_SERVICE_UNAVAILABLE (503)  
    ngx.HTTP_GATEWAY_TIMEOUT (504)   
  
Nginx log level constants：      #错误日志级别常量 ,这些参数经常在ngx.log方法中被使用.  
    ngx.STDERR  
    ngx.EMERG  
    ngx.ALERT  
    ngx.CRIT  
    ngx.ERR  
    ngx.WARN  
    ngx.NOTICE  
    ngx.INFO  
    ngx.DEBUG  
  
##################  
#API中的方法：  
##################  
print()                         #与 ngx.print()方法有区别，print() 相当于ngx.log()  
ngx.ctx                         #这是一个lua的table，用于保存ngx上下文的变量，在整个请求的生命周期内都有效,详细参考官方  
ngx.location.capture()          #发出一个子请求，详细用法参考官方文档。  
ngx.location.capture_multi()    #发出多个子请求，详细用法参考官方文档。  
ngx.status                      #读或者写当前请求的相应状态. 必须在输出相应头之前被调用.  
ngx.header.HEADER               #访问或设置http header头信息，详细参考官方文档。  
ngx.req.set_uri()               #设置当前请求的URI,详细参考官方文档  
ngx.set_uri_args(args)          #根据args参数重新定义当前请求的URI参数.  
ngx.req.get_uri_args()          #返回一个LUA TABLE，包含当前请求的全部的URL参数  
ngx.req.get_post_args()         #返回一个LUA TABLE，包括所有当前请求的POST参数  
ngx.req.get_headers()           #返回一个包含当前请求头信息的lua table.  
ngx.req.set_header()            #设置当前请求头header某字段值.当前请求的子请求不会受到影响.  
ngx.req.read_body()             #在不阻塞ngnix其他事件的情况下同步读取客户端的body信息.[详细]  
ngx.req.discard_body()          #明确丢弃客户端请求的body  
ngx.req.get_body_data()         #以字符串的形式获得客户端的请求body内容  
ngx.req.get_body_file()         #当发送文件请求的时候，获得文件的名字  
ngx.req.set_body_data()         #设置客户端请求的BODY  
ngx.req.set_body_file()         #通过filename来指定当前请求的file data。  
ngx.req.clear_header()          #清求某个请求头  
ngx.exec(uri,args)              #执行内部跳转，根据uri和请求参数  
ngx.redirect(uri, status)       #执行301或者302的重定向。  
ngx.send_headers()              #发送指定的响应头  
ngx.headers_sent                #判断头部是否发送给客户端ngx.headers_sent=true  
ngx.print(str)                  #发送给客户端的响应页面  
ngx.say()                       #作用类似ngx.print，不过say方法输出后会换行  
ngx.log(log.level,...)          #写入nginx日志  
ngx.flush()                     #将缓冲区内容输出到页面（刷新响应）  
ngx.exit(http-status)           #结束请求并输出状态码  
ngx.eof()                       #明确指定关闭结束输出流  
ngx.escape_uri()                #URI编码(本函数对逗号,不编码，而php的urlencode会编码)  
ngx.unescape_uri()              #uri解码  
ngx.encode_args(table)          #将tabel解析成url参数  
ngx.decode_args(uri)            #将参数字符串编码为一个table  
ngx.encode_base64(str)          #BASE64编码  
ngx.decode_base64(str)          #BASE64解码  
ngx.crc32_short(str)            #字符串的crs32_short哈希  
ngx.crc32_long(str)             #字符串的crs32_long哈希  
ngx.hmac_sha1(str)              #字符串的hmac_sha1哈希  
ngx.md5(str)                    #返回16进制MD5  
ngx.md5_bin(str)                #返回2进制MD5  
ngx.today()                     #返回当前日期yyyy-mm-dd  
ngx.time()                      #返回当前时间戳  
ngx.now()                       #返回当前时间  
ngx.update_time()               #刷新后返回  
ngx.localtime()                 #返回 yyyy-mm-dd hh:ii:ss  
ngx.utctime()                   #返回yyyy-mm-dd hh:ii:ss格式的utc时间  
ngx.cookie_time(sec)            #返回用于COOKIE使用的时间  
ngx.http_time(sec)              #返回可用于http header使用的时间        
ngx.parse_http_time(str)        #解析HTTP头的时间  
ngx.is_subrequest               #是否子请求（值为 true or false）  
ngx.re.match(subject,regex,options,ctx)     #ngx正则表达式匹配，详细参考官网  
ngx.re.gmatch(subject,regex,opt)            #全局正则匹配  
ngx.re.sub(sub,reg,opt)         #匹配和替换（未知）  
ngx.re.gsub()                   #未知  
ngx.shared.DICT                 #ngx.shared.DICT是一个table 里面存储了所有的全局内存共享变量  
    ngx.shared.DICT.get    
    ngx.shared.DICT.get_stale      
    ngx.shared.DICT.set    
    ngx.shared.DICT.safe_set       
    ngx.shared.DICT.add    
    ngx.shared.DICT.safe_add       
    ngx.shared.DICT.replace    
    ngx.shared.DICT.delete     
    ngx.shared.DICT.incr       
    ngx.shared.DICT.flush_all      
    ngx.shared.DICT.flush_expired      
    ngx.shared.DICT.get_keys  
ndk.set_var.DIRECTIVE           #不懂
```
