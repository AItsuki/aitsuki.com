---
title: Lnmp WordPress 搭建个人博客
titleurl: build-personal-blog-by-lemp-wordpress
date: 2020-12-19 12:35:00
mathjax: false
---

在开始前，可以先准备好用于搭建个人博客的服务器和域名，或者使用虚拟机练习搭建。服务器选择1cpu 1g内存 1M带宽的最低配置即可，后续若有需要可再升级。购买服务器时请查看平台当前正在进行的活动，通常能以非常低廉的价格购买。系统镜像这里推荐ubuntu，安装软件相对比较简单，centos则需要一定的门槛。另外，选择国内的服务器搭建网站是需要备案的，如不过不打算备案可选择香港或海外的服务器。

### MySql

```Bash
sudo apt install mysql-server
sudo mysql
# 创建wordpress数据库、账户、并授权。
mysql> create database wordpress;
mysql> create user username identified by 'password';
mysql> grant all privileges on wordpress.* to username;
mysql> flush privileges;
mysql> exit
```

将上面的username和password替换成你想要的用户名和密码，记住这个数据库和账户，之后需要写入到wordpress配置中。

另外mysql-5.6以上版本默认启用一项叫做`performance-schema`的功能，性能提升有限但是会额外占用400M左右的内存，如果你的服务器内存小于1G可以禁用此功能。

查看是否开启了`performance_schema`：

```bash
sudo mysql
mysql> show variables like 'performance_schema';
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| performance_schema | ON    |
+--------------------+-------+
1 row in set (0.00 sec)

mysql> exit
```

可以通过配置文件启用or关闭此功能：

```bash
sudo vim /etc/mysql/my.cnf
```

```
# 在配置文件中新增以下两行后保存（ON or OFF）
[mysqld]
performance_schema=OFF
```

```bash
# 重启mysql
sudo service mysql restart
```

### Nginx

```Bash
sudo apt install nginx
```

此时用浏览器你的网站就可以看到Nginx的默认首页了。

配置Gzip：https://kinsta.com/blog/enable-gzip-compression/

```Bash
sudo vim /etc/nginx/nginx.conf

gzip on;
gzip_vary on;
gzip_proxied any;
gzip_comp_level 6;
gzip_min_length 256;
gzip_buffers 16 8k;
gzip_disable "msie6";
gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

# 或者使用 sudo service nginx reload
sudo nginx -s reload
```
### PHP

```Bash
sudo apt install php-fpm
# 安装扩展，这些扩展都是wordpress必备或推荐的。
sudo apt install php-mysql php-gd php-curl php-dom php-mbstring php-imagick php-zip php-gmp php-bcmath
```

```Bash
# 7.4是目前最新版本，请替换成你安装的版本，尽量使用7.2以上的。
sudo vim /etc/php/7.4/fpm/php.ini
```

修改php配置：

```
# Nginx + PHP CGI的一个可能的安全漏洞，https://www.laruence.com/2010/05/20/1495.html
cgi.fix_pathinfo = 0
# 配置文件上传大小限制(上传媒体库，插件时)
upload_max_filesize = 128M
post_max_size = 128M
```

```Bash
# 重启fpm让配置生效
sudo service php7.4-fpm restart
```

### phpMyAdmin

phpMyAdamin是数据库管理工具，某些情况下非常有用。

```Bash
# 下载phpMyAdamin(到官网获取最新版本下载链接)
wget https://files.phpmyadmin.net/phpMyAdmin/4.9.7/phpMyAdmin-4.9.7-all-languages.tar.gz
# 解压
tar -xf phpMyAdmin-4.9.7-all-languages.tar.gz
# 移动到html目录
sudo mv phpMyAdmin-4.9.7-all-languages /var/www/html/phpMyAdmin
# 修改目录归属
sudo chown -R www-data:www-data /var/www/html/phpMyAdmin/
```

配置phpMyAdmin

```Bash
cd /var/www/html/phpMyAdmin
sudo cp config.sample.inc.php config.inc.php
sudo vim config.inc.php
```

``` PHP
// 在这里输入32位以内的密码，用于cookie加密，增加安全性。不用记住，尽量设置长点。
$cfg['blowfish_secret'] = '请随意输入32位以内密码';
```


配置phpMyAdmin的Nginx服务

```Bash
# 进入sites-available目录
cd /etc/nginx/sites-available/
# 开始为phpMyAdmin配置Nginx服务
sudo vim phpMyAdmin
```

```Nginx
server {
        # 输入你希望的端口
        listen 8100;
        listen [::]:8100;

        root /var/www/html/phpMyAdmin;

        index index.php;

        server_name _;

        client_max_body_size 128m;
        client_body_buffer_size 128m;

        location / {
                try_files $uri $uri/ =404;
        }

        location ~ \.php$ {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
        }
}
```

让配置生效

```Bash
# sites-available 目录保存我们可用的服务配置，而sites-enabled目录则相当于一个开关。
# 将我们想开启的服务软链接到sites-enabled目录即可
sudo ln -s /etc/nginx/sites-available/phpMyAdmin /etc/nginx/sites-enabled/phpMyAdmin
# 检查配置（每次修改配置后都推荐此命令检查）
sudo nginx -t
sudo nginx -s reload
```

现在登陆网站的8100端口（记得在服务器的安全组中放开对应端口）即可看到phpMyadmin的登陆界面，登陆时的账号密码就是前面MySql命令新建的。

### phpMyAdmin 高级功能（可选）

另外phpMyAdmin首页底部可能会提示“phpMyAdmin 高级功能尚未完全设置，部分功能未激活”，不知道所谓的高级功能究竟是什么，如果感兴趣的话可以尝试开启：

```Bash
sudo mysql
# 执行sql：创建了一个phpmyadmin数据库并创建若干张表
mysql> source /var/www/html/phpMyAdmin/sql/create_tabls.sql;
# 创建phpmyadmin数据库的管理账户，将pma和pmapass替换成你需要的
mysql> create user pma identified by 'pmapass';
mysql> grant all privileges on phpmyadmin.* to pma;
mysql> flush privileges;
mysql> exit
```

配置高级功能：

```BASH
sudo vim /var/www/html/phpMyAdmin/config.inc.php
```

```PHP
/* User used to manipulate with storage */
// $cfg['Servers'][$i]['controlhost'] = '';
// $cfg['Servers'][$i]['controlport'] = '';
# 这里输入刚刚创建的用户名和密码
$cfg['Servers'][$i]['controluser'] = 'pma';
$cfg['Servers'][$i]['controlpass'] = 'pmapass';

/* Storage database and tables */
# 放开下面注释
$cfg['Servers'][$i]['pmadb'] = 'phpmyadmin';
$cfg['Servers'][$i]['bookmarktable'] = 'pma__bookmark';
$cfg['Servers'][$i]['relation'] = 'pma__relation';
$cfg['Servers'][$i]['table_info'] = 'pma__table_info';
$cfg['Servers'][$i]['table_coords'] = 'pma__table_coords';
$cfg['Servers'][$i]['pdf_pages'] = 'pma__pdf_pages';
$cfg['Servers'][$i]['column_info'] = 'pma__column_info';
$cfg['Servers'][$i]['history'] = 'pma__history';
$cfg['Servers'][$i]['table_uiprefs'] = 'pma__table_uiprefs';
$cfg['Servers'][$i]['tracking'] = 'pma__tracking';
$cfg['Servers'][$i]['userconfig'] = 'pma__userconfig';
$cfg['Servers'][$i]['recent'] = 'pma__recent';
$cfg['Servers'][$i]['favorite'] = 'pma__favorite';
$cfg['Servers'][$i]['users'] = 'pma__users';
$cfg['Servers'][$i]['usergroups'] = 'pma__usergroups';
$cfg['Servers'][$i]['navigationhiding'] = 'pma__navigationhiding';
$cfg['Servers'][$i]['savedsearches'] = 'pma__savedsearches';
$cfg['Servers'][$i]['central_columns'] = 'pma__central_columns';
$cfg['Servers'][$i]['designer_settings'] = 'pma__designer_settings';
$cfg['Servers'][$i]['export_templates'] = 'pma__export_templates';
```

重新登陆phpmyadmin即可。

### Wordpress

```Bash
# 下载wordpress中文版
wget https://cn.wordpress.org/latest-zh_CN.tar.gz
# 解压
tar -xf latest-zh_CN.tar.gz
# 移动到html目录
sudo mv wordpress /var/www/html/
# 更改wordpress的owner和group，与nginx相同（www-data）
# 这是为了避免日后出现各种各样的权限问题
sudo chown -R www-data:www-data /var/www/html/wordpress
```

配置wordpress的Nginx服务

```Bash
cd /etc/nginx/sites-available/
sudo vim wordpress
```

```Nginx
# https://www.digitalocean.com/community/tutorials/how-to-implement-browser-caching-with-nginx-s-header-module-on-ubuntu-16-04
# Expires map
map $sent_http_content_type $expires {
        default off;
        text/html epoch;
        text/css 30d;
        application/javascript 30d;
        ~image/ max;
}

server {
        listen 80 default_server;
        listen [::]:80 default_server;

        # 增加上传文件限制大小
        client_max_body_size 128m;
        client_body_buffer_size 128m;

        # 缓存头
        expires $expires;

        root /var/www/html/wordpress;

        index index.php;

        # 输入你的域名
        server_name yourdomain.com www.yourdomain.com;

        location / {
                # 伪静态
                try_files $uri $uri/ /index.php?$args;
        }

        location ~ \.php$ {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
        }
}
```

```Bash
sudo ln -s /etc/nginx/sites-available/wordpress /etc/nginx/sites-enabled/wordpress
# 移除默认default配置（就是安装nginx后看到的默认首页）
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo nginx -s reload
```

现在只需要登录你的网站，即可根据wordpress向导完成后续操作。

### 配置SSL证书（可选）

各大平台都有提供免费证书，如果没有可以到 https://freessl.cn/ 这个网站申请，最终你能获得名为xxx.crt, xxx.key 的证书文件和密钥。

将证书和密钥上传到服务器，并放置在`/etc/nginx/ssl`目录，目录本身不存在需要创建。也可以放到`/etc/ssl`目录，或者其他目录，没有规定。

配置ssl

```Bash
sudo vim /etc/nginx/sites-available/wordpress
```

添加以下规则：（原来的80端口改成443端口，然后重新声明一个80端口的服务重定向到443端口的服务）

```Nginx
server {
        listen 80 default_server;
        listen [::]:80 default_server;
        server_name yourdomain.com www.yourdomain.com;
        return 301 https://$server_name$request_uri;
}

server {
        listen 443 ssl http2;
        listen [::]:443 http2;

        ssl_certificate /etc/nginx/ssl/xxx.crt;
        ssl_certificate_key /etc/nginx/ssl/xxx.key;

        add_header Strict-Transport-Security "max-age=31536000";
        ssl_session_cache shared:SSL:20m;
        ssl_session_timeout 10m;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_prefer_server_ciphers on;
        ssl_ciphers 'ECDH+AESGCM:ECDH+AES256:ECDH+AES128:DH+3DES:!ADH:!AECDH:!MD5';

        ...
}
```

```Bash
sudo nginx -t
sudo nginx -s reload
```

现在证书已部署，访问主页可能出现问题，需要到wordpress后台的`设置-常规` 配置wordpress地址和站点地址的前缀为https。

### 文件权限的注意事项

wordpress是部署Nginx上的，用户组是www-data。所以前面教程中就有了更改wordpress目录用户组的操作 `sudo chown -R www-data:www-data wordpress`，否则wordpress后台会出现很多权限导致的问题。

另外有的做法是将wordpress的读写权限放开给所有用户`chmod 777`，本质上和修改用户组没什么区别，但是前者更安全。

### 推荐插件

- WP Mail SMTP by WPForms，邮件服务。
- WP Statistics，统计插件，可配合Sakura主题使用。
- UpdraftPlus，备份。推荐只备份数据库和上传目录。
- WP-Optimize， 推荐只使用压缩和缓存功能，禁用图片压缩（主要是不起作用）和数据库清理。
- WP Githuber MD，markdown编辑器。
- Wordfence Security，安全插件。
- Akismet Anti-Spam，垃圾评论过滤。
- AMP，生成AMP页面。
- Rank Math SEO，搜索引擎优化。