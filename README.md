# 个人学习笔记


## Markdown的用法

```markdown
高亮区域代码
```
# 标题一    
## 标题二  
### 标题三

- Bulleted
- List

1. Numbered
2. List

**粗体** and _斜体_ and `编码` text
```
[Link](url) and ![Image](src)
```

## FFmpeg的使用流程


- 编码流程
![编码流程](https://upload-images.jianshu.io/upload_images/2262256-6636249a310bea3f.png?imageMogr2/auto-orient/strip|imageView2/2/w/569/format/webp)

1. 编码步骤：

  (1) av_register_all()：注册FFmpeg 的H265编码器。调用了avcodec_register_all()，avcodec_register_all()注册了H265编码器有关的组件：硬件加速器，编码器，Parser，Bitstream Filter等
  
  (2) avformat_alloc_output_context2()：初始化输出码流的AVFormatContext，获取输出文件的编码格式；
  
  (3) avio_open()：打开输出文件，调用了2个函数：ffurl_open()和ffio_fdopen()。其中ffurl_open()用于初始化URLContext，ffio_fdopen()用于根据URLContext初始化AVIOContext。URLContext中包含的URLProtocol完成了具体的协议读写等工作。AVIOContext则是在URLContext的读写函数外面加上了一层“包装”；
  
  (4) av_new_stream()：创建输出码流的AVStream结构体，为输出文件设置编码所需要的参数和格式；
  
  (5) avcodec_find_encoder()：通过 codec_id查找H265编码器
  
```c++

HEVC解码器对应的AVCodec结构体ff_hevc_decoder：
AVCodec ff_hevc_decoder = {
    .name                  = "hevc",
    .long_name             = NULL_IF_CONFIG_SMALL("HEVC (High Efficiency Video Coding)"),
    .type                  = AVMEDIA_TYPE_VIDEO,
    .id                    = AV_CODEC_ID_HEVC,
    .priv_data_size        = sizeof(HEVCContext),
    .priv_class            = &hevc_decoder_class,
    .init                  = hevc_decode_init,
    .close                 = hevc_decode_free,
    .decode                = hevc_decode_frame,
    .flush                 = hevc_decode_flush,
    .update_thread_context = hevc_update_thread_context,
    .init_thread_copy      = hevc_init_thread_copy,
    .capabilities          = AV_CODEC_CAP_DR1 | AV_CODEC_CAP_DELAY |
                             AV_CODEC_CAP_SLICE_THREADS | AV_CODEC_CAP_FRAME_THREADS,
    .caps_internal         = FF_CODEC_CAP_INIT_THREADSAFE | FF_CODEC_CAP_EXPORTS_CROPPING,
    .profiles              = NULL_IF_CONFIG_SMALL(ff_hevc_profiles),
    .hw_configs            = (const AVCodecHWConfigInternal*[]) {
#if CONFIG_HEVC_DXVA2_HWACCEL
                               HWACCEL_DXVA2(hevc),
#endif
#if CONFIG_HEVC_D3D11VA_HWACCEL
                               HWACCEL_D3D11VA(hevc),
#endif
#if CONFIG_HEVC_D3D11VA2_HWACCEL
                               HWACCEL_D3D11VA2(hevc),
#endif
#if CONFIG_HEVC_NVDEC_HWACCEL
                               HWACCEL_NVDEC(hevc),
#endif
#if CONFIG_HEVC_VAAPI_HWACCEL
                               HWACCEL_VAAPI(hevc),
#endif
#if CONFIG_HEVC_VDPAU_HWACCEL
                               HWACCEL_VDPAU(hevc),
#endif
#if CONFIG_HEVC_VIDEOTOOLBOX_HWACCEL
                               HWACCEL_VIDEOTOOLBOX(hevc),
#endif
                               NULL
                             
```
  (6) avcodec_open2()：打开编码器。调用AVCodec的libx265_encode_init()初始化H265解码器 avcodec_open2()函数；`avcodec_open2() -> libx265_encode_init() -> x265_param_alloc(), x265_param_default_preset(), x265_encoder_open()`
  
  (7) avformat_write_header()：写入编码的H265码流的文件头；
  
  (8) avcodec_encode_video2()：编码一帧视频。将AVFrame（存储YUV像素数据）编码为AVPacket（存储H265格式的码流数据）。调用H265编码器的libx265_encode_frame()函数；`avcodec_encode_video2() -> libx265_encode_frame() -> x265_encoder_encode()`
  
  (9) av_write_frame()：将编码后的视频码流写入文件中；
  
  (10) flush_encoder()：输入的像素数据读取完成后调用此函数，用于输出编码器中剩余的AVPacket；
  
  (11) av_write_trailer()：写入编码的H265码流的文件尾；
  
  (12) close():释放 AVFrame和图片buf，关闭H265编码器，调用AVCodec的libx265_encode_close()函数 `avcodec_close() -> libx265_encode_close() -> x265_param_free(), x265_encoder_close()`
  
3. 示例代码：
```c++
**
 * 基于FFmpeg的视频编码器
 * 功能：实现了YUV420像素数据编码为视频码流（H264，H265,MPEG2，VP8）。
 * ffmpeg编码yuv文件的命令:
 * H264:ffmpeg -s cif -i foreman_cif.yuv -vcodec libx264 -level 40 -profile baseline -me_method epzs -qp 23 -i_qfactor 1.0  -g 12 -refs 1 -frames 50 -r 25 output.264 
 * H265:ffmpeg -s cif -foreman_cif.yuv -vcodec libx265  -frames 100  output.265
 */
 
#include <stdio.h>
 
#define __STDC_CONSTANT_MACROS
 
#ifdef _WIN32
//Windows
extern "C"
{
#include "libavutil/opt.h"
#include "libavcodec/avcodec.h"
#include "libavformat/avformat.h"
};
#else
//Linux...
#ifdef __cplusplus
extern "C"
{
#endif
#include <libavutil/opt.h>
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#ifdef __cplusplus
};
#endif
#endif
 
//H.265码流与YUV输入的帧数不同。经过观察对比其他程序后发现需要调用flush_encoder()将编码器中剩余的视频帧输出。当av_read_frame()循环退出的时候，实际上解码器中可能还包含剩余的几帧数据。
//因此需要通过“flush_decoder”将这几帧数据输出。“flush_decoder”功能简而言之即直接调用avcodec_decode_video2()获得AVFrame，而不再向解码器传递AVPacket
int flush_encoder(AVFormatContext *fmt_ctx, unsigned int stream_index){
	int ret;
	int got_frame;
	AVPacket enc_pkt;
 
	if (!(fmt_ctx->streams[stream_index]->codec->codec->capabilities &
		CODEC_CAP_DELAY))
		return 0;
	while (1) {
		enc_pkt.data = NULL;
		enc_pkt.size = 0;
		av_init_packet(&enc_pkt);
		ret = avcodec_encode_video2(fmt_ctx->streams[stream_index]->codec, &enc_pkt,
			NULL, &got_frame);
		av_frame_free(NULL);
		if (ret < 0)
			break;
		if (!got_frame){
			ret = 0;
			break;
		}
		printf("Flush Encoder: Succeed to encode 1 frame!\tsize:%5d\n", enc_pkt.size);
		/* mux encoded frame */
		ret = av_write_frame(fmt_ctx, &enc_pkt);
		if (ret < 0)
			break;
	}
	return ret;
}
 
int main(int argc, char* argv[])
{
	AVFormatContext* pFormatCtx = NULL;
	AVOutputFormat* fmt;
	AVStream* video_st;
	AVCodecContext* pCodecCtx;
	AVCodec* pCodec;
	AVPacket pkt;
	uint8_t* picture_buf;
	AVFrame* pFrame;
	int picture_size;
	int y_size;
	int framecnt = 0;
	FILE *in_file = fopen("chezaiyundong_1280x720_30_300.yuv", "rb");
	int in_w = 1280, in_h = 720;
	int framenum = 10;
	const char* out_file = "chezaiyundong_1280x720_30_300.hevc";
 
	av_register_all();//注册FFmpeg所有编解码器
	avformat_alloc_output_context2(&pFormatCtx, NULL, NULL, out_file);//初始化输出码流的AVFormatContext(获取输出文件的编码格式)
	fmt = pFormatCtx->oformat;
 
	// 打开文件的缓冲区输入输出，flags 标识为  AVIO_FLAG_READ_WRITE ，可读写;将输出文件中的数据读入到程序的 buffer 当中，方便之后的数据写入fwrite
	if (avio_open(&pFormatCtx->pb, out_file, AVIO_FLAG_READ_WRITE) < 0){
		printf("Failed to open output file! \n");
		return -1;
	}
	video_st = avformat_new_stream(pFormatCtx, 0);//创建输出码流的AVStream。
	// 设置 码率25 帧每秒(fps=25)
	video_st->time_base.num = 1;
	video_st->time_base.den = 25;
	if (video_st == NULL){
		return -1;
	}
 
	//为输出文件设置编码的参数和格式
	pCodecCtx = video_st->codec;// 从媒体流中获取到编码结构体，一个 AVStream 对应一个  AVCodecContext
	pCodecCtx->codec_id = fmt->video_codec;// 设置编码器的 id，例如 h265 的编码 id 就是 AV_CODEC_ID_H265
	pCodecCtx->codec_type = AVMEDIA_TYPE_VIDEO;//编码器视频编码的类型
	pCodecCtx->pix_fmt = AV_PIX_FMT_YUV420P;//设置像素格式为 yuv 格式
	pCodecCtx->width = in_w; //设置视频的宽高
	pCodecCtx->height = in_h;
	pCodecCtx->time_base.num = 1;
	pCodecCtx->time_base.den = 25;
	pCodecCtx->bit_rate = 400000;  //采样的码率；采样码率越大，视频大小越大
	pCodecCtx->gop_size = 250;//每250帧插入1个I帧，I帧越少，视频越小
	pCodecCtx->qmin = 10;最大和最小量化系数 
	//（函数输出的延时仅仅跟max_b_frames的设置有关，想进行实时编码，将max_b_frames设置为0便没有编码延时了）
	pCodecCtx->max_b_frames = 3;// 设置 B 帧最大的数量，B帧为视频图片空间的前后预测帧， B 帧相对于 I、P 帧来说，压缩率比较大，采用多编码 B 帧提高清晰度
 
	//设置编码速度
	AVDictionary *param = 0;
	//preset的参数调节编码速度和质量的平衡。
	//tune的参数值指定片子的类型，是和视觉优化的参数，
	//zerolatency: 零延迟，用在需要非常低的延迟的情况下，比如电视电话会议的编码
	if (pCodecCtx->codec_id == AV_CODEC_ID_H264) {
		av_dict_set(¶m, "preset", "slow", 0);
		av_dict_set(¶m, "tune", "zerolatency", 0);
		//av_dict_set(¶m, "profile", "main", 0);
	}
	//H.265
	if (pCodecCtx->codec_id == AV_CODEC_ID_H265){
		av_dict_set(¶m, "preset", "ultrafast", 0);
		av_dict_set(¶m, "tune", "zero-latency", 0);
	}
 
	//输出格式的信息，例如时间，比特率，数据流，容器，元数据，辅助数据，编码，时间戳
	av_dump_format(pFormatCtx, 0, out_file, 1);
 
	pCodec = avcodec_find_encoder(pCodecCtx->codec_id);//查找编码器
	if (!pCodec){
		printf("Can not find encoder! \n");
		return -1;
	}
	// 打开编码器，并设置参数 param
	if (avcodec_open2(pCodecCtx, pCodec, ¶m) < 0){
		printf("Failed to open encoder! \n");
		return -1;
	}
 
	//设置原始数据 AVFrame
	pFrame = av_frame_alloc();
	if (!pFrame) {
		printf("Could not allocate video frame\n");
		return -1;
	}
	pFrame->format = pCodecCtx->pix_fmt;
	pFrame->width = pCodecCtx->width;
	pFrame->height = pCodecCtx->height;
 
	// 获取YUV像素格式图片的大小
	picture_size = avpicture_get_size(pCodecCtx->pix_fmt, pCodecCtx->width, pCodecCtx->height);
	// 将 picture_size 转换成字节数据
	picture_buf = (uint8_t *)av_malloc(picture_size);
	// 设置原始数据 AVFrame 的每一个frame 的图片大小，AVFrame 这里存储着 YUV 非压缩数据
	avpicture_fill((AVPicture *)pFrame, picture_buf, pCodecCtx->pix_fmt, pCodecCtx->width, pCodecCtx->height);
	//写封装格式文件头
	avformat_write_header(pFormatCtx, NULL);
	//创建编码后的数据 AVPacket 结构体来存储 AVFrame 编码后生成的数据  //编码前：AVFrame  //编码后：AVPacket
	av_new_packet(&pkt, picture_size);
	// 设置 yuv 数据中Y亮度图片的宽高，写入数据到 AVFrame 结构体中
	y_size = pCodecCtx->width * pCodecCtx->height;
 
	for (int i = 0; i<framenum; i++){
		//Read raw YUV data
		if (fread(picture_buf, 1, y_size * 3 / 2, in_file) <= 0){
			printf("Failed to read raw data! \n");
			return -1;
		}
		else if (feof(in_file)){
			break;
		}
		pFrame->data[0] = picture_buf;              // 亮度Y
		pFrame->data[1] = picture_buf + y_size;      // U 
		pFrame->data[2] = picture_buf + y_size * 5 / 4;  // V
		//顺序显示解码后的视频帧
		pFrame->pts = i;
		// 设置这一帧的显示时间
		//pFrame->pts=i*(video_st->time_base.den)/((video_st->time_base.num)*25);
		int got_picture = 0;
		int ret = avcodec_encode_video2(pCodecCtx, &pkt, pFrame, &got_picture);//编码一帧视频。即将AVFrame（存储YUV像素数据）编码为AVPacket（存储H.264等格式的码流数据）
		if (ret < 0){
			printf("Failed to encode! \n");
			return -1;
		}
 
		if (got_picture == 1){
			printf("Succeed to encode frame: %5d\tsize:%5d\n", framecnt, pkt.size);
			framecnt++;
			pkt.stream_index = video_st->index;
			printf("video_st->index = %d\n", video_st->index);
			av_write_frame(pFormatCtx, &pkt);//将编码后的视频码流写入文件（fwrite）
			av_free_packet(&pkt);//释放内存
		}
	}
 
	//输出编码器中剩余的AVPacket
	int ret = flush_encoder(pFormatCtx, 0);
	if (ret < 0) {
		printf("Flushing encoder failed\n");
		return -1;
	}
 
	// 写入数据流尾部到输出文件当中，表示结束并释放文件的私有数据
	av_write_trailer(pFormatCtx);
 
	if (video_st){
		// 关闭编码器
		avcodec_close(video_st->codec);
		// 释放 AVFrame
		av_free(pFrame);
		// 释放图片 buf
		av_free(picture_buf);
	}
	// 关闭输入数据的缓存
	avio_close(pFormatCtx->pb);
	// 释放 AVFromatContext 结构体
	avformat_free_context(pFormatCtx);
	// 关闭输入文件
	fclose(in_file);
 
	return 0;
}
`
```

- 解码流程![FFMpeg解码流程](https://img-blog.csdnimg.cn/20190131151319528.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0ZQR0FUT00=,size_16,color_FFFFFF,t_70#pic_center)

- 解码步骤：
1. 注册组件 
`av_register_all()`    例如：编码器、解码器等等…
2. 打开封装格式->打开文件
`avformat_open_input()` 例如：.mp4、.mov、.wmv文件等等...
3. 查看视频流
`avformat_find_stream_info()` 如果是视频解码，那么查找视频流，如果是音频解码，那么就查找音频流
4. 查找视频解码器
 -(1)、查找视频流索引位置
 -(2)、根据视频流索引，获取解码器上下文
 -(3)、根据解码器上下文，获得解码器ID，然后查找解码器
5. 打开解码器  
`avcodec_open2()`
6. 读取视频压缩数据->循环读取
7. 视频解码->播放视频->得到视频像素数据
8. 关闭解码器->解码完成
-示例代码：
```c++
/*
 * Copyright (c) 2001 Fabrice Bellard
 *
 * Permission is hereby granted, free of charge, to any person obtaining a copy
 * of this software and associated documentation files (the "Software"), to deal
 * in the Software without restriction, including without limitation the rights
 * to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
 * copies of the Software, and to permit persons to whom the Software is
 * furnished to do so, subject to the following conditions:
 *
 * The above copyright notice and this permission notice shall be included in
 * all copies or substantial portions of the Software.
 *
 * THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
 * IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
 * FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL
 * THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
 * LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
 * OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
 * THE SOFTWARE.
 */

/**
 * @file
 * video decoding with libavcodec API example
 *
 * @example decode_video.c
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include <libavcodec/avcodec.h>

#define INBUF_SIZE 4096

static void pgm_save(unsigned char *buf, int wrap, int xsize, int ysize,
                     char *filename)
{
    FILE *f;
    int i;

    f = fopen(filename,"w");
    fprintf(f, "P5\n%d %d\n%d\n", xsize, ysize, 255);
    for (i = 0; i < ysize; i++)
        fwrite(buf + i * wrap, 1, xsize, f);
    fclose(f);
}

static void decode(AVCodecContext *dec_ctx, AVFrame *frame, AVPacket *pkt,
                   const char *filename)
{
    char buf[1024];
    int ret;

    ret = avcodec_send_packet(dec_ctx, pkt);
    if (ret < 0) {
        fprintf(stderr, "Error sending a packet for decoding\n");
        exit(1);
    }

    while (ret >= 0) {
        ret = avcodec_receive_frame(dec_ctx, frame);
        if (ret == AVERROR(EAGAIN) || ret == AVERROR_EOF)
            return;
        else if (ret < 0) {
            fprintf(stderr, "Error during decoding\n");
            exit(1);
        }

        printf("saving frame %3d\n", dec_ctx->frame_number);
        fflush(stdout);

        /* the picture is allocated by the decoder. no need to
           free it */
        snprintf(buf, sizeof(buf), "%s-%d", filename, dec_ctx->frame_number);
        pgm_save(frame->data[0], frame->linesize[0],
                 frame->width, frame->height, buf);
    }
}

int main(int argc, char **argv)
{
    const char *filename, *outfilename;
    const AVCodec *codec;
    AVCodecParserContext *parser;
    AVCodecContext *c= NULL;
    FILE *f;
    AVFrame *frame;
    uint8_t inbuf[INBUF_SIZE + AV_INPUT_BUFFER_PADDING_SIZE];
    uint8_t *data;
    size_t   data_size;
    int ret;
    AVPacket *pkt;

    if (argc <= 2) {
        fprintf(stderr, "Usage: %s <input file> <output file>\n", argv[0]);
        exit(0);
    }
    filename    = argv[1];
    outfilename = argv[2];

    pkt = av_packet_alloc();
    if (!pkt)
        exit(1);

    /* set end of buffer to 0 (this ensures that no overreading happens for damaged MPEG streams) */
    memset(inbuf + INBUF_SIZE, 0, AV_INPUT_BUFFER_PADDING_SIZE);

    /* find the MPEG-1 video decoder */
    codec = avcodec_find_decoder(AV_CODEC_ID_MPEG1VIDEO);
    if (!codec) {
        fprintf(stderr, "Codec not found\n");
        exit(1);
    }

    parser = av_parser_init(codec->id);
    if (!parser) {
        fprintf(stderr, "parser not found\n");
        exit(1);
    }

    c = avcodec_alloc_context3(codec);
    if (!c) {
        fprintf(stderr, "Could not allocate video codec context\n");
        exit(1);
    }

    /* For some codecs, such as msmpeg4 and mpeg4, width and height
       MUST be initialized there because this information is not
       available in the bitstream. */

    /* open it */
    if (avcodec_open2(c, codec, NULL) < 0) {
        fprintf(stderr, "Could not open codec\n");
        exit(1);
    }

    f = fopen(filename, "rb");
    if (!f) {
        fprintf(stderr, "Could not open %s\n", filename);
        exit(1);
    }

    frame = av_frame_alloc();
    if (!frame) {
        fprintf(stderr, "Could not allocate video frame\n");
        exit(1);
    }

    while (!feof(f)) {
        /* read raw data from the input file */
        data_size = fread(inbuf, 1, INBUF_SIZE, f);
        if (!data_size)
            break;

        /* use the parser to split the data into frames */
        data = inbuf;
        while (data_size > 0) {
            ret = av_parser_parse2(parser, c, &pkt->data, &pkt->size,
                                   data, data_size, AV_NOPTS_VALUE, AV_NOPTS_VALUE, 0);
            if (ret < 0) {
                fprintf(stderr, "Error while parsing\n");
                exit(1);
            }
            data      += ret;
            data_size -= ret;

            if (pkt->size)
                decode(c, frame, pkt, outfilename);
        }
    }

    /* flush the decoder */
    decode(c, frame, NULL, outfilename);

    fclose(f);

    av_parser_close(parser);
    avcodec_free_context(&c);
    av_frame_free(&frame);
    av_packet_free(&pkt);

    return 0;
}
```


