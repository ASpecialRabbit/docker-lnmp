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
