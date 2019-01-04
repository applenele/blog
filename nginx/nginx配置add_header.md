## add_header

1. add_header 指令属于ngx_http_headers_module
2. 在配置中加了add_header 则相应的配置无法继承上一层的配置，即请求的过来的header将被重写。

## 例子

```shell
server {
        listen 80;
        server_name localhost 127.0.0.1 demo.com;
        root /www;
        location ~ \.php {
        client_body_timeout 6s;
        	if ($request_method = 'OPTIONS') {
            	add_header 'Access-Control-Allow-Origin' '*' always;
            	add_header 'Access-Control-Allow-Credentials' 'true';
            	add_header 'Access-Control-Allow-Methods' 'GET, POST, PATCH, DELETE, PUT, OPTIONS';
             	add_header 'Access-Control-Allow-Headers' 'DNT,X-Mx-ReqToken,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,  Access-Control-Expose-Headers, Token, Authorization';
            	add_header 'Access-Control-Max-Age' 1728000;
            	add_header 'Content-Type' 'text/plain charset=UTF-8';
            	add_header 'Content-Length' 0;
            	return 204;
        	}
        	add_header 'Access-Control-Allow-Origin' '*' always;
        	fastcgi_pass    unix:/run/php7.0-fpm.sock;
        	include         snippets/fastcgi-php.conf;
        	include         fastcgi_params;
        	fastcgi_param   SCRIPT_FILENAME $document_root$fastcgi_script_name;
    	}
}
```



1. Access-Control-Allow-Methods 限制请求的方法
2. Access-Control-Allow-Headers 限制请求头，其他的头都被去掉
3. 其他的一次类推