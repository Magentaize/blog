---
title: CentOS7 Ghost博客HTTPS部署指南
date: 2017-06-03 04:39:44
categories:
 - Note
---

#### 0x01 配置Ghost ####
>请确保 Nginx、OpenSSL、MySQL 已安装妥当

首先切换至网站目录，例如
```html
$ cd /home/wwwroot/magentaize.net
```
然后在此目录中下载Ghost博客，进入 [Ghost Developer Area](https://ghost.org/developers/) 得到最新的下载链接，在终端中执行
<!--more-->
```html
$ wget https://github.com/TryGhost/Ghost/releases/download/{version}/Ghost-{version}.zip
$ unzip Ghost-{version}.zip
$ rm Ghost-{version}.zip
```
从示例文件复制配置文件
```html
$ cp config.example.js config.js
```
由于仅作日常使用而非开发，所以只需关注 `production` 部分即可，将 `url` 修改为博客域名，注意是https
```javascript
url: 'https://blog.magentaize.net',
```
将原本的sqlite3数据库修改为mysql并配置连接信息
```javascript
database: {
    client: 'mysql',
    connection: {
        host: '{database_host}',
        port: '{database_port}',
        user: '{database_user}',
        password: '{database_pwd}',
        database: '{database_name}',
        charset: 'utf8' 
    },
    debug: false
}
```
`server` 部分我们不使用TCP连接，而是用UNIX域套接字，所以此处应为
```javascript
server: {
    socket: {
        path: '{custom_file_name}.sock',
        permissions: '0666'
    }
}
```
至此，Ghost 已配置完成。
#### 0x02 新建 Nginx 配置文件 ####
在你的 Nginx 配置文件夹中，例如 /etc/nginx/conf.d，新建一个以你的域名为名的 conf 文件，例如 blog.magentaize.net.conf。接下来开始编辑该文件，添加一个
 `server` 段，监听80端口。

此处配置 `location` 段是为了将同服务器上的所有域名的证书验证都放在一个文件夹中
```nginx
server{
    listen 80;
	listen [::]:80;
    server_name blog.magentaize.net;

	location ^~/.well-known/ {
	    alias /home/wwwroot/.well-known;
		default_type "text/plain";
	}
}
```
让 Nginx 重新加载配置文件
```html
$ nginx -s reload
```
#### 0x03 配置 Let’s Encrypt 与证书续期 ####
安装 Let’s Encrypt 客户端
```html
$ yum install certbot
```
获取证书，--email参数后接你的邮箱，-w参数后接通过域名访问到的真实目录，这个目录并非是网站的目录，而是通过域名访问到的验证文件目录，例如 `https://blog.magentaize.net/.well-known/acme-challenge/Sdfr4we3TSDE44Gset8hDSg`，-d参数后接申请证书的域名，可以用多个-d参数来一次获取多个证书
```html
$ certbot certonly --webroot --email admin@magentaize.net -w /home/wwwroot -d blog.magentaize.net -d magentaize.net
```
执行此命令后会生成证书, 保存在 /etc/letsencrypt/live 中对应的域名目录下面。

由于 Let’s Encrypt 的证书仅有 3 个月的有效期，所以需要配置证书自动续期，当执行续期时，cerbot 会遍历所有证书，并重新生成将在30日内过期的证书。

修改文件 /etc/crontab，添加一条计划任务，计划在每月的28日23点执行证书续期
```html
0 23 28 * * root certbot renew --quiet && nginx -s reload
```
加载任务，随后可以使用 `crontab -l` 查看已配置的全部任务
```html
$ crontab /etc/crontab
```
#### 0x04 配置 HTTPS ####
为了更加安全的 HTTPS，需要添加完全向前保密（PFS）的支持。为了生成 `dhparam.pem`，执行
```html
$ openssl dhparam -out /etc/letsencrypt/live/blog.magentaize.net/dhparam.4096.pem 4096
```
重新打开网站的 Nginx 配置文件，将 80 端口的访问全部 301 重定向至 443 端口
```nginx
server{
    listen 80;
	listen [::]:80;
    server_name blog.magentaize.net;
    add_header Strict-Transport-Security "max-age=31536000; includeSubdomains; preload";
	
	location /{
		return 301 https://$server_name$request_uri;
	}
}
```
再添加一个 `server` 段以配置 https
```nginx
server{
    # 使用HTTP/2
    listen 443 ssl http2;
	listen [::]:443 ssl http2;
    server_name blog.magentaize.net;
	
	# 添加HSTS头
	add_header Strict-Transport-Security "max-age=31536000; includeSubdomains; preload";
	
    # 指定证书位置
    ssl_certificate /etc/letsencrypt/live/blog.magentaize.net/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/blog.magentaize.net/privkey.pem;
	ssl_trusted_certificate /etc/letsencrypt/live/blog.magentaize.net/chain.pem;

    # 指定加密协议
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    # 指定PFS证书位置
    ssl_dhparam dhparam.4096.pem;
    # 使用适当的加密套件
    ssl_prefer_server_ciphers on;
    ssl_ciphers 'TLS13-CHACHA20-POLY1305-SHA256:TLS13-AES-256-GCM-SHA384:TLS13-AES-128-GCM-SHA256:EECDH+CHACHA20:EECDH+CHACHA20-draft:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';
    # 设置SSL会话缓存与超时
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
	
    # 设置在线证书状态协议
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 8.8.8.8 8.8.4.4 223.5.5.5 valid=300s;
    resolver_timeout 5s;
}
```
#### 0x05 让 Nginx 分发静态文件 ####
>以下内容都在 443 端口的 `server` 段中添加

由于之前已经配置了 Let’s Encrypt 的自动续期，所以需要在生产环境中依然可以进行网站的目录验证，于是需要添加一条规则
```nginx
location ^~/.well-known/ {
	    alias /home/wwwroot/.well-known;
		default_type "text/plain";
}
```
对于静态资源，例如图片、css、js，我们不希望由 node.js 来代为执行，毕竟 Nginx 的性能要高得多，所以需要添加以下规则
```nginx
location /content/images {
    alias /home/wwwroot/blog.magentaize.net/content/images;
    access_log off;
    expires max;        
}
location /assets {
    alias /home/wwwroot/blog.magentaize.net/content/themes/casper/assets;
    access_log off;
    expires max;         
}
location /example {
    alias /home/wwwroot/blog.magentaize.net/content/themes/casper/example;
    access_log off;
    expires max;         
}    
location /public {
    alias /home/wwwroot/blog.magentaize.net/core/built/public;
    access_log off;
    expires max;        
}
location /ghost/scripts {
    alias /home/wwwroot/blog.magentaize.net/core/built/scripts;
    access_log off;
    expires max;         
}
```
#### 0x06 配置 Node.js 的反向代理 ####
设置 upstream
```nginx
upstream blog_magentaize_net_upstream {  
    server unix:/home/wwwroot/blog.magentaize.net/ghost.sock;
    keepalive 64;
}
```
添加以下规则
```nginx
location / {
    # 让 Nginx 处理页面压缩
    proxy_set_header Accept-Encoding ""; 
    # 隐藏 Ghost 添加的 header
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    # 设置反向代理协议，因为最初的请求是https而Ghost是使用了http，如果不设置为https会导致循环重定向
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header Host $http_host;
    # 保持原请求IP
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header REMOTE-HOST $remote_addr;
    proxy_pass http://blog_magentaize_net_upstream;
}
```
可以选择开启 Nginx 自带的页面缓存，首先要禁止为后台生成缓存
```nginx
location ~^/(?:ghost|signout) {
    proxy_set_header Accept-Encoding ""; 
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header Host $http_host;
    proxy_redirect off;
    proxy_pass http://blog_magentaize_net_upstream;
    add_header Cache-Control "no-cache, private, no-store, must-revalidate, max-stale=0, post-check=0, pre-check=0";
}
```
然后再设置缓存目录
```nginx
proxy_cache_path /home/wwwroot/cache levels=1:2 keys_zone=STATIC:100m inactive=24h max_size=512m;
```
最后在 `location /` 中添加
```nginx
# 设置缓存池
proxy_cache STATIC;
proxy_cache_valid 200 60m;
proxy_cache_valid 404 1m;
# 忽略Ghost的Set-Cookie header，这会导致不使用缓存
proxy_ignore_headers Set-Cookie;
proxy_hide_header Set-Cookie;
proxy_ignore_headers X-Accel-Expires Expires Cache-Control;
# 设置浏览器缓存超时
expires 10m;
```
#### 0x07 安装 Node.js ####
Ghost 官方推荐使用的 Node.js 版本与最新版本相差较大，但运行起来并没有遇到意外，直接下载最新版即可 [Node.js](https://nodejs.org/dist/) 
```html
$ wget https://nodejs.org/dist/v7.10.0/node-v7.10.0.tar.gz
$ tar xvzf node-v7.10.0.tar.gz
$ rm node-v7.10.0.tar.gz
$ cd node-v7.10.0.tar.gz
$ ./configure
$ make && make install
```
然后打开 Ghost 的 `package.json`，将 `"node": "^4.2.0 || ^6.4.0"` 修改为下载的版本即可。
#### 0x08 启动 Ghost ####
首先安装依赖
```html
$ npm install --production
```
为了让 Ghost 能够随开机启动，需要一个守护进程，这里选择用 PM2
```html
$ npm install -g pm2
```
在 Ghost 目录下，让 PM2 来接管 Ghost 的运行，--name参数后接的是 pm2 进程名，可以随自己修改
```html
$ NODE_ENV=production pm2 start index.js --name "ghost"
```
可以查看当前由 pm2 托管的所有进程
```html
$ pm2 list
```
让 pm2 开机启动
```html
$ pm2 startup centos
```
保存当前设置
```html
$ pm2 save
```
至此，Ghost 已全部配置完成。

#### 0x09 Ghost升级 ####
仅需要从 [Ghost Developer Area](https://ghost.org/developers/) 得到最新的下载链接，解压覆盖同名文件，然后执行
```html
$ node install --production
$ pm2 restart ghost
```
即可。