---
title: 【翻译】音频文件和编码格式
date: 2016-10-09 12:42:42
tags:
---

此文章翻译自 [RayWenderlich](https://www.raywenderlich.com/69365/audio-tutorial-ios-file-data-formats-2014-edition), 作为对一个音频文件基础知识的补充。

一个音频文件包含两部分：文件格式和音频编码格式，文件格式顾名思义就是音频文件本身的格式，而这个音频文件内部的音频数据也可以通过多种不同的编码方式进行编码。比如我们有一个音频文件叫 xxx.caf ,它本身的文件格式是 CAF 格式，但是这个音频文件可以包含不同编码的音频格式，比如 MP3，liner PCM 等。

<!-- more -->

### 音频文件格式

拿 iPhone 支持的音频文件格式举例，它支持多种格式包括 MPEG-1（文件扩展名为 .mp3）、MPEG-2 ADTS（文件扩展名为 .aac）、AIFF、CAF 和 WAVE. iPhone 上比较推荐使用的文件格式是 CAF ，因为它几乎可以支持任意一种音频编码。

#### 比特率

比特率指的是单位时间播放连续的媒体的比特数量，使用“比特每秒（bit/s 或 bps）”或者“千比特每秒（kbit/s 或 kbps）”，他们之间的换算关系是 1 kbit/s = 1000 bit/s 而不是 1024 bit/s 。比特率越高，则音频文件的大小越大，减小一个音频文件的比特率也就意味着这个音频文件的音频质量会下降。

一些常见的比特率的应用场景如下：

* 32 kbit/s：AM 质量，一般只用于语音
* 48-64 kbit/s：一般用于 Podcast 音频
* 96 kbit/s：FM 质量，一般用于语音或者低质量流媒体
* 128-160 kbit/s：中档质量音频
* 256 kbit/s：常用的高质量比特率之一
* 320 kbit/s：MP3 标准支持的最高比特率，从 CD 进行播放的音频比特率
* 500-1411 kbit/s：无损音频，比如 linear PCM 编码的音频

### 音频编码格式

iPhone 支持的音频编码格式如下：

* MP3：作为一个有着多年被使用的有损压缩格式，MP3 是最被人们所熟知的编码格式。
* AAC：AAC 全称为“Advanced Audio Coding（高级音频编码）”，作为 MP3 的继任者被设计出来，比 MP3 拥有者更高的压缩比，更高的采样率和采样精度，以及更高的解码效率。
* HE-AAC：是 AAC 的超集，HE 代表的是“更高性能的”，更适合使用在低比特率的场景下，比如一些流媒体当中。
* AMR：全称为 “Adaptive multi-Rate compression（自适应多速率音频压缩）”一般用于低比特率的语音中。
* ALAC：作为苹果的无所音频压缩编码格式，可将非压缩音频压缩至原容量的 40% 至 60% 左右，编解码速度很快，且仍保持无损压缩，常被用于 iPod 中。
* iLBC：是另外一种为语音优化的编码格式，常被用于 IP 语音通话和流媒体中。
* IMA4：这是一个在16-bit音频文件下按照4：1的压缩比来进行压缩的格式。
* liner PCM：对数据不进行压缩，是 iPhone 上首选的编码格式，当然不压缩则意味着需要更大的空间。line PCM 编码的文件格式一般有 WAV、APE、FLAC 三种。

### 如何选用

如果选用 linear PCM、IMA4 或者其他一些非压缩或者轻度压缩的编码格式，那么播放起来没有任何问题。
如果选用压缩率较高的 AAC、MP3 等能够支持硬解的编码格式，那每次解码时只支持一个文件，如果需要同时播放多个需要解码的文件就只能通过软解，速度会变慢。

所以如何选择编码格式只需要基于存储空间进行考量即可，存储空间足够的话最好使用 Linear PCM，播放快且可同时播放多个音频还不占用 CPU 资源，如果存储空间有要求的话，最好使用 AAC 作为音乐播放的编码，选择 IMA4 作为系统声音的编码。
