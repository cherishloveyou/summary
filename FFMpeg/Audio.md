## FFMpeg Audio

```shell
将smaples由32bits转换为16bits
若sample是AV_SAMPLE_FMT_FLTP,则sample是float格式,且值域为[-1.0,1.0]
若sample是AV_SAMPLE_FMT_S16,则sample是int16格式,且值域为[-32768,+32767]
```

```
AV_SAMPLE_FMT_S16   kAudioFormatFlagIsSignedInteger | kAudioFormatFlagIsPacked    AV_SAMPLE_FMT_S16P  kAudioFormatFlagIsSignedInteger | kAudioFormatFlagIsNonInterleaved    AV_SAMPLE_FMT_FLT   kAudioFormatFlagIsFloat | kAudioFormatFlagIsPacked    
AV_SAMPLE_FMT_FLTP  kAudioFormatFlagIsFloat | kAudioFormatFlagIsNonInterleaved
```

音频数据存储分为 Packed和Planar两种存储方式
**Packed方式为两个声道的数据交错存储；Planar方式为两个声道分开存储。**
Packed:  L R L R L R L R ...
Planar:   L L L L ... R R R R...

PCM数据解析

```c
//AV_SAMPLE_FMT_FLTP packed
int out_buffer_size = av_samples_get_buffer_size(NULL, self.model.codecContex->channels, _temp_frame->nb_samples, _temp_frame->format, 1);
for (int i = 0; i < out_buffer_size/2; i += 4) {
  fwrite(_temp_frame->data[0] + i, 1, 4, self->fp);
  fwrite(_temp_frame->data[1] + i, 1, 4, self->fp);
}
//AV_SAMPLE_FMT_FLT
int out_buffer_size = av_samples_get_buffer_size(NULL, self.model.codecContex->channels, _temp_frame->nb_samples, AV_SAMPLE_FMT_FLT, 1);
NSData *data = [NSData dataWithBytes:audioDataBuffer length:out_buffer_size];

Byte *dst = (Byte*)data.mutableBytes;
memcpy(dst, audioDataBuffer, out_buffer_size);
```



处理 Audio Unit 之外，还可以通过混音算法来实现混音，几乎所有的混音算法都是通过对输入的音频数据做线性叠加衍生出的，比如平均值法，[自适应加权](https://zhida.zhihu.com/search?content_id=144321339&content_type=Article&match_order=1&q=自适应加权&zhida_source=entity)等



**1.直接加和**
同一个声道的数值进行简单的相加,数据是很完整的保留下来了，但是会存在溢出的可能而且混合的路数越多，溢出的可能性越大

```c
/**
* @param inputAudios
* 直接加和
* @return
*/

public static short[] mixRawAudioBytes(short[][] inputAudios) {
  int coloum = finalLength;//最终合成的音频长度
  // 音轨叠加
  short[] realMixAudio = new short[coloum];
  int mixVal;
  for (int trackOffset = 0; trackOffset < coloum; ++trackOffset) {
      mixVal = inputAudios[0][trackOffset]+inputAudios[1][trackOffset];
      realMixAudio[trackOffset] = (short) (mixVal);
  }
  return realMixAudio;
}
```

**2.平均调整权重法（平均法）**

将每一路的语音线性相加，再除以通道数，该方法虽然不会引入噪声，但是随着通道数成员的增多，各路语音的衰减将愈加严重。具体体现在随着通道数成员的增多，各路音量会逐步变小。

```c
 /**
 * @param inputAudios
 * 平均调整权重法（平均法）
 * @return
 */

public static short[] mixRawAudioBytes(short[][] inputAudios) {
    int coloum = finalLength;//最终合成的音频长度
    // 音轨叠加
    short[] realMixAudio = new short[coloum];
    int mixVal;
    for (int trackOffset = 0; trackOffset < coloum; ++trackOffset) {
        mixVal = (inputAudios[0][trackOffset]+inputAudios[1][trackOffset])/2;
        realMixAudio[trackOffset] = (short) (mixVal);
    }
    return realMixAudio;
}
```

**3. 加和并箝(qián)位**

将每一路的语音线性相加进行溢出检测，如果溢出，以最大值来替代。这样会造成声音波形的人为削峰，在破坏语音信号特性的同叫会促使噪音的产生

```c
/**
* @param inputAudios
* 加和并箝位
* @return
*/

public static short[] mixRawAudioBytes(short[][] inputAudios) {
  int coloum = finalLength;//最终合成的音频长度
  //混音溢出边界
  int MAX = 32767;
  int MIN = -32768;
  // 音轨叠加
  short[] realMixAudio = new short[coloum];
  int mixVal;
  for (int trackOffset = 0; trackOffset < coloum; ++trackOffset) {
      mixVal = inputAudios[0][trackOffset]+inputAudios[1][trackOffset];
      if (mixVal>MAX){
          mixVal = MAX;
      }
      if (mixVal<MIN){
          mixVal = MIN;
      }
      realMixAudio[trackOffset] = (short) (mixVal);
  }
  return realMixAudio;
}
```

**4. 归一化**

全部乘个系数因子，使幅值归一化，但是个人认为这个归一化因子是不好确认的。

```c
/**
 * @param inputAudios
 * 归一化
 * @return
 */

public static short[] mixRawAudioBytes(short[][] inputAudios) {
    int coloum = finalLength;//最终合成的音频长度
    float f = divisor;//归一化因子
    // 音轨叠加
    short[] realMixAudio = new short[coloum];
    float mixVal;
    for (int trackOffset = 0; trackOffset < coloum; ++trackOffset) {
        mixVal = (inputAudios[0][trackOffset]+inputAudios[1][trackOffset])*f;
        realMixAudio[trackOffset] = (short) (mixVal);
    }
    return realMixAudio;
}
```

**5. 自适应混音加权(衰减因子法)(改进后的归一化算法)**
使用可变的衰减因子对语音进行衰减，该衰减因子代表了语音的权重，该衰减因子随着数据的变化而变化，当数据溢出时，则相应的使衰减因子变小，使后续的数据在衰减后处于临界值以内，没有溢出时，让衰减因子慢慢增大，使数据变化相对平滑。
[算法详细解释可以参考这个链接](http://www.doc88.com/p-70383188302.html)

```c
/**
 * @param inputAudios
 * 自适应混音加权(衰减因子法)（改进版归一化因子法）
 * @return
 */
public static short[] mixRawAudioBytes(short[][] inputAudios) {
    int coloum = finalLength;//最终合成的音频长度
    float f = 1;//衰减因子 初始值为1
    //混音溢出边界
    int MAX = 32767;
    int MIN = -32768;
    //音轨叠加
    short[] realMixAudio = new short[coloum];
    float mixVal;
    for (int trackOffset = 0; trackOffset < coloum; ++trackOffset) {
        mixVal = (inputAudios[0][trackOffset]+inputAudios[1][trackOffset])*f;
        if (mixVal>MAX){
            f = MAX/mixVal;
            mixVal = MAX;
        }

        if (mixVal<MIN){
            f = MIN/mixVal;
            mixVal = MIN;
        }
        if (f < 1){
  //SETPSIZE为f的变化步长，通常的取值为(1-f)/VALUE,此处取SETPSIZE 为 32   VALUE值可以取 8, 16, 32,64,128.
            f += (1 - f) / 32;
        }
        realMixAudio[trackOffset] = (short) (mixVal);
    }
    return realMixAudio;
}
```

**6. 自动对齐算法**

考虑参与混音的多路音视频信号自身特点，以它们自身的比例作为权重，从而决定它们在合成后的输出中所占比重。

```c
 /**
 * @param inputAudios
 * 自动对齐算法
 * @return
 */

public static short[] mixRawAudioBytes(short[][] inputAudios) {
    int coloum = finalLength;//最终合成的音频长度
    float f1 = divisor1;//权重因子
    float f2 = divisor2;//权重因子
    //音轨叠加
    short[] realMixAudio = new short[coloum];
    float mixVal;
    for (int trackOffset = 0; trackOffset < coloum; ++trackOffset) {
        mixVal = inputAudios[0][trackOffset]*f1+inputAudios[1][trackOffset]*f2;
        realMixAudio[trackOffset] = (short) (mixVal);
    }
    return realMixAudio;
}
```

**7. 有人说的newlc中的一段算法**
算法原型：
Y = A + B - (A * B / (-(2 pow(n-1) -1)))
Y = A + B - (A * B / (2 pow(n-1))
这个算法网上有很多人在引用，当我尝试把 (2 pow(n-1))替换为常量Max后进行数学推导后发现
我推不出来。。。。。。。 我只能得出的结论是，如果A=Max，B=Max，A+B-A*B/MAX=MAX
而且也有好多人提出了质疑。

```c
/**
 * @param inputAudios
 * 有人说的newlc中的一段算法
 * @return
 */
public static short[] mixRawAudioBytes(short[][] inputAudios) {
    int coloum = finalLength;//最终合成的音频长度
    //音轨叠加
    short[] realMixAudio = new short[coloum];
    int mixVal;
    for (int trackOffset = 0; trackOffset < coloum; ++trackOffset) {
        mixVal = 0;
        if (inputAudios[0][trackOffset] < 0 && inputAudios[1][trackOffset] < 0) {
            mixVal = inputAudios[0][trackOffset] + inputAudios[1][trackOffset] - (inputAudios[0][trackOffset] * inputAudios[1][trackOffset] / MIN);
        } else {
            mixVal = inputAudios[0][trackOffset] + inputAudios[1][trackOffset] - (inputAudios[0][trackOffset] * inputAudios[1][trackOffset] / MAX);
        }
        realMixAudio[trackOffset] = (short) (mixVal);
    }
    return realMixAudio;
}
```
