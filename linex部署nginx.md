select pull是被动模式

epoll是主动模式

举例:你想请别人吃饭  epoll模式你问谁还没吃饭  会主动举手告诉你

select pull  需要你一个个去询问

Nginx 服务 学习借助第三方库Openresty 它本身就是把nginx

1.编辑文件

zjh@ubuntu:/opt/openresty/nginx/conf$ vim /opt/openresty/nginx/conf/nginx.conf

2.启动测试

zjh@ubuntu:/opt/openresty/nginx/conf$ vim /opt/openresty/nginx/conf/nginx.conf  -t

3.启动

zjh@ubuntu:/opt/openresty/nginx/conf$ ps -ef | grep nginx



分流 :

1.先把conf下的nginx.conf 复制拷贝到conf.d中 

2.在conf.d中建project.conf ip8000来主控

把location中的root和index注掉,添加proxy_pass http://名称

在头部加

``` 
upstream 名称(和上面http://名称一致){
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
    server 127.0.0.1:8003;
}
ip 和分流的conf文件ip一致
```

3.在conf.d中建其他conf  IP 来分流

4.分流中的index 加入对应的html问价