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
Packed方式为两个声道的数据交错存储；Planar方式为两个声道分开存储。
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

