# iOS音视频

AAC音频格式有ADIF和ADTS：

- ADIF：Audio Data Interchange Format 音频数据交换格式。这种格式的特征是可以确定的找到这个音频数据的开始，不需进行在音频数据流中间开始的解码，即它的解码必须在明确定义的开始处进行。故这种格式常用在磁盘文件中。

- ADTS：Audio Data Transport Stream 音频数据传输流。这种格式的特征是它是一个有同步字的比特流，解码可以在这个流中任何位置开始。它的特征类似于mp3数据流格式。

  

AAC文件处理流程

 (1)判断文件格式，确定为ADIF或ADTS

 (2)若为ADIF，解ADIF头信息，跳至第6步。

 (3)若为ADTS，寻找同步头。

 (4)解ADTS帧头信息。

 (5)若有错误检测，进行错误检测。

 (6)解块信息。

 (7)解元素信息。



采样频率(sampleRate): 也称为采样速度或者采样率，定义了每秒从连续信号中提取并组成离散信号的采样个数，它用赫兹（Hz）来表示。采样频率的倒数是采样周期，它是采样之间的时间间隔。通俗的讲采样频率是指计算机每秒钟采集多少个信号样本。采样频率越高声音的还原就越真实越自然。

 比特率=采样率 * 单个的周期音频数据长度 (sampleSize)。
 如16bit 单声道(channelCount) 48KHz音频的比特率:
 **48KHz \* (16 \* 1) =  1536kbps = 192 kBps**






[AudioToolbox实现音频编解码](https://www.jianshu.com/p/27df093f0e2e)