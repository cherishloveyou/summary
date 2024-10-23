# MP4文件格式解析

MP4文件由许多box组成，每个box包含不同的信息， 这些box以树形结构的方式组织。以下是主要box的简要说明：

![img](https://picx.zhimg.com/80/v2-a54c40412a7b55b5bfeddf79d3d92d4d_1440w.webp)

图2 主要box说明

根节点之下，主要包含三个节点：ftyp、[moov](https://zhida.zhihu.com/search?content_id=208952033&content_type=Article&match_order=1&q=moov&zhida_source=entity)、mdat。

- **ftyp**：文件类型。描述遵从的规范的版本。
- **moov box**：媒体的metadata信息。
- **mdat**：具体的媒体数据。

**说明**：在 mp4 中默认写入[字节序](https://zhida.zhihu.com/search?content_id=208952033&content_type=Article&match_order=1&q=字节序&zhida_source=entity)是 Big-Endian的。

2. mp4文件基本信息

分析mp4文件的工具：

1. [mp4box.js](https://link.zhihu.com/?target=https%3A//links.jianshu.com/go%3Fto%3Dhttps%3A%2F%2Fgpac.github.io%2Fmp4box.js%2Ftest%2Ffilereader.html)：一个在线解析mp4的工具。
2. [bento4](https://link.zhihu.com/?target=https%3A//links.jianshu.com/go%3Fto%3Dhttp%3A%2F%2Fbento4.com)：包含mp4dump、mp4edit、mp4encrypt等工具。
3. [MP4Box](https://link.zhihu.com/?target=https%3A//links.jianshu.com/go%3Fto%3Dhttps%3A%2F%2Fgpac.wp.mines-telecom.fr%2Fmp4box%2F)：类似于bento4，包含很全面的工具。
4. [mp4info.exe](https://link.zhihu.com/?target=https%3A//links.jianshu.com/go%3Fto%3Dhttps%3A%2F%2Fdl.pconline.com.cn%2Fdownload%2F409348.html): windows平台图形界面展示mp4基本信息的工具。

下图为使用mp4info.exe打开mp4文件的界面：

![img](https://pica.zhimg.com/80/v2-fd17e60a46eb3fa3c640cc8caf01398a_1440w.webp)

图3 mp4info.PNG

mp4文件基本信息
audio信息：

1. smplrate：sample rate(采样率)。
2. channel：通道个数。
3. bitrate：比特率。
4. audiosamplenum：音频sample的个数。
