# 

Android 中调用硬[解码](https://so.csdn.net/so/search?q=%E8%A7%A3%E7%A0%81&spm=1001.2101.3001.7020) API 是使用 MediaCodec 一步一步调用硬件实现的，通常需要最终调用 VPU 进行解码工作，现在先来分析其初始化过程。

![](content/assets/images/初始化过程.jpg)

下面是一段典型的硬解码初始化代码，当然在[异常处理](https://so.csdn.net/so/search?q=%E5%BC%82%E5%B8%B8%E5%A4%84%E7%90%86&spm=1001.2101.3001.7020)上也做了处理，是为了更好的容错。

1. 根据 MIME_TYPE（video/avc） 创建解码器，调用 createDecoderByType 实现；
2. 根据视频长宽以及 MIME_TYPE 创建 MediaFormat 配置（设置解码颜色空间、Profile Baseline 和 Profile Level 等）；
3. 将 MediaFormat 配置、Surface（解码后绘制图像解码可为 null）等传入解码器配置函数 configure 进行配置。

```
public class H264Decoder {
    private static final String TAG = "H264Decoder";
    private final static String MIME_TYPE = "video/avc"; // H.264 Advanced Video

    private MediaCodec mDecoderMediaCodec;

    private int mFps;
    private Surface mSurface;
    private int mWidth;
    private int mHeight;
    private int mInitMediaCodecTryTimes = 0;

    public H264Decoder(int width, int height, int fps, Surface surface) {
        mWidth = width;
        mHeight = height;
        mFps = fps;
        mSurface = surface;

        initMediaCodec(width, height, surface);
    }

    private void initMediaCodec(int width, int height, Surface surface) {
        try {
            mDecoderMediaCodec = MediaCodec.createDecoderByType(MIME_TYPE);
            //创建配置
            MediaFormat mediaFormat = MediaFormat.createVideoFormat(MIME_TYPE, width, height);
            mediaFormat.setInteger(MediaFormat.KEY_COLOR_FORMAT, MediaCodecInfo.CodecCapabilities.COLOR_FormatYUV420Flexible);
            //mediaFormat.setInteger(MediaFormat.KEY_PROFILE, MediaCodecInfo.CodecProfileLevel.AVCProfileBaseline);
            //mediaFormat.setInteger("level", MediaCodecInfo.CodecProfileLevel.AVCLevel4); // Level 4

            mDecoderMediaCodec.configure(mediaFormat, surface, null, 0);
        } catch (Exception e) {
            e.printStackTrace();
            //创建解码失败
            Log.e(TAG, "init MediaCodec fail.");
            // 重新尝试六次
            if (mInitMediaCodecTryTimes < 6) {
                mInitMediaCodecTryTimes++;
                try {
                    Thread.sleep(20);
                } catch (InterruptedException ie) {
                    ie.printStackTrace();
                }
                initMediaCodec(mWidth, mHeight, mSurface);
            }
        }
    }
    ......
}
123456789101112131415161718192021222324252627282930313233343536373839404142434445464748
```

现在分析第一步，createDecoderByType 如何查找实例化 MediaCodec ？

实例化支持给定 mime 类型的首选解码器。

下面是定义的 mime 类型及其语义的部分列表：

“video/x-vnd.on2. vp8” - VP8 视频（例如 webm 中的视频)

“video/x-vnd.on2. vp9” - VP9 视频（例如 webm 中的视频)

“video/avc” - H.264/AVC 视频

“video/hevc” - H.265/HEVC 视频

“video/mp4v-es” - MPEG4 视频

“video/3gpp” - H.263 视频

“audio/3gpp” - AMR 窄带音频

“audio/amr-wb” - AMR 宽带音频

“audio/mpeg” - MPEG1/2 音频层 III

“audio/mp4a-latm” - AAC 音频（注意，这是原始的 AAC 包，不是封装在 LATM）

“audio/vorbis” - vorbis 音频

“audio/g711-alaw” - G.711 alaw 音频

“audio/g711-mlaw” - G.711 ulaw 音频

内部其实调用了 MediaCodec 构造器，只不过后两个参数写死了，nameIsType （true）顾名思义名称是否是类型，encoder（false）是否编码器。

frameworks/base/media/java/android/media/MediaCodec.java

```
final public class MediaCodec {
    ......
    @NonNull
    public static MediaCodec createDecoderByType(@NonNull String type)
            throws IOException {
        return new MediaCodec(type, true /* nameIsType */, false /* encoder */);
    }
    ......
}
12345678
```

1. 获取 Looper 对象，如果是在普通线程中调用那么 Looper.myLooper() 必然不为 null，则用这个 Looper 对象构造 EventHandler。否则使用 Looper.getMainLooper() 获取主线程 Looper 对象并构造 EventHandler；
2. 接着将 mCallbackHandler（回调“句柄”） 和 mOnFrameRenderedHandler（帧渲染“句柄”） 均赋值为刚才创建的 EventHandler 对象，这个 EventHandler 是用来处理事件的，后面用到再去分析其处理的具体事件；
3. 创建一个 Buffer Lock，用于同步相关；
4. mNameAtCreation 则表示是否保存创建时使用的名称，此处传入的是 ture，所以并不需要，因为我们已经知道名称了；
5. 最后调用 native_setup 进行 jni 调用进行进一步初始化。

frameworks/base/media/java/android/media/MediaCodec.java

```
final public class MediaCodec {
    ......
    private EventHandler mEventHandler;
    private EventHandler mOnFrameRenderedHandler;
    private EventHandler mCallbackHandler;
    ......
    final private Object mBufferLock;
    ......
    private MediaCodec(
            @NonNull String name, boolean nameIsType, boolean encoder) {
        Looper looper;
        if ((looper = Looper.myLooper()) != null) {
            mEventHandler = new EventHandler(this, looper);
        } else if ((looper = Looper.getMainLooper()) != null) {
            mEventHandler = new EventHandler(this, looper);
        } else {
            mEventHandler = null;
        }
        mCallbackHandler = mEventHandler;
        mOnFrameRenderedHandler = mEventHandler;

        mBufferLock = new Object();

        // save name used at creation
        mNameAtCreation = nameIsType ? null : name;

        native_setup(name, nameIsType, encoder);
    }

    private String mNameAtCreation;
    ......
}
12345678910111213141516171819202122232425262728293031
```

1. 创建 JMediaCodec 对象；
2. 检查创建 JMediaCodec 对象是否发生错误，如果发生错误则分支到相应的语句抛出异常给 java 层；
3. 调用 JMediaCodec registerSelf() 注册自己；
4. 调用 setMediaCodec(…) 函数将指向 JMediaCodec 对象的指针赋给 java 层 field（MediaCodec.java mNativeContext）。

frameworks/base/media/jni/android_media_MediaCodec.cpp

```
static void android_media_MediaCodec_native_setup(
        JNIEnv *env, jobject thiz,
        jstring name, jboolean nameIsType, jboolean encoder) {
    if (name == NULL) {
        jniThrowException(env, "java/lang/NullPointerException", NULL);
        return;
    }

    const char *tmp = env->GetStringUTFChars(name, NULL);

    if (tmp == NULL) {
        return;
    }

    sp<JMediaCodec> codec = new JMediaCodec(env, thiz, tmp, nameIsType, encoder);

    const status_t err = codec->initCheck();
    if (err == NAME_NOT_FOUND) {
        // fail and do not try again.
        jniThrowException(env, "java/lang/IllegalArgumentException",
                String8::format("Failed to initialize %s, error %#x", tmp, err));
        env->ReleaseStringUTFChars(name, tmp);
        return;
    } if (err == NO_MEMORY) {
        throwCodecException(env, err, ACTION_CODE_TRANSIENT,
                String8::format("Failed to initialize %s, error %#x", tmp, err));
        env->ReleaseStringUTFChars(name, tmp);
        return;
    } else if (err != OK) {
        // believed possible to try again
        jniThrowException(env, "java/io/IOException",
                String8::format("Failed to find matching codec %s, error %#x", tmp, err));
        env->ReleaseStringUTFChars(name, tmp);
        return;
    }

    env->ReleaseStringUTFChars(name, tmp);

    codec->registerSelf();

    setMediaCodec(env,thiz, codec);
}
......
static const JNINativeMethod gMethods[] = {
    ......
    { "native_setup", "(Ljava/lang/String;ZZ)V",
      (void *)android_media_MediaCodec_native_setup },
    ......
};
123456789101112131415161718192021222324252627282930313233343536373839404142434445464748
```

1. 缓存一些 jni 对象供后续使用；
2. 创建一个 ALooper 对象，并启动，用新建的 LooperThread （runOnCallingThread 为 false 表明不再调用线程上处理）处理事件；
3. nameIsType 等于 true，调用 Native MediaCodec CreateByType 方法创建 Native MediaCodec 对象。

frameworks/base/media/jni/android_media_MediaCodec.cpp

```
JMediaCodec::JMediaCodec(
        JNIEnv *env, jobject thiz,
        const char *name, bool nameIsType, bool encoder)
    : mClass(NULL),
      mObject(NULL) {
    jclass clazz = env->GetObjectClass(thiz);
    CHECK(clazz != NULL);

    mClass = (jclass)env->NewGlobalRef(clazz);
    mObject = env->NewWeakGlobalRef(thiz);

    cacheJavaObjects(env);

    mLooper = new ALooper;
    mLooper->setName("MediaCodec_looper");

    mLooper->start(
            false,      // runOnCallingThread
            true,       // canCallJava
            ANDROID_PRIORITY_VIDEO);

    if (nameIsType) {
        mCodec = MediaCodec::CreateByType(mLooper, name, encoder, &mInitStatus);
        if (mCodec == nullptr || mCodec->getName(&mNameAtCreation) != OK) {
            mNameAtCreation = "(null)";
        }
    } else {
        mCodec = MediaCodec::CreateByComponentName(mLooper, name, &mInitStatus);
        mNameAtCreation = name;
    }
    CHECK((mCodec != NULL) != (mInitStatus != OK));
}
12345678910111213141516171819202122232425262728293031
```

最后两个入参都是 -1。

1. 调用 MediaCodecList::findMatchingCodecs(…) 查找所有匹配的解码器；
2. 遍历上一步返回的容器 Vector 中的组件名称，创建 MediaCodec cpp 对象，并根据组件名称调用其 init(…) 函数；
3. 如果初始化没有错误，则直接返回 MediaCodec cpp 对象。

frameworks/av/media/libstagefright/MediaCodec.cpp

```
sp<MediaCodec> MediaCodec::CreateByType(
        const sp<ALooper> &looper, const AString &mime, bool encoder, status_t *err, pid_t pid,
        uid_t uid) {
    Vector<AString> matchingCodecs;

    MediaCodecList::findMatchingCodecs(
            mime.c_str(),
            encoder,
            0,
            &matchingCodecs);

    if (err != NULL) {
        *err = NAME_NOT_FOUND;
    }
    for (size_t i = 0; i < matchingCodecs.size(); ++i) {
        sp<MediaCodec> codec = new MediaCodec(looper, pid, uid);
        AString componentName = matchingCodecs[i];
        status_t ret = codec->init(componentName);
        if (err != NULL) {
            *err = ret;
        }
        if (ret == OK) {
            return codec;
        }
        ALOGD("Allocating component '%s' failed (%d), try next one.",
                componentName.c_str(), ret);
    }
    return NULL;
}
12345678910111213141516171819202122232425262728
```

1. 调用 getInstance() 获取 BpMediaCodecList 对象；
2. 调用 BpMediaCodecList 的 findCodecByType(…) 方法，实际由 MediaCodecList::findCodecByType(…) 处理返回匹配的 index，当 matchIndex 小于 0 退出循环；
3. 调用 BpMediaCodecList 的 getCodecInfo(…) 方法，获取 MediaCodecInfo；
4. 通过 MediaCodecInfo 对象 getCodecName() 获取解码器名称 ；
5. flags 入参等于 0，所以将解码器组件名称直接推入容器 matches；
6. 接下来根据是否设置 debug.stagefright.swcodec 属性来判定是否要对容器元素排序。

frameworks/av/media/libstagefright/MediaCodecList.cpp

```
void MediaCodecList::findMatchingCodecs(
        const char *mime, bool encoder, uint32_t flags,
        Vector<AString> *matches) {
    matches->clear();

    const sp<IMediaCodecList> list = getInstance();
    if (list == nullptr) {
        return;
    }

    size_t index = 0;
    for (;;) {
        ssize_t matchIndex =
            list->findCodecByType(mime, encoder, index);

        if (matchIndex < 0) {
            break;
        }

        index = matchIndex + 1;

        const sp<MediaCodecInfo> info = list->getCodecInfo(matchIndex);
        CHECK(info != nullptr);
        AString componentName = info->getCodecName();

        if ((flags & kHardwareCodecsOnly) && isSoftwareCodec(componentName)) {
            ALOGV("skipping SW codec '%s'", componentName.c_str());
        } else {
            matches->push(componentName);
            ALOGV("matching '%s'", componentName.c_str());
        }
    }

    if (flags & kPreferSoftwareCodecs ||
            property_get_bool("debug.stagefright.swcodec", false)) {
        matches->sort(compareSoftwareCodecsFirst);
    }
}
1234567891011121314151617181920212223242526272829303132333435363738
```

查找名称为 media.player 的服务，实际为 MediaPlayerService，然后通过 MediaPlayerService getCodecList() 获取到 IMediaCodecList（BpMediaCodecList）。这里跨进程通信实际是通过 BpMediaPlayerService 调用到远端的 MediaPlayerService。

frameworks/av/media/libstagefright/MediaCodecList.cpp

```
sp<IMediaCodecList> MediaCodecList::getInstance() {
    Mutex::Autolock _l(sRemoteInitMutex);
    if (sRemoteList == nullptr) {
        sp<IBinder> binder =
            defaultServiceManager()->getService(String16("media.player"));
        sp<IMediaPlayerService> service =
            interface_cast<IMediaPlayerService>(binder);
        if (service.get() != nullptr) {
            sRemoteList = service->getCodecList();
            if (sRemoteList != nullptr) {
                sBinderDeathObserver = new BinderDeathObserver();
                binder->linkToDeath(sBinderDeathObserver.get());
            }
        }
        if (sRemoteList == nullptr) {
            // if failed to get remote list, create local list
            sRemoteList = getLocalInstance();
        }
    }
    return sRemoteList;
}
123456789101112131415161718192021
```

BpMediaPlayerService getCodecList() 方法使用 binder 机制写入数据，等待远端响应。

frameworks/av/media/libmedia/IMediaPlayerService.cpp

```
class BpMediaPlayerService: public BpInterface<IMediaPlayerService>
{
public:
    ......
    virtual sp<IMediaCodecList> getCodecList() const {
        Parcel data, reply;
        data.writeInterfaceToken(IMediaPlayerService::getInterfaceDescriptor());
        remote()->transact(GET_CODEC_LIST, data, &reply);
        return interface_cast<IMediaCodecList>(reply.readStrongBinder());
    }
};
1234567891011
```

实际响应方为 MediaPlayerService::getCodecList()，这个函数内部调用了 MediaCodecList::getLocalInstance()，又绕了回去（因为调用进程和响应进程是不同的，虽然都使用了同一个类的中的函数）。

frameworks/av/media/libmediaplayerservice/MediaPlayerService.cpp

```
sp<IMediaCodecList> MediaPlayerService::getCodecList() const {
    return MediaCodecList::getLocalInstance();
}

```

MediaCodecList 继承自 BnMediaCodecList，注释写道：getLocalInstance() 仅供 MediaPlayerService 使用。

1. 新建一个 MediaCodecList 对象，不过入参是另外一个函数 GetBuilders() 的返回值 ；
2. 检测上一步是否异常，无异常最终返回 BpMediaCodecList 单例对象；
3. isProfilingNeeded() 看名字和编解码器分析有关，不在主流程上不再具体分析。

frameworks/av/media/libstagefright/MediaCodecList.cpp

```
sp<IMediaCodecList> MediaCodecList::getLocalInstance() {
    Mutex::Autolock autoLock(sInitMutex);

    if (sCodecList == nullptr) {
        MediaCodecList *codecList = new MediaCodecList(GetBuilders());
        if (codecList->initCheck() == OK) {
            sCodecList = codecList;

            if (isProfilingNeeded()) {
                ALOGV("Codec profiling needed, will be run in separated thread.");
                pthread_t profiler;
                if (pthread_create(&profiler, nullptr, profilerThreadWrapper, nullptr) != 0) {
                    ALOGW("Failed to create thread for codec profiling.");
                }
            }
        } else {
            // failure to initialize may be temporary. retry on next call.
            delete codecList;
        }
    }

    return sCodecList;
}
12345678910111213141516171819202122
```

1. 遍历 GetBuilders() 返回的容器中所有的 MediaCodecListBuilderBase*；
2. 调用每个 builder 的 buildMediaCodecList(…) 方法；
3. MediaCodecListWriter 写入全局设置到 AMessage、写入 Codec 信息到 mCodecInfos 指向的容器；
4. mCodecInfos 是一个指向 std::vector<sp > 容器的变量，对容器内的元素进行排序（排序依据是 rank）；
5. 移除容器中重复的元素，前提是 debug.stagefright.dedupe-codecs 属性开关要打开。

frameworks/av/media/libstagefright/MediaCodecList.cpp

```
MediaCodecList::MediaCodecList(std::vector<MediaCodecListBuilderBase*> builders) {
    mGlobalSettings = new AMessage();
    mCodecInfos.clear();
    MediaCodecListWriter writer;
    for (MediaCodecListBuilderBase *builder : builders) {
        if (builder == nullptr) {
            ALOGD("ignored a null builder");
            continue;
        }
        mInitCheck = builder->buildMediaCodecList(&writer);
        if (mInitCheck != OK) {
            break;
        }
    }
    writer.writeGlobalSettings(mGlobalSettings);
    writer.writeCodecInfos(&mCodecInfos);
    std::stable_sort(
            mCodecInfos.begin(),
            mCodecInfos.end(),
            [](const sp<MediaCodecInfo> &info1, const sp<MediaCodecInfo> &info2) {
                // null is lowest
                return info1 == nullptr
                        || (info2 != nullptr && info1->getRank() < info2->getRank());
            });

    // remove duplicate entries
    bool dedupe = property_get_bool("debug.stagefright.dedupe-codecs", true);
    if (dedupe) {
        std::set<std::string> codecsSeen;
        for (auto it = mCodecInfos.begin(); it != mCodecInfos.end(); ) {
            std::string codecName = (*it)->getCodecName();
            if (codecsSeen.count(codecName) == 0) {
                codecsSeen.emplace(codecName);
                it++;
            } else {
                it = mCodecInfos.erase(it);
            }
        }
    }
}
123456789101112131415161718192021222324252627282930313233343536373839
```

依赖插件提供可用的 OMX codec 列表。如果插件提供输入 surface，不能使用 OMX 视频编码器。

1. 调用 StagefrightPluginLoader::GetCCodecInstance() 获取 StagefrightPluginLoader 对象，接着调用其 createInputSurface() 方法；
2. 根据上一步的返回值来分支具体添加哪一个 builder 到容器；
3. 添加 GetCodec2InfoBuilder() 返回的 builder 到容器。

GetCodec2InfoBuilder() 函数内部调用了 StagefrightPluginLoader 对象的 createBuilder() 方法返回 builder。

frameworks/av/media/libstagefright/MediaCodecList.cpp

```
OmxInfoBuilder sOmxInfoBuilder{true /* allowSurfaceEncoders */};
OmxInfoBuilder sOmxNoSurfaceEncoderInfoBuilder{false /* allowSurfaceEncoders */};

Mutex sCodec2InfoBuilderMutex;
std::unique_ptr<MediaCodecListBuilderBase> sCodec2InfoBuilder;

MediaCodecListBuilderBase *GetCodec2InfoBuilder() {
    Mutex::Autolock _l(sCodec2InfoBuilderMutex);
    if (!sCodec2InfoBuilder) {
        sCodec2InfoBuilder.reset(
                StagefrightPluginLoader::GetCCodecInstance()->createBuilder());
    }
    return sCodec2InfoBuilder.get();
}

std::vector<MediaCodecListBuilderBase *> GetBuilders() {
    std::vector<MediaCodecListBuilderBase *> builders;
    // if plugin provides the input surface, we cannot use OMX video encoders.
    // In this case, rely on plugin to provide list of OMX codecs that are usable.
    sp<PersistentSurface> surfaceTest =
        StagefrightPluginLoader::GetCCodecInstance()->createInputSurface();
    if (surfaceTest == nullptr) {
        ALOGD("Allowing all OMX codecs");
        builders.push_back(&sOmxInfoBuilder);
    } else {
        ALOGD("Allowing only non-surface-encoder OMX codecs");
        builders.push_back(&sOmxNoSurfaceEncoderInfoBuilder);
    }
    builders.push_back(GetCodec2InfoBuilder());
    return builders;
}
123456789101112131415161718192021222324252627282930
```

这里指向 StagefrightPluginLoader 对象的指针便用 unique_ptr 进行了声明，因此 sInstance 指针变量只能被"绑定”一次，也就做到了单例模式的效果。

**小知识**

> 
> 
> 
> unique_ptr 是一种对资源具有排他性拥有权的智能指针，即一个对象资源只能同时被一个 unique_ptr 指向。
> 

frameworks/av/media/libstagefright/StagefrightPluginLoader.cpp

```
const std::unique_ptr<StagefrightPluginLoader> &StagefrightPluginLoader::GetCCodecInstance() {
    Mutex::Autolock _l(sMutex);
    if (!sInstance) {
        ALOGV("Loading library");
        sInstance.reset(new StagefrightPluginLoader(kCCodecPluginPath));
    }
    return sInstance;
}

```

1. 调用 dlopen(…) 根据路径 libsfplugin_ccodec.so 打开这个 so 库；
2. 调用 dlsym 解析符号获取 CreateCodec、CreateBuilder 和 CreateInputSurface 这三个方法地址。

**小知识**

> 
> 
> 
> RTLD_NOW： 需要在 dlopen 返回前，解析出所有未定义符号，如果解析不出来，在 dlopen 会返回 NULL，错误为：undefined symbol: xxxx…
> 
> RTLD_NODELETE： 在 dlclose() 期间不卸载库，并且在以后使用 dlopen() 重新加载库时不初始化库中的静态变量。这个 flag 不是 POSIX-2001 标准。
> 

frameworks/av/media/libstagefright/StagefrightPluginLoader.cpp

```
namespace /* unnamed */ {

constexpr const char kCCodecPluginPath[] = "libsfplugin_ccodec.so";

}  // unnamed namespace

StagefrightPluginLoader::StagefrightPluginLoader(const char *libPath) {
    if (android::base::GetIntProperty("debug.stagefright.ccodec", 1) == 0) {
        ALOGD("CCodec is disabled.");
        return;
    }
    mLibHandle = dlopen(libPath, RTLD_NOW | RTLD_NODELETE);
    if (mLibHandle == nullptr) {
        ALOGD("Failed to load library: %s (%s)", libPath, dlerror());
        return;
    }
    mCreateCodec = (CodecBase::CreateCodecFunc)dlsym(mLibHandle, "CreateCodec");
    if (mCreateCodec == nullptr) {
        ALOGD("Failed to find symbol: CreateCodec (%s)", dlerror());
    }
    mCreateBuilder = (MediaCodecListBuilderBase::CreateBuilderFunc)dlsym(
            mLibHandle, "CreateBuilder");
    if (mCreateBuilder == nullptr) {
        ALOGD("Failed to find symbol: CreateBuilder (%s)", dlerror());
    }
    mCreateInputSurface = (CodecBase::CreateInputSurfaceFunc)dlsym(
            mLibHandle, "CreateInputSurface");
    if (mCreateInputSurface == nullptr) {
        ALOGD("Failed to find symbol: CreateInputSurface (%s)", dlerror());
    }
}
123456789101112131415161718192021222324252627282930
```

由上面符号解析不难查到 StagefrightPluginLoader::createInputSurface() 最终会调用 codec2/sfplugin/CCodec.cpp 下的 android::PersistentSurface *CreateInputSurface() 方法。

frameworks/av/media/libstagefright/StagefrightPluginLoader.cpp

```
PersistentSurface *StagefrightPluginLoader::createInputSurface() {
    if (mLibHandle == nullptr || mCreateInputSurface == nullptr) {
        ALOGD("Handle or CreateInputSurface symbol is null");
        return nullptr;
    }
    return mCreateInputSurface();
}

```

从 bp 文件不难看出 libsfplugin_ccodec 这个 so 文件位于 frameworks/av/media/codec2/sfplugin/ 目录下，具体编译了哪些 cpp 文件一目了然，包括依赖库。

frameworks/av/media/codec2/sfplugin/Android.bp

```
cc_library_shared {
    name: "libsfplugin_ccodec",

    srcs: [
        "C2OMXNode.cpp",
        "CCodec.cpp",
        "CCodecBufferChannel.cpp",
        "CCodecBuffers.cpp",
        "CCodecConfig.cpp",
        "Codec2Buffer.cpp",
        "Codec2InfoBuilder.cpp",
        "Omx2IGraphicBufferSource.cpp",
        "PipelineWatcher.cpp",
        "ReflectedParamUpdater.cpp",
        "SkipCutBuffer.cpp",
    ],

    cflags: [
        "-Werror",
        "-Wall",
    ],

    header_libs: [
        "libcodec2_internal",
    ],

    shared_libs: [
        "android.hardware.cas.native@1.0",
        "android.hardware.graphics.bufferqueue@1.0",
        "android.hardware.media.c2@1.0",
        "android.hardware.media.omx@1.0",
        "libbase",
        "libbinder",
        "libcodec2",
        "libcodec2_client",
        "libcodec2_vndk",
        "libcutils",
        "libgui",
        "libhidlallocatorutils",
        "libhidlbase",
        "liblog",
        "libmedia",
        "libmedia_omx",
        "libsfplugin_ccodec_utils",
        "libstagefright_bufferqueue_helper",
        "libstagefright_codecbase",
        "libstagefright_foundation",
        "libstagefright_omx",
        "libstagefright_omx_utils",
        "libstagefright_xmlparser",
        "libui",
        "libutils",
    ],

    sanitize: {
        cfi: true,
        misc_undefined: [
            "unsigned-integer-overflow",
            "signed-integer-overflow",
        ],
    },
}
110111213141516171819202122232425262728293031323334353637383940414243444546474849505152535455565758596061
```

现在来分析 sfplugin 中的 CreateInputSurface() 方法具体实现。

1. 调用 Codec2Client::CreateInputSurface() 获取指向 Codec2Client::InputSurface 共享指针；
2. 由于在瑞芯微 rk 平台上 debug.stagefright.c2inputsurface 属性值没有设置，所以默认为 0，Codec2Client::CreateInputSurface() 函数直接返回空指针；
3. 最终 android::PersistentSurface *CreateInputSurface() 也返回空指针。

frameworks/av/media/codec2/sfplugin/CCodec.cpp

```
extern "C" android::PersistentSurface *CreateInputSurface() {
    using namespace android;
    // Attempt to create a Codec2's input surface.
    std::shared_ptr<Codec2Client::InputSurface> inputSurface =
            Codec2Client::CreateInputSurface();
    if (!inputSurface) {
        if (property_get_int32("debug.stagefright.c2inputsurface", 0) == -1) {
            sp<IGraphicBufferProducer> gbp;
            sp<OmxGraphicBufferSource> gbs = new OmxGraphicBufferSource();
            status_t err = gbs->initCheck();
            if (err != OK) {
                ALOGE("Failed to create persistent input surface: error %d", err);
                return nullptr;
            }
            return new PersistentSurface(
                    gbs->getIGraphicBufferProducer(),
                    sp<IGraphicBufferSource>(
                        new Omx2IGraphicBufferSource(gbs)));
        } else {
            return nullptr;
        }
    }
    return new PersistentSurface(
            inputSurface->getGraphicBufferProducer(),
            static_cast<sp<android::hidl::base::V1_0::IBase>>(
            inputSurface->getHalInterface()));
}
11011121314151617181920212223242526
```

Codec2Client 结构体定义在 codec2/hidl/client.h 中。

frameworks/av/media/codec2/hidl/client/include/codec2/hidl/client.h

```
struct Codec2Client : public Codec2ConfigurableClient {
    ......
    // Create an input surface.
    static std::shared_ptr<InputSurface> CreateInputSurface(
            char const* serviceName = nullptr);
    ......
}

```

瑞芯微 rk 平台上 debug.stagefright.c2inputsurface 属性值没有设置，所以默认为 0。这里直接返回 nullptr。

frameworks/av/media/codec2/hidl/client/client.cpp

```
std::shared_ptr<Codec2Client::InputSurface> Codec2Client::CreateInputSurface(
        char const* serviceName) {
    int32_t inputSurfaceSetting = ::android::base::GetIntProperty(
            "debug.stagefright.c2inputsurface", int32_t(0));
    if (inputSurfaceSetting <= 0) {
        return nullptr;
    }
    ......
}

```

现在继续回到 MediaCodecList.cpp GetBuilders() 函数，目前已经知道会添加 sOmxInfoBuilder 到容器。sOmxInfoBuilder 是一个 OmxInfoBuilder 类的实例，并且默认允许 surface encoder，除了添加 sOmxInfoBuilder 还会继续添加 GetCodec2InfoBuilder() 函数返回的 builder，这个函数内部调用了 sfplugin 中的函数 CreateBuilder() 。

可以看到此处直接返回了 Codec2InfoBuilder 对象。

frameworks/av/media/codec2/sfplugin/Codec2InfoBuilder.cpp

Codec2InfoBuilder 类构造器什么也没干。

frameworks/av/media/codec2/sfplugin/include/media/stagefright/Codec2InfoBuilder.h

```
namespace android {

class Codec2InfoBuilder : public MediaCodecListBuilderBase {
public:
    Codec2InfoBuilder() = default;
    ~Codec2InfoBuilder() override = default;
    status_t buildMediaCodecList(MediaCodecListWriter* writer) override;
};

}  // namespace android
1
```

在 MediaCodecList 构造函数中，根据以上分析不难得出 builder 调用其 buildMediaCodecList(…)，其一是 OmxInfoBuilder，其二是 Codec2InfoBuilder。至于如何构造 MediaCodec 列表下一节再来分析。回到主线上，MediaCodecList.cpp 中 MediaCodecList::findMatchingCodecs(…) 函数内部调用 BpMediaCodecList 的 findCodecByType(…) 方法，实际由 MediaCodecList::findCodecByType(…) 处理返回匹配的 index。

1. 遍历容器中所有的 MediaCodecInfo；
2. 如果发现 MediaCodecInfo isEncoder() 返回不是 false，则跳过这一项，说明不是解码器；
3. 如果发现 MediaCodecInfo getCapabilitiesFor(…) 返回为 nullptr，也跳过这一项，说明这一项没有匹配的解码能力；
4. 检查高级特性是否支持加密的解码器、隧道播放解码器，在不支持这两种特性时返回对应的 index。

frameworks/av/media/libstagefright/MediaCodecList.cpp

```
ssize_t MediaCodecList::findCodecByType(
        const char *type, bool encoder, size_t startIndex) const {
    static const char *advancedFeatures[] = {
        "feature-secure-playback",
        "feature-tunneled-playback",
    };

    size_t numCodecInfos = mCodecInfos.size();
    for (; startIndex < numCodecInfos; ++startIndex) {
        const MediaCodecInfo &info = *mCodecInfos[startIndex];

        if (info.isEncoder() != encoder) {
            continue;
        }
        sp<MediaCodecInfo::Capabilities> capabilities = info.getCapabilitiesFor(type);
        if (capabilities == nullptr) {
            continue;
        }
        const sp<AMessage> &details = capabilities->getDetails();

        int32_t required;
        bool isAdvanced = false;
        for (size_t ix = 0; ix < ARRAY_SIZE(advancedFeatures); ix++) {
            if (details->findInt32(advancedFeatures[ix], &required) &&
                    required != 0) {
                isAdvanced = true;
                break;
            }
        }

        if (!isAdvanced) {
            return startIndex;
        }
    }

    return -ENOENT;
}

```

再来看 MediaCodecList.cpp 中 MediaCodecList::findMatchingCodecs(…) 函数内部调用 BpMediaCodecList 的 getCodecInfo(…) 方法，实际由 MediaCodecList::getCodecInfo(…) 返回 MediaCodecInfo，其实就是根据 index 直接返回 mCodecInfos 指向的容器（std::vector<sp >）中的对应项。

frameworks/av/media/libstagefright/include/media/stagefright/MediaCodecList.h

```
struct MediaCodecList : public BnMediaCodecList {
    ......
    virtual sp<MediaCodecInfo> getCodecInfo(size_t index) const {
        if (index >= mCodecInfos.size()) {
            ALOGE("b/24445127");
            return NULL;
        }
        return mCodecInfos[index];
    }
    ......
}

```

现在终于可以分析 MediaCodec cpp 构造函数以及其 init 流程。构造器中给各种字段进行了初始化。

frameworks/av/media/libstagefright/MediaCodec.cpp

```
MediaCodec::MediaCodec(const sp<ALooper> &looper, pid_t pid, uid_t uid)
    : mState(UNINITIALIZED),
      mReleasedByResourceManager(false),
      mLooper(looper),
      mCodec(NULL),
      mReplyID(0),
      mFlags(0),
      mStickyError(OK),
      mSoftRenderer(NULL),
      mAnalyticsItem(NULL),
      mResourceManagerClient(new ResourceManagerClient(this)),
      mResourceManagerService(new ResourceManagerServiceProxy(pid)),
      mBatteryStatNotified(false),
      mIsVideo(false),
      mVideoWidth(0),
      mVideoHeight(0),
      mRotationDegrees(0),
      mDequeueInputTimeoutGeneration(0),
      mDequeueInputReplyID(0),
      mDequeueOutputTimeoutGeneration(0),
      mDequeueOutputReplyID(0),
      mHaveInputSurface(false),
      mHavePendingInputBuffers(false),
      mCpuBoostRequested(false),
      mLatencyUnknown(0) {
    if (uid == kNoUid) {
        mUid = IPCThreadState::self()->getCallingUid();
    } else {
        mUid = uid;
    }

    initAnalyticsItem();
}

```

1. 名称如果包含 .secure 字符串，将 secureCodec 置为 true；
2. 根据名称获取相应的 MediaCodecInfo，并检查是否支持视频解码；
3. 调用 GetCodecBase(…) 创建继承自 CodecBase 的子类获取解码器；
4. 如果是视频编解码器创建专用的 looper，此时对应注册的 handler 就是上一步创建的 CodecBase 子类 ；
5. mLooper 对应注册的 handler 就是 MediaCodec 对象本身；
6. 设置 CodecBase 子类回调，设置 BufferChannelBase 子类回调；
7. 调用 ResourceManagerService reclaimResource(…) 进行资源回收。

frameworks/av/media/libstagefright/MediaCodec.cpp

```
status_t MediaCodec::init(const AString &name) {
    mResourceManagerService->init();

    // save init parameters for reset
    mInitName = name;

    // Current video decoders do not return from OMX_FillThisBuffer
    // quickly, violating the OpenMAX specs, until that is remedied
    // we need to invest in an extra looper to free the main event
    // queue.

    mCodecInfo.clear();

    bool secureCodec = false;
    AString tmp = name;
    if (tmp.endsWith(".secure")) {
        secureCodec = true;
        tmp.erase(tmp.size() - 7, 7);
    }
    const sp<IMediaCodecList> mcl = MediaCodecList::getInstance();
    if (mcl == NULL) {
        mCodec = NULL;  // remove the codec.
        return NO_INIT; // if called from Java should raise IOException
    }
    for (const AString &codecName : { name, tmp }) {
        ssize_t codecIdx = mcl->findCodecByName(codecName.c_str());
        if (codecIdx < 0) {
            continue;
        }
        mCodecInfo = mcl->getCodecInfo(codecIdx);
        Vector<AString> mediaTypes;
        mCodecInfo->getSupportedMediaTypes(&mediaTypes);
        for (size_t i = 0; i < mediaTypes.size(); i++) {
            if (mediaTypes[i].startsWith("video/")) {
                mIsVideo = true;
                break;
            }
        }
        break;
    }
    if (mCodecInfo == nullptr) {
        return NAME_NOT_FOUND;
    }

    mCodec = GetCodecBase(name, mCodecInfo->getOwnerName());
    if (mCodec == NULL) {
        return NAME_NOT_FOUND;
    }

    if (mIsVideo) {
        // video codec needs dedicated looper
        if (mCodecLooper == NULL) {
            mCodecLooper = new ALooper;
            mCodecLooper->setName("CodecLooper");
            mCodecLooper->start(false, false, ANDROID_PRIORITY_AUDIO);
        }

        mCodecLooper->registerHandler(mCodec);
    } else {
        mLooper->registerHandler(mCodec);
    }

    mLooper->registerHandler(this);

    mCodec->setCallback(
            std::unique_ptr<CodecBase::CodecCallback>(
                    new CodecCallback(new AMessage(kWhatCodecNotify, this))));
    mBufferChannel = mCodec->getBufferChannel();
    mBufferChannel->setCallback(
            std::unique_ptr<CodecBase::BufferCallback>(
                    new BufferCallback(new AMessage(kWhatCodecNotify, this))));

    sp<AMessage> msg = new AMessage(kWhatInit, this);
    msg->setObject("codecInfo", mCodecInfo);
    // name may be different from mCodecInfo->getCodecName() if we stripped
    // ".secure"
    msg->setString("name", name);

    if (mAnalyticsItem != NULL) {
        mAnalyticsItem->setCString(kCodecCodec, name.c_str());
        mAnalyticsItem->setCString(kCodecMode, mIsVideo ? kCodecModeVideo : kCodecModeAudio);
    }

    status_t err;
    Vector<MediaResource> resources;
    MediaResource::Type type =
            secureCodec ? MediaResource::kSecureCodec : MediaResource::kNonSecureCodec;
    MediaResource::SubType subtype =
            mIsVideo ? MediaResource::kVideoCodec : MediaResource::kAudioCodec;
    resources.push_back(MediaResource(type, subtype, 1));
    for (int i = 0; i <= kMaxRetry; ++i) {
        if (i > 0) {
            // Don't try to reclaim resource for the first time.
            if (!mResourceManagerService->reclaimResource(resources)) {
                break;
            }
        }

        sp<AMessage> response;
        err = PostAndAwaitResponse(msg, &response);
        if (!isResourceError(err)) {
            break;
        }
    }
    return err;
}
100101102103104105
```

MediaCodec::GetCodecBase(…) 方法中根据不同的 owner 和 name 创建不同的对象或结构体 ACodec 、android::CCodec 或 MediaFilter。

添加 Log 后 rk3399 平台上 Log 打印如下：

MediaCodec: GetCodecBase name=OMX.rk.video_decoder.avc owner=default

所以实际创建了 ACodec 结构体。

frameworks/av/media/libstagefright/MediaCodec.cpp

```
static CodecBase *CreateCCodec() {
    return StagefrightPluginLoader::GetCCodecInstance()->createCodec();
}

//static
sp<CodecBase> MediaCodec::GetCodecBase(const AString &name, const char *owner) {
    if (owner) {
        if (strcmp(owner, "default") == 0) {
            return new ACodec;
        } else if (strncmp(owner, "codec2", 6) == 0) {
            return CreateCCodec();
        }
    }

    if (name.startsWithIgnoreCase("c2.")) {
        return CreateCCodec();
    } else if (name.startsWithIgnoreCase("omx.")) {
        // at this time only ACodec specifies a mime type.
        return new ACodec;
    } else if (name.startsWithIgnoreCase("android.filter.")) {
        return new MediaFilter;
    } else {
        return NULL;
    }
}

```

这个初始化给一系列字段赋了初值，对九种状态进行了初始化，并在构造函数中将状态改为未初始化。

frameworks/av/media/libstagefright/ACodec.cpp

```
ACodec::ACodec()
    : mSampleRate(0),
      mNodeGeneration(0),
      mUsingNativeWindow(false),
      mNativeWindowUsageBits(0),
      mLastNativeWindowDataSpace(HAL_DATASPACE_UNKNOWN),
      mIsVideo(false),
      mIsImage(false),
      mIsEncoder(false),
      mFatalError(false),
      mShutdownInProgress(false),
      mExplicitShutdown(false),
      mIsLegacyVP9Decoder(false),
      mEncoderDelay(0),
      mEncoderPadding(0),
      mRotationDegrees(0),
      mChannelMaskPresent(false),
      mChannelMask(0),
      mDequeueCounter(0),
      mMetadataBuffersToSubmit(0),
      mNumUndequeuedBuffers(0),
      mRepeatFrameDelayUs(-1LL),
      mMaxPtsGapUs(0LL),
      mMaxFps(-1),
      mFps(-1.0),
      mCaptureFps(-1.0),
      mCreateInputBuffersSuspended(false),
      mTunneled(false),
      mDescribeColorAspectsIndex((OMX_INDEXTYPE)0),
      mDescribeHDRStaticInfoIndex((OMX_INDEXTYPE)0),
      mDescribeHDR10PlusInfoIndex((OMX_INDEXTYPE)0),
      mStateGeneration(0),
      mVendorExtensionsStatus(kExtensionsUnchecked) {
    memset(&mLastHDRStaticInfo, 0, sizeof(mLastHDRStaticInfo));

    mUninitializedState = new UninitializedState(this);
    mLoadedState = new LoadedState(this);
    mLoadedToIdleState = new LoadedToIdleState(this);
    mIdleToExecutingState = new IdleToExecutingState(this);
    mExecutingState = new ExecutingState(this);

    mOutputPortSettingsChangedState =
        new OutputPortSettingsChangedState(this);

    mExecutingToIdleState = new ExecutingToIdleState(this);
    mIdleToLoadedState = new IdleToLoadedState(this);
    mFlushingState = new FlushingState(this);

    mPortEOS[kPortIndexInput] = mPortEOS[kPortIndexOutput] = false;
    mInputEOSResult = OK;

    mPortMode[kPortIndexInput] = IOMX::kPortModePresetByteBuffer;
    mPortMode[kPortIndexOutput] = IOMX::kPortModePresetByteBuffer;

    memset(&mLastNativeWindowCrop, 0, sizeof(mLastNativeWindowCrop));

    changeState(mUninitializedState);
}

```

