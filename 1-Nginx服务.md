###                                                        Nginx服务

### 1   Nginx现状

​      nginx 是当前的`使用最广泛的webserver ,支持http正向/反向代理，支持TCP/UDP层代理`，来看下netcraft的数据

[![webserver_all](http://cuihuan.net/wp_content/new/nginx1/nginx_trend_all.png)](http://cuihuan.net/wp_content/new/nginx1/nginx_trend_all.png)
[![webserver_top](http://cuihuan.net/wp_content/new/nginx1/nginx_trend_top.png)](http://cuihuan.net/wp_content/new/nginx1/nginx_trend_top.png)

​    nginx在`全部网站中占比达到18%`，在`top millon busest 达到28%`，而且一直在增加。当下最时尚的webserver非nginx莫属



### 2   Nginx特点

- 性能好
  - 非阻塞IO/高并发,支持文件IO
  - 多worker，thread pool
  - 基于rbtree的定时器
  - 系统特性的持续支持
- 功能强大
  - webserver/cache/keepalive/pipeline等等
  - 各种upstream的支持【fastcgi/http/…】
  - 输出灵活【chunk/zip/…】
  - 在不断的发展 http2,tcp,udp,proxy…
- 运维的友好【这个对于开发和部署很重要】
  - 配置非常规范【个人认为：约定及规范是最好的实践】
  - 热加载和热更新【后文会详细介绍，能在二进制的层面热更新】
  - 日志强大【真的很强的，很多变量支撑】
- 扩展强大

​     下图是nginx、apache和lighttpd的一个对比。系统压力，内存占用，upstream支持等多个方面都非常不错
[![nginx对比图](http://cuihuan.net/wp_content/new/nginx1/nginx_compare.png)](http://cuihuan.net/wp_content/new/nginx1/nginx_compare.png)



HTTPS = HTTP + SSL

### 3  Nginx工作原理

​       Nginx的运行方式：`master-worker多进程模式运行，单线程/非阻塞执行`

​       Nginx 启动后生成master，master会启动conf数量的`worker进程`，当用户的请求过来之后，由不同的worker调起`执行线程`，非阻塞的执行请求。这种运行方式相对于`apache的进程执行`相对轻量很多，支撑的并发性也会高很多。   

​      Nginx 默认采用守护模式启动，守护模式让master进程启动后在后台运行。在Nginx运行期间主要由一个master主进程和多个worker进程（worker数目一般与cpu数目相同）

（1）master主进程

- master主进程主要是管理worker进程，对网络事件进行收集和分发：

1. 接收来自外界的信号
2. 向各worker进程发送信号
3. 监控worker进程的运行状态，当worker进程退出后(异常情况下)，会自动重新启动新的worker进程



（2）worker 工作进程

​     nginx用一个独立的worker进程来处理一个请求，一个worker进程可以处理多个请求：

​     a.  当一个worker进程在accept这个连接之后，就开始读取请求，解析请求，处理请求，产生数据后，再返回给客户端，最后才断开连接。

​     b.  一个请求，完全由worker进程来处理，而且只在一个worker进程中处理。采用这种方式的好处：

- - 节省锁带来的开销。对于每个worker进程来说，独立的进程，不需要加锁，所以省掉了锁带来的开销，同时在编程以及问题查上时，也会方便很多
  - 独立进程，减少风险。
  - 采用独立的进程，可以让互相之间不会影响，一个进程退出后，其它进程还在工作，服务不会中断，master进程则很快重新启动新的worker进程。
  - 在一次请求里无需进程切换



(3) Nginx的IO处理过程

​    一般的Tcp Socket处理过程：

​    服务端：fd = socket.socket   —> fd.bind()   —> fd.listen()   —> accept()  等待客户端连接  —>  send / recv  —> close()

​    客户端：fd = socket.socket  —> fd.connect()  与服务端建立连接  —> recv / send   —> close()



​     Nginx的网络IO处理通常使用epoll，epoll函数使用了I/O复用模型。与I/O阻塞模型比较，I/O复用模型的优势在于可以同时等待多个（而不只是一个）套接字描述符就绪。Nginx的epoll工作流程如下：

- - master进程先建好需要listen的socket后，然后再fork出多个woker进程，这样每个work进程都可以去accept这个socket
  - 当一个client连接到来时，所有accept的work进程都会受到通知，但只有一个进程可以accept成功，其它的则会accept失败，Nginx提供了一把共享锁accept_mutex来保证同一时刻只有一个work进程在accept连接，从而解决惊群问题。
  - 当一个worker进程accept这个连接后，就开始读取请求，解析请求，处理请求，产生数据后，再返回给客户端，最后才断开连接，这样一个完成的请求就结束了.



## 4  Nginx 安装与配置

​       Nginx服务学习我们借助于一个第三方库Openresty，它本身就是把nginx核心代码做了一层封装，你完全可以把它当成Nginx使用。

​        OpenResty 是一个基于 [Nginx](https://openresty.org/cn/nginx.html) 与 Lua 的高性能 Web 平台，其内部集成了大量精良的 Lua 库、第三方模块以及大多数的依赖项。用于方便地搭建能够处理超高并发、扩展性极高的动态 Web 应用、Web 服务和动态网关。

​        OpenResty 的目标是让你的Web服务直接跑在 [Nginx](https://openresty.org/cn/nginx.html) 服务内部，充分利用 [Nginx](https://openresty.org/cn/nginx.html) 的非阻塞 I/O 模型，不仅仅对 HTTP 客户端请求,甚至于对远程后端诸如 MySQL、PostgreSQL、Memcached 以及 Redis 等都进行一致的高性能响应。

Openresty下载页：

https://openresty.org/cn/download.html

下载最新版本：wget  https://openresty.org/download/openresty-1.11.2.5.tar.gz

性能测试工具 Apache AB test

安装过程：

（1）安装依赖库：

​     sudo apt-get install libpcre3 libpcre3-dev

​    sudo apt-get install openssl libssl-dev

（2）安装openresty 

```
#将openresty安装到/opt/openresty目录下
sudo mkdir /opt/openresty   
#修改组和用户权限
sudo chown -Rf zhouguangyou /opt/openresty/
sudo chgrp -Rf zhouguangyou /opt/openresty/
tar -xzvf openresty-1.11.2.5.tar.gz
cd openresty-1.11.2.5 
./configure  --prefix=/opt/openresty --with-http_gzip_static_module    (注:./configure --help 查看更多的配置选项)
make
sudo make install
```

至此，安装完成，安装openresty就在/opt/openresty目录下。

ls

（3）目录介绍

  bin                          openresty 的启动文件

COPYRIGHT             版权文件

 luajit                        lua虚拟环境luajit

lualib                       lua实现的第三方库，包括redis， mysql， upload， upstream，websocket等等。

nginx                      nginx核心功能块

pod     resty.index    site



​       openresty不只是提供了nginx功能，而且提供了丰富的工具集，我们可以做除了负载均衡和反向代理之外的很多事情，快速搭建出高性能web服务。

修改  vim  /opt/openresty/nginx/conf/nginx.conf 中，将listen 80改成 8080:	:

启动nginx服务：

$ /opt/openresty/nginx/sbin/nginx

打开页面：http://172.16.245.180:8080/

可以看到 Welcome   to  OpenResty ！ 的页面，表示已经安装成功！



## 5 OpenResty 使用

#### 5.1  配置文件

   Openresty 框架的配置和nginx配置方法一样，配置文件: /opt/openresty/nginx/conf/nginx.conf

​    Nginx主要通过nginx.conf文件进行配置使用。在nginx.conf文件中主要分为：

- - 全局块：一些全局的属性，在运行时与具体业务功能（比如http服务或者email服务代理）无关的一些参数，比如工作进程数，运行的身份等
  - event块：参考事件模型，单个进程最大连接数等
  - http块：设定http服务器
  - server块：配置虚拟主机
  - location块：配置请求路由及页面的处理情况等



   关键参数说明：

```
#nginx进程数，建议设置为等于CPU总核心数。
worker_processes 8;

#全局错误日志定义类型，[ debug | info | notice | warn | error | crit ]
error_log /usr/local/nginx/logs/error.log info;

#进程pid文件
pid /opt/openresty/nginx/logs/nginx.pid;

#指定进程可以打开的最大描述符：数目
#工作模式与连接数上限
#这个指令是指当一个nginx进程打开的最多文件描述符数目，理论值应该是最多打开文件数（ulimit -n）与nginx进程数相除，但是nginx分配请求并不是那么均匀，所以最好与ulimit -n 的值保持一致。
worker_rlimit_nofile 65535;


#虚拟主机的配置
    server
    {
        #监听端口
        listen 80;

        #域名可以有多个，用空格隔开
        server_name www.jd.com jd.com;
        index index.html index.htm index.php;
        root /data/www/jd;
        
        #url 请求路由
        location  /hello {
            default_type text/html;
            content_by_lua '
                ngx.say("<p>Hello, World!</p>")
            ';
        }
        
    }
    
    
   #负载均衡配置
    upstream piao.jd.com {
        #upstream的负载均衡，weight是权重，可以根据机器配置定义权重。weight参数表示权值，权值越高被分配到的几率越大。
        server 192.168.80.121:80 weight=3;
        server 192.168.80.122:80 weight=2;
        server 192.168.80.123:80 weight=3;
        
   }
```



#### 5.2   启动测试

拷贝我们的工程文件 artproject.zip，解压到某一个目录 eg:  /home/zhouguangyou根目录下

配置nginx配置文件nginx.conf，添加如下信息：

```
location /hello
{
   default_type   text/html;
   content_by_lua  '
       ngx.say("<p>Hello, World!</p>")';
}


# 静态文件，nginx自己处理
location  ~ ^/(images|javascript|js|css|flash|media|static)/ {
   root   /home/zhouguangyou/artproject/art;
   # 过期1天，静态文件不怎么更新，过期可以设大一点，如果频繁更新，则可以设置得小一点。
   expires  1d;
}

```



$ /opt/openresty/nginx/sbin/nginx -s reload
$ /opt/openresty/nginx/sbin/nginx -t
nginx: the configuration file /opt/openresty/nginx/conf/nginx.conf syntax is ok
nginx: configuration file /opt/openresty/nginx/conf/nginx.conf test is successful



查看页面效果

http://172.16.245.180:8080/hello

http://172.16.245.180:8080/static/admin/pages/index.html





### 5.3  负载均衡策略

​      

​      负载均衡也是Nginx常用的一个功能，负载均衡其意思就是分摊到多个操作单元上进行执行，例如Web服务器、FTP服务器、企业关键应用服务器和其它关键任务服务器等，从而共同完成工作任务。

​      Nginx目前支持自带3种负载均衡策略，还有2种常用的第三方策略

1.  RR （轮询策略）

​        按照轮询（默认）方式进行负载，每个请求按时间顺序逐一分配到不同的后端服务器，如果后端服务器down掉，能自动剔除。虽然这种方式简便、成本低廉。但缺点是：可靠性低和负载分配不均衡。

2. 权重 
   指定轮询几率，weight和访问比率成正比，用于后端服务器性能不均的情况。

   ```
   upstream test{
        server localhost:8080 weight=9;
        server localhost:8081 weight=1;
   }
   ```

   此时8080和8081分别占90%和10%。

3. ip_hash 
   ​      上面的2种方式都有一个问题，那就是下一个请求来的时候请求可能分发到另外一个服务器，当我们的程序不是无状态的时候（采用了session保存数据），这时候就有一个很大的很问题了，比如把登录信息保存到了session中，那么跳转到另外一台服务器的时候就需要重新登录了，所以很多时候我们需要一个客户只访问一个服务器，那么就需要用iphash了，iphash的每个请求按访问ip的hash结果分配，这样每个访客固定访问一个后端服务器，可以解决session的问题。

   ```
   upstream test {
       ip_hash;
       server localhost:8080;
       server localhost:8081;
   }
   ```

4. fair(第三方) 
   按后端服务器的响应时间来分配请求，响应时间短的优先分配。

   ```
   upstream backend { 
       fair; 
       server localhost:8080;
       server localhost:8081;
   }
   ```

5. url_hash(第三方) 
   按访问url的hash结果来分配请求，使每个url定向到同一个后端服务器，后端服务器为缓存时比较有效。 在upstream中加入hash语句，server语句中不能写入weight等其他的参数，hash_method是使用的hash算法。

   ```
   upstream backend { 
       hash $request_uri; 
       hash_method crc32; 
       server localhost:8080;
       server localhost:8081;
   } 
   ```



   处理动态请求转发到某一个服务

​         location = / {  

​                  proxy_pass   http://localhost:8080  

​       }  

​       此处的proxy_pass 对应的服务，会导到上述upstream入口

### 5.4 作为静态资源服务器

​       

​       Nginx本身也是一个静态资源的服务器，当只有静态资源的时候，就可以使用Nginx来做服务器，同时现在也很流行动静分离，就可以通过Nginx来实现，动静分离是让动态网站里的动态网页根据一定规则把不变的资源和经常变的资源区分开来，动静资源做好了拆分以后，我们就可以根据静态资源的特点将其做缓存操作（CDN），这就是网站静态化处理的核心思路。

```
# 静态文件，nginx自己处理
location  ~ ^/(images|javascript|js|css|flash|media|static)/ {
   root   /home/zhouguangyou/artproject/art;
   # 过期1天，静态文件不怎么更新，过期可以设大一点，如果频繁更新，则可以设置得小一点。
   expires  1d;
}
```





### 5.5  关于URL路由规则

语法规则：

 location [=|~|~*|^~] /uri/ 

{ 

​     … 

}

= 开头表示精确匹配
^~ 开头表示uri以某个常规字符串开头，理解为匹配 url路径即可。

~ 开头表示区分大小写的正则匹配
~*  开头表示不区分大小写的正则匹配
!~和!~*分别为区分大小写不匹配及不区分大小写不匹配 的正则
/ 通用匹配，任何请求都会匹配到。
多个location配置的情况下匹配顺序为：

首先匹配 =，其次匹配^~, 其次是按文件中顺序的正则匹配，最后是交给 / 通用匹配。当有匹配成功时候，停止匹配，按当前匹配规则处理请求。



例子，有如下匹配规则：

```
location = / {
   #规则A
}
location = /login {
   #规则B
}
location ^~ /static/ {
   #规则C
}
location ~ \.(gif|jpg|png|js|css)$ {
   #规则D
}
location ~* \.png$ {
   #规则E
}
location !~ \.xhtml$ {
   #规则F
}
location !~* \.xhtml$ {
   #规则G
}
location / {
   #规则H
}
```



那么产生的效果如下:

访问根目录/， 比如http://localhost/ 将匹配规则A
访问 http://localhost/login 将匹配规则B，http://localhost/register 则匹配规则H
访问 http://localhost/static/a.html 将匹配规则C
访问 http://localhost/a.gif, http://localhost/b.jpg 将匹配规则D和规则E，但是规则D顺序优先，规则E不起作用，而 http://localhost/static/c.png 则优先匹配到规则C
访问 http://localhost/a.PNG 则匹配规则E，而不会匹配规则D，因为规则E不区分大小写。

访问 http://localhost/a.xhtml 不会匹配规则F和规则G，http://localhost/a.XHTML不会匹配规则G，因为不区分大小写。规则F，规则G属于排除法，符合匹配规则但是不会匹配到，所以想想看实际应用中哪里会用到。

访问 http://localhost/category/id/1111 则最终匹配到规则H，因为以上规则都不匹配，这个时候应该是nginx转发请求给后端应用服务器，比如FastCGI（php），tomcat（jsp），nginx作为方向代理服务器存在。







