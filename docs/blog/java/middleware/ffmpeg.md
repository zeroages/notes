# ffmpeg+SRS实现推拉流
* 下载 ffmpeg 
https://www.gyan.dev/ffmpeg/builds/


* 配置环境变量
* ffmpeg -version  查看 FFmpeg 版本


* 推流命令
* MP4
ffmpeg -stream_loop -1 -re -i ./龙凤视频素材.mp4 -c copy -f flv rtmp://ip:6010/live/livestream
* rtsp格式
ffmpeg -i "rtsp://ip/h264/ch1/main/av_stream" -c:v libx264 -preset veryfast -c:a aac -f flv "rtmp://ip:1935/live/livestream"


* VLC测试  
rtmp://ip:6010/live/livestream 
http://ip:8080/live/livestream.flv