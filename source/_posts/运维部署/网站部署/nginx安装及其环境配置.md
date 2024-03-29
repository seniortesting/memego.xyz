---
title: Nginx Linux安装及其环境配置
categories: []
abbrlink: 2Ix82hIx7y
date: 2021-03-25 23:16:38
description:
cover:
top_img:
---

参考：[修改nginx的Server名称,修改两个文件:ngx_http_header_filter_module.c和nginx.h](https://serverfault.com/questions/214242/can-i-hide-all-server-os-info/279389#279389)


## docker setup

```bash
# get the nginx image
$ sudo apt install certbot
$ sudo docker pull nginx
$ sudo docker images
# setup the files
$ sudo mkdir -p /www /var/log/nginx
$ sudo echo "Hello World" | sudo tee /www/index.html  # add -a for append (>>)
# run the test instance
$ sudo docker run --name tmp-nginx-container -d nginx
$ sudo docker ps -a
# copy all the nginx docker config files, nginx docker default config file path in: /etc/nginx/conf.d/default.conf
$ sudo docker cp tmp-nginx-container:/etc/nginx/ /etc/
# delete the default conf files
$ sudo rm -rf /etc/nginx/conf.d
# delete previous image, then rerun the docker instance
$ sudo docker rm -f tmp-nginx-container
$ sudo docker run -d -p 80:80 -v /var/log/nginx:/var/log/nginx -v /etc/nginx/nginx.conf:/etc/nginx/nginx.conf --name nginx nginx 

```

## Nginx安装步骤

有关各个安装类库的版本要求参见：<https://nginx.org/en/docs/configure.html>

```
$ cd /opt
$ sudo mkdir nginx-modules
$ cd nginx-modules

// zlib安装
$ wget http://zlib.net/zlib-1.2.11.tar.gz
$ tar xvf zlib-1.2.11.tar.gz

// ngx-brotli安装
$ apt install git
$ git clone https://github.com/google/ngx_brotli
$ cd ngx_brotli && git submodule update --init --recursive --progress

// pcre安装
$ cd ..
$ wget https://ftp.pcre.org/pub/pcre/pcre-8.43.tar.gz
$ tar xvf pcre-8.43.tar.gz

// OpenSSL安装
$ wget https://www.openssl.org/source/openssl-1.1.1f.tar.gz
$ tar xvf openssl-1.1.1f.tar.gz
$ sudo apt install build-essential -y

// nginx安装
配置可参考： [Compile from source](https://github.com/jukbot/setup-nginx-webserver/blob/master/README.md)
$ wget http://nginx.org/download/nginx-1.18.0.tar.gz
$ tar zxvf nginx-1.18.0.tar.gz
$ cd nginx-1.18.0
$ sudo nano src/http/ngx_http_header_filter_module.c
$ sudo nano src/core/nginx.h
$ sudo nano src/http/ngx_http_special_response.c
$ sudo mkdir /var/lib/nginx

// 运行安装
$ ./configure --prefix=/usr/local/nginx \
            --sbin-path=/usr/sbin/nginx \
            --modules-path=/usr/lib/nginx/modules \
            --conf-path=/etc/nginx/nginx.conf \
            --error-log-path=/var/log/nginx/error.log \
            --http-log-path=/var/log/nginx/access.log \
            --pid-path=/run/nginx.pid \
            --lock-path=/var/lock/nginx.lock \
            --user=www-data \
            --group=www-data \
            --http-client-body-temp-path=/var/lib/nginx/body \
            --http-fastcgi-temp-path=/var/lib/nginx/fastcgi \
            --http-proxy-temp-path=/var/lib/nginx/proxy \
            --http-scgi-temp-path=/var/lib/nginx/scgi \
            --http-uwsgi-temp-path=/var/lib/nginx/uwsgi \
            --with-openssl=../nginx-modules/openssl-1.1.1f \
            --with-openssl-opt=enable-ec_nistp_64_gcc_128 \
            --with-openssl-opt=no-nextprotoneg \
            --with-openssl-opt=no-weak-ssl-ciphers \
            --with-openssl-opt=no-ssl3 \
            --with-pcre=../nginx-modules/pcre-8.43 \
            --with-pcre-jit \
            --with-zlib=../nginx-modules/zlib-1.2.11 \
            --with-compat \
            --with-file-aio \
            --with-threads \
            --with-http_addition_module \
            --with-http_auth_request_module \
            --with-http_dav_module \
            --with-http_flv_module \
            --with-http_mp4_module \
            --with-http_gunzip_module \
            --with-http_gzip_static_module \
            --with-http_random_index_module \
            --with-http_realip_module \
            --with-http_slice_module \
            --with-http_ssl_module \
            --with-http_sub_module \
            --with-http_stub_status_module \
            --with-http_v2_module \
            --with-http_secure_link_module \
            --with-mail \
            --with-mail_ssl_module \
            --with-stream \
            --with-stream_realip_module \
            --with-stream_ssl_module \
            --with-stream_ssl_preread_module \
            --with-debug \
            --with-cc-opt='-g -O2 -fPIE -fstack-protector-strong -Wformat -Werror=format-security -Wdate-time -D_FORTIFY_SOURCE=2' \
            --with-ld-opt='-Wl,-Bsymbolic-functions -fPIE -pie -Wl,-z,relro -Wl,-z,now' \
            --add-module=../nginx-modules/ngx_brotli

$ sudo make & sudo make install  #这个操作需要花费十分钟左右 
```

## 安装nginx.service服务

```shell
$ sudo  nano /lib/systemd/system/nginx.service 
[Unit]
Description=A high performance web server and a reverse proxy server
Documentation=man:nginx(8)
After=network.target

[Service]
Type=forking
PIDFile=/run/nginx.pid
ExecStartPre=/usr/sbin/nginx -t -q -g 'daemon on; master_process on;'
ExecStartPost=/bin/sleep 0.1
ExecStart=/usr/sbin/nginx -g 'daemon on; master_process on;'
ExecReload=/usr/sbin/nginx -g 'daemon on; master_process on;' -s reload
ExecStop=-/sbin/start-stop-daemon --quiet --stop --retry QUIT/5 --pidfile /run/nginx.pid
TimeoutStopSec=5
KillMode=mixed

[Install]
WantedBy=multi-user.target

$ sudo systemctl enable nginx
$ sudo systemctl restart nginx
$ sudo systemctl status nginx

// 删除默认的nginx网站目录
$ sudo rm -rf /usr/local/nginx/*
```

遇到异常的报错信息:

> nginx.service: Failed to read PID from file `/run/nginx.pid`: Invalid argument

That warning with the nginx.pid file is a know bug (at least for Ubutnu if not for other distros as well). More details here: <https://bugs.launchpad.net/ubuntu/+source/nginx/+bug/1581864>

Workaround (on a ssh console, as root, use the commands bellow):

```shell
mkdir /etc/systemd/system/nginx.service.d
printf "[Service]\nExecStartPost=/bin/sleep 0.1\n" > /etc/systemd/system/nginx.service.d/override.conf
systemctl daemon-reload
systemctl restart nginx 
```

# nginx配置助手

1. 安装cerbot配置SSL

```
sudo apt-get install certbot -y
```

2. 配置nginx和SSL

参考配置文件,每次只需要修改对应的域名(Domain)和目录(Path):
 [nginxconfig助手](https://www.digitalocean.com/community/tools/nginx#?0.domain=memego.xyz&0.path=%2Fwww%2Fweb%2Fmemego.xyz&0.document_root=&0.redirect=false&0.email=alterhu2020@gmail.com&0.php=false&0.proxy&0.index=index.html&0.access_log_domain&0.error_log_domain&directory_letsencrypt=%2Fwww%2F_letsencrypt%2F&brotli&log_not_found&symlink=false)

以后每次配置子域名，只需要配置好对应的conf文件然后执行如下命令:

```
certbot certonly --webroot -d res.memego.xyz --email alterhu2020@gmail.com -w /www/_letsencrypt -n --agree-tos --force-renewal
```

::: warning Symlink vhost配置文件
注意: 在`Symlink vhost` 中不要勾选enable。
完成以上的配置后不行看任意修改文件
:::

3. 配置letsencrypt自动续期

参考文档：
 - <https://serverfault.com/questions/790772/cron-job-for-lets-encrypt-renewal>
 - <https://stackoverflow.com/questions/41535546/how-do-i-schedule-the-lets-encrypt-certbot-to-automatically-renew-my-certificat>
   
Let's Encrypt申请的证书会有三个月的有效期，可以到期前手动续约，也可以自己写定时脚本任务自动续约。嫌手动麻烦，就写了个简单的续期脚本。一般安装完后`certbot`也会自动创建一个cronjob文件在目录： `/etc/cron.d/certbot`. 但是需要你执行如下的命令才能生效，默认是每天执行一次检查。

Note: in 18.04 LTS the `letsencrypt` package has been (finally) renamed to `certbot`. It now includes a `systemd` timer which you can enable to schedule certbot renewals, with `systemctl enable certbot.timer` and `systemctl start certbot.timer`. However, Ubuntu did not provide a way to specify hooks. You'll need to set up an override for `certbot.service` to override ExecStart= with your desired command line, until Canonical fixes this.


3.1 python脚本版：

```bash
$ echo "0 0,12 * * * root python -c 'import random; import time; time.sleep(random.random() * 3600)' && certbot renew --force-renew" | sudo tee -a /etc/crontab > /dev/null

```
3.2 （推荐）直接使用上述的配置文件`/etc/cron.d/certbot`

```bash
$ nano /etc/cron.d/certbot
$ systemctl enable certbot.timer
$ systemctl start certbot.timer
```
检查是否设置成功:
```bash
$ systemctl list-timers (or systemctl list-timers --all if you also want to show inactive timers).
```

The certbot timer should be here `/lib/systemd/system/certbot.timer` and it will execute the command specified in `/lib/systemd/system/certbot.service`


## 配置静态域名：res.memego.xyz

对于静态资源展示，提供了几种方式配置：

- 简易配置Github风格：（推荐）[autoindex.html - 一行配置美化 nginx 目录成 github 风格。](https://www.91yunbbs.com/discussion/441/autoindex-html-%E4%B8%80%E8%A1%8C%E9%85%8D%E7%BD%AE%E7%BE%8E%E5%8C%96-nginx-%E7%9B%AE%E5%BD%95%E6%88%90-github-%E9%A3%8E%E6%A0%BC)

- 复杂配置，比较好看 [ngx-fancyindex](https://www.cnblogs.com/linkenpark/p/9184491.html)

```
# 代理配置
location / {
  proxy_pass http://127.0.0.1:8080;
  include nginxconfig.io/proxy.conf;
}
# 静态资源配置
location / {
  autoindex on; #开启目录浏览
  autoindex_format html; #以html风格将目录展示在浏览器中
  autoindex_exact_size off; #切换为 off 后，以可读的方式显示文件大小，单位为 KB、MB 或者 GB
  autoindex_localtime on; #以服务器的文件时间作为显示的时间
  charset utf-8,gbk; #展示中文文件名
  # 美化显示
  add_after_body /autoindex.html;
  # try_files $uri $uri/ index.html;
}

```

其中的`autoindex.html`内容如下:

```

<!-- autoindex.html 19.08, see https://phuslu.github.io -->
<script>
!function(){
 var website_title = ''
 var datetime_format = '%d-%b-%Y %H:%M'
 var show_readme_md = true
 var max_name_length = 50

 var dom = {
  element: null,
  get: function (o) {
   var obj = Object.create(this)
   obj.element = (typeof o == "object") ? o : document.createElement(o)
   return obj
  },
  add: function (o) {
   var obj = dom.get(o)
   this.element.appendChild(obj.element)
   return obj
  },
  text: function (t) {
   this.element.appendChild(document.createTextNode(t))
   return this
  },
  html: function (s) {
   this.element.innerHTML = s
   return this
  },
  attr: function (k, v) {
   this.element.setAttribute(k, v)
   return this
  }
 }

 head = dom.get(document.head)
 head.add('meta').attr('charset', 'utf-8')
 head.add('meta').attr('name', 'viewport').attr('content', 'width=device-width,initial-scale=1')

 if (!document.title) {
  document.write(["<div class=\"container\">",
  "<h3>nginx.conf</h3>",
  "<textarea rows=8 cols=50>",
  "# download autoindex.html to /wwwroot/",
  "location ~ ^(.*)/$ {",
  "    charset utf-8;",
  "    autoindex on;",
  "    autoindex_localtime on;",
  "    autoindex_exact_size off;",
  "    add_after_body /autoindex.html;",
  "}",
  "</textarea>",
  "</div>"].join("\n"))
  return
 }

 var bodylines = document.body.innerHTML.split('\n')
 document.body.innerHTML = ''

 var titlehtml = document.title.replace(/\/$/, '').split('/').slice(1).reduce(function(acc, v, i, a) {
  return acc + '<a href="/' + a.slice(0, i+1).join('/') + '/">' + v + '</a>/'
 }, 'Index of /')
 if (website_title) {
  document.title = website_title + ' - ' + document.title
 }

 div = dom.get('div').attr('class', 'container')
 div.add('table').add('tbody').add('tr').add('th').html(titlehtml)
 tbody = div.add('table').attr('class', 'table-hover').add('tbody')

 names = ['Name', 'Date', 'Size']
 thead = tbody.add('tr')
 for (i = 0; i < names.length; i++)
  thead.add('td').add('a').attr('href', 'javascript:sortby('+i+')').attr('class', 'octicon arrow-up').text(names[i]);

 var insert = function(filename, datetime, size) {
  if (/\/$/.test(filename)) {
   css = 'file-directory'
   size = ''
  } else if (/\.(zip|7z|bz2|gz|tar|tgz|tbz2|xz|cab)$/.test(filename)) {
   css = 'file-zip'
  } else if (/\.(py|js|php|pl|rb|sh|bash|lua|sql|go|rs|java|c|h|cpp|cxx|hpp)$/.test(filename)) {
   css = 'file-code'
  } else if (/\.(jpg|png|bmp|gif|ico|webp)$/.test(filename)) {
   css = 'file-media'
  } else if (/\.(flv|mp4|mkv|avi|mkv|vp9)$/.test(filename)) {
   css = 'device-camera-video'
  } else {
   css = 'file'
  }

  displayname = decodeURIComponent(filename.replace(/\/$/, ''))
  if (displayname.length > max_name_length)
   displayname = displayname.substring(0, max_name_length-3) + '..>';

  if (!isNaN(Date.parse(datetime))) {
   d = new Date(datetime)
   pad = function (s) {return s < 10 ? '0' + s : s}
   mon = function (m) {return ['Jan','Feb','Mar','Apr','May','Jun','Jul','Aug','Sep','Oct','Nov','Dec'][m]}
   datetime = datetime_format
     .replace('%Y', d.getFullYear())
     .replace('%m', pad(d.getMonth()+1))
     .replace('%d', pad(d.getDate()))
     .replace('%H', pad(d.getHours()))
     .replace('%M', pad(d.getMinutes()))
     .replace('%S', pad(d.getSeconds()))
     .replace('%b', mon(d.getMonth()))
  }

  tr = tbody.add('tr')
  tr.add('td').add('a').attr('class', 'octicon ' + css).attr('href', filename).text(displayname)
  tr.add('td').text(datetime)
  tr.add('td').text(size)
 }

 var readme = ''
 insert('../', '', '-')
 for (var i in bodylines) {
  if (m = /\s*<a href="(.+?)">(.+?)<\/a>\s+(\S+)\s+(\S+)\s+(\S+)\s*/.exec(bodylines[i])) {
   filename = m[1]
   datetime = m[3] + ' ' + m[4]
   size = m[5]
   insert(filename, datetime, size)
   if (filename.toLowerCase() == 'readme.md') {
    readme = filename
   }
  }
 }

 document.body.appendChild(div.element)

 if (show_readme_md && readme !== '') {
  tbody = div.add('table').add('tbody');
  tbody.add('tr').add('th').attr('class', 'octicon octicon-book').text(readme)
  tbody.add('tr').add('td').add('div').attr('id', 'readme').attr('class', 'markdown-body')
  var xhr = new XMLHttpRequest()
  xhr.open('GET', location.pathname.replace(/[^/]+$/, '')+readme, true)
  xhr.onload = function() {
   if (xhr.status >= 200 && xhr.status < 400) {
    wait = function (name, callback) {
     var interval = 10; // ms
     window.setTimeout(function() {
      if (window[name]) {
       callback(window[name])
      } else {
       window.setTimeout(arguments.callee, interval)
      }
     }, interval)
    }
    wait('marked', function() {
     document.getElementById("readme").innerHTML = marked(xhr.responseText)
    })
   }
  }
  xhr.send()

  div.add('script').attr('src', 'https://cdn.staticfile.org/marked/0.7.0/marked.min.js')
  div.add('link').attr('rel', 'stylesheet').attr('href', 'https://cdn.staticfile.org/github-markdown-css/3.0.1/github-markdown.min.css')
 }
}()

function sortby(index) {
 rows = document.getElementsByClassName('table-hover')[0].rows
 link = rows[0].getElementsByTagName('a')[index]
 arrow = link.className == 'octicon arrow-down' ? 1 : -1
 link.className = 'octicon arrow-' + (arrow == 1 ? 'up' : 'down');
 [].slice.call(rows).slice(2).map(function (e, i) {
  type = e.getElementsByTagName('a')[0].className == 'octicon file-directory' ? 0 : 1
  text = e.getElementsByTagName('td')[index].innerText
  if (index === 0) {
   value = text
  } else if (index === 1) {
   value = new Date(text).getTime()
  } else if (index === 2) {
   m = {'G':1024*1024*1024, 'M':1024*1024, 'K':1024}
   value = parseInt(text || 0) * (m[text[text.search(/[KMG]B?$/)]] || 1)
  }
  return {type: type, value: value, index: i, html: e.innerHTML}
 }).sort(function (a, b) {
  if (a.type != b.type)
   return a.type - b.type
  if (a.value != b.value)
   return a.value < b.value ? -arrow : arrow
  return a.index < b.index ? -arrow : arrow
 }).forEach(function (e, i) {
  rows[2+i].innerHTML = e.html
 })
}
</script>

<style>
body {
 margin: 0;
 font-family: "ubuntu", "Tahoma", "Microsoft YaHei", Arial, Serif;
}
.container {
 padding-right: 15px;
 padding-left: 15px;
 margin-right: auto;
 margin-left: auto;
}
@media (min-width: 768px) {
 .container {
  max-width: 750px;
 }
}
@media (min-width: 992px) {
 .container {
  max-width: 970px;
 }
}
@media (min-width: 1200px) {
 .container {
  max-width: 1170px;
 }
}
table {
 width: 100%;
 max-width: 100%;
 margin-bottom: 20px;
 border: 1px solid #ddd;
 padding: 0;
 border-collapse: collapse;
}
table th {
 font-size: 14px;
}
table tr {
 border: 1px solid #ddd;
 padding: 5px;
}
table tr:nth-child(odd) {
 background: #f9f9f9
}
table th, table td {
 border: 1px solid #ddd;
 font-size: 14px;
 line-height: 20px;
 padding: 3px;
 text-align: left;
}
a {
 color: #337ab7;
 text-decoration: none;
}
a:hover, a:focus {
 color: #2a6496;
 text-decoration: underline;
}
table.table-hover > tbody > tr:hover > td,
table.table-hover > tbody > tr:hover > th {
 background-color: #f5f5f5;
}
.markdown-body {
 float: left;
 font-family: "ubuntu", "Tahoma", "Microsoft YaHei", Arial, Serif;
}
/* octicons */
.octicon {
 background-position: center left;
 background-repeat: no-repeat;
 padding-left: 16px;
}
.file {
 background-image: url("data:image/svg+xml;charset=utf8,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='16' viewBox='0 0 12 16'%3E%3Cpath d='M6 5L2 5 2 4 6 4 6 5 6 5ZM2 8L9 8 9 7 2 7 2 8 2 8ZM2 10L9 10 9 9 2 9 2 10 2 10ZM2 12L9 12 9 11 2 11 2 12 2 12ZM12 4.5L12 14C12 14.6 11.6 15 11 15L1 15C0.5 15 0 14.6 0 14L0 2C0 1.5 0.5 1 1 1L8.5 1 12 4.5 12 4.5ZM11 5L8 2 1 2 1 14 11 14 11 5 11 5Z' fill='%237D94AE'/%3E%3C/svg%3E");
}
.file-directory {
 background-image: url("data:image/svg+xml;charset=utf8,%3Csvg xmlns='http://www.w3.org/2000/svg' width='14' height='16' viewBox='0 0 14 16'%3E%3Cpath d='M13 4L7 4 7 3C7 2.3 6.7 2 6 2L1 2C0.5 2 0 2.5 0 3L0 13C0 13.6 0.5 14 1 14L13 14C13.6 14 14 13.6 14 13L14 5C14 4.5 13.6 4 13 4L13 4ZM6 4L1 4 1 3 6 3 6 4 6 4Z' fill='%237D94AE'/%3E%3C/svg%3E");
}
.file-zip {
 background-image: url("data:image/svg+xml;charset=utf8,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='16' viewBox='0 0 12 16'%3E%3Cpath d='M8.5 1L1 1C0.4 1 0 1.4 0 2L0 14C0 14.6 0.4 15 1 15L11 15C11.6 15 12 14.6 12 14L12 4.5 8.5 1ZM11 14L1 14 1 2 4 2 4 3 5 3 5 2 8 2 11 5 11 14 11 14ZM5 4L5 3 6 3 6 4 5 4 5 4ZM4 4L5 4 5 5 4 5 4 4 4 4ZM5 6L5 5 6 5 6 6 5 6 5 6ZM4 6L5 6 5 7 4 7 4 6 4 6ZM5 8L5 7 6 7 6 8 5 8 5 8ZM4 9.3C3.4 9.6 3 10.3 3 11L3 12 7 12 7 11C7 9.9 6.1 9 5 9L5 8 4 8 4 9.3 4 9.3ZM6 10L6 11 4 11 4 10 6 10 6 10Z' fill='%237D94AE'/%3E%3C/svg%3E");
}
.file-code {
 background-image: url("data:image/svg+xml;charset=utf8,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='16' viewBox='0 0 12 16'%3E%3Cpath d='M8.5,1 L1,1 C0.45,1 0,1.45 0,2 L0,14 C0,14.55 0.45,15 1,15 L11,15 C11.55,15 12,14.55 12,14 L12,4.5 L8.5,1 L8.5,1 Z M11,14 L1,14 L1,2 L8,2 L11,5 L11,14 L11,14 Z M5,6.98 L3.5,8.5 L5,10 L4.5,11 L2,8.5 L4.5,6 L5,6.98 L5,6.98 Z M7.5,6 L10,8.5 L7.5,11 L7,10.02 L8.5,8.5 L7,7 L7.5,6 L7.5,6 Z' fill='%237D94AE' /%3E%3C/svg%3E");
}
.file-media {
 background-image: url("data:image/svg+xml;charset=utf8,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='16' viewBox='0 0 12 16'%3E%3Cpath d='M6 5L8 5 8 7 6 7 6 5 6 5ZM12 4.5L12 14C12 14.6 11.6 15 11 15L1 15C0.5 15 0 14.6 0 14L0 2C0 1.5 0.5 1 1 1L8.5 1 12 4.5 12 4.5ZM11 5L8 2 1 2 1 13 4 8 6 12 8 10 11 13 11 5 11 5Z' fill='%237D94AE'/%3E%3C/svg%3E");
}
.device-camera-video {
 background-image: url("data:image/svg+xml;charset=utf8,%3Csvg xmlns='http://www.w3.org/2000/svg' width='16' height='16' viewBox='0 0 16 16'%3E%3Cpath d='M15.2,2.09 L10,5.72 L10,3 C10,2.45 9.55,2 9,2 L1,2 C0.45,2 0,2.45 0,3 L0,12 C0,12.55 0.45,13 1,13 L9,13 C9.55,13 10,12.55 10,12 L10,9.28 L15.2,12.91 C15.53,13.14 16,12.91 16,12.5 L16,2.5 C16,2.09 15.53,1.86 15.2,2.09 L15.2,2.09 Z' fill='%237D94AE' /%3E%3C/svg%3E");
}
.octicon-book {
 padding-left: 20px;
 background-image: url("data:image/svg+xml;charset=utf8,%3Csvg xmlns='http://www.w3.org/2000/svg' width='16' height='16' viewBox='0 0 16 16'%3E%3Cpath d='M3,5 L7,5 L7,6 L3,6 L3,5 L3,5 Z M3,8 L7,8 L7,7 L3,7 L3,8 L3,8 Z M3,10 L7,10 L7,9 L3,9 L3,10 L3,10 Z M14,5 L10,5 L10,6 L14,6 L14,5 L14,5 Z M14,7 L10,7 L10,8 L14,8 L14,7 L14,7 Z M14,9 L10,9 L10,10 L14,10 L14,9 L14,9 Z M16,3 L16,12 C16,12.55 15.55,13 15,13 L9.5,13 L8.5,14 L7.5,13 L2,13 C1.45,13 1,12.55 1,12 L1,3 C1,2.45 1.45,2 2,2 L7.5,2 L8.5,3 L9.5,2 L15,2 C15.55,2 16,2.45 16,3 L16,3 Z M8,3.5 L7.5,3 L2,3 L2,12 L8,12 L8,3.5 L8,3.5 Z M15,3 L9.5,3 L9,3.5 L9,12 L15,12 L15,3 L15,3 Z' /%3E%3C/svg%3E");
}
.arrow-down {
 font-weight: bold;
 text-decoration: none !important;
 background-image: url("data:image/svg+xml;charset=utf8,%3Csvg xmlns='http://www.w3.org/2000/svg' width='10' height='16' viewBox='0 0 10 16'%3E%3Cpolygon id='Shape' points='7 7 7 3 3 3 3 7 0 7 5 13 10 7'%3E%3C/polygon%3E%3C/svg%3E");
}
.arrow-up {
 font-weight: bold;
 text-decoration: none !important;
 background-image: url("data:image/svg+xml;charset=utf8,%3Csvg xmlns='http://www.w3.org/2000/svg' width='10' height='16' viewBox='0 0 10 16'%3E%3Cpolygon id='Shape' points='5 3 0 9 3 9 3 13 7 13 7 9 10 9'%3E%3C/polygon%3E%3C/svg%3E");
}
</style>
<!-- autoindex.html 19.08, see https://phuslu.github.io -->
<script>
!function(){
 var website_title = ''
 var datetime_format = '%d-%b-%Y %H:%M'
 var show_readme_md = true
 var max_name_length = 50

 var dom = {
  element: null,
  get: function (o) {
   var obj = Object.create(this)
   obj.element = (typeof o == "object") ? o : document.createElement(o)
   return obj
  },
  add: function (o) {
   var obj = dom.get(o)
   this.element.appendChild(obj.element)
   return obj
  },
  text: function (t) {
   this.element.appendChild(document.createTextNode(t))
   return this
  },
  html: function (s) {
   this.element.innerHTML = s
   return this
  },
  attr: function (k, v) {
   this.element.setAttribute(k, v)
   return this
  }
 }

 head = dom.get(document.head)
 head.add('meta').attr('charset', 'utf-8')
 head.add('meta').attr('name', 'viewport').attr('content', 'width=device-width,initial-scale=1')

 if (!document.title) {
  document.write(["<div class=\"container\">",
  "<h3>nginx.conf</h3>",
  "<textarea rows=8 cols=50>",
  "# download autoindex.html to /wwwroot/",
  "location ~ ^(.*)/$ {",
  "    charset utf-8;",
  "    autoindex on;",
  "    autoindex_localtime on;",
  "    autoindex_exact_size off;",
  "    add_after_body /autoindex.html;",
  "}",
  "</textarea>",
  "</div>"].join("\n"))
  return
 }

 var bodylines = document.body.innerHTML.split('\n')
 document.body.innerHTML = ''

 var titlehtml = document.title.replace(/\/$/, '').split('/').slice(1).reduce(function(acc, v, i, a) {
  return acc + '<a href="/' + a.slice(0, i+1).join('/') + '/">' + v + '</a>/'
 }, 'Index of /')
 if (website_title) {
  document.title = website_title + ' - ' + document.title
 }

 div = dom.get('div').attr('class', 'container')
 div.add('table').add('tbody').add('tr').add('th').html(titlehtml)
 tbody = div.add('table').attr('class', 'table-hover').add('tbody')

 names = ['Name', 'Date', 'Size']
 thead = tbody.add('tr')
 for (i = 0; i < names.length; i++)
  thead.add('td').add('a').attr('href', 'javascript:sortby('+i+')').attr('class', 'octicon arrow-up').text(names[i]);

 var insert = function(filename, datetime, size) {
  if (/\/$/.test(filename)) {
   css = 'file-directory'
   size = ''
  } else if (/\.(zip|7z|bz2|gz|tar|tgz|tbz2|xz|cab)$/.test(filename)) {
   css = 'file-zip'
  } else if (/\.(py|js|php|pl|rb|sh|bash|lua|sql|go|rs|java|c|h|cpp|cxx|hpp)$/.test(filename)) {
   css = 'file-code'
  } else if (/\.(jpg|png|bmp|gif|ico|webp)$/.test(filename)) {
   css = 'file-media'
  } else if (/\.(flv|mp4|mkv|avi|mkv|vp9)$/.test(filename)) {
   css = 'device-camera-video'
  } else {
   css = 'file'
  }

  displayname = decodeURIComponent(filename.replace(/\/$/, ''))
  if (displayname.length > max_name_length)
   displayname = displayname.substring(0, max_name_length-3) + '..>';

  if (!isNaN(Date.parse(datetime))) {
   d = new Date(datetime)
   pad = function (s) {return s < 10 ? '0' + s : s}
   mon = function (m) {return ['Jan','Feb','Mar','Apr','May','Jun','Jul','Aug','Sep','Oct','Nov','Dec'][m]}
   datetime = datetime_format
     .replace('%Y', d.getFullYear())
     .replace('%m', pad(d.getMonth()+1))
     .replace('%d', pad(d.getDate()))
     .replace('%H', pad(d.getHours()))
     .replace('%M', pad(d.getMinutes()))
     .replace('%S', pad(d.getSeconds()))
     .replace('%b', mon(d.getMonth()))
  }

  tr = tbody.add('tr')
  tr.add('td').add('a').attr('class', 'octicon ' + css).attr('href', filename).text(displayname)
  tr.add('td').text(datetime)
  tr.add('td').text(size)
 }

 var readme = ''
 insert('../', '', '-')
 for (var i in bodylines) {
  if (m = /\s*<a href="(.+?)">(.+?)<\/a>\s+(\S+)\s+(\S+)\s+(\S+)\s*/.exec(bodylines[i])) {
   filename = m[1]
   datetime = m[3] + ' ' + m[4]
   size = m[5]
   insert(filename, datetime, size)
   if (filename.toLowerCase() == 'readme.md') {
    readme = filename
   }
  }
 }

 document.body.appendChild(div.element)

 if (show_readme_md && readme !== '') {
  tbody = div.add('table').add('tbody');
  tbody.add('tr').add('th').attr('class', 'octicon octicon-book').text(readme)
  tbody.add('tr').add('td').add('div').attr('id', 'readme').attr('class', 'markdown-body')
  var xhr = new XMLHttpRequest()
  xhr.open('GET', location.pathname.replace(/[^/]+$/, '')+readme, true)
  xhr.onload = function() {
   if (xhr.status >= 200 && xhr.status < 400) {
    wait = function (name, callback) {
     var interval = 10; // ms
     window.setTimeout(function() {
      if (window[name]) {
       callback(window[name])
      } else {
       window.setTimeout(arguments.callee, interval)
      }
     }, interval)
    }
    wait('marked', function() {
     document.getElementById("readme").innerHTML = marked(xhr.responseText)
    })
   }
  }
  xhr.send()

  div.add('script').attr('src', 'https://cdn.staticfile.org/marked/0.7.0/marked.min.js')
  div.add('link').attr('rel', 'stylesheet').attr('href', 'https://cdn.staticfile.org/github-markdown-css/3.0.1/github-markdown.min.css')
 }
}()

function sortby(index) {
 rows = document.getElementsByClassName('table-hover')[0].rows
 link = rows[0].getElementsByTagName('a')[index]
 arrow = link.className == 'octicon arrow-down' ? 1 : -1
 link.className = 'octicon arrow-' + (arrow == 1 ? 'up' : 'down');
 [].slice.call(rows).slice(2).map(function (e, i) {
  type = e.getElementsByTagName('a')[0].className == 'octicon file-directory' ? 0 : 1
  text = e.getElementsByTagName('td')[index].innerText
  if (index === 0) {
   value = text
  } else if (index === 1) {
   value = new Date(text).getTime()
  } else if (index === 2) {
   m = {'G':1024*1024*1024, 'M':1024*1024, 'K':1024}
   value = parseInt(text || 0) * (m[text[text.search(/[KMG]B?$/)]] || 1)
  }
  return {type: type, value: value, index: i, html: e.innerHTML}
 }).sort(function (a, b) {
  if (a.type != b.type)
   return a.type - b.type
  if (a.value != b.value)
   return a.value < b.value ? -arrow : arrow
  return a.index < b.index ? -arrow : arrow
 }).forEach(function (e, i) {
  rows[2+i].innerHTML = e.html
 })
}
</script>

<style>
body {
 margin: 0;
 font-family: "ubuntu", "Tahoma", "Microsoft YaHei", Arial, Serif;
}
.container {
 padding-right: 15px;
 padding-left: 15px;
 margin-right: auto;
 margin-left: auto;
}
@media (min-width: 768px) {
 .container {
  max-width: 750px;
 }
}
@media (min-width: 992px) {
 .container {
  max-width: 970px;
 }
}
@media (min-width: 1200px) {
 .container {
  max-width: 1170px;
 }
}
table {
 width: 100%;
 max-width: 100%;
 margin-bottom: 20px;
 border: 1px solid #ddd;
 padding: 0;
 border-collapse: collapse;
}
table th {
 font-size: 14px;
}
table tr {
 border: 1px solid #ddd;
 padding: 5px;
}
table tr:nth-child(odd) {
 background: #f9f9f9
}
table th, table td {
 border: 1px solid #ddd;
 font-size: 14px;
 line-height: 20px;
 padding: 3px;
 text-align: left;
}
a {
 color: #337ab7;
 text-decoration: none;
}
a:hover, a:focus {
 color: #2a6496;
 text-decoration: underline;
}
table.table-hover > tbody > tr:hover > td,
table.table-hover > tbody > tr:hover > th {
 background-color: #f5f5f5;
}
.markdown-body {
 float: left;
 font-family: "ubuntu", "Tahoma", "Microsoft YaHei", Arial, Serif;
}
/* octicons */
.octicon {
 background-position: center left;
 background-repeat: no-repeat;
 padding-left: 16px;
}
.file {
 background-image: url("data:image/svg+xml;charset=utf8,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='16' viewBox='0 0 12 16'%3E%3Cpath d='M6 5L2 5 2 4 6 4 6 5 6 5ZM2 8L9 8 9 7 2 7 2 8 2 8ZM2 10L9 10 9 9 2 9 2 10 2 10ZM2 12L9 12 9 11 2 11 2 12 2 12ZM12 4.5L12 14C12 14.6 11.6 15 11 15L1 15C0.5 15 0 14.6 0 14L0 2C0 1.5 0.5 1 1 1L8.5 1 12 4.5 12 4.5ZM11 5L8 2 1 2 1 14 11 14 11 5 11 5Z' fill='%237D94AE'/%3E%3C/svg%3E");
}
.file-directory {
 background-image: url("data:image/svg+xml;charset=utf8,%3Csvg xmlns='http://www.w3.org/2000/svg' width='14' height='16' viewBox='0 0 14 16'%3E%3Cpath d='M13 4L7 4 7 3C7 2.3 6.7 2 6 2L1 2C0.5 2 0 2.5 0 3L0 13C0 13.6 0.5 14 1 14L13 14C13.6 14 14 13.6 14 13L14 5C14 4.5 13.6 4 13 4L13 4ZM6 4L1 4 1 3 6 3 6 4 6 4Z' fill='%237D94AE'/%3E%3C/svg%3E");
}
.file-zip {
 background-image: url("data:image/svg+xml;charset=utf8,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='16' viewBox='0 0 12 16'%3E%3Cpath d='M8.5 1L1 1C0.4 1 0 1.4 0 2L0 14C0 14.6 0.4 15 1 15L11 15C11.6 15 12 14.6 12 14L12 4.5 8.5 1ZM11 14L1 14 1 2 4 2 4 3 5 3 5 2 8 2 11 5 11 14 11 14ZM5 4L5 3 6 3 6 4 5 4 5 4ZM4 4L5 4 5 5 4 5 4 4 4 4ZM5 6L5 5 6 5 6 6 5 6 5 6ZM4 6L5 6 5 7 4 7 4 6 4 6ZM5 8L5 7 6 7 6 8 5 8 5 8ZM4 9.3C3.4 9.6 3 10.3 3 11L3 12 7 12 7 11C7 9.9 6.1 9 5 9L5 8 4 8 4 9.3 4 9.3ZM6 10L6 11 4 11 4 10 6 10 6 10Z' fill='%237D94AE'/%3E%3C/svg%3E");
}
.file-code {
 background-image: url("data:image/svg+xml;charset=utf8,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='16' viewBox='0 0 12 16'%3E%3Cpath d='M8.5,1 L1,1 C0.45,1 0,1.45 0,2 L0,14 C0,14.55 0.45,15 1,15 L11,15 C11.55,15 12,14.55 12,14 L12,4.5 L8.5,1 L8.5,1 Z M11,14 L1,14 L1,2 L8,2 L11,5 L11,14 L11,14 Z M5,6.98 L3.5,8.5 L5,10 L4.5,11 L2,8.5 L4.5,6 L5,6.98 L5,6.98 Z M7.5,6 L10,8.5 L7.5,11 L7,10.02 L8.5,8.5 L7,7 L7.5,6 L7.5,6 Z' fill='%237D94AE' /%3E%3C/svg%3E");
}
.file-media {
 background-image: url("data:image/svg+xml;charset=utf8,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='16' viewBox='0 0 12 16'%3E%3Cpath d='M6 5L8 5 8 7 6 7 6 5 6 5ZM12 4.5L12 14C12 14.6 11.6 15 11 15L1 15C0.5 15 0 14.6 0 14L0 2C0 1.5 0.5 1 1 1L8.5 1 12 4.5 12 4.5ZM11 5L8 2 1 2 1 13 4 8 6 12 8 10 11 13 11 5 11 5Z' fill='%237D94AE'/%3E%3C/svg%3E");
}
.device-camera-video {
 background-image: url("data:image/svg+xml;charset=utf8,%3Csvg xmlns='http://www.w3.org/2000/svg' width='16' height='16' viewBox='0 0 16 16'%3E%3Cpath d='M15.2,2.09 L10,5.72 L10,3 C10,2.45 9.55,2 9,2 L1,2 C0.45,2 0,2.45 0,3 L0,12 C0,12.55 0.45,13 1,13 L9,13 C9.55,13 10,12.55 10,12 L10,9.28 L15.2,12.91 C15.53,13.14 16,12.91 16,12.5 L16,2.5 C16,2.09 15.53,1.86 15.2,2.09 L15.2,2.09 Z' fill='%237D94AE' /%3E%3C/svg%3E");
}
.octicon-book {
 padding-left: 20px;
 background-image: url("data:image/svg+xml;charset=utf8,%3Csvg xmlns='http://www.w3.org/2000/svg' width='16' height='16' viewBox='0 0 16 16'%3E%3Cpath d='M3,5 L7,5 L7,6 L3,6 L3,5 L3,5 Z M3,8 L7,8 L7,7 L3,7 L3,8 L3,8 Z M3,10 L7,10 L7,9 L3,9 L3,10 L3,10 Z M14,5 L10,5 L10,6 L14,6 L14,5 L14,5 Z M14,7 L10,7 L10,8 L14,8 L14,7 L14,7 Z M14,9 L10,9 L10,10 L14,10 L14,9 L14,9 Z M16,3 L16,12 C16,12.55 15.55,13 15,13 L9.5,13 L8.5,14 L7.5,13 L2,13 C1.45,13 1,12.55 1,12 L1,3 C1,2.45 1.45,2 2,2 L7.5,2 L8.5,3 L9.5,2 L15,2 C15.55,2 16,2.45 16,3 L16,3 Z M8,3.5 L7.5,3 L2,3 L2,12 L8,12 L8,3.5 L8,3.5 Z M15,3 L9.5,3 L9,3.5 L9,12 L15,12 L15,3 L15,3 Z' /%3E%3C/svg%3E");
}
.arrow-down {
 font-weight: bold;
 text-decoration: none !important;
 background-image: url("data:image/svg+xml;charset=utf8,%3Csvg xmlns='http://www.w3.org/2000/svg' width='10' height='16' viewBox='0 0 10 16'%3E%3Cpolygon id='Shape' points='7 7 7 3 3 3 3 7 0 7 5 13 10 7'%3E%3C/polygon%3E%3C/svg%3E");
}
.arrow-up {
 font-weight: bold;
 text-decoration: none !important;
 background-image: url("data:image/svg+xml;charset=utf8,%3Csvg xmlns='http://www.w3.org/2000/svg' width='10' height='16' viewBox='0 0 10 16'%3E%3Cpolygon id='Shape' points='5 3 0 9 3 9 3 13 7 13 7 9 10 9'%3E%3C/polygon%3E%3C/svg%3E");
}
</style>

```

## 配置501,404默认页面

修改文件: `/etc/nginx/nginx.conf`,参考链接: [One NGINX error page to rule them all](https://blog.adriaan.io/one-nginx-error-page-to-rule-them-all.html)

```

http {
     # error pages
 map $status $status_text {
  400 'Bad Request';
  401 'Unauthorized';
  402 'Payment Required';
  403 'Forbidden';
  404 'Not Found';
  405 'Method Not Allowed';
  406 'Not Acceptable';
  407 'Proxy Authentication Required';
  408 'Request Timeout';
  409 'Conflict';
  410 'Gone';
  411 'Length Required';
  412 'Precondition Failed';
  413 'Payload Too Large';
  414 'URI Too Long';
  415 'Unsupported Media Type';
  416 'Range Not Satisfiable';
  417 'Expectation Failed';
  418 'I\'m a teapot';
  421 'Misdirected Request';
  422 'Unprocessable Entity';
  423 'Locked';
  424 'Failed Dependency';
  426 'Upgrade Required';
  428 'Precondition Required';
  429 'Too Many Requests';
  431 'Request Header Fields Too Large';
  451 'Unavailable For Legal Reasons';
  500 'Internal Server Error';
  501 'Not Implemented';
  502 'Bad Gateway';
  503 'Service Unavailable';
  504 'Gateway Timeout';
  505 'HTTP Version Not Supported';
  506 'Variant Also Negotiates';
  507 'Insufficient Storage';
  508 'Loop Detected';
  510 'Not Extended';
  511 'Network Authentication Required';
  default 'Oppos, Something is wrong';
    }
}

```

in the `/etc/nginx/nginxconfig.io/general.conf` file, put code as following also create file: `error.html` in directory: `/www/_error`:

```
error_page 400 401 402 403 404 405 406 407 408 409 410 411 412 413 414 415 416 417 418 421 422 423 424 426 428 429 431 451 500 501 502 503 504 505 506 507 508 510 511 /error.html;

location = /error.html {
  ssi on;
  internal;
  root /www/_error;
}

```

## 配置内网映射

### frps 服务端配置脚本 `frps.ini`

```
[common]
bind_port = 7000

# 配置frps面板
dashboard_port = 7500
dashboard_user = alterhu2020
dashboard_pwd = guchan1026
 
```

## 文档备份

备份文档脚本：

```
 sudo tar -zcvf backup-20200420.tar.gz /etc/profile /www/ /etc/nginx/ /etc/letsencrypt/ /etc/mysql/mysql.conf.d/mysqld.cnf /etc/redis/redis.conf

```
