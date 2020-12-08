###1、代理介绍

*   实践中客户端无法直接跟服务端发起请求的时候，我们就需要代理服务。代理可以实现客户端与服务端之间的通信，我们的Nginx也可以实现相应的代理服务。代理分为正向代理和反向代理。

###2、正向代理

​	正向代理示例图：

​	![img](https://f10.baidu.com/it/u=3132587007,2920873354&fm=173&app=49&f=JPEG?w=640&h=391&s=80124F326F0A5443066D14D60200D0B2&access=215967316)

*   客户端 <--> 代理 -->服务端
*   说到正向代理，曾经看过一篇文章，说正向代理好比租房子。
*   客户想租房东的房子，但是客户并不认识房东，所以租不到房子。中介(代理)认识房东，能租这个房子所以你找了中介(代理)帮忙租到了这个房子。
*   在这个租借过程中，房东并不认识客户，只认识中介(代理)，这时候房东并不知道谁租了他的房子，他只知道房子租给了中介(代理)。

1.  现实中的场景

    1.  在如今的网络环境下，由于技术需要，要去访问国外的某些网站，此时你发现位于国外的某网站我们通过浏览器是没有办法访问的，此时大家可能都会用一个操作FQ进行访问，FQ的方式主要是找到一个可以访问国外网站的代理服务器，我们将请求发送给代理服务器，代理服务器去访问国外的网站，然后将访问到的数据传递给我们！
    2.  正向代理模式屏蔽或者隐藏了真实客户端信息。

2.  配置正向代理

    ~~~nginx
    server {
    	resolver 8.8.8.8; #为DNS解析,这里填写的IP为Google提供的免费DNS服务器的IP地址
    	resolver_timeout 5s; #解析超时时间
    	listen 80;  #监听的端口
    	location / {
    		if ( $remote_addr !~* "^127\.0\.0\.1") {
    		return403;
    	}
    	proxy_pass http://$http_host$request_uri; #$http_host 要访问的主机域名,$request_uri 后面所加的参数。
    	proxy_set_header HOST $http_host; #重写上面代理主机域名
    	proxy_buffers 256 4k; #配置缓存大小
    	proxy_max_temp_file_size 0k; #关闭磁盘缓存读写减少I/O
    	proxy_connect_timeout 30; #代理连接超时时间
    	proxy_send_timeout 60; #代理发送超时时间
    	proxy_read_timeout 60; #代理读取超时时间
    	.....
    }
    }
    ~~~

3.  测试示例配置

    ~~~mysql
    server {
    	resolver 8.8.8.8;
    	listen 8080;
    	location / {
    		proxy_pass http://$http_host$request_uri;
    	}
    }
    ~~~

    终端中输入 curl -l --proxy 127.0.0.1:8080 "www.baidu.com"，看到可以被重定向到百度。

###3、反向代理

​	反向代理示例图

​	![img](https://f12.baidu.com/it/u=475559834,2653493006&fm=173&app=49&f=JPEG?w=543&h=271&s=5AA83C6227405D431C5074CC0000E0B0&access=215967316)

*   客户端 -->代理 <--> 服务端
*   反向代理还是租房子例子：
    *   客户端想租一个房子，二房东(代理)就把这个房子租给了他。这个房子实际上不是他的，客户也不知道他是不是真的房东，房东和客户都不知道对方的存在。

1.  现实中的场景

    *   某宝网站，每天同时连接到网站的访问量大大超过单个服务器所能承受的能力，单个服务器远远不能满足业务的需求了，因此就出现了分布式部署；分布式就是通过部署多台服务器来解决服务器压力的问题。
    *   多个客户端给服务器发送的请求，服务器接收到之后，按照一定的规则分发给了后端的业务服务器进行处理。请求的来源是明确的，就是客户端。但是请求是哪台服务器处理我们并不知道。
    *   反向代理隐藏了服务器的信息。
    *   由上的例子和图我们可以知道正向代理和反向代理的区别在于代理的对象不一样，正向代理的代理对象是客户端，反向代理的代理对象是服务端。

2.  配置反向代理

    ~~~nginx
    upstream typefunc {
    	server ip1:8080; #Apache
    	server ip2:8080; #tomcat
    }
    server {
    	server_name www.test.cn;
    	listen 80; #监听的端口
    	location / {
    		proxy_pass http://typefunc;
    		proxy_redirect off;
    		proxy_set_header Host $host;
    		proxy_set_header X-Real-IP $remote_addr;
    		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    		.....
    	}
    }
    ~~~

    浏览器输入www.test.cn就会随机访问上面两台服务器。