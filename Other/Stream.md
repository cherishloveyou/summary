## RTSP服务器搭建

`live555 Media Server`(Mac) VLC(iOS Client)

#### 步骤

1. [`live555 Media Server`](https://link.jianshu.com?t=http://www.live555.com/)下载

2. 变成可执行文件

   ```undefined
   chmod 777 live555MediaServer
   ```

3. 创建个文件夹，把`live555MediaServer`放进去，执行命令，看到一堆信息

    ```
    Live55Server ./live555MediaServer 
    ```

    ```
    LIVE555 Media Server
        version 0.89 (LIVE555 Streaming Media library version 2016.05.16).
    Play streams from this server using the URL
        rtsp://192.168.1.100:8554/<filename>
    where <filename> is a file present in the current directory.
    Each file's type is inferred from its name suffix:
        ".264" => a H.264 Video Elementary Stream file
        ".265" => a H.265 Video Elementary Stream file
        ".aac" => an AAC Audio (ADTS format) file
        ".ac3" => an AC-3 Audio file
        ".amr" => an AMR Audio file
        ".dv" => a DV Video file
        ".m4e" => a MPEG-4 Video Elementary Stream file
        ".mkv" => a Matroska audio+video+(optional)subtitles file
        ".mp3" => a MPEG-1 or 2 Audio file
        ".mpg" => a MPEG-1 or 2 Program Stream (audio+video) file
        ".ogg" or ".ogv" or ".opus" => an Ogg audio and/or video file
        ".ts" => a MPEG Transport Stream file
            (a ".tsx" index file - if present - provides server 'trick play' support)
        ".vob" => a VOB (MPEG-2 video with AC-3 audio) file
        ".wav" => a WAV Audio file
        ".webm" => a WebM audio(Vorbis)+video(VP8) file
    See http://www.live555.com/mediaServer/ for additional documentation.
    (We use port 8000 for optional RTSP-over-HTTP tunneling, or for HTTP live streaming (for indexed Transport Stream files only).)
    ```

4. 找个test.264文件或者其他支持的格式放在跟`live555MediaServer`同目录

     生成流地址 rtsp://192.168.1.100:8554/test.264

    > 很多使用第三方或者其他`Demo`生成的`h264`裸流后缀是`.h264`,记住要改成`.264`才能被`live555MediaServer`识别。



## Nginx+RTMP推流服务器

1、clone nginx项目

```Shell
brew tap denji/nginx
```

2、执行安装

```Shell
brew install nginx-full --with-rtmp-module
```

3、启动nginx，输入以下命令

```Shell
nginx
```

在浏览器打开[http://localhost:8080](https://links.jianshu.com/go?to=http://localhost:8080)，如果出现Welcome to nginx!即为启动成功

4、查询安装信息（非必须）

```Shell
brew info nginx-full
```

5、配置rtmp

```Shell
#打开配置文件
vim /usr/local/etc/nginx/nginx.conf
```

在http节点下面(也就是文件的尾部)加上rtmp配置：

```
rtmp {
  server {  
      listen 1935;
      application live {
          live on;
          record off;
      }
   }
}

字段说明：
1、rtmp：协议名称
2、server：说明内部中是服务器相关配置
3、listen：监听的端口号，rtmp协议的默认端口号1935
4、application：访问的应用路径是live.(配置后生成推流地址是rtmp://localhost:1935/live/test这里live是application配置的,test任意取的名字)
5、live on; 开启实时流
6、record off; 不记录数据
```

```Shell
#修改完配置文件之后执行
nginx -s reload

#查询1935端口是否开启
sudo lsof -i -P | grep -i "listen"
```

6、直播测试->安装ffmpeg->安装vlc播放器->准备mp4文件->推流

```
ffmpeg -re -i /xxx/xxx.mp4 -vcodec libx264 -acodec aac -f flv rtmp://127.0.0.1:1935/live/test
```
7、在vlc播放器上输入 [rtmp://localhost:1935/live/test](https://links.jianshu.com/go?to=rtmp://localhost:1935/live/test)进行播放
