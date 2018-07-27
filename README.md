[代码示例](https://github.com/ASpecialRabbit/docker-lnmp)  

我理解的PHP项目docker化有两种。  
1、一种是把 PHP代码、nginx、PHP环境、mysql 当作一个整体打包成镜像。  
2、另一种是，nginx打包成一个镜像，PHP打包成一个镜像，mysql再打包成一个镜像，项目代码不打包进docker，通过link实现nginx与PHP、PHP与mysql的通信，如果需要切换PHP版本，只需要再打包一个别的版本PHP镜像即可。  

如果是第一种，把所有的东西都装到一个镜像里，感觉跟虚拟机没什么区别，官方也不推荐这种。  
下面讲一下第二种。  

构建docker镜像最主要就是写好Dockerfile，附一个[Dockerfile命令详解](https://www.cnblogs.com/dazhoushuoceshi/p/7066041.html)。  

# 定制镜像
## MySQL镜像Dockerfile文件编写
先从mysql开始写，最简单。新建一个mysql文件夹，新建Dockerfile文件，内容如下：
```
# 指定基础镜像，必须是第一条指令。
FROM mysql:5.7

# 作者
MAINTAINER ASpecialRabbit

# 设置环境变量
ENV TZ "Asia/Shanghai"
```
意思就是继承一下官方的mysql5.7镜像。  

## PHP镜像Dockerfile文件编写
再写php的，同样新建一个文件夹，新建Dockerfile文件，内容如下：
```
# 指定基础镜像，必须是第一条指令。
FROM centos:centos7

# 作者
MAINTAINER ASpecialRabbit

# 设置环境变量
ENV TZ "Asia/Shanghai"

# 在镜像里创建这个目录
RUN mkdir -p /usr/local/nginx/html

# 在镜像里用Yum安装一些组件
RUN yum -y update && \
    yum install -y gcc automake autoconf libtool make gcc-c++ vixie-cron  wget zlib  file openssl-devel sharutils zip  bash vim cyrus-sasl-devel libmemcached libmemcached-devel libyaml libyaml-devel unzip libvpx-devel openssl-devel ImageMagick-devel  autoconf  tar gcc libxml2-devel gd-devel libmcrypt-devel libmcrypt mcrypt mhash libmcrypt libmcrypt-devel libxml2 libxml2-devel bzip2 bzip2-devel curl curl-devel libjpeg libjpeg-devel libpng libpng-devel freetype-devel bison libtool-ltdl-devel net-tools && \
    yum clean all

# 在镜像里更换一下rpm源并且安装一个组件
RUN rpm -ivh http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm && \
    yum install -y libmcrypt-devel

# 在镜像里源码编译安装PHP7.2.7
RUN cd /tmp && \
  wget http://cn2.php.net/distributions/php-7.2.7.tar.gz && \
  tar xzf php-7.2.7.tar.gz && \
  cd /tmp/php-7.2.7 && \
  ./configure \
    --prefix=/usr/local/php \
    --with-mysqli --with-pdo-mysql --with-iconv-dir --with-freetype-dir --with-jpeg-dir --with-png-dir --with-zlib --with-libxml-dir --enable-simplexml --enable-xml --disable-rpath --enable-bcmath --enable-soap --enable-zip --with-curl --enable-fpm --with-fpm-user=nobody --with-fpm-group=nobody --enable-mbstring --enable-sockets --with-mcrypt --with-gd --enable-gd-native-ttf --with-openssl --with-mhash --enable-opcache --disable-fileinfo && \
    make && \
    make install

RUN cp /tmp/php-7.2.7/php.ini-production /usr/local/php/lib/php.ini && \
    cp /usr/local/php/etc/php-fpm.conf.default /usr/local/php/etc/php-fpm.conf && \
    cp /usr/local/php/etc/php-fpm.d/www.conf.default /usr/local/php/etc/php-fpm.d/www.conf

# 对外暴露9000端口
EXPOSE 9000

# 修改php-fpm配置文件，让php监听所有ip的9000端口请求，而不只是本机的，因为nginx和php不在一个容器
RUN sed -i -e 's/listen = 127.0.0.1:9000/listen = 9000/' /usr/local/php/etc/php-fpm.d/www.conf

# 镜像启动的时候启动php
ENTRYPOINT ["/usr/local/php/sbin/php-fpm", "-F", "-c", "/usr/local/php/lib/php.ini"]

```

## Nginx镜像Dockerfile文件编写
新建一个文件夹，新建Dockerfile文件，内容如下：
```
# 指定基础镜像，必须是第一条指令。
FROM centos:centos7

# 作者
MAINTAINER ASpecialRabbit

# 设置环境变量
ENV TZ "Asia/Shanghai"

# 在镜像里用Yum安装一些组件
RUN yum -y update && \
    yum install -y gcc automake autoconf libtool make gcc-c++ vixie-cron  wget zlib  file openssl-devel sharutils zip  bash vim cyrus-sasl-devel libmemcached libmemcached-devel libyaml libyaml-devel unzip libvpx-devel openssl-devel ImageMagick-devel  autoconf  tar gcc libxml2-devel gd-devel libmcrypt-devel libmcrypt mcrypt mhash libmcrypt libmcrypt-devel libxml2 libxml2-devel bzip2 bzip2-devel curl curl-devel libjpeg libjpeg-devel libpng libpng-devel freetype-devel bison libtool-ltdl-devel net-tools && \
    yum clean all

# 在镜像里源码编译安装Nginx
RUN cd /tmp && \
  wget http://nginx.org/download/nginx-1.15.0.tar.gz && \
  tar xzf nginx-1.15.0.tar.gz && \
  cd /tmp/nginx-1.15.0 && \
  ./configure \
    --prefix=/usr/local/nginx \
    --with-http_ssl_module --with-http_sub_module --with-http_dav_module --with-http_flv_module \
    --with-http_gzip_static_module --with-http_stub_status_module --with-debug && \
    make && \
    make install

# 在镜像里创建目录，删掉原有的nginx配置文件
RUN mkdir -p /usr/local/nginx/conf/vhost && \
  rm -rf /usr/local/nginx/conf/nginx.conf

# 把自己的配置文件加到镜像里
ADD nginx.conf /usr/local/nginx/conf/nginx.conf 

# 对外暴露80和443端口
EXPOSE 80 443

# 镜像启动的时候启动nginx
ENTRYPOINT ["/usr/local/nginx/sbin/nginx", "-g", "daemon off;"]
```
把它自带的nginx配置文件删了，用自己的配置文件，新建一个nginx.conf，内容如下：
```
worker_processes  1;

events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
    include vhost/*.conf;
}
```

## 生成镜像
Dockerfile写完之后，就可以build镜像了，命令如下：
### MySQL镜像
在mysql目录下运行 docker build --tag asr/mysql .  
### PHP镜像
在php目录下运行 docker build --tag asr/php7 . 
生成PHP镜像的时候可能会报错，[gcc 编译出现 internal compiler error: Killed](https://blog.csdn.net/qq_29573053/article/details/69665996)
### Nginx镜像
在nginx目录下运行 docker build --tag asr/nginx .  
注意最后面这个点，会影响到Dockerfile里你的COPY或者ADD文件时的路径怎么写。[详细解释](https://yeasy.gitbooks.io/docker_practice/content/image/build.html)

# 启动
因为nginx需要和php通信，所以启动nginx之前要启动php，同理，启动php之前要启动mysql。
## 后台启动mysql
```
docker run -d --name docker-lnmp-mysql -p 7203:3306 -v /dockerdata/mysql:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=uqh7ybwbf66t21ybbf -it asr/mysql
```
-d表示后台运行，-name指定容器名字，-p把本机7203端口映射到容器的3306端口，-v挂载本机的/dockerdata/mysql目录到容器的/var/lib/mysql目录，这样mysql的数据可以持久化到本机，-e指定环境变量。
## 后台启动php
```
docker run -d --name docker-lnmp-php7 -p 7202:9000 -v /home/dockerwww:/usr/local/nginx/html --link docker-lnmp-mysql:mysql -it asr/php7
```
link指定要连接的容器 容器名:容器别名
## 后台启动nginx
```
docker run -d --name docker-lnmp-nginx -p 7201:80 -v /home/dockerwww:/usr/local/nginx/html -v /home/dockeretc/nginx/vhost/:/usr/local/nginx/conf/vhost --link docker-lnmp-php7:php7 -it asr/nginx
```
将本机/home/dockeretc/nginx/vhost挂载到容器的/usr/local/nginx/conf/vhost，这样在本机就可以配置容器的站点

# 站点配置
在本机/home/dockeretc/nginx/vhost新建一个docker.hjply.com.conf文件，内容如下：
```
server{
    # 这里listen监听容器的80端口，但是我们访问的网址应该是docker.hjply.com:7201，因为本机7201端口映射到容器80端口
    listen 80;

    server_name wiitu.club;

    # root是容器里nginx的站点目录
    root /usr/local/nginx/html;

    index index.php index.html;

    location ~ \.php$ {
        # 这里不能再转发给127.0.0.1:9000了，而要转发给php容器的9000端口
        fastcgi_pass   php7:9000;
        fastcgi_index  index.php;
        # 这里/usr/local/nginx/html就是启动php容器的时候挂载的目录，为了直观最好跟nginx挂载的保持一致
        fastcgi_param  SCRIPT_FILENAME /usr/local/nginx/html$fastcgi_script_name;
        include        fastcgi_params;
    }
}
```

# 重启nginx
先用docker container ls看一下nginx容器的id是多少，然后docker restart 容器id

# 测试
在本机/home/dockerwww目录新建index.php文件，内容如下:
```
<?php
echo date("Y-m-d H:i:s")."<br/>";

try {
    // 这里的host=mysql，mysql是启动php容器的时候指定的docker-lnmp-mysql容器别名
    $conn = new PDO('mysql:host=mysql;port=3306;dbname=mysql;charset=utf8', 'root', 'uqh7ybwbf66t21ybbf');
} catch (PDOException $e) {
    echo $e->getMessage()."<br/>";
}
$sql = "SELECT * FROM `user`";
$result = $conn->query($sql);
while($rows = $result->fetch(PDO::FETCH_ASSOC)) {
    echo $rows['Host'] . '------' . $rows['User']."<br/>";
}

phpinfo();
```
访问www.wiitu.club:7201查看效果

# 改进
这样3个容器分别启动之后，项目就可以跑起来了。但是容器多了之后（比如再加个redis、memcache等等），一个个启动很麻烦，下面讲一下用docker-compose一键启动容器。

## docker-compose安装
[官方文档参考](https://docs.docker.com/compose/install/#install-compose)
```
sudo curl -L https://github.com/docker/compose/releases/download/1.21.2/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version
```

## docker-compose.yml文件编写,注意缩进
```
# mysql
docker-lnmp-mysql:
    # Dockerfile所在目录
    build: ./mysql
    # 本机端口:镜像对外暴露的端口
    ports:
        - "7203:3306"
    # 环境变量
    environment:
        - MYSQL_DATABASE=mysql
        - MYSQL_USER=root
        - MYSQL_ROOT_PASSWORD=uqh7ybwbf66t21ybbf
    # 挂载 本机目录：镜像里的目录
    volumes:
        - /home/dockerdata/mysql:/var/lib/mysql
    restart: always

# php7
docker-lnmp-php7:
    build: ./php7
    ports:
        - "7202:9000"
    # 连接 连接之后，在php镜像里可以和mysql镜像通信 镜像名:别名
    links:
        - docker-lnmp-mysql:mysql
    # /usr/local/nginx/html这个可以和nginx镜像的项目目录不一样，但是配置成一样的比较直观
    volumes:
        - /home/dockerwww:/usr/local/nginx/html
    restart: always

# nginx
docker-lnmp-nginx:
    build: ./nginx
    ports:
        - "7201:80"
        - "7204:443"
    links:
        - docker-lnmp-php7:php7
    # 本机PHP项目目录：nginx镜像的项目目录
    # 本机nginx的vhost目录：nginx镜像的vhost目录
    volumes:
        - /home/dockerwww:/usr/local/nginx/html
        - /home/dockeretc/nginx/vhost/:/usr/local/nginx/conf/vhost
    restart: always

```
里面的一些参数跟刚才手动启动容器的时候一样就行了。

## 定制镜像并启动容器
### 删掉刚才的镜像和容器
>- 首先查看一下在运行的容器id docker container ls
>- 然后停掉刚才的容器 docker stop 容器id或者直接停掉在运行的所有容器 docker stop $(docker ps -aq)
>- 再删掉所有已退出的容器 docker rm $(docker ps --all -q -f status=exited)
>- 再查看所有镜像 docker images
>- 删除刚才生成的镜像 docker rmi -f 镜像id
### 用docker-compose生成镜像并启动
进入docker-compose.yml所在目录，运行
```
docker-compose up --build
```
再访问www.wiitu.club:7201，结果跟刚才一样。ctrl+c停止，用
```
docker-compose start
```
后台启动所有容器，访问www.wiitu.club:7201查看效果  

[docker-compose常用命令参考](https://blog.csdn.net/skh2015java/article/details/80410306)  

# 关于容器编排
[docker-compose、docker swarm、kubernetes的区别](https://stackoverflow.com/questions/47536536/whats-the-difference-between-docker-compose-and-kubernetes#)  

docker-compose只能一键启动同一个机器上的多个容器，而后两者可以启动多个机器上的容器。
git@github.com:ASpecialRabbit/docker-lnmp.git