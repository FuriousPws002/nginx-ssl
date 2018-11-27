# nginx-ssl
阿里云服务器采用acme.sh配置nginx ssl

## 先决条件
+ 阿里云服务器
+ 域名

## 本机环境
阿里云服务器使用`centos 7.4`

## 操作步骤
+ **安装nginx**
+ **获取阿里云api_key、api_secret**
+ **安装acme.sh**
+ **生成域名证书**
+ **配置nginx ssl**

### 安装nginx  
安装nginx，参考nginx[安装](http://www.nginx.cn/install "Markdown")   
安装完成后，设置nginx为服务[参考](https://www.nginx.com/resources/wiki/start/topics/examples/systemd/ "Markdown")

### 获取阿里云api_key、api_secret
生成[阿里云api_key](https://ram.console.aliyun.com/ "Markdown")   
可参考[博客](https://frontenddev.org/article/use-acme-sh-deployment-let-s-encrypt-by-ali-cloud-dns-generic-domain-https-authentication.html "Markdown")设置key

### 安装acme.sh
参考文档[安装acme.sh](https://github.com/Neilpang/acme.sh/wiki/%E8%AF%B4%E6%98%8E "Markdown")

### 生成域名证书  
执行如下命令校验dns

    acme.sh --issue --dns dns_ali -d <domain>
其中`<domain>`为实际域名   
创建证书路径

     mkdir /etc/nginx/ssl/<domain>
复制域名证书

    acme.sh --installcert -d <domain> \
      --key-file /etc/nginx/ssl/<domain>/<domain>.key \
      --fullchain-file /etc/nginx/ssl/<domain>/fullchain.cer \
      --reloadcmd  "sudo systemctl reload nginx"
若采用`非root`用户安装acme.sh，配置当前用户sudo`免密`，方法如下

    1.添加写权限
    chmod u+w /etc/sudoers
    2.编辑/etc/sudoers
    添加如下一行
    <user> ALL=(ALL) NOPASSWD:ALL
    其中<user>为实际的用户   
    3.移除写权限
    chmod u-w /etc/sudoers

### 配置nginx ssl
生成DH

    openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
创建nginx ssl配置全局文件

    vi /etc/nginx/snippets/ssl.conf
添加如下配置

    ssl_dhparam /etc/ssl/certs/dhparam.pem;
    
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers 'ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:ECDHE-ECDSA-DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!DSS';
    ssl_prefer_server_ciphers on;
    
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 30s;

    add_header Strict-Transport-Security "max-age=15768000; includeSubdomains; preload";
    add_header X-Frame-Options SAMEORIGIN;
    add_header X-Content-Type-Options nosniff;

添加nginx读取配置文件的路径

    vi /usr/local/nginx/nginx.conf
在http 节点下添加如下一行

    include       /etc/nginx/conf.d/*.conf;

创建当前域名配置文件

    mkdir /etc/nginx/conf.d
    vi /etc/nginx/conf.d/<domain>.conf
添加如下内容

    server {
      listen 80;
      server_name <domain>;
      access_log off;
      return 301 https://$host$request_uri;
    }
    
    server {
        listen 443 ssl;
        server_name <domain>;
    
        ssl_certificate /etc/nginx/ssl/<domain>/fullchain.cer;
        ssl_certificate_key /etc/nginx/ssl/<domain>/<domain>.key;
        include /etc/nginx/snippets/ssl.conf;
    }
其中`<domain>`为实际域名

重载nginx配置

    systemctl reload nginx
到此，完成nginx ssl的配置，可在浏览器访问域名测试是否跳转到`https://<domain>`

## 参考
http://www.nginx.cn/install   
https://www.nginx.com/resources/wiki/start/topics/examples/systemd/   
https://linuxize.com/post/secure-nginx-with-let-s-encrypt-on-centos-7/   
https://github.com/Neilpang/acme.sh/wiki/%E8%AF%B4%E6%98%8E   
https://ram.console.aliyun.com/   
https://frontenddev.org/article/use-acme-sh-deployment-let-s-encrypt-by-ali-cloud-dns-generic-domain-https-authentication.html