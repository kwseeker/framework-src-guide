# [LiveAndLive](https://pengrl.com/lal/#/)

[lalserver](https://github.com/q191201771/lal)是纯Golang开发的流媒体（直播音视频网络传输）服务器，源码算上注释等3W5K行。

一直以来对音视频的存储和传输方式等内部原理抱有兴趣，但是由于工作中基本很少接触这部分内容一直没有研究过，偶然看到有资料介绍这部分内容且现在视频直播比较火就深入学习一下。

音视频服务器的客户端偏前端开发。

## 音视频服务器工作原理

### 音视频上传

**音视频文件上传**：

**FFmpeg** 是一个开源的、跨平台的多媒体处理工具和库集合，广泛用于处理视频、音频和其他多媒体文件。它可以执行多种任务，如**编码**、**解码**、**转码**、**流媒体处理**、**滤镜应用**等。

```shell
# 下载 lal 源码，编译启动
cd lal
go mod tidy
make build
./bin/lalserver -c conf/lalserver.conf.json	# 没必要-c, 会自动加载默认配置

# -re：表示以实时速度读取输入文件
#     如果不加 -re，FFmpeg 会尽可能快地读取文件并推流，可能导致推流速度过快
#     适用于模拟实时推流场景（如直播）
# -i 指定输入文件, 这里上传的音视频文件是 demo.flv
# -c:a copy：指定音频流的处理方式为直接复制（不重新编码）
#     -c:a 表示音频编解码器，copy 表示直接复制原始音频流
# -c:v copy：指定视频流的处理方式为直接复制（不重新编码）
#     -c:v 表示视频编解码器，copy 表示直接复制原始视频流
# -f flv：
#     指定输出格式为 flv（Flash Video）
# rtmp://127.0.0.1:1935/live/test110：指定推流的目标地址
#     rtmp:// 表示使用 RTMP 协议
#     127.0.0.1 是 RTMP 服务器的 IP 地址（这里是本地服务器）
#     1935 是 RTMP 协议的默认端口
#     /live/test110 是推流的路径和流名称（test110 是流名称，可以自定义）
ffmpeg -re -i demo.flv -c:a copy -c:v copy -f flv rtmp://127.0.0.1:1935/live/test110
# 重新编码推流
# -c:v libx264 指定视频编码器为 libx264，即使用 H.264 编码。H.264 是一种广泛使用的视频编码格式，具有良好的压缩效率和兼容性
# -preset fast 设置 H.264 编码的预设模式, preset 模式控制编码速度和压缩效率之间的平衡
# -b:v 1000k 设置视频的目标比特率为 1000 kbps，比特率越高，视频质量越好，但文件体积也越大
ffmpeg -re -i demo.flv -c:v libx264 -preset fast -b:v 1000k -c:a aac -b:a 128k -f flv rtmp://127.0.0.1:1935/live/test110
# 推送摄像头或屏幕内容
ffmpeg -f avfoundation -i "1:0" -c:v libx264 -preset fast -b:v 1000k -c:a aac -b:a 128k -f flv rtmp://127.0.0.1:1935/live/test110
# 推送 mp4 文件
# -stream_loop -1 表示无限循环输入文件，-1 表示无限循环，0 表示不循环，1 表示循环一次，依此类推。
./ffmpeg -re -stream_loop -1 -i ../../视频/26110790646-1-192.mp4 -c copy -f flv rtmp://127.0.0.1:1935/live/test1

# 查看视频文件的音视频编码， ffprobe 是 FFmpeg 工具集中的一部分专门用于分析多媒体文件
./ffprobe ../../视频/26110790646-1-192.mp4
# MP4文件、MPEG-4封装格式、视频编码格式 h264(high) 即 H.264 编码，High Profile、
Input #0, mov,mp4,m4a,3gp,3g2,mj2, from '../../视频/26110790646-1-192.mp4':
  Metadata:
    major_brand     : isom
    minor_version   : 512
    compatible_brands: isomiso2avc1mp41
    encoder         : Lavf59.27.100
    description     : Packed by Bilibili XCoder v2.0.2
  Duration: 00:17:21.79, start: 0.000000, bitrate: 578 kb/s
  Stream #0:0[0x1](und): Video: h264 (High) (avc1 / 0x31637661), yuv420p(tv, bt709, progressive), 1152x720 [SAR 1:1 DAR 8:5], 488 kb/s, 30 fps, 30 tbr, 16k tbn (default)
      Metadata:
        handler_name    : VideoHandler
        vendor_id       : [0][0][0][0]
  Stream #0:1[0x2](und): Audio: aac (LC) (mp4a / 0x6134706D), 44100 Hz, stereo, fltp, 80 kb/s (default)
      Metadata:
        handler_name    : SoundHandler
        vendor_id       : [0][0][0][0]

# ffmpeg 工具集安装
# sudo apt install FFmpeg	# apt 没有找到工具配置信息可以手动安装，参考 linux安装ffmpeg：https://www.pengrl.com/p/20029/， 这里下载的是 ffmpeg-release-i686-static.tar.xz
ffmpeg -version
```

**边直播边上传**：

比如推送摄像头内容，直播音视频上传估计就是直接前端上传给音视频服务器，应该不会走Web服务器；像一些房间数据走Web服务器上报和音视频分开处理。

### 音视频编码

常用视频编码：

+ **H.264 (AVC)**：
  - **特点**：高压缩率、广泛兼容、硬件支持好。
  - **应用场景**：视频流媒体（如 YouTube、Netflix）、视频会议、监控摄像头。
  - **优点**：压缩效率高，适合低带宽传输。
  - **缺点**：相比新一代编码格式（如 H.265），压缩效率较低。
+ **H.265 (HEVC)**：
  - **特点**：H.264 的升级版，压缩效率提高约 50%。
  - **应用场景**：4K/8K 超高清视频、VR/AR 视频。
  - **优点**：更高的压缩效率，节省带宽和存储空间。
  - **缺点**：编码复杂度高，硬件支持不如 H.264 广泛。
+ **VP9**：
  - **特点**：Google 开发的开放视频编码格式。
  - **应用场景**：YouTube、WebRTC。
  - **优点**：免费开源，压缩效率接近 H.265。
  - **缺点**：硬件支持较少，编码速度较慢。
+ **AV1**：
  - **特点**：新一代开放视频编码格式，由开放媒体联盟（AOMedia）开发。
  - **应用场景**：流媒体、WebRTC。
  - **优点**：压缩效率高于 H.265 和 VP9，免费开源。
  - **缺点**：编码复杂度高，硬件支持较少。

常用音频编码：

+ **AAC (Advanced Audio Coding)**：
  - **特点**：高效音频编码，广泛兼容。
  - **应用场景**：流媒体（如 YouTube、Spotify）、视频文件（如 MP4）。
  - **优点**：压缩效率高，音质好。
  - **缺点**：相比 Opus，延迟较高。
+ **Opus**：
  - **特点**：低延迟、高音质、开放标准。
  - **应用场景**：WebRTC、语音通话、流媒体。
  - **优点**：低延迟，适合实时通信；音质优于 AAC。
  - **缺点**：兼容性不如 AAC。
+ **MP3**：
  - **特点**：经典的音频编码格式。
  - **应用场景**：音乐播放、存储。
  - **优点**：广泛兼容。
  - **缺点**：压缩效率较低，音质不如 AAC 和 Opus。

### 传输协议

+ **RTMP (Real-Time Messaging Protocol)**：

  - **特点**：Adobe 开发的流媒体传输协议，基于 TCP。
  - **应用场景**：直播推流（如 YouTube Live、Twitch）。
  - **优点**：延迟较低（1-3 秒），广泛支持。
  - **缺点**：不支持现代浏览器的原生播放。

  [RTMP协议规范](https://chenlichao.gitbooks.io/rtmp-zh_cn/content/1-introduction.html)（这个中文文档内容不太全）：

  [握手流程](https://chenlichao.gitbooks.io/rtmp-zh_cn/content/assets/handshake_graph.png)

  RTMP**块流**（Chunk Stream）和RTMP**消息**（Message）的关系：消息是逻辑上的数据单元，可能比较大；实际传输时需要拆分为更小的块（Chunk）传输。
  **块流**是说以块为最小传输单元的流式传输。块流支持多路复用，即多个消息可以通过一个块流传输，只需要块中标识消息ID即可。
  同时也支持使用多个块流，每个块流有自己的块流ID。

  比如：音频、视频、控制数据分别使用一个块流传输，比如cs1 cs2 cs3，同时多个视频可以复用这个视频块流(cs2)传输。

  每一个**块流**对应一个**数据缓冲**，比如 lal 中 Stream 类：

  ```go
  type Stream struct {
  	header base.RtmpHeader
  	msg    StreamMsg
  	absTsFlag bool   // 标记当这个stream收到新的msg的时候，是否收到过绝对时间
  	timestamp uint32 // 注意，是rtmp chunk协议header中的时间戳，可能是绝对的，也可能是相对的。上层不应该使用这个字段，而应该使用Header.TimestampAbs
  }
  
  type StreamMsg struct {
  	buff *nazabytes.Buffer	// 这里可以看到是数据缓冲
  }
  ```

  AMF 编码：用于封装指令等复杂格式的编码方式，有AMF0 AMF3 两种编码格式。

  

+ **RTSP (Real-Time Streaming Protocol)**：

  - **特点**：用于控制实时流媒体的协议。
  - **应用场景**：监控摄像头、IPTV。
  - **优点**：支持实时控制（如播放、暂停）。
  - **缺点**：兼容性较差。

+ **HLS (HTTP Live Streaming)**：

  - **特点**：Apple 开发的基于 HTTP 的流媒体协议。
  - **应用场景**：点播和直播（如 iOS 设备、YouTube）。
  - **优点**：兼容性好，支持自适应码率（ABR）。
  - **缺点**：延迟较高（10-30 秒）。

+ **WebRTC (Web Real-Time Communication)**：

  - **特点**：基于 UDP 的实时通信协议。
  - **应用场景**：视频会议、实时直播（如 Google Meet、Zoom）。
  - **优点**：延迟极低（< 1 秒），支持点对点传输。
  - **缺点**：不适合大规模直播。

常用视频封装格式：

+ **MP4**：
  - **特点**：广泛使用的视频封装格式。
  - **应用场景**：点播视频、存储。
  - **优点**：兼容性好，支持 H.264 和 AAC。
+ **FLV (Flash Video)**：
  - **特点**：Adobe 开发的流媒体封装格式。
  - **应用场景**：RTMP 推流。
  - **优点**：适合流媒体传输。
  - **缺点**：逐渐被淘汰。
+ **TS (Transport Stream)**：
  - **特点**：用于流媒体传输的封装格式。
  - **应用场景**：HLS、DASH。
  - **优点**：适合分段传输。

### 音视频转推

由音视频服务器实现。另一个 lal 服务器作为当前 lal 服务器的订阅者。

### 音视频录制

将数据流写成 FLV 等格式的文件中。

### 音视频存储

lal 服务器读取音视频发布者推过来的数据流并不会长时间存储，而是存储在一个名为 Stream 的缓冲中，Stream 是一个带有读写指针的byte数组，最大缓存1MB 数据，存满1MB后新数据会覆盖旧数据。

### 音视频拉流

使用 VLC 拉取音视频流（打开网络串流）。

```shell
sudo snap install vlc
# 输入网络 URL，支持多种协议
rtsp://localhost:5544/live/test1
http://127.0.0.1:8080/live/test1.flv
http://127.0.0.1:8080/hls/test1.m3u8
http://127.0.0.1:8080/live/test1.ts
# 前面设置的无线循环推流,所以VLC中可以一直拉流，无线循环播放，看起来就像是直播
./ffmpeg -re -stream_loop -1 -i ../../视频/26110790646-1-192.mp4 -c copy -f flv rtmp://127.0.0.1:1935/live/test1
```

### 拉流回源

回源是指当用户请求的音视频内容在边缘节点（如 CDN 节点）上没有缓存时，CDN 会向源站（Origin Server）请求内容，然后将内容缓存到边缘节点并返回给用户。边缘节点也是一个音视频服务器。

其实和转推一样，仅仅是多了一层转发。

