# acme-tiny 半自动获取LETS ENCRYPT SSL证书脚本


## 步骤 (以下假设你要为主域名domain.com申请证书)

 - 克隆本仓库到主机并进入acme-tiny目录

 - 创建private key
 通过openssl创建key，当然你也可以用2048(推荐，速度快)
 ```
 openssl genrsa 2048 > account.key
 or
 openssl genrsa 4096 > account.key
 ```
 - 为你的域名创建证书
 Create a certificate signing request (CSR) for your domains.
 ```
 openssl genrsa 2048 > domain.key
 #or
 openssl genrsa 4096 > domain.key
 #然后执行
 
 #单域名
 openssl req -new -sha256 -key domain.key -subj "/CN=domain.com" > domain.csr
 #多域名
 openssl req -new -sha256 -key domain.key -subj "/" -reqexts SAN -config <(cat /etc/ssl/openssl.cnf <(printf "[SAN]\nsubjectAltName=DNS:domain.com,DNS:www.domain.com")) > domain.csr
 ```


 - 配置域名验证
lets encrypt会要求访问你所要验证的域名的指定目录(.well-known/acme-challenge/)来进行验证，
```
#首先 在网站根目录创建验证目录 假设网站根目录为 /var/rootpath/
mkdir -p /var/rootpath/.well-known/acme-challenge/
#注意如果是nginx服务器并且配置文件含有location ~ /\.的先暂时注释掉才能访问.开头的文件夹
#然后执行命令
python acme_tiny.py --account-key ./account.key --csr ./domain.csr --acme-dir /var/rootpath/.well-known/acme-challenge/ > ./signed.crt
```

 - 配置证书
 首先创建中间证书
 ```
wget -O - https://letsencrypt.org/certs/lets-encrypt-x3-cross-signed.pem > intermediate.pem
cat signed.crt intermediate.pem > chained.pem
 ```
 然后在配置文件引入证书，以nginx为例，在server字段中加入:
 ```
     listen 443;
    server_name www.domain.com;
    root  /var/www/www.domain.com/;
    ssl on;
    ssl_certificate /path/chained.pem;
    ssl_certificate_key /path/domain.key;
    ssl_session_timeout 5m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_session_cache shared:SSL:50m;
    ssl_prefer_server_ciphers on;
 ```
 重启服务即可

 - 自动更新证书配置
  lets encrypt的免费证书只有三个月期限，所以可以配置自动更新脚本，每隔一段时间自动更新
  #auto_renew.sh
  ```
  #!/usr/bin/sh
python acme_tiny.py --account-key ./account.key --csr ./domain.csr --acme-dir /var/rootpath/.well-known/acme-challenge/ > ./signed.crt
wget -O - https://letsencrypt.org/certs/lets-encrypt-x3-cross-signed.pem > intermediate.pem
cat /tmp/signed.crt intermediate.pem > /path/to/chained.pem
service nginx reload
  ```
 添加crond计划任务每月1日执行执行
 ```
0 0 1 * * /path/to/auto_renew.sh 2>> /var/log/acme_tiny.log
```
 
