# built-docker
中国大陆自建Docker教程

## 自建Docker Hub加速镜像
在使用Docker的过程中会大量的涉及Pull镜像的操作，但是由于官方的镜像服务器在国外再加上某个防火墙的阻拦，导致直接拉取镜像非常困难（看脸）。所以通常的操作是设置一个由国内厂商、机构提供的加速镜像，来提高拉取镜像的速度。但是随着Docker hub限制了未注册用户的拉取频率、各大厂商、机构开始将加速镜像转为内部使用，个人用户拉取镜像变得越来越困难。在长期拉取镜像速度看脸的头疼之下，尝试通过 Nginx 和 Cloudflare Worker 两种方案以及两种方案的组合方案自建Docker hub加速镜像来解决这个问题。

## 自建加速镜像

在尝试搭建之前找了挺多资料，主流的方式有：使用官方提供的 registry，第三方的 Nexus、Harbor。但是使用 registry 搭建一直没有成功，客户端一直报找不到指定镜像；使用 Nexus 搭建又有些太复杂。最后自己总结出来了这两个比较方便的方案。

一点小发现：
在使用 Nginx 搭建时，发现服务器的流量很小，经过检查 Nginx 的日志后发现，Docker hub 镜像仓库返回的下载地址是需要 307 跳转的，而跳转后的地址依然下载很慢，所以需要在服务端处理这个跳转，将跳转后的数据返回客户端。

# 方案一：使用 Nginx 搭建
系统：Debian 12 服务器：[棉花云](https://www.88sup.com) 洛杉矶 
提到要加速一个网站，自然就能想到使用 Nginx 反代一下了。接下来是具体的配置方案

## 安装 Nginx
```
sudo apt update 
sudo apt install nginx
```
## 防火墙放行指定端口
我这里使用的防火墙是系统自带的 UFW，并且没有开厂商提供的防火墙（忘记是关掉了还是本来就没有，反正没开），所以只需要 sudo ufw allow ‘Nginx Full’ 一条命令即可，这样就会放行 IPv4 和 IPv6 的 80 和 443 端口（当然也可以手动 sudo ufw allow 443 这样只开放 443 端口）
如果没有使用防火墙，就不用设置这一步，如果还使用了厂商提供的防火墙，就需要在厂商的面板处同样开放这些端口

## 配置 Nginx 

使用命令 sudo vim /etc/nginx/nginx.conf 编辑 Nginx 配置，在 http 块下增加一个 server 块

/etc/nginx/nginx.conf

```
#反代docker hub镜像源
     server {
             listen 443 ssl;
             server_name 域名;

             ssl_certificate 证书地址;
             ssl_certificate_key 密钥地址;

             ssl_session_timeout 24h;
             ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256';
             ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;

             location / {
                     proxy_pass https://registry-1.docker.io;  # Docker Hub 的官方镜像仓库
                     proxy_set_header Host registry-1.docker.io;
                     proxy_set_header X-Real-IP $remote_addr;
                     proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                     proxy_set_header X-Forwarded-Proto $scheme;

                     # 关闭缓存
                     proxy_buffering off;

                     # 转发认证相关的头部
                     proxy_set_header Authorization $http_authorization;
                     proxy_pass_header  Authorization;

                     # 对 upstream 状态码检查，实现 error_page 错误重定向
                     proxy_intercept_errors on;
                     # error_page 指令默认只检查了第一次后端返回的状态码，开启后可以跟随多次重定向。
                     recursive_error_pages on;
                     # 根据状态码执行对应操作，以下为301、302、307状态码都会触发
                     error_page 301 302 307 = @handle_redirect;

             }
             location @handle_redirect {
                     resolver 1.1.1.1;
                     set $saved_redirect_location '$upstream_http_location';
                     proxy_pass $saved_redirect_location;
             }
     }
```

然后按 Esc ，输入 : wq 保存退出即可

重新加载 Nginx 配置 输入命令 sudo nginx -s reload，没有报错就说明配置已经生效

# 方案二：使用 CloudFlare Worker 搭建
作为一个贫穷（doge）的用户，可以免费使用的 CloudFlare Worker 自然要想方设法的用用了，虽然 CloudFlare Worker 的访问速度在国内也不算稳定，但在 CloudFlare 的边缘网络的加持下，白天的速度还是非常可观的，晚上会比较慢但还是比直接使用官方的镜像源要快上很多（又不要钱，要啥自行车.jpg） 这里是在 [基于 Cloudflare Worker 的容器镜像加速器](https://github.com/Doublemine/container-registry-worker) 的基础上稍作修改

在面板左侧找到 Workers 和 Pages，然后点击右侧的 创建应用程序、创建 Worker，修改一个好记的名字，部署

接下来编辑代码，将 worker.js 的内容替换为下面内容

worker.js
```
import HTML from './docker.html';

export default {
    async fetch(request) {
        const url = new URL(request.url);
        const path = url.pathname;
        const originalHost = request.headers.get("host");
        const registryHost = "registry-1.docker.io";

        if (path.startsWith("/v2/")) {
        const headers = new Headers(request.headers);
        headers.set("host", registryHost);

        const registryUrl = `https://${registryHost}${path}`;
        const registryRequest = new Request(registryUrl, {
            method: request.method,
            headers: headers,
            body: request.body,
            // redirect: "manual",
            redirect: "follow",
        });

        const registryResponse = await fetch(registryRequest);

        console.log(registryResponse.status);

        const responseHeaders = new Headers(registryResponse.headers);
        responseHeaders.set("access-control-allow-origin", originalHost);
        responseHeaders.set("access-control-allow-headers", "Authorization");
        return new Response(registryResponse.body, {
            status: registryResponse.status,
            statusText: registryResponse.statusText,
            headers: responseHeaders,
        });
        } else {
        return new Response(HTML.replace(/{{host}}/g, originalHost), {
            status: 200,
            headers: {
            "content-type": "text/html"
            }
        });
        }
    }
}
```
这里相比原项目，将 redirect: “manual” 修改为了 redirect: “follow”，目的是为了让脚本自行处理 307 跳转，直接返回给我们跳转后的数据。
新建一个名为 docker.html 的 文件，内容如下

docker.html
```
<!DOCTYPE html>
<html>
    <head>
        <meta charset="utf-8" />
        <meta name="viewport" content="width=device-width, initial-scale=1" />
        <title>Mirror Usage</title>
        <style>
        html {
        height: 100%;
        }
        body {
        font-family: "Roboto", "Helvetica", "Arial", sans-serif;
        font-size: 16px;
        color: #333;
        margin: 0;
        padding: 0;
        height: 100%;
        display: flex;
        flex-direction: column;
        justify-content: space-between;

        }
        .container {
            margin: 0 auto;
            max-width: 600px;
        }

        .header {
            background-color: #438cf8;
            color: white;
            padding: 10px;
            display: flex;
            align-items: center;
        }

        h1 {
            font-size: 24px;
            margin: 0;
            padding: 0;
        }

        .content {
            padding: 32px;
        }

        .footer {
            background-color: #f2f2f2;
            padding: 10px;
            text-align: center;
            font-size: 14px;
        }
        </style>
    </head>
    <body>
        <div class="header">
        <h1>Mirror Usage</h1>
        </div>
        <div class="container">
        <div class="content">
            <p>镜像加速说明</p>
            <p>
            为了加速镜像拉取,你可以使用以下命令设置registery mirror:
            </p>
            <pre>
            sudo tee /etc/docker/daemon.json &lt;&lt;EOF
            {
                "registry-mirrors": ["https://{{host}}"]
            }
            EOF
            </pre>
            </br>
            <p>
            为了避免 Worker 用量耗尽,你可以手动 pull 镜像然后 re-tag 之后 push 至本地镜像仓库:
            </p>
            <pre>
            docker pull {{host}}/library/alpine:latest # 拉取 library 镜像
            docker pull {{host}}/coredns/coredns:latest # 拉取 library 镜像
            </pre>
        </div>
        </div>
        <div class="footer">
        <p>Powered by Cloudflare Workers</p>
        </div>
    </body>
</html>
```
接下来，点击右上角的 部署，稍等片刻

最后，返回面板，在 设置，触发器 处设置一个自己的域名，一切就大功告成了
不建议使用自带的 workers.dev 的域名，被墙了

# 方案一、二整合
本来上面的两个方案是独立的，一个使用 Nginx 部署，一个使用 CloudFlare Worker 部署，但是就在我写这篇博客的时候，突然想到，为什么不能把上面的两个方案整合起来呢？
利用服务器搭建的 Nginx 作为中转，优先由服务器直连 Docker hub 的官方源，当服务器的 IP 请求次数超限后（会报 429 错误），就把请求转发到 CloudFlare Worker 部署的镜像源上，利用 CloudFlare Worker 再做一次中转。这样就即保证了使用服务器中转提高速度，又保证了不会因为服务器的 IP 请求速度过多而受限制，唯一的限制就是服务器的带宽和流量了，几乎完美！！！

## 部署方法：
将上面部署的 Nginx 配置替换为下面的配置并使用 sudo nginx -s reload 重新加载即可

/etc/nginx/nginx.conf
```
#反代docker hub镜像源
    server {
            listen 443 ssl;
            server_name 域名;

            ssl_certificate 证书地址;
            ssl_certificate_key 密钥地址;

            proxy_ssl_server_name on; # 启用SNI

            ssl_session_timeout 24h;
            ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256';
            ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;

            location / {
                    proxy_pass https://registry-1.docker.io;  # Docker Hub 的官方镜像仓库

                    proxy_set_header Host registry-1.docker.io;
                    proxy_set_header X-Real-IP $remote_addr;
                    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                    proxy_set_header X-Forwarded-Proto $scheme;

                    # 关闭缓存
                    proxy_buffering off;

                    # 转发认证相关的头部
                    proxy_set_header Authorization $http_authorization;
                    proxy_pass_header  Authorization;

                    # 对 upstream 状态码检查，实现 error_page 错误重定向
                    proxy_intercept_errors on;
                    # error_page 指令默认只检查了第一次后端返回的状态码，开启后可以跟随多次重定向。
                    recursive_error_pages on;
                    # 根据状态码执行对应操作，以下为301、302、307状态码都会触发
                    #error_page 301 302 307 = @handle_redirect;

                    error_page 429 = @handle_too_many_requests;
            }
            #处理重定向
            location @handle_redirect {
                    resolver 1.1.1.1;
                    set $saved_redirect_location '$upstream_http_location';
                    proxy_pass $saved_redirect_location;
            }
            # 处理429错误
            location @handle_too_many_requests {
                    proxy_set_header Host 替换为在CloudFlare Worker设置的域名;  # 替换为另一个服务器的地址
                    proxy_pass http://替换为在CloudFlare Worker设置的域名;
                    proxy_set_header Host $http_host;
            }
    }
```
如果想要反代 ghcr 镜像源呢？只要参考上面的配置，将域名、header 修改一下即可
并且因为 ghcr 好像不像 docker hub 有下载频率的限制，所以也不用去 Cloudflare Worker 部署了，直接在服务器上部署一个就行。
```
#反代ghcr镜像源
server {
        listen 443 ssl;
        server_name 域名;

        ssl_certificate 证书地址;
        ssl_certificate_key 密钥地址;
        proxy_ssl_server_name on;
        ssl_protocols TLSv1.1 TLSv1.2 TLSv1.3;
        #error_log /home/ubuntuago/proxy_docker.log debug;
        if ($blocked_agent) {
                return 403;
        }

        location / {
                proxy_pass https://ghcr.io;  # Docker Hub 的官方镜像仓库

                proxy_set_header Host ghcr.io;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;

                # 关闭缓存
                proxy_buffering off;

                # 转发认证相关的头部
                proxy_set_header Authorization $http_authorization;
                proxy_pass_header  Authorization;
                # 对 upstream 状态码检查，实现 error_page 错误重定向
                proxy_intercept_errors on;
                # error_page 指令默认只检查了第一次后端返回的状态码，开启后可以跟随多次重定向。
                recursive_error_pages on;
                # 根据状态码执行对应操作，以下为301、302、307状态码都会触发
                error_page 301 302 307 = @handle_redirect;

                #error_page 429 = @handle_too_many_requests;

        }
        #处理重定向
        location @handle_redirect {
                resolver 1.1.1.1;
                set $saved_redirect_location '$upstream_http_location';
                proxy_pass $saved_redirect_location;
        }

}
```
## 新增的一些代码的作用：

proxy_ssl_server_name on;
HTTPS, 需要在握手时候验证证书, 所以在握手时候需要将域名告诉对方, 找到匹配的证书, 这就是 SNI 的工作. 否则会导致证书找不到而请求失败.
默认情况下, nginx 并不会开启 proxy_ssl_server_name, 也就是说不启用 SNI. 如果使用 nginx 反代一个虚拟主机的服务, 比如 Cloudflare Workers, 此时如果不开启 SNI, 会导致与 CF 握手时候, CF 并不清楚请求哪一个域名下的服务, 所以找不到匹配的证书, 因此会报 502 错误. 当然也可以使用 proxy_ssl_name 字段复写于最终服务器收到的域名.
引自：[SNI](https://faichou.com/sni/)

在尝试整合这两个方案的时候不知道这个参数，一直报 502..人都麻了

error_page 429 = @handle_too_many_requests;
当错误代码为 429 时（即请求次数超限）转发到在 Cloudflare Workers 部署的镜像源


proxy_set_header Host 替换为在 CloudFlare Worker 设置的域名;
将发给 CloudFlare Worker 的请求加上正确的域名方可让请求到达我们搭建的镜像源，如果不加会报 404

# 总结
根据测试，自建一个 Docker hub 加速镜像完全可行，使用 Nginx 搭建会比较看重服务器的线路，但只要不算太差，就完全可用了。不处理跳转会比较节省流量，但是为了速度，节省这一点流量也没什么必要。有一定的成本，但是如果本身就已经买了服务器的话，搭建个镜像源就几乎是顺带的事情了。
而使用 CloudFlare Worker 搭建..只能说稍微好一点点，如果服务器需要大量拉取镜像的话或许会用到这个方案。
而将方案一、二整合，几乎就是现阶段最完美的方案了，速度快且不受拉取次数的限制，只要 Docker hub 别乱变，就可以用超久。

# 如果失效，显示
```
root@VM-4-14-debian:~# docker pull hub1.nat.tf/library/alpine:latest
Error response from daemon: Head "https://hub1.nat.tf/v2/library/alpine/manifests/latest": Get "https://auth.docker.io/token?scope=repository%3Alibrary%2Falpine%3Apull&service=registry.docker.io": net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
```
# 那就把Nginx的改成如下，根据自己配置来调整，我是基于1panel的OpenResty搭建，记得重载配置
```
user  root;
worker_processes  auto;
error_log  /var/log/nginx/error.log notice;
error_log  /dev/stdout notice;
pid        /var/run/nginx.pid;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    server_tokens off;
    access_log  /var/log/nginx/access.log  main;
    access_log /dev/stdout main;
    sendfile        on;

    server_names_hash_bucket_size 512;
    client_header_buffer_size 32k;
    client_max_body_size 50m;
    keepalive_timeout 60;
    keepalive_requests 100000;

    gzip on;
    gzip_min_length  1k;
    gzip_buffers     4 16k;
    gzip_http_version 1.1;
    gzip_comp_level 2;
    gzip_types     text/plain application/javascript application/x-javascript text/javascript text/css application/xml;
    gzip_vary on;
    gzip_proxied   expired no-cache no-store private auth;
    gzip_disable   "MSIE [1-6]\.";

    limit_conn_zone $binary_remote_addr zone=perip:10m;
    limit_conn_zone $server_name zone=perserver:10m;

    # 反代docker hub镜像源的服务器配置
    server {
        listen 443 ssl;
        server_name 域名;

        ssl_certificate 证书地址;
        ssl_certificate_key 密钥地址;

        ssl_session_timeout 24h;
        ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256';
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;

        location /v2/ {
            proxy_pass https://registry-1.docker.io;  # Docker Hub 的官方镜像仓库
            proxy_set_header Host registry-1.docker.io;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # 关闭缓存
            proxy_buffering off;

            # 转发认证相关的头部
            proxy_set_header Authorization $http_authorization;
            proxy_pass_header Authorization;

            # 重写 www-authenticate 头为你的反代地址
            proxy_hide_header www-authenticate;
            add_header www-authenticate 'Bearer realm="https://域名/token",service="registry.docker.io"' always;

            # 对 upstream 状态码检查，实现 error_page 错误重定向
            proxy_intercept_errors on;
            recursive_error_pages on;
            error_page 301 302 307 = @handle_redirect;
        }

        # 处理 Docker OAuth2 Token 认证请求
        location /token {
            resolver 1.1.1.1 valid=600s;
            proxy_pass https://auth.docker.io;  # Docker 认证服务器

            proxy_set_header Host auth.docker.io;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_set_header Authorization $http_authorization;
            proxy_pass_header Authorization;

            proxy_buffering off;
        }

        location @handle_redirect {
            resolver 1.1.1.1;
            set $saved_redirect_location '$upstream_http_location';
            proxy_pass $saved_redirect_location;
        }
    }

    include /usr/local/openresty/nginx/conf/conf.d/*.conf;
    include /usr/local/openresty/1pwaf/data/conf/waf.conf;
}
```




