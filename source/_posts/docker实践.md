---
title: docker 实践
date: 2018-05-25 15:17:16
tags: docker
---

公司的项目都切换成了docker，不同的项目进行了隔离，维护起来很方便。抽空记录一下常用的命令和自己捣鼓的东西。

镜像相当于装系统时的iso文件，容器相当于系统，一个iso文件系统盘可以装无数个系统，同理一个镜像可以运行无数个容器
### 查看本地镜像 & 删除镜像

``` bash
docker images
docker rmi fe4a64cc837f
```

### 构建镜像
构建镜像 -t镜像名，-f 构建镜像脚本路径
``` bash
docker build -t php7 .  // .为当前目录，docker会自动寻找当前目录下的dockerfile文件 如果使用其他路径加-f 跟其他路径
```
<!-- more -->
### 远程拉取镜像
远程镜像 Jessie stretch wheezy 都是 Debian 发行版本的代称
debian 系安装vim
更新: apt-get update
安装vim: apt-get install vim 
alpine 为最小镜像
如何在 Alpine Linux 中安装 bash？
``` bash
# apk update
# apk upgrade
# apk add bash
```
安装 bash 文档，请输入：
``` bash
# apk add bash-doc

```
安装 bash 自动命令补全，请运行
``` bash
#  apk add bash-completion

```
[apline bash 参考链接] https://www.oschina.net/translate/alpine-linux-install-bash-using-apk-command


``` bash
docker pull golang:1.11.5-stretch //从golang仓库拉取标签为1.11.5-stretch的镜像
```

### 运行容器 & 停止容器
``` bash
docker start
docekr stop 
```

### 查看容器 & 查看所有容器 

``` bash
docker ps
docekr ps -a 
```

### 进入容器

``` bash
docker exec -it d07e9dfbe8ad bash
```
### 删除容器

``` bash
docker rm d07e9dfbe8ad 
```

### 查看所有容器的ip地址

``` bash
docker inspect -f '{{.Name}} - {{.NetworkSettings.IPAddress }}' $(docker ps -aq)
```

### 查看容器挂载的目录

``` bash
docker inspect d07e9dfbe8ad | grep Mounts -A 20
```
### 配置容器间通讯
同一台机器上容器之间可以通过 --link 进行连接通讯
不通机器的docker之间可以通过安装docker集群工具进行通讯，原理与局域网的路由器相当

``` bash
docker run -d --name mysql -p 3306:3306 -v /opt/var/log/mysql:/opt/var/log/mysql -e MYSQL_ROOT_PASSWORD=thiasdfsismfas -it mysql:5.6
docker run -d --name xiaomo -p 9000:9000 -v /opt/var/log/xiaomo:/opt/var/log/xiaomo --link mysql:mysql -it 4f31fe61b1b0
docker run --name nginx -p 80:80 -v /var/www/html:/usr/local/nginx/html --link php7:php7 -it nginx
```
另需配置php容器 里面的/etc/php-fpm.d/www.conf
1 listen=127.0.0.1:9000 改为：9000
2  注释掉listen.allowed_clients = 127.0.0.1一行

Nginx如何支持PHP脚本？
Nginx容器启动时候，通过--link php7:php7参数共享PHP容器的网络，配置nginx.conf文件（见nginx/Dockerfile），当处理PHP脚本时，转给PHP容器解析：

### docker 搭建 wordpress
注意要使用5.6 版本的数据库 否则安装WordPress 会报错
``` bash
docker run -d --name wushixuan -e WORDPRESS_DB_HOST=mysql -e WORDPRESS_DB_USER=root -e WORDPRESS_DB_PASSWORD=thisismysql2018, -p9523:80 --link mysql:mysql -it wordpress
docker run -d --name wushixuan-wordpress   -p9961:80 --link mysql5.6:mysql -it wordpress
```



### docker 搭建 lnmp
启动容器：
整个流程可以看到，Nginx、PHP、MySQL三者的关系：
Nginx容器---->PHP容器，PHP容器---->MySQL容器。即容器之间是有关联的，两两容器的数据通信通过容器启动命令docker run加参数--link解决

```bash

docker run --name mysql -p 3306:3306 -v /root/bo/data/mysql:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=123456 -it addcn/mysql
docker run --name php7 -p 9000:9000 -v /var/www/html:/usr/local/nginx/html --link mysql:mysql -it addcn/php7
docker run --name nginx -p 80:80 -v /var/www/html:/usr/local/nginx/html --link php7:php7 -it addcn/nginx

```
Nginx如何支持PHP脚本？
Nginx容器启动时候，通过--link php7:php7参数共享PHP容器的网络，配置nginx.conf文件（见nginx/Dockerfile），当处理PHP脚本时，转给PHP容器解析

```bash
location ~ \\.php$ {
    root           html;
    fastcgi_pass   php7:9000;  #此处为关键！！其中php7为PHP容器的名称，见启动PHP容器docker run --name指定的值
    fastcgi_index  index.php;
    fastcgi_param  SCRIPT_FILENAME  /usr/local/nginx/html$fastcgi_script_name; #关键！！/usr/local/nginx/html为web目录
    include        fastcgi_params;
}
```

PHP如何读取MySQL数据？
PHP容器启动时候，通过--link mysql:mysql参数，与MySQL容器共享网络，类似两者处于同一台机器，因此PHP代码连接的时候使用$conn = new PDO('mysql:host=mysql;port=3306;dbname=mysql;charset=utf8', 'root', '123456');就可以连接上MySQL（其中host=mysql的mysql为MySQL容器的名称，见启动MySQL容器docker run --name指定的值）。
```php
<?php
//date
echo date("Y-m-d H:i:s")."<br />\\n";

//mysql
try {
    $conn = new PDO('mysql:host=mysql;port=3306;dbname=mysql;charset=utf8', 'root', '123456');
} catch (PDOException $e) {
    echo 'Connection failed: ' . $e->getMessage();
}
//$conn->exec('set names utf8');
$sql = "SELECT * FROM `user` WHERE 1";
$result = $conn->query($sql);
while($rows = $result->fetch(PDO::FETCH_ASSOC)) {
    echo $rows['Host'] . ' ' . $rows['User']."<br />\\n";
}

//phpinfo
phpinfo();
?>
```
