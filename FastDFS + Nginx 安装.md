FastDFS + Nginx 安装

系统：Ubuntu16.04

**1、下载安装**libfastcommon（需root权限）

由于fastdfs5.0.5依赖libfastcommon,先安装libfastcommon

```alloy
wget https://github.com/happyfish100/libfastcommon/archive/V1.0.7.tar.gz
```

解压libfastcommon，命令：

```text
tar -zxvf V1.0.7.tar.gz
```

编译，进入libfastcommon-1.0.7目录，命令：

```text
cd libfastcommon-1.0.7
  ./make.sh
```

安装，命令：

```text
 ./make.sh install
```

![屏幕快照 2019-02-27 23.17.48](/Users/zouziwei/Desktop/屏幕快照 2019-02-27 23.17.48.png)

显示如上图，libfastcommon 安装成功，

设置软连接，命令：

```text
ln -s /usr/lib64/libfastcommon.so /usr/local/lib/libfastcommon.so
ln -s /usr/lib64/libfastcommon.so /usr/lib/libfastcommon.so
ln -s /usr/lib64/libfdfsclient.so /usr/local/lib/libfdfsclient.so
ln -s /usr/lib64/libfdfsclient.so /usr/lib/libfdfsclient.so
```

**2、下载安装FastDFS**

下载（版本不能错），命令：

```text
wget https://github.com/happyfish100/fastdfs/archive/V5.05.tar.gz
```

解压FastDFS，命令：

```text
tar -zxvf V5.05.tar.gz
```

编译，进入fastfds-5.05目录，命令：

```text
cd fastdfs-5.05
./make.sh
```

安装，命令：

```text
./make.sh install
```

![屏幕快照 2019-02-27 23.39.57](/Users/zouziwei/Desktop/屏幕快照 2019-02-27 23.39.57.png)

安装成功

#### **配置Tracker与Storage**

Tracker 192.168.7.73:22122

FastDFS安装成功后，会在/etc目录下会有个fdfs目录，进入fdfs，会发现三个.sample后缀的示例文件。

**1、配置Tracker服务器**

在/etc/fdfs目录下，修改tracker.conf，命令：

```text
 cp tracker.conf.sample tracker.conf
   vim tracker.conf
```

打开tracker.conf，修改如下处：

```text
# the base path to store data and log files
base_path=/data/fastdfs/tracker
```

当然前提是，首先要创建/data/fastdfs/tracker目录，命令：

```text
mkdir -p /data/fastdfs/tracker
```

修改保存， 启动tracker服务，命令：

```text
fdfs_trackerd /etc/fdfs/tracker.conf start
```

类似的命令，关闭tracker服务：

```text
fdfs_trackerd /etc/fdfs/tracker.conf stop
```

启动tracker服务后，查看监听，命令：

```text
netstat -unltp|grep fdfs
```

**2、配置Storage服务器**

两台服务器，同样进入/etc/fdfs目录下，命令：

```text
cp storage.conf.sample storage.conf
vim storage.conf
```

打开storage.conf，修改如下处：

```text
# the base path to store data and log files
base_path=/data/fastdfs/storage

# store_path#, based 0, if store_path0 not exists, it's value is base_path
# the paths must be exist
store_path0=/data/fastdfs/storage

# tracker_server can ocur more than once, and tracker_server format is
#  "host:port", host can be hostname or ip address
#配置tracker跟踪器ip端口
tracker_server=如果是在本地启动，就设置成本地ip.22122
```

当然前提是，首先要创建/data/fastdfs/storage目录，命令：

```text
mkdir -p /data/fastdfs/storage
```

修改保存后，启动storage服务，命令：可能会有点慢

```text
fdfs_storaged /etc/fdfs/storage.conf start
```

启动有错误，可以通过/data/fastdfs/storage/logs查看

查看/data/fastdfs/storage下文件内容，生成logs、data两个目录

查看下端口监听，命令：

```text
netstat -unltp|grep fdfs
```

storage默认端口23000



**至此Storage存储节点安装成功。**

所有存储节点都启动之后，可以在任一存储节点上使用如下命令查看集群的状态信息：

```text
/usr/bin/fdfs_monitor /etc/fdfs/storage.conf
```

![img](https://pic4.zhimg.com/80/v2-bd303ab61d3161724b8da5bfbbd801cf_hd.png)



![img](https://pic4.zhimg.com/80/v2-44b0bf570c09239c193071195326db3f_hd.png)

通过上两图可以看到，两台storage都为**Active**，配置成功



**五、测试上传文件**

三台服务器随便选择一台服务器，这里我选择192.168.7.44服务器

同样进入/etc/fdfs目录，编译client.conf，命令如下：

```text
cp /etc/fdfs/client.conf.sample /etc/fdfs/client.conf
vim /etc/fdfs/client.conf
```

![img](https://pic1.zhimg.com/80/v2-faff255eb5846011bc87b83d182777f8_hd.png)



修改如下：

```text
# the base path to store log files
base_path=/data/fastdfs/client

# tracker_server can ocur more than once, and tracker_server format is
#  "host:port", host can be hostname or ip address
#配置tracker跟踪器ip端口
tracker_server=192.168.7.73:22122
```

同样前提是，首先要创建/data/fastdfs/client目录，命令：

```text
mkdir -p /data/fastdfs/client
```



上传/opt目录的一张图片（名为：14.jpg），命令：

```text
fdfs_test /etc/fdfs/client.conf upload /opt/14.jpg
```

![img](https://pic1.zhimg.com/80/v2-c47aa205923b6d6e2c15a795b47e1544_hd.png)

如上图，上传成功，分别进入两台storage服务器目录/data/fastdfs/storage/data/00/00下，都可以发现，文件保存成功

![img](https://pic2.zhimg.com/80/v2-ec7eed3905795683f55074aa7dfd823d_hd.png)

**至此，文件上传测试成功。**



#### Nginx配置

**2、在所有Storage节点下载安装fastdfs-nginx-module**

a、fastdfs-nginx-module介绍

FastDFS 通过 Tracker 服务器，将文件放在 Storage服务器存储，但是同组存储服务器之间需要进入文件复制，有同步延迟的问题。假如Tracker 服务器将文件上传到192.168.7.149，上传成功后文件ID已经返回给客户端。此时 FastDFS 存储集群机制会将这个文件同步到同组存储 192.168.7.44，在文件还没有复制完成的情况下，客户端如果用这个文件 ID192.168.7.44
上取文件，就会出现文件无法访问的错误。而 **fastdfs-nginx-module 可以重定向文件连接到源服务器取文件，避免客户端由于复制延迟导致的文件无法访问错误。**

b、下载fastdfs-nginx-module，本文下载在/opt/fastdfs文件夹中，命令：

```text
wget http://jaist.dl.sourceforge.NET/project/fastdfs/FastDFS%20Nginx%20Module%20Source%20Code/fastdfs-nginx-module_v1.16.tar.gz
```

![img](https://pic1.zhimg.com/80/v2-fe3f5ce9698ba0e461fd59660ac9e95c_hd.png)

c、解压，命令：

```text
tar -zxvf fastdfs-nginx-module_v1.16.tar.gz
```

![img](https://pic1.zhimg.com/80/v2-7ac1013d5248f84976fc524fa671ffa4_hd.png)

d、配置config文件，命令：

```text
cd /opt/fastdfs/fastdfs-nginx-module/src
vim config
```

修改如下：**不修改后面编译nginx会报错**

找到下面一行包含有 local 字眼去掉

CORE_INCS="$CORE_INCS /usr/local/include/fastdfs /usr/local/include/fastcommon/"

改为如下：

CORE_INCS="$CORE_INCS /usr/include/fastdfs /usr/include/fastcommon/"



e、复制 fastdfs-nginx-module 源码中的配置文件到/etc/fdfs 目录， 并修改，命令：

进入fastdfs-nginx-module/src目录下，复制mod_fastdfs.conf文件到 /etc/fdfs目录，进入/etc/fdfs目录，修改mod_fastdfs.conf配置文件

```text
cd fastdfs-nginx-module/src
cp mod_fastdfs.conf /etc/fdfs
cd /etc/fdfs
vim mod_fastdfs.conf
```

![img](https://pic4.zhimg.com/80/v2-a85f163161abf6978f4042b7026b0ec3_hd.png)

修改如下三处：

tracker_server=192.168.7.73:22122 # tracker服务IP和端口
url_have_group_name=true # 访问链接前缀加上组名
store_path0=/data/fastdfs/storage # 文件存储路径

![img](https://pic2.zhimg.com/80/v2-aca8e842d092418e3e9043a8ebe0c3d5_hd.png)

修改保存。

**3、在所有Storage节点下载安装nginx**

a、Nginx安装依赖如下（gcc/pcre/zlib/openssl）插件，先要安装如下插件，命令：

```text
apt-get install openssl libssl-dev
apt-get install libpcre3 libpcre3-dev
apt-get install zlib1g-dev
apt-get install build-essential
```



b、下载nginx，命令

```text
wget -c https://nginx.org/download/nginx-1.10.1.tar.gz
```

![img](https://pic4.zhimg.com/80/v2-5c2a29d78f915de7b29840cee15a93b3_hd.png)



c、解压，命令：

```text
tar -zxvf nginx-1.10.1.tar.gz
```

![img](https://pic3.zhimg.com/80/v2-6230d4b4d27598c570aa1000204daa12_hd.png)

d、编译配置，命令：

- 注意最后一行，配置fastdfs-nginx模块的路径（这个路径根据自己实际情况而定）
- --add-module=/opt/fastdfs/fastdfs-nginx-module/src 这个要和自己fastdfs-nginx-module安装的位置对应上

```text
cd nginx-1.10.1

./configure \
--prefix=/usr/local/nginx \
--pid-path=/var/local/nginx/nginx.pid \
--lock-path=/var/lock/nginx/nginx.lock \
--error-log-path=/var/log/nginx/error.log \
--http-log-path=/var/log/nginx/access.log \
--with-http_gzip_static_module \
--http-client-body-temp-path=/var/temp/nginx/client \
--http-proxy-temp-path=/var/temp/nginx/proxy \
--http-fastcgi-temp-path=/var/temp/nginx/fastcgi \
--http-uwsgi-temp-path=/var/temp/nginx/uwsgi \
--http-scgi-temp-path=/var/temp/nginx/scgi \
--add-module=/opt/fastdfs/fastdfs-nginx-module/src
```

![img](https://pic3.zhimg.com/80/v2-f4d8477f5c9794aae25cbef5ca4d238e_hd.png)



e、编译安装，命令：

```text
make && make install  
```



f、启动，命令：

```text
/usr/local/nginx/sbin/nginx
```

![img](https://pic3.zhimg.com/80/v2-82f858def6af8497127734095e3eb4a6_hd.png)

我这里启动报错，找不到那个目录，于是手动创建，再启动

```text
mkdir -p /var/temp/nginx/client
```

![img](https://pic1.zhimg.com/80/v2-90018c888c84e97d817936557469cab4_hd.png)

nginx默认端口80，查看命令：

```text
netstat -anp|grep 80
```

![img](https://pic1.zhimg.com/80/v2-4167015e759ba03ea2cb9c1b97476b88_hd.png)

**如上图，至此nginx安装成功**

**4、其他配置**

复制 FastDFS 的部分配置文件到/etc/fdfs 目录，命令：

```text
cd /opt/fastdfs/fastdfs-5.05/conf
cp http.conf mime.types /etc/fdfs
```

![img](https://pic2.zhimg.com/80/v2-1aff32d8f7124ce04690353dde2f8841_hd.png)



配置nginx.conf文件，命令：

```text
vim /usr/local/nginx/conf/nginx.conf
```

在配置文件中加入如下内容：

```text
location /group1/M00 {
    	root /data/fastdfs/storage/;
    	ngx_fastdfs_module;
}
```

单点部署配置完成，分布式还没做