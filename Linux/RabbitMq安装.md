# [Linux] RabbitMq 的安装

## 一, 更新yum源(可选择)
>1. `yum update -y`
>2. `reboot`(更新)

## 二, 安装相关的依赖
`yum -y install gcc glibc-devel make ncurses-devel openssl-devel xmlto perl wget`

## 三, 安装 erlang 语言环境
>1. `wget http://www.erlang.org/download/otp_src_20.3.tar.gz`(下载erlang安装包)
>2. `tar -xzvf otp_src_20.3.tar.gz`(解压)
>3. `cd otp_src_20.3`
>4. `./configure --prefix=你打算安装的路径`(设置erlang安装的路径，我的是安装在/opt/erlang-20.3)
>5. `make && make install` (编译安装)

## 四, 配置 erlang 的环境变量
>1. `vim /etc/profile` (打开profile文件，此处的安装在全局的环境变量里面的，既无论哪个用户登录了，都能操作erlang，如果只想把erlang安装在当前用户的话，请使用`vim ~/.bash_profile`)
>2. `i` (进入编辑/etc/profile文件模式,右下角会显示 `insert`)
>3. 在文件的最底部添加如下内容
```shell
# erlang
export ERL_HOME=/opt/erlang-20.3

PATH=.:$PATH:$ERL_HOME/bin
export PATH=.:$PATH:$ERL_HOME/bin
```
>4. `esc`(退出编辑模式,右下角的`insert`标志消失)
>5. `shift+;`(右下角会出现`:`标志)
>6. `wq`(输入w,q,对编辑的内容进行保存，如果不想要保存输入q,!)
>7. `enter`
>8. `source /etc/profile ` (使配置生效)
>9. `erl`(验证erlang是否按装成功,如何进入了erlang的shell说明安装成功了)
>10. `halt().` (退出erlang的shell)

# 五, 安装 RabbitMq
因为通过`wget`貌似无法下载最新版的rabbitMq,所以选择通过上传的方法。
>1. 到rabbitMq的github地址`https://github.com/rabbitmq/rabbitmq-server/releases/tag/v3.7.4`下载这个`rabbitmq-server-generic-unix-3.7.4.tar.xz`,然后上传到服务器
>2. `xz -d rabbitmq-server-generic-unix-3.7.4.tar.xz` (解压.xz文件)
>3. `tar -xf rabbitmq-server-generic-unix-3.7.4.tar`  (解压.tar文件)
>4. `mv rabbitmq_server-3.7.4 rabbitmq-3.7.4` (可选操作，因为解压出来的文件名有个server,感觉有点不好看，重命名了)

# 六, 配置 RabbitMq 的环境变量
>1. `vim /etc/profile`
>2. 在文件的最底部添加如下内容
```shell
# rabbitmq
export RABBITMQ_SERVER=/opt/rabbitmq-3.7.4

# 此处是在原来的Path路径后面追加
PATH=一开始的内容:$RABBITMQ_SERVER/sbin
```
>3. `source /etc/profile `
>4. `rabbitmq-server -detached` （后台启动服务）
>5. `rabbitmqctl status` (查看rabbitMq的状态，如果显示出的日志里面有mq的运行进程号，mq的版本号等信息说明安装成功了。)
>6. `rabbitmqctl stop` (关闭服务)

# 七, 配置 RabbitMq 的网页插件
>1. `mkdir /etc/rabbitmq` (在/etc下创建一个rabbitmq的文件夹)
>2. `rabbitmq-plugins enable rabbitmq_management` (启用插件)

# 八,配置防火墙(如果没有设置防火墙的话，可以过)
>1. `firewall-cmd --permanent --add-port=15672/tcp` (开启15672端口，15672为网页插件端口)
>2. `firewall-cmd --permanent --add-port=5672/tcp`  (开启5672端口，5672为mq提供服务的端口)
>3. `firewall-cmd --reload` (重启防火墙)
>4. 现在可以通过`IP:15672`访问到rabbitMq的管理界面(注：进行访问时，记得先把服务启动了`rabbitmq-server -detached
>5. 虽然rabbitMq已经为我们初始了一个超级管理员的账号`用户名:guest, 密码:guest`,但是也对这个用户做了安全设置，只能在本机使用，既只能在用`localhost`的情况下使用，所以需要我们只能新建一个超级管理员
>6. `rabbitmqctl add_user 用户名 密码` (新建一个用户)
>7. `rabbitmqctl set_permissions -p / 用户名 ".*" ".*" ".*" ` (分配权限)
>8. `rabbitmqctl set_user_tags 用户名 administrator` (修改角色)
>9. 现在可以登录了，

**至此 rabbitMq 的安装完成**