
博客是一种记录和分享技术知识的优秀平台，由于个人很不喜欢 CSDN 遍地广告的风格，所以就有了搭建属于自己的博客网站的想法。

由于还在读书阶段，有幸能够享受阿里云提供的学生优惠，所以以一个很低的价格购买了阿里云的 ECS 和属于自己的一个域名 [coolxxy.cn](https://www.coolxxy.cn)，以后会尽量把自己学到的知识在这个网站上面做记录。某种程度上，建立自己的博客站也是想以此多做鞭策多学多写吧。

搭建这个站点，我选择的博客系统是 [Ghost](https://ghost.org) , Ghost 相比老牌的 Wordpress 非常轻量，功能方面也满足我的需求。使用 docker 搭建一方面这种方式很简便，容易管理，另一方面也是像体验一下各种文档和书中描写 Docker 在实际开发和运维方面的种种优点。而使用 Nginx 只是为了实现 https 访问。

接下来是具体安装过程：

- 准备虚拟主机和域名

从阿里云的云翼计划页面购买学生专属产品，我选择的是云服务器 ECS, 每月￥9.5，其实如果服务器只用来建立博客站的话，可以选择 **轻量应用服务器**，这种方式能够获得更高的网络带宽。

从阿里云购买域名，由于新的规定，国内网站必须完成备案才能正常访问，而目前备案最方便的是 cn 和 com 顶级域名，如果想用其他的顶级域名，提前确认好是否支持备案。我的网站备案从阿里云的备案系统完成，备案过程相对会消耗你很多的时间和精力。

在开始安装具体应用前，我们首先完成服务器和域名的一些基础配置。

我选择的操作系统是 Ubuntu Server 18.04, 也可以选择更为稳定的 Ubuntu Server 16.04 或者其他你喜欢的发行版。这里主要修改服务器的默认安全组规则，开放需要的端口，开放 `3001` 用于 ghost 测试和反向代理，开放 `80` 和 `443` 用于 nginx 的服务。另外，为了安全，我修改了默认的 ssh 端口，因此还需要开放修改后的 ssh 端口。

域名配置主要是完成 DNS 解析配置，添加两个 A 记录类型的记录，主机记录分别为 `www` 和 `@`, 记录值全部为 ECS 的公网 IP.

- 安装 docker

这个步骤可以利用一键安装脚本完成
```bahs
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
```
也可以使用命令行自己完成，Docker 官方文档提供了详细的步骤，在这里我们只需要把其中的官方网址换成阿里云镜像的地址便可
```bash
安装依赖的系统工具
sudo apt update
sudo apt -y install apt-transport-https ca-certificates curl software-properties-common
安装GPG证书
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
添加 Docker 软件源信息
sudo add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
更新并安装 Docker-CE
sudo apt update
sudo apt -y install docker-ce
```
- 安装 ghost

使用docker命令启动ghost服务

```bash
docker run -d --name some-ghost -p 3001:2368 -v /path/to/ghost/blog:/var/lib/ghost/content -e url=<你的域名> ghost
```
命令中的参数 `-p 3001:2368` 将容器中的ghost服务的端口 2368 映射到了宿主机的 3001端口；

参数 `-v /path/to/ghost/blog:/var/lib/ghost/content` 在启动 ghost 容器时挂载了一个数据卷，容器里面的 ghost 的博客数据会存储到宿主机的 `/path/to/ghost/blog` 路径下，以后重新启动新的 ghost 容器时，可以从这个目录恢复原有的博客。也可以使用阿里云的其他服务备份这个目录，防止数据丢失。

参数 `-e url=<你的域名>` 是告诉 ghost 容器你的博客站的域名，ghost 容器启动时会用这个域名完成一些初始配置。

以上命令执行完成后，在浏览器访问 http://<你的ECS的公网IP>:3001, 会看到初始的 ghost 服务，可以在此完成初始账户设置。

- 安装 nginx

安装 nginx 是为了给网站配置 https, 这里的主要思路是给 nginx 配置证书以便于 https 访问，然后配置 nginx 反向代理到 ghost 的 3001 端口。

首先安装 nginx 服务

```bash
docker run --name some-nginx -d -p 80:80 -p 443:443 nginx 
```
启动完毕后在浏览器访问 http://<你的ECS的公网IP>, 会看到 nginx 的欢迎界面。

- 配置 https 和反向代理

我使用了 [freessl.cn](https://freessl.cn/) 提供的免费证书，在 freessl 的网站填写自己的域名，点击 `创建免费的ssl 证书`，依照弹出的提示，配置之前购买的域名，完成域名所有权验证。如果采用 DNS 验证，就是按照它提供的字段创建一个 dns 解析记录，一般是 TXT 类型，主机记录是 _dnsauth, 记录值是它给的一长串字符串。配置完成后，点击验证，验证通过后便可以下载到自己的密钥， 一般是两个文件，后缀名分别是 `pem` 和 `key`。

接下来首先使用 scp 将两个密钥文件上传到 ECS, 在 nginx 容器里面，建立一个路径为 `/etc/nginx/cert` 的文件夹，并用 `docker cp` 命令将两个密钥文件拷贝到容器里面的 `/etc/nginx/cert` 文件夹下。

由于 nginx 容器里面精简掉了文件编辑器，我在宿主机完成 nginx 配置文件修改后拷贝进容器的方式进行配置，我选择修改容器里面的 `/etc/nginx/conf.d/default.conf`，注释掉默认的配置，然后添加以下内容
```conf
upstream coolxxy_cn {
    server www.coolxxy.cn:3001;
    keepalive 8;
}

server {
    listen 443 ssl;

    server_name coolxxy.cn www.coolxxy.cn;
    ssl_certificate cert/full_chain.pem;
    ssl_certificate_key cert/private.key;

    ssl_session_timeout 5m;

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    add_header Strict-Transport-Security "max-age=63072000; includeSubdomains; preload";
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    ssl_stapling on;
    ssl_stapling_verify on;

    location / {
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_set_header X-NginX-Proxy true;
        proxy_pass http://coolxxy_cn/;
        proxy_redirect off;
    }
}
server {
    listen      80;
    server_name coolxxy.cn www.coolxxy.cn;
    return 301 https://$server_name$request_uri;
}
```
配置文件中的 upsteam 是建立了一个指向 ghost 服务的虚拟主机，然后建立 443 端口的 https 监听服务，再将 80 端口的 http 访问强制重定向至 https 访问。

至此，如果能够从 https://www.coolxxy.cn 正常访问到博客站，并且在浏览器输入 http://www.coolxxy.cn 和 http://coolxxy.cn 中的任意一个，都会重定向到 https://www.coolxxy.cn ，说明网站搭建成功。

另外，我们之前配置的安全规则中，为了测试 nginx 是能够从外网访问主机IP的 3001 端口的，我们在一切配置完毕后，在阿里云的管理界面把指向 3001 端口的安全组规则修改为 `授权类型=从地址段访问 授权对象=<你的ECS的公网IP/32>`，防止从公网直接 3001 端口带来的风险。