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
![编码流程](https://upload-images.jianshu.io/upload_images/2262256-6636249a310bea3f.png?imageMogr2/auto-orient/strip|imageView2/2/w/569/format/webp “编码流程”)
1. 编码步骤：
  (1) av_register_all()：注册FFmpeg 的H265编码器。调用了avcodec_register_all()，avcodec_register_all()注册了H265编码器有关的组件：硬件加速器，编码器，Parser，Bitstream Filter等
  
  (2) avformat_alloc_output_context2()：初始化输出码流的AVFormatContext，获取输出文件的编码格式；
  
  (3) avio_open()：打开输出文件，调用了2个函数：ffurl_open()和ffio_fdopen()。其中ffurl_open()用于初始化URLContext，ffio_fdopen()用于根据URLContext初始化AVIOContext。URLContext中包含的URLProtocol完成了具体的协议读写等工作。AVIOContext则是在URLContext的读写函数外面加上了一层“包装”；
  
  (4) av_new_stream()：创建输出码流的AVStream结构体，为输出文件设置编码所需要的参数和格式；
  
  (5) avcodec_find_encoder()：通过 codec_id查找H265编码器
  
```

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
```
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

- 解码流程![FFMpeg解码流程](https://img-blog.csdnimg.cn/20190131151319528.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0ZQR0FUT00=,size_16,color_FFFFFF,t_70#pic_center “FFMpeg解码流程”)

1. 

