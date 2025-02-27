# code to select hardware to decode video
```cpp

#include <iostream>
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libavutil/hwcontext.h>
#include <libavutil/opt.h>

AVBufferRef *hw_device_ctx = nullptr;
AVHWFramesContext *frames_ctx = nullptr;
AVBufferRef *hw_frames_ref = nullptr;

// 初始化硬件设备上下文
int init_hw_device_ctx(AVHWDeviceType type) {
    int err = av_hwdevice_ctx_create(&hw_device_ctx, type, nullptr, nullptr, 0);
    if (err < 0) {
        std::cerr << "Failed to create hardware device context: " << av_err2str(err) << std::endl;
        return err;
    }
    return 0;
}

// 初始化硬件帧上下文
int init_hw_frames_ctx(AVCodecContext *decoder_ctx) {
    int err;

    hw_frames_ref = av_hwframe_ctx_alloc(hw_device_ctx);
    if (!hw_frames_ref) {
        std::cerr << "Failed to create hardware frame context" << std::endl;
        return AVERROR(ENOMEM);
    }

    frames_ctx = (AVHWFramesContext *)(hw_frames_ref->data);
    frames_ctx->format = AV_PIX_FMT_CUDA; // 硬件加速格式
    frames_ctx->sw_format = decoder_ctx->sw_pix_fmt; // 软件解码格式
    frames_ctx->width = decoder_ctx->width;
    frames_ctx->height = decoder_ctx->height;
    frames_ctx->initial_pool_size = 20;

    err = av_hwframe_ctx_init(hw_frames_ref);
    if (err < 0) {
        std::cerr << "Failed to initialize hardware frame context: " << av_err2str(err) << std::endl;
        av_buffer_unref(&hw_frames_ref);
        return err;
    }

    decoder_ctx->hw_frames_ctx = av_buffer_ref(hw_frames_ref);
    if (!decoder_ctx->hw_frames_ctx) {
        std::cerr << "Failed to set hardware frame context" << std::endl;
        return AVERROR(ENOMEM);
    }

    return 0;
}

// 自动选择硬解码器
AVCodec* find_hw_decoder(AVCodecParameters *codecpar, AVCodecContext **decoder_ctx) {
    const AVCodecHWConfig *config = nullptr;
    AVCodec *decoder = nullptr;
    void *i = nullptr;

    while ((decoder = av_codec_iterate(&i))) {
        if (decoder->id == codecpar->codec_id && av_codec_is_decoder(decoder)) {
            for (int j = 0;; j++) {
                config = avcodec_get_hw_config(decoder, j);
                if (!config) break;

                if (config->methods & AV_CODEC_HW_CONFIG_METHOD_HW_DEVICE_CTX &&
                    init_hw_device_ctx(config->device_type) == 0) {
                    *decoder_ctx = avcodec_alloc_context3(decoder);
                    if (*decoder_ctx) {
                        return decoder;
                    }
                }
            }
        }
    }

    return avcodec_find_decoder(codecpar->codec_id);
}

int main(int argc, char *argv[]) {
    const char *input_file = "input.mp4";

    av_register_all();
    avcodec_register_all();
    avformat_network_init();

    AVFormatContext *fmt_ctx = nullptr;
    if (avformat_open_input(&fmt_ctx, input_file, nullptr, nullptr) < 0) {
        std::cerr << "Failed to open input file" << std::endl;
        return -1;
    }

    if (avformat_find_stream_info(fmt_ctx, nullptr) < 0) {
        std::cerr << "Failed to find stream info" << std::endl;
        return -1;
    }

    int video_stream_index = -1;
    for (unsigned int i = 0; i < fmt_ctx->nb_streams; ++i) {
        if (fmt_ctx->streams[i]->codecpar->codec_type == AVMEDIA_TYPE_VIDEO) {
            video_stream_index = i;
            break;
        }
    }

    if (video_stream_index == -1) {
        std::cerr << "Failed to find video stream" << std::endl;
        return -1;
    }

    AVCodecParameters *codecpar = fmt_ctx->streams[video_stream_index]->codecpar;
    AVCodecContext *decoder_ctx = nullptr;
    AVCodec *decoder = find_hw_decoder(codecpar, &decoder_ctx);

    if (!decoder) {
        std::cerr << "Failed to find decoder" << std::endl;
        return -1;
    }

    if (avcodec_parameters_to_context(decoder_ctx, codecpar) < 0) {
        std::cerr << "Failed to copy codec parameters to codec context" << std::endl;
        return -1;
    }

    if (decoder_ctx->hw_device_ctx && init_hw_frames_ctx(decoder_ctx) < 0) {
        return -1;
    }

    if (avcodec_open2(decoder_ctx, decoder, nullptr) < 0) {
        std::cerr << "Failed to open codec" << std::endl;
        return -1;
    }

    AVFrame *frame = av_frame_alloc();
    AVPacket packet;
    while (av_read_frame(fmt_ctx, &packet) >= 0) {
        if (packet.stream_index == video_stream_index) {
            if (avcodec_send_packet(decoder_ctx, &packet) == 0) {
                while (avcodec_receive_frame(decoder_ctx, frame) == 0) {
                    // 处理解码后的帧
                    std::cout << "Decoded frame: " << frame->pts << std::endl;
                }
            }
        }
        av_packet_unref(&packet);
    }

    av_frame_free(&frame);
    avcodec_free_context(&decoder_ctx);
    av_buffer_unref(&hw_frames_ref);
    av_buffer_unref(&hw_device_ctx);
    avformat_close_input(&fmt_ctx);

    return 0;
}
```
