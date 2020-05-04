---
layout: post 
title: Mac环境搭建RTSP、RTMP及HLS服务器
date: 2016-11-30
tags:
---

> 因为工作项目中需要用到点播、直播等内容，但是服务器端的网速太渣，为了节省等待时间，因此在本地搭建了RTSP、RTMP以及HLS服务器。

## RTSP服务器的搭建

首先感谢强大的VLC播放器，这款播放器不仅仅是个万能解码的播放器，而且还可以充当RTSP点播服务器，用来与人分享音视频。先去[官网](http://www.videolan.org/vlc/)下载安装文件，安装后VLC的路径就是```/Applications/VLC.app```。

下面我们将使用命令行来启动点播服务器，命令如下:
```
/Applications/VLC.app/Contents/MacOS/VLC --ttl 12 -vvv --color -I telnet --telnet-password videolan --rtsp-host 0.0.0.0 --rtsp-port=50055
```
这里面碰到了几个坑，一个是刚开始```--rtsp-host 0.0.0.0:50055```来启动，但是莫名的跳转到544端口，然后需要sudo权限，sudo权限又不能启动，但是拆开成```--rtsp-host 0.0.0.0 --rtsp-port=50055```就可以正常运行。第二个坑就是端口被占用了，此时最好是改另外一个端口，不要用1-1000之内的端口，当然也可以去查找被占用的端口，然后将其kill掉。

启动里面参数我也不太了解，需要深入了解的，可以自己去找资料。
如果一切正常的话，VLC会开启一个telnet，并且监听4212端口。

此时我们需要额外开启一个终端窗口，用来登录telnet，输入命令。
输入命令登录telnet : ```telnet localhost 4212```，
密码就是前面参数```--telnet-password```后面的```videolan```。
登录进入以后，用命令创建一个```new Test vod enabled```创建一个名字叫做```Test```的点播，然后设置输入源```setup Test input /Users/aijun/Documents/Test.mp4```，后面就是你要点播的文件。

这样一个点播源就建立好了，在局域网内用VLC播放器打开```rtsp://192.168.1.103:50055/Test```就可以播放此点播媒体文件了。

看网上教程，不少可以直接用VLC图形界面来建立点播源的， 但是大多都是windows系统下的，我在Mac系统下测试一直没成功，有知道的可以给我留言，不胜感激。

## 搭建RTMP和HLS服务器

网上关于RTMP和HLS服务器的教程有多，毕竟现在直播非常火，不过自己去尝试安装一下，也有不少坑，下面写一下自己的一些步骤。

### 1、安装Homebrow
已经安装的话， 此步可以略过。
```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

### 2、安装Nginx服务器及配置文件

首先下载nginx扩展
```
brew tap homebrew/nginx
```

安装Nginx服务器及rtmp模块

```
brew install nginx-full --with-rtmp-module
```

打开配置文件```/usr/local/etc/nginx/nginx.conf```，修改如下:
```

worker_processes  1;

events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    keepalive_timeout  65;

    server {
        listen       8080;
        server_name  localhost;

        location / {
            root   html;
            index  index.html index.htm;
        }

        #rtmp stat
        location /stat {
          rtmp_stat all;
          rtmp_stat_stylesheet stat.xsl;
        }
        location /stat.xsl {
            root /usr/local/Cellar/rtmp-nginx-module/1.1.7.10/share/rtmp-nginx-module;
        }

        location /control {
          rtmp_control all;
        }

        #HLS配置开始,这个配置为了`客户端`能够以http协议获取HLS的拉流
        location /hls {
            # Serve HLS fragments
            types {
                application/vnd.apple.mpegurl m3u8;
                video/mp2t ts;
            }
            root html;
            add_header Cache-Control no-cache;
        }
       #HLS配置结束

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

    }
    include servers/*;
}

rtmp {
	server {
		listen 1935;
        ping 30s;
        notify_method get;

		application myapp {
			live on;
			record off;
		}

    #增加对HLS支持开始
    application hls {
        live on;
        hls on;
        hls_path /usr/local/var/www/hls;
        hls_fragment 5s;
    }
    #增加对HLS支持结束
	}
}

```
修改后，直接运行```nginx```启动服务器。

> Nginx常用目录
* 配置文件 ```/usr/local/etc/nginx/nginx.conf```
* 安装地址 ```/usr/local/Cellar/nginx-full/1.10.2/bin/nginx```
* 默认根目录 ```/usr/local/var/www```

> Nginx常用操作命令
* 服务器启动   ```nginx```
* 服务器重新加载配置文件	```nginx -s reload ```
* 服务器停止  ```nginx -s stop```
* 服务器退出 ```nginx -s quit```

### 3、安装FFMPEG

```
brew install ffmpeg --with-ffplay
```

### 设置推流

* 设置文件推流
```
ffmpeg -re -i /Users/aijun/Documents/Test.mp4 -vcodec copy -ar 22050 -f flv rtmp://localhost:1935/hls/movie
```
网上不少教程都没有```-ar 22050```的参数，如果你的转码flv也碰到和我一样的问题，加上这个参数就可以解决了。

HLS服务器地址: http://192.168.1.103:8080/hls/movie.m3u8
RTMP服务器地址: rtmp://192.168.1.103/hls/movie
可以使用VLC打开网络地址测试是否运行正常。
** 记住必须先运行命令进行推流， 然后才能打开直播视频流 **
> 其中movie可以改成其他的任何名字，对应的服务器地址的movie也需要做出相应更改

* 设置本地桌面摄像头的推流
```
ffmpeg -f avfoundation -framerate 30 -i "1:0" -f avfoundation -framerate 25 -video_size 640x480 -i "0" -c:v libx264 -preset ultrafast -filter_complex 'overlay=main_w-overlay_w-10:main_h-overlay_h-10' -acodec libmp3lame -ar 44100 -ac 1  -f flv rtmp://localhost:1935/hls/movie
```
使用摄像头的详细命令就不多解释了，不过我用iMac看起来效果一般，毕竟帧数不够高，而且延迟比较大。