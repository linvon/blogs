---
title: "Nginx+PHP+yaml\u6269\u5C55\u914D\u7F6E"
date: "2019-08-06"
toc: false
tags: ~
categories:
- "\u96F6\u6563\u77E5\u8BC6"
...
--- ``` bash
apt-get install nginx
apt-get install php
```

安装完成后,修改Nginx的配置文件，把默认支持的PHP打开，新的Nginx配置都做到分离了，现在应该是修改/etc/nginx/sites-available/default 

``` tex
# Add index.php to the list if you are using PHP 如注释所言，如果使用php，就增加一个index.php
index index.php index.html index.htm index.nginx-debian.html;



# pass PHP scripts to FastCGI server
# 这里取消部分注释，php-fpm默认使用sock，将sock的放开就可以了
location ~ \.php$ {
include snippets/fastcgi-php.conf;
#
#       # With php-fpm (or other unix sockets):
fastcgi_pass unix:/var/run/php/php7.2-fpm.sock;
#       # With php-cgi (or other tcp sockets):
#       fastcgi_pass 127.0.0.1:9000;
}
```

修改后重启服务就可以解析PHP了，接下来为PHP添加yaml扩展

``` bash
apt install php7.2-dev
apt-get install libyaml-dev
pecl install yaml
```

安装完成之后修改自己的php.ini，如果是用fpm解析，就直接修改对应fpm目录下的就可以了：/etc/php/7.2/fpm/php.ini

找到extension=xxx的部分，添加一行取消注释的extension=yaml.so，至此重启php-fpm服务，便可以解析yaml了
