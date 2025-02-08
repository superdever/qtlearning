# play video with ffmpeg using hardware acceleration and opengl drawing

```

#include <QApplication>
#include <QOpenGLWidget>
#include <QThread>
#include <QOpenGLFunctions>
#include <QOpenGLTexture>
#include <QVBoxLayout>
#include <QMutex>
#include <QMutexLocker>
#include <QWaitCondition>
#include <QDebug>

extern "C" {
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libswscale/swscale.h>
#include <libavutil/hwcontext.h>
#include <libavutil/opt.h>
#include <libavutil/imgutils.h>
}

class VideoWidget : public QOpenGLWidget, protected QOpenGLFunctions {
    Q_OBJECT
public:
    VideoWidget(QWidget *parent = nullptr)
        : QOpenGLWidget(parent), frame(nullptr), swsContext(nullptr), texture(nullptr) {
        initializeFFmpeg();
    }

    ~VideoWidget() {
        cleanupFFmpeg();
    }

    void setFrame(AVFrame *newFrame) {
        QMutexLocker locker(&mutex);
        if (frame) {
            av_frame_free(&frame);
        }
        frame = newFrame;
        update(); // 请求重绘
    }

protected:
    void initializeGL() override {
        initializeOpenGLFunctions();
        glGenTextures(1, &textureID);
        glBindTexture(GL_TEXTURE_2D, textureID);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
    }

    void resizeGL(int w, int h) override {
        glViewport(0, 0, w, h);
    }

    void paintGL() override {
        if (!frame) return;

        QMutexLocker locker(&mutex);
        int width = frame->width;
        int height = frame->height;

        if (!swsContext) {
            swsContext = sws_getContext(
                width, height, (AVPixelFormat)frame->format,
                width, height, AV_PIX_FMT_RGB24,
                SWS_BILINEAR, nullptr, nullptr, nullptr);
        }

        if (!swsContext) {
            qDebug() << "Failed to initialize swsContext";
            return;
        }

        buffer.resize(av_image_get_buffer_size(AV_PIX_FMT_RGB24, width, height, 1));
        uint8_t *data[1] = { buffer.data() };
        int linesize[1] = { width * 3 };

        int result = sws_scale(
            swsContext,
            frame->data, frame->linesize, 0, height,
            data, linesize
        );
        if (result <= 0) {
            qDebug() << "sws_scale failed";
            return;
        }

        glBindTexture(GL_TEXTURE_2D, textureID);
        glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, width, height, 0, GL_RGB, GL_UNSIGNED_BYTE, buffer.data());

        glClear(GL_COLOR_BUFFER_BIT);
        glEnable(GL_TEXTURE_2D);
        glBegin(GL_QUADS);
            //the follow code can flip y-coordinates
            //glTexCoord2f(0.0f, 0.0f); glVertex2f(-1.0f, -1.0f);  
            //glTexCoord2f(1.0f, 0.0f); glVertex2f( 1.0f, -1.0f);  
            //glTexCoord2f(1.0f, 1.0f); glVertex2f( 1.0f,  1.0f);  
            //glTexCoord2f(0.0f, 1.0f); glVertex2f(-1.0f,  1.0f);  
            //glTexCoord2f(0.0f, 1.0f); glVertex2f(-1.0f, -1.0f); // Flip y-coordinates
            glTexCoord2f(1.0f, 1.0f); glVertex2f( 1.0f, -1.0f); // Flip y-coordinates
            glTexCoord2f(1.0f, 0.0f); glVertex2f( 1.0f,  1.0f); // Flip y-coordinates
            glTexCoord2f(0.0f, 0.0f); glVertex2f(-1.0f,  1.0f); // Flip y-coordinates
        glEnd();
        glDisable(GL_TEXTURE_2D);
    }

private:
    AVFrame *frame;
    SwsContext *swsContext;
    QMutex mutex;
    GLuint textureID;
    QVector<uint8_t> buffer;

    void initializeFFmpeg() {
        av_register_all();
        avcodec_register_all();
        buffer.resize(1920 * 1080 * 3); // 预分配缓冲区
    }

    void cleanupFFmpeg() {
        if (swsContext) {
            sws_freeContext(swsContext);
            swsContext = nullptr;
        }
        if (frame) {
            av_frame_free(&frame);
            frame = nullptr;
        }
        glDeleteTextures(1, &textureID);
    }
};

class DecodeThread : public QThread {
    Q_OBJECT
public:
    DecodeThread(VideoWidget *widget, const QString &filePath, QObject *parent = nullptr)
        : QThread(parent), videoWidget(widget), filePath(filePath) {}

protected:
    void run() override {
        AVFormatContext *formatContext = avformat_alloc_context();
        if (avformat_open_input(&formatContext, filePath.toUtf8().data(), nullptr, nullptr) != 0) {
            qDebug() << "Failed to open input file";
            return;
        }

        if (avformat_find_stream_info(formatContext, nullptr) < 0) {
            qDebug() << "Failed to find stream info";
            return;
        }

        AVCodec *codec = nullptr;
        AVCodecContext *codecContext = nullptr;
        int videoStreamIndex = -1;
        bool useHardware = false;
        AVHWDeviceType hwDeviceType = AV_HWDEVICE_TYPE_NONE;
        AVBufferRef *hwDeviceCtx = nullptr;

        // 尝试多种硬件加速类型
        const AVHWDeviceType hwDeviceTypes[] = {
            AV_HWDEVICE_TYPE_CUDA,
            AV_HWDEVICE_TYPE_DXVA2,
            AV_HWDEVICE_TYPE_VAAPI,
            AV_HWDEVICE_TYPE_QSV
        };

        for (unsigned int i = 0; i < formatContext->nb_streams; i++) {
            if (formatContext->streams[i]->codecpar->codec_type == AVMEDIA_TYPE_VIDEO) {
                videoStreamIndex = i;
                for (const AVHWDeviceType &type : hwDeviceTypes) {
                    codec = avcodec_find_decoder(formatContext->streams[i]->codecpar->codec_id);
                    codecContext = avcodec_alloc_context3(codec);
                    avcodec_parameters_to_context(codecContext, formatContext->streams[i]->codecpar);

                    if (type != AV_HWDEVICE_TYPE_NONE && av_hwdevice_ctx_create(&hwDeviceCtx, type, nullptr, nullptr, 0) >= 0) {
                        codecContext->hw_device_ctx = av_buffer_ref(hwDeviceCtx);
                        av_buffer_unref(&hwDeviceCtx);
                        hwDeviceType = type;
                        useHardware = true;
                        break;
                    }
                }

                if (!useHardware) {
                    codec = avcodec_find_decoder(formatContext->streams[i]->codecpar->codec_id);
                    codecContext = avcodec_alloc_context3(codec);
                    avcodec_parameters_to_context(codecContext, formatContext->streams[i]->codecpar);
                }

                avcodec_open2(codecContext, codec, nullptr);
                break;
            }
        }

        if (!codecContext) {
            qDebug() << "Failed to find video stream";
            return;
        }

        AVPacket *packet = av_packet_alloc();
        AVFrame *frame = av_frame_alloc();
        AVFrame *sw_frame = av_frame_alloc();

        while (av_read_frame(formatContext, packet) >= 0) {
            if (packet->stream_index == videoStreamIndex) {
                avcodec_send_packet(codecContext, packet);
                while (avcodec_receive_frame(codecContext, frame) >= 0) {
                    AVFrame *frameToUse = frame;
                    if (useHardware && frame->format != AV_PIX_FMT_YUV420P) {
                        if (av_hwframe_transfer_data(sw_frame, frame, 0) < 0) {
                            qDebug() << "Error transferring the data to system memory";
                            continue;
                        }
                        frameToUse = sw_frame;
                    }
                    AVFrame *clonedFrame = av_frame_clone(frameToUse);
                    emit frameReady(clonedFrame);
                    msleep(1000 / 30); // 模拟30帧每秒的视频播放
                }
            }
            av_packet_unref(packet);
        }

        av_frame_free(&frame);
        av_frame_free(&sw_frame);
        av_packet_free(&packet);
        avcodec_free_context(&codecContext);
        avformat_close_input(&formatContext);
    }

signals:
    void frameReady(AVFrame *frame);

private:
    VideoWidget *videoWidget;
    QString filePath;
};

int main(int argc, char *argv[]) {
    QApplication app(argc, argv);

    QWidget window;
    QVBoxLayout *layout = new QVBoxLayout;
    VideoWidget *videoWidget = new VideoWidget;
    layout->addWidget(videoWidget);
    window.setLayout(layout);
    window.resize(800, 600);
    window.show();

    DecodeThread *decodeThread = new DecodeThread(videoWidget, "input.mp4");
    QObject::connect(decodeThread, &DecodeThread::frameReady, videoWidget, &VideoWidget::setFrame);
    decodeThread->start();

    return app.exec();
}

#include "main.moc"



```
