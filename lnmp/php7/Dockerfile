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

# 配置php监听所有ip的9000端口请求，而不只是本机的，因为nginx和php不在一个容器
RUN sed -i -e 's/listen = 127.0.0.1:9000/listen = 0.0.0.0:9000/' /usr/local/php/etc/php-fpm.d/www.conf

# 镜像启动的时候启动php
ENTRYPOINT ["/usr/local/php/sbin/php-fpm", "-F", "-c", "/usr/local/php/lib/php.ini"]
