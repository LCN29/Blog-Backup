# [Linux] Nginx 的安装

## 环境
1. Centos 7.6 64位
2. 安装的路径 /opt/software(可以根据个人的习惯，进行选择)
3. 下文出现的软件的版本，根据个人的情况进行选择，因为个人只是当做学习用，所以直接选了最新的版本


## 开始
1. 下载 nginx 的源码包
```shell
wget https://nginx.org/download/nginx-1.17.6.tar.gz
```

2. nginx 源码包的编译需要 GCC 编译器，此处直接通过 yum 安装 G++ 编译器
```shell
yum install -y gcc-c++
```

3. 下载 Pcre 库(可选)  
Pcre（Perl Compatible Regular Expressions，Perl兼容正则表达式），如果我们后续的配置文件 `nginx.conf` 里需要使用到正则表达式，就需要安装这个模块。(个人建议还是安装了，如果百分比确定后续不用到正则，可以不装)
```shell
wget https://ftp.pcre.org/pub/pcre/pcre-8.43.tar.gz
```

4. 下载 zlib 库(可选)    
zlib库用于对 http 包的内容做gzip格式的压缩，如果我们在 `nginx.conf` 里配置了 gzip on，并指定对于某些类型（content-type）的HTTP响应使用gzip来进行压缩以减少网络传输量，那么，在编译时就必须把 zlib 编译进 Nginx
```shell
wget http://www.zlib.net/zlib-1.2.11.tar.gz
```

5. 下载 openSSL(可选)  
需要使用到 https 时，需要安装。 注意，阿里云服务已经装好了 openSSL了，但是这个版本有个 `心脏滴血` 漏洞(具体是否为当前的版本，我也不清楚)，所以个人重新安装了新版本的 openSSL。
```shell
wget http://www.openssl.org/source/openssl-1.1.1d.tar.gz
```

6. 安装 zlib  
```shell
# 解压
tar -zxf zlib-1.2.11.tar.gz

# 进入对应的目录
cd zlib-1.2.11.tar.gz

# 配置，讲解后面说
./configure --shared

# 编译 安装
make && make install
```

这里主要说一下 `./configure --shared`   
1. 通过源码包进行安装时，如果 `./configure` 后面没有加上 `--prefix`，默认情况会将库安装在 `/usr/local/lib`，所以如果你需要把库安装在另一个位置，可以通过 `--prefix=你想安装的位置`  
2. `--shared` 该选项指定生成动态连接库（让连接器生成T类型的导出符号表，有时候也生成弱连接W类型的导出符号），不用该标志外部程序无法连接。相当于一个可执行文件
3. 动态连接库

Linux 系统上有两类根本不同的 Linux 可执行程序：静态链接的可执行程序和动态链接的可执行程序。 2者的区别就像是 java 中的 jar 包，静态链接可执行程序就是把这个 jar 依赖的其他 jar 到打包在了一起。 而 动态连接的可执行程序就像是 jar 只打包了自身的代码，第三方的依赖到不引入。 可以通过 `ldd 查看的执行程序` 判断某个程序是否为动态链接程序。

动态连接程序的依赖则是通过动态装入器(dynamic loader) 实现的。 动态装入器负责装入动态链接的可执行程序运行所需的共享库。   
动态装入器则是通过  /etc/ld.so.conf 和 /etc/ld.so.cache实现的。 可以简单的看出 cache 是 conf 的缓存，conf 配置了所有的 jar。 我们的动态联军程序需要的 jar，可以通过 cache 找到。
conf 默认只会到 /usr/lib，/lib 查找程序需要的jar。

但是我们的 zlib 安装在 /usr/local/lib，是找不到的。 所以需要把 /usr/local 放到 /etc/ld.so/conf 里面
```shell

# 查看 /etc/ld.so.conf 如果里面有  /usr/local/lib 则忽略下面的操作
vim /etc/ld.so.conf

# 添加我们的目录
echo "/usr/local/lib" >> /etc/ld.so.conf

# 刷新缓存
ldconfig
```

## 7. 安装 Pcre
```shell
cd /opt/software

tar -zxvf pcre-8.43.tar.gz 

cd pcre-8.43

./configure --shared /usr/local

make && make install
```

## 8. 安装 openSSL
```shell
cd /opt/software

tar -zxvf openssl-1.1.1d.tar.gz

cd openssl-1.1.1d

./config shared zlib

make && make install


# 替换旧版 OpenSSL  阿里云默认已经有一个安装了
mv /usr/bin/openssl /usr/bin/openssl.old

# 建立新的软连接
ln -s /usr/local/bin/openssl /usr/bin/openssl

ln -s /usr/local/include/openssl/ /usr/include/openssl

# 新版本需要2个依赖，会到 /usr/lib64 寻找，建立一个新的软件连接，连过去就行了
ln -s /usr/local/lib64/libssl.so.1.1 /usr/lib64/libssl.so.1.1
ln -s /usr/local/lib64/libcrypto.so.1.1 /usr/lib64/libcrypto.so.1.1

# 查看当前的版本
openssl version -a 
```

## 9. 开始安装 nginx
```shell
cd /opt/software

# 用来存放解压后的文件
mkdir nginx

3 用来存放 nginx 安装路径
mkdir nginx-1.17.6

# 新建一个 nginx 的组
groupadd nginx

# 新建一个 nginx 的用户
useradd -r -g nginx nginx

# 进入 nginx的安装目录
cd nginx/nginx-1.17.6

# 修改当前目录所属的用户和组
chown -R nginx .
chgrp -R nginx .

# 进入到 nginx 的 源码包
cd /opt/software/nginx/nginx-1.17.6

# 进入 man 目录
cd /man

# 拷贝当前目录的的 nginx.8 到 man8 目录
# cp ./nginx.8 /usr/share/man/man8

# 压缩 nginx.8 把 nginx 的手册放到 man里面  后续可以通过 man nginx 进行查询
gzip /usr/share/man/man8/nginx.8


# 配置 注意里面的 --prefix 指定了我们新建的 nginx-1.17.6目录，而不是当前目录
./configure --user=nginx --group=nginx --prefix=/opt/software/nginx-1.17.6 --with-http_stub_status_module --with-http_ssl_module  --with-poll_module --with-pcre  --with-http_gzip_static_module --with-debug

make && make install

# 进入我们的安装目录
cd /opt/software/nginx-1.17.6

# 进入到执行目录
cd sbin

# 打印出了版本就是安装成功了
./nginx -v
```
1. --user 和 --group 指定了Nginx Worker 线程的所属的组合和用户，2. --prefix 指定了前缀，简单的理解就是按照目录
3. --with-http_stub_status_module 开启性能查看页面，获取当前 nginx 的情况
4. --with-http_ssl_module 开启 ssl 模块，因为我们的 依赖包都装在默认环境，所以可以不用配置 --with-openssl=openssl的安装路径
5. --with-poll_module 使用 poll moudle 处理事件驱动
6. --with-pcre 使用 pcre 库，如果 pcre 库没有安装在默认路径，使用 --with-pcre=pcre安装的路径说明
7. --with-http_gzip_static_module gzip模块可以在开启了 gzip 模式时，对文件压缩前先查找缓存，没有在压缩
8. --with-debug 以 debug 模式运行，可以多一下日志，了解 nginx 的性能等 


# 10. Nginx 加入系统服务

1. 在 /etc/systemd/system 下创建文件 nginx.server (在 Centos 7 推荐使用 systemctl 配置应用，老式的 chkconfig 不推荐了)
```shell
vim /etc/systemd/system/nginx.service
```

2. 然后把下面的配置复制到 nginx.service 里面，把里面的 nginx 的配置修改为自己的
```shell
[Unit]
Description=nginx - high performance web server
Documentation=https://nginx.org/en/docs/
After=network-online.target remote-fs.target nss-lookup.target
Wans=network-online.target

[Service]
Type=forking
ExecStartPre=/opt/software/nginx-1.17.6/sbin/nginx -t -c /opt/software/nginx-1.17.6/conf/nginx.conf
ExecStart=/opt/software/nginx-1.17.6/sbin/nginx -c /opt/software/nginx-1.17.6/conf/nginx.conf
ExecReload=/opt/software/nginx-1.17.6/sbin/nginx -s reload
ExecStop=/opt/software/nginx-1.17.6/sbin/nginx -s quit
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```
3. 查看我们的配置
```shell
systemctl list-unit-files | grep nginx
```

4. 将 Nginx 服务加到自启
```shell
systemctl enable nginx.service
```

5. 启动服务
```shell
systemctl start nginx.service
```

## 11. 最后
最后说一句：整个 /opt/software 里面的压缩包，源码包，安装这种方式安装的话，出了 nginx 的安装目录 (/opt/software/nginx-1.17.6) 外，全部都可以删掉了。
