

title: nginx 限制url的IP访问
date: 2016-10-14 09:44:10
tags:
- lua
- nginx

categories: devops
---

针对某些IP开放特苏的URL访问，比如说网站的管理后台。在nginx里我们可以使用`geo`和`map`的使用，通过变量的判断来实现。当然，最便捷的方式就是通过多域名的方式。


`geo`和`map`使用的变量是全局有效的，在各`server`段是共享的
``` powershell
geo $remote_addr $denied1 {
        default 1;
        124.127.138.32/27 0;
        124.243.227.224/32 0;
        106.75.33.149/32  0;
}

map $request_uri $denied2 {
        default 0;
        ~^/admin.php 1;
}

```
在`server`段里添加判断指令
```powershell

   if ($denied1) {
        set $denied o;
    }
    if ($denied2) {
        set $denied "${denied}o";
    }

    if ($denied = 'oo') {
        return 403;
    }
```

如果后端解析是PHP,也可以通过`location`指令，不过在`location`里必须添加完整`PHP`解析，不然会报错
``` powershell

    location ~* ^/admin.php  {
        allow 1.1.1.1;
        allow 12.12.12.0/24;
        deny all;
        location ~ .*\.php {
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            fastcgi_index  index.php;
            fastcgi_split_path_info ^(.+\.php)(.*)$;
            fastcgi_param   SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_param   PATH_INFO $fastcgi_path_info;
            fastcgi_param   PATH_TRANSLATED $document_root$fastcgi_path_info;
            include fastcgi_params;
        }
    }
```
