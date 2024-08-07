FFMpeg一些命令

摄像头推流

```shell
ffmpeg -f avfoundation -video_size 640x480 -framerate 30 -i 0:0 -vcodec libx264 -preset veryfast -f flv rtmp://localhost:1935/live/test
```

文件推流

```shell
ffmpeg -re -i /Users/xxx/Desktop/xxx.mp4 -vcodec copy -f flv rtmp://localhost:1935/live/test

ffmpeg -re -i /Users/xxx/Desktop/xxx.mp4 -vcodec libx264 -acodec aac -f flv rtmp://127.0.0.1:1935/live/test
```

转码切片

```shell
ffmpeg -i rtsp://192.168.2.42 -c copy -f segment -segment_time 10 -movflags frag_keyframe+empty_moov -strftime 1 " /Users/moon/xxx/%Y%m%d/output_%s.flv"
```

转码

```shell
ffmpeg -i rtsp://192.168.1.103:554/1.h264 -f flv rtmp://127.0.0.1:1935/live/test
```

ffplay

```shell
ffplay [rtmp://127.0.0.1:1935/live/test](rtmp://127.0.0.1:1935/live/test) -fflags nobuffer
```





参数设置

```shell
av_dict_set(&options, "buffer_size", "2048000", 0);
av_dict_set(&options, "max_delay", "500000", 0);
av_dict_set(&options, "stimeout", "10000000", 0);  //设置超时断开连接时间 10s
av_dict_set(&options, "rtsp_transport", "tcp", 0);  //以udp方式打开，如果以tcp方式打开将udp替换为tcp

//防止编码延迟修改参数
av_opt_set(c->priv_data, "preset", "superfast", 0); 
av_opt_set(c->priv_data, "tune", "zerolatency", 0);

//av_dict_set(&options, "fflags", "nobuffer", 0);
//网络资料显示会减少解码延时,实际会造成第一帧丢弃,导致出现非I帧
```

分析

```shell
ffprobe -show_streams rtsp://192.168.124.3:554/live/mainstream
```



