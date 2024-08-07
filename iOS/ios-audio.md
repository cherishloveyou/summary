# iOS音视频

AAC音频格式有ADIF和ADTS：

- ADIF：Audio Data Interchange Format 音频数据交换格式。这种格式的特征是可以确定的找到这个音频数据的开始，不需进行在音频数据流中间开始的解码，即它的解码必须在明确定义的开始处进行。故这种格式常用在磁盘文件中。

- ADTS：Audio Data Transport Stream 音频数据传输流。这种格式的特征是它是一个有同步字的比特流，解码可以在这个流中任何位置开始。它的特征类似于mp3数据流格式。

AAC头部信息分为：固定头部信息和可变头部信息

![cda2d0e185456ee939d3eb93928f8462](/Users/moon/Documents/summary/iOS/Assets/Audio/cda2d0e185456ee939d3eb93928f8462.png)

固定头:
syncword ：同步头代表着1个ADTS帧的开始，所有bit置1，即 0xFFF
ID：MPEG标识符，0标识MPEG-4，1标识MPEG-2
Layer： 直接置00,解码时忽略这个参数
protection_absent：表示是否误码校验。1 no CRC , 0 has CRC
profile：AAC 编码级别, 0: Main Profile, 1:LC(最常用), 2: SSR, 3: reserved.
sampling_frequency_index：采样率标识,重要！
Private bit：直接置0，解码时忽略这个参数
channel_configuration: 声道数标识,重要！
original_copy: 直接置0，解码时忽略这个参数
home:直接置0，解码时忽略这个参数

![b01e87a1092453766948092f556ad430](/Users/moon/Documents/summary/iOS/Assets/Audio/b01e87a1092453766948092f556ad430.png)

可变头:
copyright_identification_bit: 直接置0，解码时忽略这个参数
copyright_identification_start: 直接置0，解码时忽略这个参数
aac_frame_lenght: 当前音频帧的字节数. 重要！
adts_buffer_fullness: 当设置为0x7FF时表示时可变码率
number_of_raw_data_blocks_in_frames: 当前音频包里面包含的音频编码帧数,为0代表1frame.



```c
const int sampling_frequencies[] = {
    96000,  // 0x0
    88200,  // 0x1
    64000,  // 0x2
    48000,  // 0x3
    44100,  // 0x4
    32000,  // 0x5
    24000,  // 0x6
    22050,  // 0x7
    16000,  // 0x8
    12000,  // 0x9
    11025,  // 0xa
    8000   // 0xb
    // 0xc d e f是保留的
};

int adts_header(char * const p_adts_header, const int data_length,
                const int profile, const int samplerate,
                const int channels)
{
    int sampling_frequency_index = 3; // 默认使用48000hz
    int adtsLen = data_length + 7;  //这里不做校验，直接+7个字节用于存放ADTS header

    int frequencies_size = sizeof(sampling_frequencies) / sizeof(sampling_frequencies[0]);
    int i = 0;
    for(i = 0; i < frequencies_size; i++)
    {
        if(sampling_frequencies[i] == samplerate)
        {
            sampling_frequency_index = i;
            break;
        }
    }
    if(i >= frequencies_size)
    {
        printf("unsupport samplerate:%d\n", samplerate);
        return -1;
    }

    p_adts_header[0] = 0xff;         //syncword:0xfff                          高8bits
    p_adts_header[1] = 0xf0;         //syncword:0xfff                          低4bits
    p_adts_header[1] |= (0 << 3);    //MPEG Version:0 for MPEG-4,1 for MPEG-2  1bit
    p_adts_header[1] |= (0 << 1);    //Layer:0                                 2bits
    p_adts_header[1] |= 1;           //protection absent:1                     1bit

    p_adts_header[2] = (profile)<<6;            //profile:profile               2bits
    p_adts_header[2] |= (sampling_frequency_index & 0x0f)<<2; //sampling frequency index:sampling_frequency_index  4bits
    p_adts_header[2] |= (0 << 1);             //private bit:0                   1bit
    p_adts_header[2] |= (channels & 0x04)>>2; //channel configuration:channels  高1bit

    p_adts_header[3] = (channels & 0x03)<<6; //channel configuration:channels 低2bits
    p_adts_header[3] |= (0 << 5);               //original：0                1bit
    p_adts_header[3] |= (0 << 4);               //home：0                    1bit
    p_adts_header[3] |= (0 << 3);               //copyright id bit：0        1bit
    p_adts_header[3] |= (0 << 2);               //copyright id start：0      1bit
    p_adts_header[3] |= ((adtsLen & 0x1800) >> 11);           //frame length：value   高2bits

    p_adts_header[4] = (uint8_t)((adtsLen & 0x7f8) >> 3);     //frame length:value    中间8bits
    p_adts_header[5] = (uint8_t)((adtsLen & 0x7) << 5);       //frame length:value    低3bits
    p_adts_header[5] |= 0x1f;                                 //buffer fullness:0x7ff 高5bits
    p_adts_header[6] = 0xfc;      //‭11111100‬       //buffer fullness:0x7ff 低6bits
    // number_of_raw_data_blocks_in_frame：
    // 表示ADTS帧中有number_of_raw_data_blocks_in_frame + 1个AAC原始帧。
    return 0;
}
```


AAC文件处理流程：

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
