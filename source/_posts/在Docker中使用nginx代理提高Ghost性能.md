title: 在Docker中使用nginx代理提高Ghost性能
permalink: zai-dockershi-yong-nginxdai-li-ti-gao-ghostxing-neng
id: 4
updated: '2015-08-25 12:45:48'
date: 2015-08-12 10:59:30
---

> 前几天看到了[High Performance Ghost Configuration with NGINX](http://pnommensen.com/2014/09/07/high-performance-ghost-configuration-with-nginx/) 就想到了在Docker中使用NGINX链接Ghost，并且缓存部分内容。

> 觉得使用nginx的最大好处是能够最大限度的提升博客的性能。

> **注意，使用nginx缓存后，新发布或者更新的博文刷新将会比较慢！**
## Docker上运行Ghost
```
docker run -d -p 80:2368 --name your-ghost -e NODE_ENV=production -v [your ghost content path]:/var/lib/ghost luck9star/ghost-zh
```

> 这里运行的是中文版的Ghost镜像。

这样打来浏览器直接输入网址就可以看到运行中的Ghost了，不过现在这个网站还只是一个node.js的web应用而已，跟nginx没有任何关系。
## 运行nginx
```
#我们先停止并且删掉之前的Ghost，因为占用的80端口得给nginx使用。
docker stop your-ghost
docker rm your-ghost

#然后再次运行Ghost，不过不在对外开放端口。
docker run -d --name your-ghost -e NODE_ENV=production -v [your ghost content path]:/var/lib/ghost luck9star/ghost-zh

#最后则是运行nginx并且使用“--link”链接到ghost。
docker run -d -p 80:80 -p 443:443 --link your-ghost:ghost -v [your path]:/var/log/nginx -v [your path]:/etc/nginx/certs -v [your site config path]:/etc/nginx/conf.d --name=nginx nginx
```

这时候就可以看到nginx的默认网页。
> Volume文件上你也可以使用-volumn-from的方式链接Storage镜像.具体可以参照[Docker上nginx页面](https://hub.docker.com/_/nginx/)

接下去就需要在nginx中配置Ghost了。

我们可以[your site config path]的路径下建立并且编辑个ghost.conf的文件，当然文件名自然是随你喜好而定的咯。

> 配置可以参照[译文：通过 nginx 代理 Ghost 的高性能配置](https://idiotwu.me/high-performance-ghost-configuration-with-nginx/)

不过现在Ghost的** upstream**要修改为：

    upstream ghost_upstream {  
        server ghost:2368;
        keepalive 64;
    }

不过目前由于nginx 和 ghost容器文件不互通，只是相互链接了而已，无法做使用nginx处理静态资源，只能做缓存而已，配置文件暂时如下。

```
upstream ghost_upstream {
    server ghost:2368;
    keepalive 64;
}
proxy_cache_path /var/run/cache levels=1:2 keys_zone=STATIC:75m inactive=24h max_size=128m;
server {
  listen 80;
  server_name 0.0.0.0; #replace this line with your domain
  access_log /var/log/nginx/ghost.log; #replace this with any log name
  add_header X-Cache $upstream_cache_status;

  location / {
   add_header X-Cache $upstream_cache_status;
   location / {
        proxy_cache STATIC;
        proxy_cache_valid 200 30m;
        proxy_cache_valid 404 1m;
        proxy_pass http://ghost_upstream;
        proxy_ignore_headers X-Accel-Expires Expires Cache-Control;
        proxy_ignore_headers Set-Cookie;
        proxy_hide_header Set-Cookie;
        proxy_hide_header X-powered-by;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        expires 10m;
    }
    location ~ ^/(?:ghost|signout) {
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $http_host;
        proxy_pass http://ghost_upstream;
        add_header Cache-Control "no-cache, private, no-store, must-revalidate, max-stale=0, post-check=0, pre-check=0";
    }
  }
}
```

现在我们又得停删除之前的Ghost和nginx容器，建立一个公共的存储容器，然后重新创建各自链接到同一个`storage容器`的实例。

在Ghost的容器中，Ghost自身的内容是放在`/usr/src/ghost`路径下的（具体得参照Dockerfile，这里只代表了我的`luck9star/ghost-zh`和Docker股官方的`ghost`镜像中得路径。），所以，首先建立一个名为`ghost-storage`的公共存储容器：
```
docker create -v /usr/src/ghost -v [你的Ghost的Content路径]:/var/lib/ghost --name ghost-storage luck9star/ghost-zh /bin/true
```

然后再次创建Ghost和nginx并且添加这个存储容器作为Volumn。
```
docker run -d --name your-ghost -e NODE_ENV=production --volumes-from=ghost-storage luck9star/ghost-zh

docker run -d -p 80:80 -p 443:443 --link your-ghost:ghost -v [your path]:/var/log/nginx -v [your path]:/etc/nginx/certs -v [your site config path]:/etc/nginx/conf.d --volumes-from=ghost-storage --name=nginx nginx
```

现在，nginx中就可以访问到Ghost中得静态的脚本等内容了。
接下里我们就可以开始编辑ghost.conf中的内容了。（具体配置参照[High Performance Ghost Configuration with NGINX](http://pnommensen.com/2014/09/07/high-performance-ghost-configuration-with-nginx/)）
下面是完成的配置文件：

```
upstream ghost_upstream {
    server ghost:2368;
    keepalive 64;
}
proxy_cache_path /var/run/cache levels=1:2 keys_zone=STATIC:75m inactive=24h max_size=512m;

server {
   server_name 0.0.0.0;
   add_header X-Cache $upstream_cache_status;
   location / {
        proxy_cache STATIC;
        proxy_cache_valid 200 30m;
        proxy_cache_valid 404 1m;
        proxy_pass http://ghost_upstream;
        proxy_ignore_headers X-Accel-Expires Expires Cache-Control;
        proxy_ignore_headers Set-Cookie;
        proxy_hide_header Set-Cookie;
        proxy_hide_header X-powered-by;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        expires 10m;
    }
    location /content/images {
        alias /var/lib/ghost/images;
        access_log off;
        expires max;
    }
    location /assets {
        alias /var/lib/ghost/themes/casper-zh/assets;
        access_log off;
        expires max;
    }
    location /public {
        alias /usr/src/ghost/core/built/public;
        access_log off;
        expires max;
    }
    location /ghost/scripts {
        alias /usr/src/ghost/core/built/scripts;
        access_log off;
        expires max;
    }
    location ~ ^/(?:ghost|signout) {
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $http_host;
        proxy_pass http://ghost_upstream;
        add_header Cache-Control "no-cache, private, no-store, must-revalidate, max-stale=0, post-check=0, pre-check=0";
    }
}
```

**值得注意的是其中`/assets`下的路径需要你当前所使用的主题的名字，这边每次换主题都需要修改，略有一些麻烦吧。**

最后重启ghost以及nginx，现在你的博客就的静态资源已经是通过nginx来处理的了，而且也做了缓存。**（使用了缓存后修改或者添加文章都会有半小时的缓存延时.需要进入到nginx容器中,删除proxy__cache__path (/var/run/cache)下的文件才能够刷新出新的内容.）**

全篇完！
