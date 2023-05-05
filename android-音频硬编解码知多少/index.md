# Android 音频硬编解码知多少


> 一、使用 AAC 音频硬解码的背景因为各种原因，在日常的开发中开发者或多或少都要接触一些音视频编解码相关的功能，所以有时候选择编解码工具就变得尤为重要，取决于你的项目属性又或者知识广度等等，转自<https://xie.infoq.cn/article/e21903be1dce4df66e1963f33>

一、使用 AAC 音频硬解码的背景
=================

因为各种原因，在日常的开发中开发者或多或少都要接触一些音视频编解码相关的功能，所以有时候选择编解码工具就变得尤为重要，取决于你的项目属性又或者知识广度等等，下面作者结合自己的实际项目经验给大家分析一下

开发成本
----

**开发成本**在企业管理者的角度来说尤为重要，关系到企业的盈利与生存。所以为了降低成本很多开发者会考虑去使用 Android 原生提供的一些 API，而不是去使用第三方的一些开源库或者收费库，因为那样急需要花费额外的金钱并且还需要花费时间与精力去熟悉，所以也不推荐，除非时间和成本都在允许的范围内

维护成本
----

当项目迭代至成熟期时，**维护成本**就成了后续开发者要关注的事情，首先假设我们使用了第三方的库，如果你的产品已经卖出去了，而这时候第三方库不维护并且出现了一个致命的问题，那这样就会导致卖出去的产品都会被投诉并且短时间内还要花时间去移除之前使用的第三方库，如果耦合性过多，将导致无法挽回的经济损失。而如果使用的是 Android 原生的 API 的话，因为本身是做产品的，所以只考虑当前设备，无须关心移植到其他平台或其他系统版本，前期做稳定，后期就不会有任何问题

二、使用 AAC 音频硬解码的优缺点
==================

优点
--

开发方便快捷，有成熟的 API 调用，使用简单，网上也有大部分的参考资料

缺点
--

可移植性差，如果公司其他项目需要移植到新的硬件平台时，会有兼容性问题，大部分需要向原厂提工单才可解决

三、AAC 音频硬解码的 API 介绍
===================

MediaCodec 方法介绍
---------------

**MediaCodec** 是 Android 原生提供的 API，支持音视频的硬编码和硬解码，Android 常用的源文件格式与编码后格式是音频的 **PCM** 编码成 **AAC**，视频的 **NV21/YV12** 编码成 H264，值得一提的是在选择和设置视频编码质量的时候，~MediaFormat.KEY_PROFILE~ 在官方 API 介绍中，其可以控制视频的质量，实际则是 **Android7.0** 以下默认 **baseline**，不管怎么设置都是默认 baseline，所以这个变量属性，作者采用了删除线，在视频编码时，不推荐大家使用，避免出现问题

**getInputBuffers()**

> 从当前编解码器中获取输入缓冲区数组，用于向输入缓冲区中添加要编解码的数据

**getOutputBuffers()**

> 从当前编解码器中获取输出缓冲区数组，用于提取编解码之后的数据缓冲区

**dequeueInputBuffer(long timeoutUs)**

> 获取输入缓冲区数组中待使用 (空闲) 的缓冲区数组下标索引，timeoutUs 为 0 时立即返回，小于 0 时表示一直等待直至输入缓冲区数组中有可用的缓冲区为止，大于 0 则表示等待时间为 timeoutUs

**getInputBuffer(int index)**

> 获取输入缓冲区数组中待使用 (空闲) 的缓冲区，index 参数为 dequeueInputBuffer(long timeoutUs)的返回值，返回值大于等于 0 即表示有可用的输入缓冲区

**queueInputBuffer(int index, int offset, int size, long presentationTimeUs, int flags)**

> 向输入缓冲区数组中添加要编解码的数据，index 参数为 dequeueInputBuffer(long timeoutUs) 的返回值，offset 为要编解码数据的起始偏移，size 为要编解码数据的长度，presentationTimeUs 为 PTS，flags 为标记，正常使用时可默认填 0，编解码至结尾时可填 MediaCodec.BUFFER_FLAG_END_OF_STREAM 值

**dequeueOutputBuffer(BufferInfo info, long timeoutUs)**

> 从输出缓冲区数组中获取编解码成功的缓冲区下标索引，info 参数表示传入一个 BufferInfo Java bean class ， 编解码器会把处理完后的数据信息等以 bean 类型返回给开发者，timeoutUs 意义跟之前介绍的 dequeueInputBuffer(long timeoutUs) 方法大致相同，返回值大于等于 0 即表示有可用的输出缓冲区

**getOutputBuffer(int index)**

> 获取输出缓冲区数组中编解码完成的缓冲区，index 参数为 dequeueOutputBuffer(BufferInfo info, long timeoutUs) 方法的返回值，返回值大于等于 0 即表示有可用的输出缓冲区

**releaseOutputBuffer(int index, boolean render)**

> 释放编解码器输出缓冲区数组中的缓冲区，index 为要释放的缓冲区数组下标索引，它为 dequeueOutputBuffer(BufferInfo info, long timeoutUs) 方法的返回值，render 参数为渲染控制，如果在编解码时设置了可用的 surface，render 为 true 时则表示将此数据缓冲区输出到 surface 渲染

**stop()**

> 关闭编解码

**release()**

> 释放编解码资源

MediaCodec 参数介绍
---------------

本篇文章关于 MediaCodec 参数的介绍只描述日常开发中出现频率最频繁的，其他一些参数很少使用或者使用之后没效果，这里就不再做过多阐述

**MediaFormat.KEY_AAC_PROFILE**

> 要使用的 AAC 配置文件的键（仅 AAC 音频格式时使用）, 常量在 android.media.MediaCodecInfo.CodecProfileLevel 中声明，音频编码中最常用的变量是 MediaCodecInfo.CodecProfileLevel.AACObjectLC

**MediaFormat.KEY_CHANNEL_MASK**

> 音频内容的通道组成的键，在音频编码中需要根据硬件支持去有选择性的选择支持范围内的通道号

**MediaFormat.KEY_BIT_RATE**

> 音视频平均比特率，以位 / 秒为单位 (bit/s) 的键

**MediaFormat.KEY_CHANNEL_COUNT**

> 音频通道数的键

**MediaFormat.KEY_COLOR_FORMAT**

> 输入视频源的颜色格式，日常开发中可根据查询设备颜色格式支持进行选择

**MediaFormat.KEY_FRAME_RATE**

> 视频帧速率的键，以帧 / 秒 (frame/s) 为单位

**MediaFormat.KEY_I_FRAME_INTERVAL**

> 关键帧间隔的键

**MediaFormat.KEY_MAX_INPUT_SIZE**

> 编解码器中数据缓冲区最大大小的键，以字节 (byte) 为单位

四、AAC 音频硬解码
===========

本地音视频文件里的 AAC 音频硬解码介绍，MediaExtractor 方法详解
-----------------------------------------

解析本地音视频文件里的 AAC 音频，需要我们借助一些 **MediaCodec** 之外的 API 即 **MediaExtractor**，如果不熟悉或之前没使用过，没关系！作者会在本篇文章中做一个详细的概述，帮助你加深印象

**setDataSource(String path)**

> 设置音视频文件的绝对路径或音视频文件的 http 地址，path 参数可以是本地音视频文件的绝对路径或网络上的音视频文件 http 地址

**getTrackCount()**

> 获取音视频数据中的轨道数，正常情况下的音视频有 audio/xxx 及 video/xxx

**getTrackFormat(int index)**

> 获取音视频数据中音频或视频的 android.media.MediaFormat，这个很重要后面还会有代码示例来介绍，index 参数为音频或视频数据轨道的索引，返回值是 android.media.MediaFormat

**selectTrack(int index)**

> 选择要 extract 的数据轨道，index 参数为指定的音频或视频轨道的索引，后面也是会通过代码示例详细介绍

**readSampleData(ByteBuffer byteBuf, int offset)**

> 读取音频或视频轨道中的数据到给定的 ByteBuffer 缓冲区中，byteBuf 参数为要保存数据的目标缓冲区，offset 参数为音频或视频的数据起始偏移量，返回值为 int 类型，大于 0 表示还有数据未处理完，否则表示数据已经全部处理完成

**getSampleTime()**

> 获取该帧音频或视频的的时间戳即 PTS，返回值为 long 类型，以微秒 (us) 为单位，如无可用返回 - 1

**advance()**

> 此方法表示开始处理下一帧音频或视频，如果还有数据返回 true，已无数据则返回 false

**release()**

> 释放资源，在 advance() 返回 false 或中断 read 操作后使用，表示数据处理完毕或不再读取数据

实时 AAC 音频硬解码介绍
--------------

实时 AAC 音频硬解码其实跟本地音视频 AAC 音频硬解码大同小异，唯一差异就是实时的不需要去使用 **MediaExtractor** 进行音频轨与视频轨进行分离，可以直接使用 **MediaCodec** 进行音频硬解码，但需要解析实时流里的 ADTS 音频头，否则 **MediaCodec** 解码器是无法识别出该数据源是否是 AAC 音频。正常情况下需要开发者解析 ADTS 头中的一些关键信息，如采样率索引 (可根据采样率进行换算)、通道数。

下面作者就给大家介绍关于 ADTS 头的解析及 ADTS 其他位的意义：

> [ADTS 头的解析及 ADTS 其他位的意义](https://xie.infoq.cn/link?target=https%3A%2F%2Fwiki.multimedia.cx%2Findex.php%3Ftitle%3DADTS)  

**ADTS 头结构:**

AAAAAAAA AAAABCCD EEFFFFGH HHIJKLMM MMMMMMMM MMMOOOOO OOOOOOPP (QQQQQQQQ QQQQQQQQ)

![][img-0] AAC 音频头构成

在实时 AAC 音频硬解码时，我们只需要解析采样率索引 (可根据采样率进行换算)、通道数即可，音频采样率索引见 [MPEG-4 Sampling Frequency Index](https://xie.infoq.cn/link?target=https%3A%2F%2Fwiki.multimedia.cx%2Findex.php%2FMPEG-4%253Ci%253EAudio%23Sampling%253C%2Fi%253EFrequencies)，接下来还会向各位介绍更重要的音视频参数

五、音视频编解码的 CSD 参数
================

音频编解码的 CSD 参数介绍
---------------

在 Android 中如果调用麦克风进行录音，结合视频使用 **MediaMuxer** 进行音视频合成时，是需要开发者传入 CSD 参数的，否则 Android 在播放或展示时会出现不识别等其他问题，所以需要开发者在编解码时需要调用 **MediaFormat** 设置 CSD 参数

在音频编解码中，CSD 参数只需要设置一个，那就是 **csd-0** 即 ADTS 音频头，在解析本地音视频中的 AAC 音频时，开发者可以调用 **MediaFormat** 取到这个 **csd-0** 参数对应的 ADTS 音频头，然后进行后续的其他操作，后续代码示例还会再次介绍。如果解析的是实时 AAC 音频，那就需要参照第四步骤对 ADTS 头进行解析，然后计算 CSD 参数并设置到 **MediaFormat** 中，然后配置到 **MediaCodec** 中进行解码，具体算法将在后面的代码示例中提到

视频编解码的 CSD 参数介绍
---------------

在 Android 中如果调用摄像头进行录像，结合音频使用 **MediaMuxer** 进行音视频合成时，是需要开发者传入 CSD 参数的，否则 Android 在播放或展示时会出现不识别等其他问题，所以需要开发者在编解码时需要调用 **MediaFormat** 设置 CSD 参数

在视频编解码中，CSD 参数需要设置 2 个，那就是 **csd-0**、**csd-1** 即 **sps** 视频头和 **pps** 视频头，在解码本地 **h264** 编码视频时可以调用 **MediaFormat** 获取 **sps** 视频头和 **pps** 视频头，减少 sps/pps 视频头运算和查找的操作，简单快捷且高效！具体使用会在代码示例中再次提及

六、代码示例
======

本地音视频文件中的 AAC 音频硬解码
-------------------

```java
/**     
* set decode file path     *     
* @param decodeFilePath decode file path     
*/    
public void setDecodeFilePath(String decodeFilePath) {        
    if (TextUtils.isEmpty(decodeFilePath)) {

        throw new RuntimeException("decode file path must not be null!");
}
mediaExtractor = getMediaExtractor(decodeFilePath);    }
```

上述代码片段为设置一个需要解码的文件的绝对路径，路径为 null 时抛出一个运行时异常，提示路径不能为 null，然后就是获取 **MediaExtractor** 对象，为提取音频做准备

```java
/**     * get media extractor     *     
* @param videoPath need extract of tht video file absolute path     
* @return {@link MediaExtractor} media extractor instance object     
* @throws IOException     
*/    
protected MediaExtractor getMediaExtractor(String videoPath) {
    MediaExtractor mMediaExtractor = new MediaExtractor();
    try {
        // set file path
        mMediaExtractor.setDataSource(videoPath);
        // get source file track count
        int trackCount = mMediaExtractor.getTrackCount();
        for (int i = 0; i < trackCount; i++) {

// get current media track media format

MediaFormat mediaFormat = mMediaExtractor.getTrackFormat(i);

// if media format object not be null

if (mediaFormat != null) {

// get media mime type

String mimeType = mediaFormat.getString(MediaFormat.KEY_MIME);

// media mime type match audio

if (mimeType.startsWith(AUDIO_MIME_TYE)) {

    // set media track is audio

    mMediaExtractor.selectTrack(i);

    // you can using media format object call getByteBuffer method and input key "csd-0" get it value , if you  want.

    // it is aac adts audio header.

    adtsAudioHeader = mediaFormat.getByteBuffer(CSD_MIME_TYPE_0).array();

    // get audio sample

    sampleRate = mediaFormat.getInteger(MediaFormat.KEY_SAMPLE_RATE);

    // get audio channel count

    channelCount = mediaFormat.getInteger(MediaFormat.KEY_CHANNEL_COUNT);

    return mMediaExtractor;

}

// >>>>>>>>>>> expand start >>>>>>>>>>>

// media mime type match video

// else if (mimeType.startsWith(VIDEO_MIME_TYE)) {

// get video sps header

// byte[] spsVideoHeader = mediaFormat.getByteBuffer(CSD_MIME_TYPE_0).array();

// get video pps header

// byte[] ppsVideoHeader = mediaFormat.getByteBuffer(CSD_MIME_TYPE_1).array();

// }

// <<<<<<<<<<< expand end <<<<<<<<<<<

}

        }        
    } catch (IOException e) {

        Log.d(TAG, "happened io exception : " + e.toString());

        if (mMediaExtractor != null) {

mMediaExtractor.release();

        }        
    }        
    return null;    
}
```

上述代码片段为获取 **MediaExtractor** 对象，在设置文件路径后调用其 getTrackCount() 方法获取文件的所有轨道数，再使用 for 循环去逐一匹配我们需要的媒体源轨道，调用其 getTrackFormat(int index) 方法获取该轨道的 **MediaFormat**，最后再去匹配该轨道 **MediaFormat** 的 mime type，如果匹配到其 mime type 以关注的 mime type 字符开始时，获取其 **csd-0** 参数的值 (音频中对应 ADTS 头)、采样率、通道数并调用 selectTrack(int index) 方法将该轨道设置为选定的轨道。

视频相关的参数获取也在代码片段中的 **expand** 范围内给出，大家可以了解一下，作者也将其添加上来了，只不过是在代码中注释了，为的就是给大家拓展一下这方面的知识

```java
@Override    
public void start() {        
    if (mediaExtractor == null) {
        Log.e(TAG, "media extractor is null , so return!");
        return;        
    }        
    if (adtsAudioHeader == null || adtsAudioHeader.length == 0) {
        Log.e(TAG, "aac audio adts header is null , so return!");
        return;        
    }        
    aacDecoder = createDefaultDecoder();        
    if (aacDecoder == null) {
        Log.e(TAG, "aac audio decoder is null , so return!");
        return;        
    }        
    if (worker == null) {
        isDecoding = true;
        worker = new Thread(this, TAG);
        worker.start();        
    }    
}
```

上述代码片段为准备开始提取 AAC 音频并进行 **MediaCodec** 硬解码，首先判断前面代码片段中 **MediaExtractor** 对象是否为空，完事在判断获取轨道时的 ADTS 头是否正常取到，最后生成一个 AAC 音频解码器，如果生成无异常，开启一个工作线程进行音频的提取和解码

```java
/**     * create default aac decoder     *     
* @return {@link MediaCodec} aac audio decoder     
*/    
private MediaCodec createDefaultDecoder() {        
    try {
        MediaFormat mediaFormat = new MediaFormat();
        mediaFormat.setString(MediaFormat.KEY_MIME, AUDIO_DECODE_MIME_TYPE);
        mediaFormat.setInteger(MediaFormat.KEY_SAMPLE_RATE, sampleRate);
        mediaFormat.setInteger(MediaFormat.KEY_CHANNEL_COUNT, channelCount);
        ByteBuffer byteBuffer = ByteBuffer.allocate(adtsAudioHeader.length);
        byteBuffer.put(adtsAudioHeader);
        byteBuffer.flip();
        mediaFormat.setByteBuffer(CSD_MIME_TYPE_0, byteBuffer);
        MediaCodec aacDecoder = MediaCodec.createDecoderByType(AUDIO_DECODE_MIME_TYPE);
        aacDecoder.configure(mediaFormat, null, null, 0);
        aacDecoder.start();
        return aacDecoder;        
    } catch (IOException e) {
        Log.e(TAG, "create aac audio decoder happened io exception : " + e.toString());        
    }        
    return null;    
}
```

上述代码片段为创建音频解码器，**sampleRate**、**channelCount**、**adtsAudioHeader** 都是前面代码片段中通过 **MediaExtractor** 从文件的媒体轨道中的 **MediaFormat** 获取的

```java
/**     
* aac audio format decode to pcm audi format     
*/    
private void aacDecodeToPcm() {        
    isLowVersion = android.os.Build.VERSION.SDK_INT < android.os.Build.VERSION_CODES.LOLLIPOP;        
    ByteBuffer[] aacDecodeInputBuffers = null;        
    ByteBuffer[] aacDecodeOutputBuffers = null;        
    if (isLowVersion) {
        aacDecodeInputBuffers = aacDecoder.getInputBuffers();
        aacDecodeOutputBuffers = aacDecoder.getOutputBuffers();        
    }        
    MediaCodec.BufferInfo info = new MediaCodec.BufferInfo();        
    // initialization audio track , use for play pcm audio data        
    // audio output channel param       channelConfig according device support select        int buffsize = AudioTrack.getMinBufferSize(sampleRate,     channelConfig, AudioFormat.ENCODING_PCM_16BIT);        AudioTrack audioTrack = new AudioTrack(AudioManager.STREAM_MUSIC,      sampleRate, channelConfig,
    AudioFormat.ENCODING_PCM_16BIT, buffsize, AudioTrack.MODE_STREAM);        
    audioTrack.play();        
    Log.d(TAG, "aac audio decode thread start");        
    while (isDecoding) {
        // This method will return immediately if timeoutUs == 0
        // wait indefinitely for the availability of an input buffer if timeoutUs < 0
        // wait up to "timeoutUs" microseconds if timeoutUs > 0.
        int aacDecodeInputBuffersIndex = aacDecoder.dequeueInputBuffer(2000);
        // no such buffer is currently available , if aacDecodeInputBuffersIndex is -1
        if (aacDecodeInputBuffersIndex >= 0) {
            ByteBuffer sampleDataBuffer;
            if (isLowVersion) {
                sampleDataBuffer = aacDecodeInputBuffers[aacDecodeInputBuffersIndex];
            } else {
                sampleDataBuffer = aacDecoder.getInputBuffer(aacDecodeInputBuffersIndex);
            }
            int sampleDataSize = mediaExtractor.readSampleData(sampleDataBuffer, 0);
            if (sampleDataSize < 0) {
                aacDecoder.queueInputBuffer(aacDecodeInputBuffersIndex, 0, 0, 0, MediaCodec.BUFFER_FLAG_END_OF_STREAM);
            } else {
                try {
                    long presentationTimeUs = mediaExtractor.getSampleTime();
                    aacDecoder.queueInputBuffer(aacDecodeInputBuffersIndex, 0, sampleDataSize, presentationTimeUs, 0);
                    mediaExtractor.advance();
                } catch (Exception e) {
                    Log.e(TAG, "aac decode to pcm happened Exception : " + e.toString());
                    continue;
                }
            }
            int aacDecodeOutputBuffersIndex = aacDecoder.dequeueOutputBuffer(info, 2000);
            if (aacDecodeOutputBuffersIndex >= 0) {
                if (((info.flags & MediaCodec.BUFFER_FLAG_END_OF_STREAM) != 0)) {
                    Log.d(TAG, "aac decode thread read sample data done!");
                    break;
                } else {
                    ByteBuffer pcmOutputBuffer;
                    if (isLowVersion) {
                    
            pcmOutputBuffer = aacDecodeOutputBuffers[aacDecodeOutputBuffersIndex];
                    } else {
            pcmOutputBuffer = aacDecoder.getOutputBuffer(aacDecodeOutputBuffersIndex);
                    }
                    ByteBuffer copyBuffer = ByteBuffer.allocate(pcmOutputBuffer.remaining());
                    copyBuffer.put(pcmOutputBuffer);
                    copyBuffer.flip();
                    final byte[] pcm = new byte[info.size];
                    copyBuffer.get(pcm);
                    copyBuffer.clear();
                    audioTrack.write(pcm, 0, info.size);
                    aacDecoder.releaseOutputBuffer(aacDecodeOutputBuffersIndex, false);
                }
            }
        }        
    }           
    Log.d(TAG, "aac audio decode thread stop");        
    aacDecoder.stop();        
    aacDecoder.release();        
    aacDecoder = null;        
    mediaExtractor.release();        
    mediaExtractor = null;        
    isDecoding = false;        
    worker = null;    
}
```

上述代码片段稍长，作者就做一个简单的概括吧，先获取 **MediaCodec** 的输入输出缓冲区数组，然后读取文件的音频轨道数据填充到可用的输入缓冲区中，在进行音频的硬解码，最后从解码成功后存放的输出缓冲区数组中拿到解码后的 **PCM** 数据，通过 **AudioTrack** 播放出来，这个播放动作是为了验证解码出来的数据是否有异常

实时 AAC 音频文件的硬解码
---------------

实时的解码先不上代码，而是先帮助大家理解，我们需要怎么去解析 AAC 音频的 ADTS 头？要取哪些对我们有用的字节？别急，作者会细细的说

**FF F1 6C 40 18 02 3C**

上述数据是作者从实际项目开发中提取出来的 AAC 实时流的 ADTS 音频头，想通过这样的方式来解答之前提到的两个问题，这段字符表示 7 个 16 进制的字节，将其进行补全则为如下数据：

**0xFF 0xF1 0x6C 0x40 0x18 0x02 0x3C**

根据最前面提到的 ADTS 头结构可知，我们只需要关注 7 个字节中的前面个 4 字节，也就是 0~3 字节即可并取出其对应位的值用于生成解码器，所以我们只需要关心如下数据：

**0xFF 0xF1 0x6C 0x40**

然后接下来一一解析给大家看，首先是 **0xFF 0xF1**

```java
// 解析 0xFF 0xF1        
// 将第0字节0xFF和第1字节0xF1通过位运算，将其转换成int类型，即65521        
// 再将65521 转换成二进制类型，即 1111111111110001        
// syncword : 111111111111 (即固定0xfff) 12位        
// MPEG Version: 0 (表 MPEG-4) 1位        
// Layer: 00 (固定 0) 2位        
// protection absent : 1 (表无CRC数据) 1位
```

接下来再解析 **0x6C 0x40**，计算到前面低 10 位就行了，后续位用不上

```java
// 解析 0x6C 0x40        
// 将第2字节0x6C和第3字节0x40通过位运算，将其转换成int类型，即27712        
// 再将27712 转换成二进制类型，即 110110001000000，因不足16位，所以在高位补0，满足16位，补足后 0110110001000000        
// profile : 01 (aac profile) 2位        
// Sampling Frequency Index : 1011 (值为11，即采样率8000) 4位        
// private bit ：0 (编码时设为0 ，解码可忽略) 1位        
// Channel Configuration : 001 (通道参数) 3位        
// ......
```

结合作者刚刚举例的案例，也可以自己写出 ADTS 音频头解析，下面作者开始贴实时 AAC 音频硬解码的代码片段

```java
@Override    
public void start() {        
    if (worker == null) {
        isDecoding = true;
        waitTimeSum = 0;
        worker = new Thread(this, TAG);
        worker.start();        
    }    
}
```

上述代码片段为启动工作线程，开始进行 **MediaCodec** 硬解码操作

```java
@Override    
public void run() {
    final long timeOut = 5 * 1000;        
    final long waitTime = 500;        
    while (isDecoding) {
        while (!aacFrameQueue.isEmpty()) {
            byte[] aac = aacFrameQueue.poll();
            if (aac != null) {
                if (!hasAacDecoder(aac)) {
                    Log.d(TAG, "aac decoder create failure , so break!");
                    break;
                }
            // todo decode aac audio data.
            // remove aac audio adts header
            byte[] aacTemp = new byte[aac.length - 7];
            // data copy
            System.arraycopy(aac, 7, aacTemp, 0, aacTemp.length);
            // decode aac audio
            decode(aacTemp, aacTemp.length);
            }
        }
        // Waiting for next frame
        synchronized (decodeLock) {
            try {
                // isEmpty() may take some time, so we set timeout to detect next frame
                decodeLock.wait(waitTime);
                waitTimeSum += waitTime;
                if (waitTimeSum >= timeOut) {
                    Log.d(TAG, "realtime aac decode thread read timeout , so break!");
                    break;
                }
            } catch (InterruptedException ie) {
                worker.interrupt();
            }
        }        
    }        
    Log.d(TAG, "realtime aac decode thread stop!");        
    if (aacDecoder != null) {
        aacDecoder.stop();
        aacDecoder.release();
        aacDecoder = null;        
    }        
    if (audioTrack != null) {
        audioTrack.stop();
        audioTrack.release();
        audioTrack = null;       
    }        
    aacFrameQueue.clear();        
    adtsAudioHeader = null;        
    isDecoding = false;        
    worker = null;    
}
```

上述代码片段为线程执行解码，判断 AAC 队列中是否有数据，如果有就取出一个数据，先判空然后再检测 AAC 的 ADTS 音频头是否符合规范，如不符合或创建解码器发生异常都将直接退出循环结束线程工作，如果队列中没有数据则等待 500ms，继续轮询队列里的数据，当线程工作结束，释放相关 API 的资源，任何时候都要对相关的一些创建操作进行回收且**形成闭环**，避免发生**内存泄漏**！

```java
/**     
* put realtime aac audio data     *     
* @param aac aac audio data     
*/    
public void putAacData(byte[] aac) {        
    if (isDecoding) {
        aacFrameQueue.add(aac);
        synchronized (decodeLock) {
            waitTimeSum = 0;
            decodeLock.notifyAll();
        }        
    }    
}
```

上述代码为添加 AAC 实时数据到缓存队列中，这个数据可以是来自 TCP 等的实时流媒体数据，如果当前解码工作线程正在解码，则添加一个 AAC 到缓存队列中，重置等待时间并且唤醒等待中的对象锁，让线程拿到锁后继续执行

```java
/**     
* @param aac aac audio data     
* @return true means has aad decoder     
*/    
private boolean hasAacDecoder(byte[] aac) {        
    if (aacDecoder != null) {
        return true;        
    }        
    return checkAacAdtsHeader(aac);    
}
```

上述代码片段为校验 AAC 的 ADTS 音频头，如果 **accDecoder** 非空表示之前已经判断过，该 AAC 数据为正常 AAC 数据，这里不考虑极端情况，AAC 音频混搭其他格式的音频，这样会导致播放出问题，正常情况交互下也不会这样干！如果 **accDecoder** 为空则先对 ADTS 进行一次校验

```java
/**
* check aac adts audio header     *     
* @param aac aac audio data     
*/    
private boolean checkAacAdtsHeader(byte[] aac) {
    byte[] dtsFixedHeader = new byte[2];
    System.arraycopy(aac, 0, dtsFixedHeader, 0, dtsFixedHeader.length);
    int bitMoveValue = dtsFixedHeader.length * 8 - ADTS_HEADER_START_FLAG_BIT_SIZE;
    int adtsFixedHeaderValue = bytesToInt(dtsFixedHeader);
    int syncwordValue = ADTS_HEADER_START_FLAG << bitMoveValue;
    boolean isAdtsHeader = (adtsFixedHeaderValue & syncwordValue) >> bitMoveValue == ADTS_HEADER_START_FLAG;
    if (!isAdtsHeader) {
        Log.e(TAG, "adts header start flag not match , so return!");
        return false;
    }
    System.arraycopy(aac, 2, dtsFixedHeader, 0, dtsFixedHeader.length);
    return parseAdtsHeaderKeyData(dtsFixedHeader);    
}
```

上述代码片段为取出 AAC 音频中的第 0、1 两个字节，因为 short 双字节转换成 Int 不会造成精度丢失，先进行数据拷贝，完后计算数据 bit 的左右移动值，然后将第 0、1 两个字节转换成 int 类型，经过位运算得到 ADTS 的 **syncword** 即 AAC 的 ADTS 固定标识，如匹配不上表示不是 AAC 数据直接 return，否则接着处理第 2、3 两个字节然后将其拷贝到数组中，接着再进行 ADTS 音频头的关键数据的解析

```java
/**     
* parse adts header key byte array data     *     
* @param adtsHeaderValue adts fixed header byte array     
*/    
private boolean parseAdtsHeaderKeyData(byte[] adtsHeaderValue) {
    int adtsFixedHeaderValue = bytesToInt(adtsHeaderValue);
    // bitMoveValue = 16(2 * 8) - 2(aac profile 3bit)
    int bitMoveValue = adtsHeaderValue.length * 8 - ADTS_HEADER_PROFILE_BIT_SIZE;
    // profile : 01 (aac profile) 2 bit
    int audioProfile = adtsFixedHeaderValue & (ADTS_HEADER_PROFILE_FLAG << bitMoveValue);
    // 1: AAC Main -- MediaCodecInfo.CodecProfileLevel.AACObjectMain
    // 2: AAC LC (Low Complexity)  -- MediaCodecInfo.CodecProfileLevel.AACObjectLC
    // 3: AAC SSR (Scalable Sample Rate) -- MediaCodecInfo.CodecProfileLevel.AACObjectSSR
    audioProfile = audioProfile >> bitMoveValue;
    // bitMoveValue = 16(2 * 8) - 2(aac profile 3bit) - 4(Sampling Frequency Index 4 bit)
    bitMoveValue -= ADTS_HEADER_SAMPLE_INDEX_BIT_SIZE;
    // Sampling Frequency Index : 1011 (value is 11，sample rate 8000) 4 bit
    int sampleIndex = adtsFixedHeaderValue & (ADTS_HEADER_SAMPLE_INDEX_FLAG << bitMoveValue);
    sampleIndex = sampleIndex >> bitMoveValue;
    sampleRate = samplingFrequencys[sampleIndex];
    // private bit ：0 (encoding set 0 ，decoding ignore) 1 bit
    // Channel Configuration : 001 (Channel Configuration) 3 bit
    // bitMoveValue = bitMoveValue - 1(private bit 1bit) +  3(Channel Configuration 3bit)
    bitMoveValue -= (1 + ADTS_HEADER_CHANNEL_CONFIG_BIT_SIZE);
    channelConfig = adtsFixedHeaderValue & (ADTS_HEADER_SAMPLE_INDEX_FLAG << bitMoveValue);
    channelConfig = channelConfig >> bitMoveValue;
    // ......
    // create csd-0(audio adts header)
    adtsAudioHeader = new byte[2];
    adtsAudioHeader[0] = (byte) ((audioProfile << 3) | (sampleIndex >> 1));
    adtsAudioHeader[1] = (byte) ((byte) ((sampleIndex << 7) & 0x80) | (channelConfig << 3));
    Log.d(TAG, "audioProfile = " + audioProfile + " , sampleIndex = " + sampleIndex + "(" + sampleRate + ")" + " ,     channelConfig = " + channelConfig
    + " , audio csd-0 = " + Utils.bytesToHexStringNo0xChar(adtsAudioHeader));
    return createDefaultDecoder();    
}
```

上述代码片段为先将前面取到的 AAC 的第 2、3 字节转换成 int 类型，接着计算数据位的位移值，然后分别计算 **audioProfile**、**sampleIndex**、**sampleRate**、**channelConfig**，最后再根据这些参数中的部分参数进行音频 **csd-0** 配置头的计算，如果一切正常，最后会调用 **createDefaultDecoder** 方法进行解码器的创建

```java
/**     
* create default decoder     
*/    
private boolean createDefaultDecoder() {
    if (adtsAudioHeader == null || adtsAudioHeader.length == 0) {
        Log.e(TAG, "realtime aac decoder create failure , adts audio header is null , so return false!");
        return false;
    }
    try {
        aacDecoder = MediaCodec.createDecoderByType(AUDIO_DECODE_MIME_TYPE);
        MediaFormat mediaFormat = new MediaFormat();
        mediaFormat.setString(MediaFormat.KEY_MIME, AUDIO_DECODE_MIME_TYPE);
        mediaFormat.setInteger(MediaFormat.KEY_SAMPLE_RATE, sampleRate);
        mediaFormat.setInteger(MediaFormat.KEY_CHANNEL_COUNT, channelConfig);
        ByteBuffer byteBuffer = ByteBuffer.allocate(adtsAudioHeader.length);
        byteBuffer.put(adtsAudioHeader);
        byteBuffer.flip();
        mediaFormat.setByteBuffer(CSD_MIME_TYPE_0, byteBuffer);
        aacDecoder.configure(mediaFormat, null, null, 0);
    } catch (IOException e) {
        Log.e(TAG, "realtime aac decoder create failure , happened exception : " + e.toString());
        if (aacDecoder != null) {
            aacDecoder.stop();
            aacDecoder.release();
        }
        aacDecoder = null;
    }
    if (aacDecoder == null) {
        return false;
    }
    // initialization audio track , use for play pcm audio data
    // audio output channel param channelConfig according device support select
    int buffsize = AudioTrack.getMinBufferSize(sampleRate, channelConfig, AudioFormat.ENCODING_PCM_16BIT);
    // author using channelConfig is AudioFormat.CHANNEL_OUT_MONO
    audioTrack = new AudioTrack(AudioManager.STREAM_MUSIC, sampleRate, channelConfig,
    AudioFormat.ENCODING_PCM_16BIT, buffsize, AudioTrack.MODE_STREAM);
    audioTrack.play();
    aacDecoder.start();
    return true;    
}
```

上述代码片段为如果前面代码的音频 **csd-0** 配置头不合法则直接 return，接着就是把之前步骤解析 AAC 音频头得到的参数设置到解码器当中，如发生异常则 return false，否则创建一个 **AudioTrack** 进行解码音频数据后的播放，验证数据是否正常被解析

``` java
/*** aac audio data decode     
** @param buf    aac audio data     
* @param length aac audio data length     
*/    
private void decode(byte[] buf, int length) {
    try {
        ByteBuffer[] codecInputBuffers = aacDecoder.getInputBuffers();
        ByteBuffer[] codecOutputBuffers = aacDecoder.getOutputBuffers();
        long kTimeOutUs = 0;
        int inputBufIndex = aacDecoder.dequeueInputBuffer(kTimeOutUs);
        if (inputBufIndex >= 0) {
            ByteBuffer dstBuf = codecInputBuffers[inputBufIndex];
            dstBuf.clear();
            dstBuf.put(buf, 0, length);
            aacDecoder.queueInputBuffer(inputBufIndex, 0, length, 0, 0);
        }
        ByteBuffer outputBuffer;
        MediaCodec.BufferInfo info = new MediaCodec.BufferInfo();
        int outputBufferIndex = aacDecoder.dequeueOutputBuffer(info, kTimeOutUs);
        while (outputBufferIndex >= 0) {
            outputBuffer = codecOutputBuffers[outputBufferIndex];
            byte[] outData = new byte[info.size];
            outputBuffer.get(outData);
            outputBuffer.clear();
            if (audioTrack != null) {
                audioTrack.write(outData, 0, info.size);
            }
            aacDecoder.releaseOutputBuffer(outputBufferIndex, false);
            outputBufferIndex = aacDecoder.dequeueOutputBuffer(info, kTimeOutUs);
        }        
    } catch (Exception e) {
        Log.e(TAG, "realtime aac decode happened exception : " + e.toString());        
    }    
}
```

最后代码片段就是介绍 **MediaCodec** 的硬解码，这里就大概描述一下，因为跟本地音视频解码是一样的流程了，首先是获取输入输出缓冲区数组，然后将要解码的 AAC 音频数据填充到可用的输入缓冲区中，具体那个输入缓冲区可用，可以调用 **dequeueInputBuffer** 方法，该方法返回值大于等于 0 时表示该返回值对应的输入缓冲区数组的索引，然后就是解码了，从输出缓冲区数组中取得已经解码成功的输出缓冲区，可以调 **dequeueOutputBuffer** 方法获取它的有效索引，最后就是取出解码后的 **PCM** 数据进行播放，播放完毕后释放索引对应的输出缓冲区

-----------

[img-0]:data:image/png;base64,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K4gbPeiFFF6XpnE1lsL3sPf8qoPT5lStcFx6bJTI0fKrTRMR+fjN76Ptf2k424C9XysPfzkptZ4+TwFadMXHgrXYlhds5UY6PLV+TtZ6+oioxhnDjfXHu13deuiaexsPQ2oAlsohZUA82kcKQO1zO4PM6XQ83973ebHW5/yeLywWxioi/VMJ8T9EUK1i+j9hs+3vxBMWXEG1FICbnz/Y8+0r1guUBCaubOG+sSaL0YPvaIz33s5JHSe/+b2o2DtfPm9Y/yGi/WMgfFQCBcIKem7dAJ5XcwO3nvtrFARKYhxjbe8OrdWbKqv1c7LW00dERF72Uy8gIqL9ONKh0VSG3kpdTC/hflWFnT63ZlQaGlRFayoRERHVqgJePHh40okgIqoL7BEm2odaPydrPX1ERERERCfpr046AURERERERETHiYEwERERERER1RUGwkRERERERFRXGAgTERERERFRXWEgTERERERERHWFgTARERERERHVFQbCREREREREVFcYCBMREREREVFdYSBMREREREREdYWBMBEREREREdWV3/w+2v6XhrMNJ50OIiIiIiIiomPxW+Mff/rzn046Lb4++91nNZ0+qj+1fk7WevqIiIiIiE4Sh0YTERERERFRXWEgTERERERERHWFgTARERERERHVFQbCREREREREVFcYCBMREREREVFdYSBMREREREREdYWBMBEREREREdUVBsJERERERERUVxgIExERERERUV1hIExERERERER1hYEwERERERER1RUGwkRERERERFRXfnvSCSAiIqIa0tSN0eGz+Pnuc7yVXm7u/grDHR+RcrwONOHyyBA6QvbN5PN5tLa2evxA3mMbZROEyyNDaPj5Hr7ZMF6L4ubta0BKei3ah6mE8/dKWH/8NV7tyPumIhTod4G8tP32/km0v3X8XvsG7jzbCLi1Q+ZxnNr7J5HwyPLS+jIW13ACx0lOq4qPKY/3qtTeP4mLe8tYXNs52IaIqO4dXiAc7cPUxV/xeGkN2ye9V6eWR8Hvp6kbo8MRvJcrALZNlT9e7f2TSOClTwFfRTrsifIvJANnwSk9z07rfhHRJ6f9SxGwvPUMcENI3J5EwvgzL8qJV0v38AoewaJLFDdHzun3Oa/tBw2+NvDNXeDm7UnchPR7ebnciuLm7S88vmv9hi29tvuwXl5J33r77CXanb8X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VcL6f/mIjv+jFdUqrf9nvD//b8Uzb11luWOxR1RaZKuyij25Td0Y/fIDFvWy3uzBN49Fue2c1HHym9+LqjpM3PsUbOVqIiIvDISPWKBAONqHqfaN6gJY5xBkVF6wgw5PrZ+TtZ4+IiLy0oTLI0ngOwZ2RERHjXOET5TRkppHqtrl/32GIDMIJiIi+lTt4NXS1yedCCKiusAeYaJ9qPVzstbTR0RERER0kv7qpBNAREREREREdJwYCBMREREREVFdYSBMREREREREdYWBMBEREREREdUVBsJERERERERUVxgIExERERERUV1hIExERERERER1hYEwERERERER1RUGwkRERERERFRXGAgTERERERFRXfmt8Y/PfvfZSaelrFpPH9WfWj8naz19REREREQnxQyE937dO+m0+Go421DT6aP6U+vnZK2nj4iIiIjoJHFoNBEREREREdUVBsJERERERERUVxgIExERERERUV1hIExERERERER1hYEwERERERER1RUGwkRERERERFRXGAgTERERERFRXWEgTERERERERHWFgTARERERERHVFQbCREREREREVFcYCBMREREREVFdYSBMREREREREdeVfNDU3/8fPfvcZ/vznP590WnzVevqo/tT6OXlc6VMHp3EVb5AtAICCnlv/Hl/8s/G3ByWBif/wb/DPr3OwPhLge7ZNjOPf/c3/wH/9p/8ZLJFKAhN/9xm0lhuY/Psr+NuuLu//Yn+Nf/yHf8L/lH5n8ot/xutcgER5/zB6bv17/P3VGP76H/8B/z1gcgOLDWD2CvC60ImJ/3AT//pf/rfgeeK9QQzNjKCryu1Um09en7efR17puoEWZx56nkv+76uD0xhLeh971bHPrnOs0m8F2Y+A2zDPm39tPx+JapW4tv4VShXPbaIqxAYwO9rrW0bXGnVwGmNq6QB1htp02q/v3550AoiofiidFxDW3qHl1jSSYcebyWnMJq0/i+kl3E+5b7uF1EO8HpzG7K0MHj1I6TdmBT23bgDPH+KF/BUlgYnRC9hcTKFQyGEuJb3eBzwzv+8Uw9V4GNpKbp97GsPQTC8i2irmXscwOzoOLDrSBoiCPhnx+H4Raa/PeymkcH8+BXVwGkOxBSwbSVYSmBjthJHN2or0ngd1sBeRYhHF+A30ZAL+NoBC6lukb41gIlHwPF7e+9iL2Zle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2fZ3ns/h9s+LsPk9+XFXvotlSemWn39b/t7i9zgtI1++Bfo8Vq12Kvf82wDnjS0I9JxPa6wfYJyb5Y6bx0q4fvO8y7BGuojf2P/56bvXjganIHOtq+xhrzXmcSh4Pi/eYJtqY5wPK0DSWPjtgceK4Eb+S8/OdeWRWSZkEJfz/oBr5PjPET95h1VHOQ0YCBPtQ62fk7WePqKTVFOLP1UtyDNmPw2H0ttFnxwedyKqFQyEifah1s/JWk8fEREREdFJ+quTTgARERERERHRcWIgTERERERERHWFgTARERERERHVFQbCREREREREVFcYCBMREREREVFdYSBMREREREREdYWBMBEREREREdUVBsJERERERERUVxgIExERERERUV1hIExERERERER1hYEwERERERER1ZXf/D7a/peGsw0nnQ4iIiIiIiKiY/Fb4x9/+vOfTjotvj773Wc1nT6qP7V+TtZ6+oiIiIiIThKHRhMREREREVFdYSBMREREREREdYWBMBEREREREdUVBsJERERERERUVxgIExERERERUV1hIExERERERER1hYEwERERERER1RUGwkRERERERFRXGAgTERERERFRXWEgTERERERERHWFgTARERERERHVFQbCREREREREVFd+e9IJICIiau7+CtexgsW1neBfaurG6LCKUMUPlrD++Gu8qmLT9Olq75/Exb3l4OdStA9TiVaU1pex+OEPmEq02t7Op+7hmw3jryZcHkkC31nn077O3SMUJD3N3V9huKPylQNA5Iu5rShu3o7i7d3neGv+/QX2Hn+NV819mEoAKfM9J3ve2dLZ1I3R4bP42fe7nxopXwKdFh6fb+rG6HXgu6U1bHt8o7n7Kww3/BF3nm3YXvc9/+XtRfsw1b6BO8820N4/ifa39/DNhvPY2r4sHbsobt6+hlb44f2WPh2HGAg34fLIEGz31fxL1wVKRIBvIalXyAz2CgjR6dXcEMLHt/Zz3bOyXsrisVwxdPztDgL0Ctxx7otPBbV6jvtEUzdGhyN4f+yVTFHxhS0gPK6f7sPUxV/tx7yCt89eov32EG5+8E+v/dzKI3X3nqj8R2E7p9r7J9Fu++YOXn2nYXT4K1yu6ji460j5g+ano7ywG8JUh/tV4ze3177GnbUD/LaZj1+gFSG0Dk/C+LnE7UkkzB98iRSuwZZM6bPOdJrf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

