# 深入理解构建 MediaCodec 列表：7-MediaCodec configure



> 使用 MediaCodec 的关键一步是 configure，在 start 之前配置是必须的。以[解码](https://so.csdn.net/so/search?q=%E8%A7%A3%E7%A0%81&spm=1001.2101.3001.7020)为例，并且直接解码到外部提供的 surface 上，所以输入的 surface 不为空。

![](/content/assets/images/configure%E6%B5%81%E7%A8%8B.jpg)

#### configure(…) 提供给外部调用的 public 方法内部调用了同名重载版本的方法。

1. 遍历从 MediaFormat 返回的 map，构造为 key 和 value 数组；
2. 调用 native_configure(…) 交给 native 层处理。

> frameworks/base/media/java/android/media/MediaCodec.java

```java
    public void configure(
            @Nullable MediaFormat format,
            @Nullable Surface surface, @Nullable MediaCrypto crypto,
            @ConfigureFlag int flags) {
        configure(format, surface, crypto, null, flags);
    }
    ......
    private void configure(
            @Nullable MediaFormat format, @Nullable Surface surface,
            @Nullable MediaCrypto crypto, @Nullable IHwBinder descramblerBinder,
            @ConfigureFlag int flags) {
        if (crypto != null && descramblerBinder != null) {
            throw new IllegalArgumentException("Can't use crypto and descrambler together!");
        }

        String[] keys = null;
        Object[] values = null;

        if (format != null) {
            Map<String, Object> formatMap = format.getMap();
            keys = new String[formatMap.size()];
            values = new Object[formatMap.size()];

            int i = 0;
            for (Map.Entry<String, Object> entry: formatMap.entrySet()) {
                if (entry.getKey().equals(MediaFormat.KEY_AUDIO_SESSION_ID)) {
                    int sessionId = 0;
                    try {
                        sessionId = (Integer)entry.getValue();
                    }
                    catch (Exception e) {
                        throw new IllegalArgumentException("Wrong Session ID Parameter!");
                    }
                    keys[i] = "audio-hw-sync";
                    values[i] = AudioSystem.getAudioHwSyncForSession(sessionId);
                } else {
                    keys[i] = entry.getKey();
                    values[i] = entry.getValue();
                }
                ++i;
            }
        }

        mHasSurface = surface != null;
        mCrypto = crypto;

        native_configure(keys, values, surface, crypto, descramblerBinder, flags);
    }
```

#### native_configure(…) jni 具体实现

native_configure(…) jni 实现具体为 android_media_MediaCodec.cpp::android_media_MediaCodec_native_configure(…) 

1. 调用 getMediaCodec(…) 获取 JMediaCodec；
2. 调用 ConvertKeyValueArraysToMessage(…) 将 key 和 value 数组转化为 AMessage；
3. 将 java surface 对象转化为 native Surface，并调用 native Surface getIGraphicBufferProducer() 获取 IGraphicBufferProducer；
4. 调用 JMediaCodec configure(…) 方法。

> frameworks/base/media/jni/android_media_MediaCodec.cpp

```cpp
static void android_media_MediaCodec_native_configure(
        JNIEnv *env,
        jobject thiz,
        jobjectArray keys, jobjectArray values,
        jobject jsurface,
        jobject jcrypto,
        jobject descramblerBinderObj,
        jint flags) {
    sp<JMediaCodec> codec = getMediaCodec(env, thiz);

    if (codec == NULL) {
        throwExceptionAsNecessary(env, INVALID_OPERATION);
        return;
    }

    sp<AMessage> format;
    status_t err = ConvertKeyValueArraysToMessage(env, keys, values, &format);

    if (err != OK) {
        jniThrowException(env, "java/lang/IllegalArgumentException", NULL);
        return;
    }

    sp<IGraphicBufferProducer> bufferProducer;
    if (jsurface != NULL) {
        sp<Surface> surface(android_view_Surface_getSurface(env, jsurface));
        if (surface != NULL) {
            bufferProducer = surface->getIGraphicBufferProducer();
        } else {
            jniThrowException(
                    env,
                    "java/lang/IllegalArgumentException",
                    "The surface has been released");
            return;
        }
    }

    sp<ICrypto> crypto;
    if (jcrypto != NULL) {
        crypto = JCrypto::GetCrypto(env, jcrypto);
    }

    sp<IDescrambler> descrambler;
    if (descramblerBinderObj != NULL) {
        descrambler = GetDescrambler(env, descramblerBinderObj);
    }

    err = codec->configure(format, bufferProducer, crypto, descrambler, flags);

    throwExceptionAsNecessary(env, err);
}
```

这里只是简单的调用 gFields.lockAndGetContextID 指向的 java 方法（lockAndGetContext()），并将返回值强转为 JMediaCodec * 指针。此方法是 setMediaCodec(…) 的逆操作。

> frameworks/base/media/jni/android_media_MediaCodec.cpp

```cpp
static sp<JMediaCodec> getMediaCodec(JNIEnv *env, jobject thiz) {
    sp<JMediaCodec> codec = (JMediaCodec *)env->CallLongMethod(thiz, gFields.lockAndGetContextID);
    env->CallVoidMethod(thiz, gFields.setAndUnlockContextID, (jlong)codec.get());
    return codec;
}
```

方法代码有点长，实际上核心逻辑很简单：[遍历](https://so.csdn.net/so/search?q=%E9%81%8D%E5%8E%86&spm=1001.2101.3001.7020)所有的 key 和 value 数组从其 java 对象中获取值用来构建 AMessage。

> frameworks/base/media/jni/android_media_Streams.cpp

```cpp
status_t ConvertKeyValueArraysToMessage(
        JNIEnv *env, jobjectArray keys, jobjectArray values,
        sp<AMessage> *out) {
    ScopedLocalRef<jclass> stringClass(env, env->FindClass("java/lang/String"));
    CHECK(stringClass.get() != NULL);
    ScopedLocalRef<jclass> integerClass(env, env->FindClass("java/lang/Integer"));
    CHECK(integerClass.get() != NULL);
    ScopedLocalRef<jclass> longClass(env, env->FindClass("java/lang/Long"));
    CHECK(longClass.get() != NULL);
    ScopedLocalRef<jclass> floatClass(env, env->FindClass("java/lang/Float"));
    CHECK(floatClass.get() != NULL);
    ScopedLocalRef<jclass> byteBufClass(env, env->FindClass("java/nio/ByteBuffer"));
    CHECK(byteBufClass.get() != NULL);

    sp<AMessage> msg = new AMessage;

    jsize numEntries = 0;

    if (keys != NULL) {
        if (values == NULL) {
            return -EINVAL;
        }

        numEntries = env->GetArrayLength(keys);

        if (numEntries != env->GetArrayLength(values)) {
            return -EINVAL;
        }
    } else if (values != NULL) {
        return -EINVAL;
    }

    for (jsize i = 0; i < numEntries; ++i) {
        jobject keyObj = env->GetObjectArrayElement(keys, i);

        if (!env->IsInstanceOf(keyObj, stringClass.get())) {
            return -EINVAL;
        }

        const char *tmp = env->GetStringUTFChars((jstring)keyObj, NULL);

        if (tmp == NULL) {
            return -ENOMEM;
        }

        AString key = tmp;

        env->ReleaseStringUTFChars((jstring)keyObj, tmp);
        tmp = NULL;

        if (key.startsWith("android._")) {
            // don't propagate private keys (starting with android._)
            continue;
        }

        jobject valueObj = env->GetObjectArrayElement(values, i);

        if (env->IsInstanceOf(valueObj, stringClass.get())) {
            const char *value = env->GetStringUTFChars((jstring)valueObj, NULL);

            if (value == NULL) {
                return -ENOMEM;
            }

            msg->setString(key.c_str(), value);

            env->ReleaseStringUTFChars((jstring)valueObj, value);
            value = NULL;
        } else if (env->IsInstanceOf(valueObj, integerClass.get())) {
            jmethodID intValueID =
                env->GetMethodID(integerClass.get(), "intValue", "()I");
            CHECK(intValueID != NULL);

            jint value = env->CallIntMethod(valueObj, intValueID);

            msg->setInt32(key.c_str(), value);
        } else if (env->IsInstanceOf(valueObj, longClass.get())) {
            jmethodID longValueID =
                env->GetMethodID(longClass.get(), "longValue", "()J");
            CHECK(longValueID != NULL);

            jlong value = env->CallLongMethod(valueObj, longValueID);

            msg->setInt64(key.c_str(), value);
        } else if (env->IsInstanceOf(valueObj, floatClass.get())) {
            jmethodID floatValueID =
                env->GetMethodID(floatClass.get(), "floatValue", "()F");
            CHECK(floatValueID != NULL);

            jfloat value = env->CallFloatMethod(valueObj, floatValueID);

            msg->setFloat(key.c_str(), value);
        } else if (env->IsInstanceOf(valueObj, byteBufClass.get())) {
            jmethodID positionID =
                env->GetMethodID(byteBufClass.get(), "position", "()I");
            CHECK(positionID != NULL);

            jmethodID limitID =
                env->GetMethodID(byteBufClass.get(), "limit", "()I");
            CHECK(limitID != NULL);

            jint position = env->CallIntMethod(valueObj, positionID);
            jint limit = env->CallIntMethod(valueObj, limitID);

            sp<ABuffer> buffer = new ABuffer(limit - position);

            void *data = env->GetDirectBufferAddress(valueObj);

            if (data != NULL) {
                memcpy(buffer->data(),
                       (const uint8_t *)data + position,
                       buffer->size());
            } else {
                jmethodID arrayID =
                    env->GetMethodID(byteBufClass.get(), "array", "()[B");
                CHECK(arrayID != NULL);

                jbyteArray byteArray =
                    (jbyteArray)env->CallObjectMethod(valueObj, arrayID);
                CHECK(byteArray != NULL);

                env->GetByteArrayRegion(
                        byteArray,
                        position,
                        buffer->size(),
                        (jbyte *)buffer->data());

                env->DeleteLocalRef(byteArray); byteArray = NULL;
            }

            msg->setBuffer(key.c_str(), buffer);
        }
    }

    *out = msg;

    return OK;
}
```

此方法在主流程上实际调用了 [native](https://so.csdn.net/so/search?q=native&spm=1001.2101.3001.7020) MediaCodec configure(…) 方法。由于下面分析会用到 Surface 的创建，这里重点再去看下 Surface 的构造函数。

> frameworks/base/media/jni/android_media_MediaCodec.cpp

```cpp
status_t JMediaCodec::configure(
        const sp<AMessage> &format,
        const sp<IGraphicBufferProducer> &bufferProducer,
        const sp<ICrypto> &crypto,
        const sp<IDescrambler> &descrambler,
        int flags) {
    sp<Surface> client;
    if (bufferProducer != NULL) {
        mSurfaceTextureClient =
            new Surface(bufferProducer, true /* controlledByApp */);
    } else {
        mSurfaceTextureClient.clear();
    }

    return mCodec->configure(
            format, mSurfaceTextureClient, crypto, descrambler, flags);
}
```

请把目光聚集到初始化 ANativeWindow 函数指针这块代码，不难知道 ANativeWindow::perform 指向了 Surface 中的方法 hook_perform(…)。

> frameworks/native/libs/gui/Surface.cpp

```cpp
Surface::Surface(const sp<IGraphicBufferProducer>& bufferProducer, bool controlledByApp)
      : mGraphicBufferProducer(bufferProducer),
        mCrop(Rect::EMPTY_RECT),
        mBufferAge(0),
        mGenerationNumber(0),
        mSharedBufferMode(false),
        mAutoRefresh(false),
        mSharedBufferSlot(BufferItem::INVALID_BUFFER_SLOT),
        mSharedBufferHasBeenQueued(false),
        mQueriedSupportedTimestamps(false),
        mFrameTimestampsSupportsPresent(false),
        mEnableFrameTimestamps(false),
        mFrameEventHistory(std::make_unique<ProducerFrameEventHistory>()) {
    // Initialize the ANativeWindow function pointers.
    ANativeWindow::setSwapInterval  = hook_setSwapInterval;
    ANativeWindow::dequeueBuffer    = hook_dequeueBuffer;
    ANativeWindow::cancelBuffer     = hook_cancelBuffer;
    ANativeWindow::queueBuffer      = hook_queueBuffer;
    ANativeWindow::query            = hook_query;
    ANativeWindow::perform          = hook_perform;

    ANativeWindow::dequeueBuffer_DEPRECATED = hook_dequeueBuffer_DEPRECATED;
    ANativeWindow::cancelBuffer_DEPRECATED  = hook_cancelBuffer_DEPRECATED;
    ANativeWindow::lockBuffer_DEPRECATED    = hook_lockBuffer_DEPRECATED;
    ANativeWindow::queueBuffer_DEPRECATED   = hook_queueBuffer_DEPRECATED;

    const_cast<int&>(ANativeWindow::minSwapInterval) = 0;
    const_cast<int&>(ANativeWindow::maxSwapInterval) = 1;

    mReqWidth = 0;
    mReqHeight = 0;
    mReqFormat = 0;
    mReqUsage = 0;
    mTimestamp = NATIVE_WINDOW_TIMESTAMP_AUTO;
    mDataSpace = Dataspace::UNKNOWN;
    mScalingMode = NATIVE_WINDOW_SCALING_MODE_FREEZE;
    mTransform = 0;
    mStickyTransform = 0;
    mDefaultWidth = 0;
    mDefaultHeight = 0;
    mUserWidth = 0;
    mUserHeight = 0;
    mTransformHint = 0;
    mConsumerRunningBehind = false;
    mConnectedToCpu = false;
    mProducerControlledByApp = controlledByApp;
    mSwapIntervalZero = false;
}
```

首先新建 AMessage，消息 what ID 为 kWhatConfigure，然后从 format 中检出各种字段用来设置 MediaAnalyticsItem 中适当的值。将入参 format、nativeWindow 和 flags 设置到 AMessage。最后调用 ResourceManagerService reclaimResource(…) 进行资源回收，并调用 PostAndAwaitResponse(…) 等待消息处理。

> frameworks/av/media/libstagefright/MediaCodec.cpp

```cpp
status_t MediaCodec::configure(
        const sp<AMessage> &format,
        const sp<Surface> &nativeWindow,
        const sp<ICrypto> &crypto,
        uint32_t flags) {
    return configure(format, nativeWindow, crypto, NULL, flags);
}

status_t MediaCodec::configure(
        const sp<AMessage> &format,
        const sp<Surface> &surface,
        const sp<ICrypto> &crypto,
        const sp<IDescrambler> &descrambler,
        uint32_t flags) {
    sp<AMessage> msg = new AMessage(kWhatConfigure, this);

    if (mAnalyticsItem != NULL) {
        int32_t profile = 0;
        if (format->findInt32("profile", &profile)) {
            mAnalyticsItem->setInt32(kCodecProfile, profile);
        }
        int32_t level = 0;
        if (format->findInt32("level", &level)) {
            mAnalyticsItem->setInt32(kCodecLevel, level);
        }
        mAnalyticsItem->setInt32(kCodecEncoder, (flags & CONFIGURE_FLAG_ENCODE) ? 1 : 0);
    }

    if (mIsVideo) {
        format->findInt32("width", &mVideoWidth);
        format->findInt32("height", &mVideoHeight);
        if (!format->findInt32("rotation-degrees", &mRotationDegrees)) {
            mRotationDegrees = 0;
        }

        if (mAnalyticsItem != NULL) {
            mAnalyticsItem->setInt32(kCodecWidth, mVideoWidth);
            mAnalyticsItem->setInt32(kCodecHeight, mVideoHeight);
            mAnalyticsItem->setInt32(kCodecRotation, mRotationDegrees);
            int32_t maxWidth = 0;
            if (format->findInt32("max-width", &maxWidth)) {
                mAnalyticsItem->setInt32(kCodecMaxWidth, maxWidth);
            }
            int32_t maxHeight = 0;
            if (format->findInt32("max-height", &maxHeight)) {
                mAnalyticsItem->setInt32(kCodecMaxHeight, maxHeight);
            }
        }

        // Prevent possible integer overflow in downstream code.
        if (mVideoWidth < 0 || mVideoHeight < 0 ||
               (uint64_t)mVideoWidth * mVideoHeight > (uint64_t)INT32_MAX / 4) {
            ALOGE("Invalid size(s), width=%d, height=%d", mVideoWidth, mVideoHeight);
            return BAD_VALUE;
        }
    }

    msg->setMessage("format", format);
    msg->setInt32("flags", flags);
    msg->setObject("surface", surface);

    if (crypto != NULL || descrambler != NULL) {
        if (crypto != NULL) {
            msg->setPointer("crypto", crypto.get());
        } else {
            msg->setPointer("descrambler", descrambler.get());
        }
        if (mAnalyticsItem != NULL) {
            mAnalyticsItem->setInt32(kCodecCrypto, 1);
        }
    } else if (mFlags & kFlagIsSecure) {
        ALOGW("Crypto or descrambler should be given for secure codec");
    }

    // save msg for reset
    mConfigureMsg = msg;

    status_t err;
    Vector<MediaResource> resources;
    MediaResource::Type type = (mFlags & kFlagIsSecure) ?
            MediaResource::kSecureCodec : MediaResource::kNonSecureCodec;
    MediaResource::SubType subtype =
            mIsVideo ? MediaResource::kVideoCodec : MediaResource::kAudioCodec;
    resources.push_back(MediaResource(type, subtype, 1));
    // Don't know the buffer size at this point, but it's fine to use 1 because
    // the reclaimResource call doesn't consider the requester's buffer size for now.
    resources.push_back(MediaResource(MediaResource::kGraphicMemory, 1));
    for (int i = 0; i <= kMaxRetry; ++i) {
        if (i > 0) {
            // Don't try to reclaim resource for the first time.
            if (!mResourceManagerService->reclaimResource(resources)) {
                break;
            }
        }

        sp<AMessage> response;
        err = PostAndAwaitResponse(msg, &response);
        if (err != OK && err != INVALID_OPERATION) {
            // MediaCodec now set state to UNINITIALIZED upon any fatal error.
            // To maintain backward-compatibility, do a reset() to put codec
            // back into INITIALIZED state.
            // But don't reset if the err is INVALID_OPERATION, which means
            // the configure failure is due to wrong state.

            ALOGE("configure failed with err 0x%08x, resetting...", err);
            reset();
        }
        if (!isResourceError(err)) {
            break;
        }
    }

    return err;
}
```

1. 调用 handleSetSurface(…) 处理 surface 设置；
2. 调用 setState(…) 将状态修改为 CONFIGURING；
3. 调用 extractCSD(…) 提出 CSD；
4. 调用 initiateConfigureComponent(…) 初始化组件。

此处需要注意一下 mAllowFrameDroppingBySurface 的值，由于 surface 不为空，因此就会查找 allow-[frame](https://so.csdn.net/so/search?q=frame&spm=1001.2101.3001.7020)-drop 为 key 的 value，但这个 key-value 对并未设置，所以 mAllowFrameDroppingBySurface 会被默认赋值为 true。

> frameworks/av/media/libstagefright/MediaCodec.cpp

```cpp
void MediaCodec::onMessageReceived(const sp<AMessage> &msg) {
    switch (msg->what()) {
        ......
        case kWhatConfigure:
        {
            sp<AReplyToken> replyID;
            CHECK(msg->senderAwaitsResponse(&replyID));

            if (mState != INITIALIZED) {
                PostReplyWithError(replyID, INVALID_OPERATION);
                break;
            }

            sp<RefBase> obj;
            CHECK(msg->findObject("surface", &obj));

            sp<AMessage> format;
            CHECK(msg->findMessage("format", &format));

            int32_t push;
            if (msg->findInt32("push-blank-buffers-on-shutdown", &push) && push != 0) {
                mFlags |= kFlagPushBlankBuffersOnShutdown;
            }

            if (obj != NULL) {
                if (!format->findInt32("allow-frame-drop", &mAllowFrameDroppingBySurface)) {
                    // allow frame dropping by surface by default
                    mAllowFrameDroppingBySurface = true;
                }

                format->setObject("native-window", obj);
                status_t err = handleSetSurface(static_cast<Surface *>(obj.get()));
                if (err != OK) {
                    PostReplyWithError(replyID, err);
                    break;
                }
            } else {
                // we are not using surface so this variable is not used, but initialize sensibly anyway
                mAllowFrameDroppingBySurface = false;

                handleSetSurface(NULL);
            }

            mReplyID = replyID;
            setState(CONFIGURING);

            void *crypto;
            if (!msg->findPointer("crypto", &crypto)) {
                crypto = NULL;
            }

            ALOGV("kWhatConfigure: Old mCrypto: %p (%d)",
                    mCrypto.get(), (mCrypto != NULL ? mCrypto->getStrongCount() : 0));

            mCrypto = static_cast<ICrypto *>(crypto);
            mBufferChannel->setCrypto(mCrypto);

            ALOGV("kWhatConfigure: New mCrypto: %p (%d)",
                    mCrypto.get(), (mCrypto != NULL ? mCrypto->getStrongCount() : 0));

            void *descrambler;
            if (!msg->findPointer("descrambler", &descrambler)) {
                descrambler = NULL;
            }

            mDescrambler = static_cast<IDescrambler *>(descrambler);
            mBufferChannel->setDescrambler(mDescrambler);

            uint32_t flags;
            CHECK(msg->findInt32("flags", (int32_t *)&flags));

            if (flags & CONFIGURE_FLAG_ENCODE) {
                format->setInt32("encoder", true);
                mFlags |= kFlagIsEncoder;
            }

            extractCSD(format);

            mCodec->initiateConfigureComponent(format);
            break;
        }
        ......
    }
}
```

由于第一次调用 mSurface 必然为空，所以此处就会转向 connectToSurface(…) 调用。

> frameworks/av/media/libstagefright/MediaCodec.cpp

```cpp
status_t MediaCodec::handleSetSurface(const sp<Surface> &surface) {
    status_t err = OK;
    if (mSurface != NULL) {
        (void)disconnectFromSurface();
    }
    if (surface != NULL) {
        err = connectToSurface(surface);
        if (err == OK) {
            mSurface = surface;
        }
    }
    return err;
}
```

1. 调用 nativeWindowConnect(…) 连接此窗口；
2. 生成唯一标志（使用了 pid 作为计算分量），调用 Surface setGenerationNumber(…) 设置唯一标志；
3. 先调用 nativeWindowDisconnect(…) 断开连接后，再次调用 nativeWindowConnect(…) 进行连接窗口。根据注释可以看出是为了清除所有可用的缓冲区（因为消费者可能持有旧的帧，它可以在断开/连接后重新连接到这个 surface，而那些旧的空闲帧将继承新的标志。在设置唯一标志之后断开连接可以防止这种情况）。
4. 由于 mAllowFrameDroppingBySurface 为 true，所以不再调用 disableLegacyBufferDropPostQ(…) 处理。

Surface 是 ANativeWindow 的一个实现，它将图形缓冲区提供给 BufferQueue。所以才可以将 Surface 作为入参直接调用 nativeWindowConnect(…) 和 nativeWindowDisconnect(…) 。

frameworks/av/media/libstagefright/MediaCodec.cpp

```cpp
status_t MediaCodec::connectToSurface(const sp<Surface> &surface) {
    status_t err = OK;
    if (surface != NULL) {
        uint64_t oldId, newId;
        if (mSurface != NULL
                && surface->getUniqueId(&newId) == NO_ERROR
                && mSurface->getUniqueId(&oldId) == NO_ERROR
                && newId == oldId) {
            ALOGI("[%s] connecting to the same surface. Nothing to do.", mComponentName.c_str());
            return ALREADY_EXISTS;
        }

        err = nativeWindowConnect(surface.get(), "connectToSurface");
        if (err == OK) {
            // Require a fresh set of buffers after each connect by using a unique generation
            // number. Rely on the fact that max supported process id by Linux is 2^22.
            // PID is never 0 so we don't have to worry that we use the default generation of 0.
            // TODO: come up with a unique scheme if other producers also set the generation number.
            static uint32_t mSurfaceGeneration = 0;
            uint32_t generation = (getpid() << 10) | (++mSurfaceGeneration & ((1 << 10) - 1));
            surface->setGenerationNumber(generation);
            ALOGI("[%s] setting surface generation to %u", mComponentName.c_str(), generation);

            // HACK: clear any free buffers. Remove when connect will automatically do this.
            // This is needed as the consumer may be holding onto stale frames that it can reattach
            // to this surface after disconnect/connect, and those free frames would inherit the new
            // generation number. Disconnecting after setting a unique generation prevents this.
            nativeWindowDisconnect(surface.get(), "connectToSurface(reconnect)");
            err = nativeWindowConnect(surface.get(), "connectToSurface(reconnect)");
        }

        if (err != OK) {
            ALOGE("nativeWindowConnect returned an error: %s (%d)", strerror(-err), err);
        } else {
            if (!mAllowFrameDroppingBySurface) {
                disableLegacyBufferDropPostQ(surface);
            }
        }
    }
    // do not return ALREADY_EXISTS unless surfaces are the same
    return err == ALREADY_EXISTS ? BAD_VALUE : err;
}
```

nativeWindowConnect(…) 内部调用了 native_window_api_connect(…) 处理，而 nativeWindowDisconnect(…) 则是调用 native_window_api_disconnect(…) 处理。调用时 api 入参均为 NATIVE_WINDOW_API_MEDIA，它表示缓冲将由 Stagefright 在视频解码器填充后进行排队，视频解码器可以是一个软件或硬件解码器。

> frameworks/av/media/libstagefright/SurfaceUtils.cpp

```cpp
status_t nativeWindowConnect(ANativeWindow *surface, const char *reason) {
    ALOGD("connecting to surface %p, reason %s", surface, reason);

    status_t err = native_window_api_connect(surface, NATIVE_WINDOW_API_MEDIA);
    ALOGE_IF(err != OK, "Failed to connect to surface %p, err %d", surface, err);

    return err;
}

status_t nativeWindowDisconnect(ANativeWindow *surface, const char *reason) {
    ALOGD("disconnecting from surface %p, reason %s", surface, reason);

    status_t err = native_window_api_disconnect(surface, NATIVE_WINDOW_API_MEDIA);
    ALOGE_IF(err != OK, "Failed to disconnect from surface %p, err %d", surface, err);

    return err;
}
```

从注释不难知道 native_window_api_connect(…) 的主要作用是将 API 连接到此窗口（ANativeWindow）。一次只能连接一个 API。

native_window_api_disconnect(…) 刚好相反，从这个窗口断开 API。

frameworks/native/libs/nativewindow/include/system/window.h

```cpp
/*
 * native_window_api_connect(..., int api)
 * connects an API to this window. only one API can be connected at a time.
 * Returns -EINVAL if for some reason the window cannot be connected, which
 * can happen if it's connected to some other API.
 */
static inline int native_window_api_connect(
        struct ANativeWindow* window, int api)
{
    return window->perform(window, NATIVE_WINDOW_API_CONNECT, api);
}

/*
 * native_window_api_disconnect(..., int api)
 * disconnect the API from this window.
 * An error is returned if for instance the window wasn't connected in the
 * first place.
 */
static inline int native_window_api_disconnect(
        struct ANativeWindow* window, int api)
{
    return window->perform(window, NATIVE_WINDOW_API_DISCONNECT, api);
}
```

分析到这里结合前面的铺垫，不难知道 ANativeWindow perform(…) 函数调用实际上具体由 Surface hook_perform(…) 实现。

hook_perform(…) 内部仅仅调用了 perform(…) 函数。

frameworks/native/libs/gui/Surface.cpp

```cpp
int Surface::hook_perform(ANativeWindow* window, int operation, ...) {
    va_list args;
    va_start(args, operation);
    Surface* c = getSelf(window);
    int result = c->perform(operation, args);
    va_end(args);
    return result;
}
```

perform(…) 函数内部根据 operation case 到不同分支，具体到此处就会调用 dispatchConnect(…) 和 dispatchDisconnect(…)。

frameworks/native/libs/gui/Surface.cpp

```cpp
int Surface::perform(int operation, va_list args)
{
    int res = NO_ERROR;
    switch (operation) {
    ......
    case NATIVE_WINDOW_API_CONNECT:
        res = dispatchConnect(args);
        break;
    case NATIVE_WINDOW_API_DISCONNECT:
        res = dispatchDisconnect(args);
        break;
    ......
    default:
        res = NAME_NOT_FOUND;
        break;
    }
    return res;
}
```

dispatchConnect(…) 内部仅仅调用了 connect(…)。dispatchDisconnect(…) 调用 disconnect(…)。

frameworks/native/libs/gui/Surface.cpp

```cpp
int Surface::dispatchConnect(va_list args) {
    int api = va_arg(args, int);
    return connect(api);
}

int Surface::dispatchDisconnect(va_list args) {
    int api = va_arg(args, int);
    return disconnect(api);
}
```

connect(int api) 最终会调用其三个入参的重载版本，内部重点关注 BpGraphicBufferProducer 的 connect(…) 方法。

frameworks/native/libs/gui/Surface.cpp

```cpp
int Surface::connect(int api) {
    static sp<IProducerListener> listener = new DummyProducerListener();
    return connect(api, listener);
}

int Surface::connect(int api, const sp<IProducerListener>& listener) {
    return connect(api, listener, false);
}

int Surface::connect(
        int api, const sp<IProducerListener>& listener, bool reportBufferRemoval) {
    ATRACE_CALL();
    ALOGV("Surface::connect");
    Mutex::Autolock lock(mMutex);
    IGraphicBufferProducer::QueueBufferOutput output;
    mReportRemovedBuffers = reportBufferRemoval;
    int err = mGraphicBufferProducer->connect(listener, api, mProducerControlledByApp, &output);
    if (err == NO_ERROR) {
        mDefaultWidth = output.width;
        mDefaultHeight = output.height;
        mNextFrameNumber = output.nextFrameNumber;

        // Ignore transform hint if sticky transform is set or transform to display inverse flag is
        // set. Transform hint should be ignored if the client is expected to always submit buffers
        // in the same orientation.
        if (mStickyTransform == 0 && !transformToDisplayInverse()) {
            mTransformHint = output.transformHint;
        }

        mConsumerRunningBehind = (output.numPendingBuffers >= 2);
    }
    if (!err && api == NATIVE_WINDOW_API_CPU) {
        mConnectedToCpu = true;
        // Clear the dirty region in case we're switching from a non-CPU API
        mDirtyRegion.clear();
    } else if (!err) {
        // Initialize the dirty region for tracking surface damage
        mDirtyRegion = Region::INVALID_REGION;
    }

    return err;
}
```

mode 默认值为 IGraphicBufferProducer::DisconnectMode::Api。Surface::disconnect(…) 实现内部会调用 BpGraphicBufferProducer 的 disconnect(…) 方法。

frameworks/native/libs/gui/Surface.cpp

```cpp
int Surface::disconnect(int api, IGraphicBufferProducer::DisconnectMode mode) {
    ATRACE_CALL();
    ALOGV("Surface::disconnect");
    Mutex::Autolock lock(mMutex);
    mRemovedBuffers.clear();
    mSharedBufferSlot = BufferItem::INVALID_BUFFER_SLOT;
    mSharedBufferHasBeenQueued = false;
    freeAllBuffers();
    int err = mGraphicBufferProducer->disconnect(api, mode);
    if (!err) {
        mReqFormat = 0;
        mReqWidth = 0;
        mReqHeight = 0;
        mReqUsage = 0;
        mCrop.clear();
        mScalingMode = NATIVE_WINDOW_SCALING_MODE_FREEZE;
        mTransform = 0;
        mStickyTransform = 0;

        if (api == NATIVE_WINDOW_API_CPU) {
            mConnectedToCpu = false;
        }
    }
    return err;
}
```

BpGraphicBufferProducer connect(…)、disconnect(…) 会调用到远端 BnGraphicBufferProducer onTransact(…) 去处理。

frameworks/native/libs/gui/IGraphicBufferProducer.cpp

```cpp
class BpGraphicBufferProducer : public BpInterface<IGraphicBufferProducer>
{
    ......
    virtual status_t connect(const sp<IProducerListener>& listener,
            int api, bool producerControlledByApp, QueueBufferOutput* output) {
        Parcel data, reply;
        data.writeInterfaceToken(IGraphicBufferProducer::getInterfaceDescriptor());
        if (listener != nullptr) {
            data.writeInt32(1);
            data.writeStrongBinder(IInterface::asBinder(listener));
        } else {
            data.writeInt32(0);
        }
        data.writeInt32(api);
        data.writeInt32(producerControlledByApp);
        status_t result = remote()->transact(CONNECT, data, &reply);
        if (result != NO_ERROR) {
            return result;
        }
        reply.read(*output);
        result = reply.readInt32();
        return result;
    }

    virtual status_t disconnect(int api, DisconnectMode mode) {
        Parcel data, reply;
        data.writeInterfaceToken(IGraphicBufferProducer::getInterfaceDescriptor());
        data.writeInt32(api);
        data.writeInt32(static_cast<int32_t>(mode));
        status_t result =remote()->transact(DISCONNECT, data, &reply);
        if (result != NO_ERROR) {
            return result;
        }
        result = reply.readInt32();
        return result;
    }
    ......
}
```

BnGraphicBufferProducer 中收到 CONNECT 或 DISCONNECT 消息之后，只是简单的获取消息中的内容后再次调用其 BufferQueueProducer 具体实现中的 connect(…) 和 disconnect(…) 方法。

frameworks/native/libs/gui/IGraphicBufferProducer.cpp

```cpp
status_t BnGraphicBufferProducer::onTransact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    switch(code) {
        ......
        case CONNECT: {
            CHECK_INTERFACE(IGraphicBufferProducer, data, reply);
            sp<IProducerListener> listener;
            if (data.readInt32() == 1) {
                listener = IProducerListener::asInterface(data.readStrongBinder());
            }
            int api = data.readInt32();
            bool producerControlledByApp = data.readInt32();
            QueueBufferOutput output;
            status_t res = connect(listener, api, producerControlledByApp, &output);
            reply->write(output);
            reply->writeInt32(res);
            return NO_ERROR;
        }
         case DISCONNECT: {
            CHECK_INTERFACE(IGraphicBufferProducer, data, reply);
            int api = data.readInt32();
            DisconnectMode mode = static_cast<DisconnectMode>(data.readInt32());
            status_t res = disconnect(api, mode);
            reply->writeInt32(res);
            return NO_ERROR;
        }
        ......
 }
    return BBinder::onTransact(code, data, reply, flags);
}
```

connect(…) 尝试将生产者 API 连接到 BufferQueue。除了getAllocator(…)，必须在调用其他任何 IGraphicBufferProducer 方法之前调用它。消费者必须已经连接。

如果之前在 BufferQueue 上调用了 connect，并且没有进行相应的 disconnect 调用（即如果它仍然连接到一个生产者），这个方法将会失败。

disconnect(…) 尝试从 IGraphicBufferProducer 断开客户端 API。调用此方法将导致对其他 IGraphicBufferProducer 方法的后续调用失败，除了getAllocator(…) 和 connect(…) 。

frameworks/native/libs/gui/include/gui/BufferQueueProducer.h

```cpp
class BufferQueueProducer : public BnGraphicBufferProducer,
                            private IBinder::DeathRecipient {
public:
    ......
    virtual status_t connect(const sp<IProducerListener>& listener,
            int api, bool producerControlledByApp, QueueBufferOutput* output);

    // See IGraphicBufferProducer::disconnect
    virtual status_t disconnect(int api, DisconnectMode mode = DisconnectMode::Api);
    ......
}
```

BufferQueueProducer::connect(…) 方法有点长，实际上却不难阅读。mCore 指向 BufferQueueCore 对象。

1. 检查是否可以正常 connect，不能正常 connect，返回 NO_INIT 或 BAD_VALUE；
2. 调用 BufferQueueCore getMaxBufferCountLocked(…) 返回最大缓冲区数，计算出差异 delta 后调用 BufferQueueCore adjustAvailableSlotsLocked(…) 进行调整；
3. 给入参 output 添加一系列值；
4. 设置死亡通知，以便当远程 producer 死亡时，可以自动断开连接；
5. 最后给 mCore 指向 BufferQueueCore 对象一些字段赋值。

BufferQueueProducer::disconnect(…) 调用 BufferQueueCore freeAllBuffersLocked() 释放资源，并将 mCore 指向 BufferQueueCore 对象的一些字段值恢复到未连接前。

frameworks/native/libs/gui/BufferQueueProducer.cpp

```cpp
status_t BufferQueueProducer::connect(const sp<IProducerListener>& listener,
        int api, bool producerControlledByApp, QueueBufferOutput *output) {
    ATRACE_CALL();
    std::lock_guard<std::mutex> lock(mCore->mMutex);
    mConsumerName = mCore->mConsumerName;
    BQ_LOGV("connect: api=%d producerControlledByApp=%s", api,
            producerControlledByApp ? "true" : "false");

    if (mCore->mIsAbandoned) {
        BQ_LOGE("connect: BufferQueue has been abandoned");
        return NO_INIT;
    }

    if (mCore->mConsumerListener == nullptr) {
        BQ_LOGE("connect: BufferQueue has no consumer");
        return NO_INIT;
    }

    if (output == nullptr) {
        BQ_LOGE("connect: output was NULL");
        return BAD_VALUE;
    }

    if (mCore->mConnectedApi != BufferQueueCore::NO_CONNECTED_API) {
        BQ_LOGE("connect: already connected (cur=%d req=%d)",
                mCore->mConnectedApi, api);
        return BAD_VALUE;
    }

    int delta = mCore->getMaxBufferCountLocked(mCore->mAsyncMode,
            mDequeueTimeout < 0 ?
            mCore->mConsumerControlledByApp && producerControlledByApp : false,
            mCore->mMaxBufferCount) -
            mCore->getMaxBufferCountLocked();
    if (!mCore->adjustAvailableSlotsLocked(delta)) {
        BQ_LOGE("connect: BufferQueue failed to adjust the number of available "
                "slots. Delta = %d", delta);
        return BAD_VALUE;
    }

    int status = NO_ERROR;
    switch (api) {
        case NATIVE_WINDOW_API_EGL:
        case NATIVE_WINDOW_API_CPU:
        case NATIVE_WINDOW_API_MEDIA:
        case NATIVE_WINDOW_API_CAMERA:
            mCore->mConnectedApi = api;

            output->width = mCore->mDefaultWidth;
            output->height = mCore->mDefaultHeight;
            output->transformHint = mCore->mTransformHint;
            output->numPendingBuffers =
                    static_cast<uint32_t>(mCore->mQueue.size());
            output->nextFrameNumber = mCore->mFrameCounter + 1;
            output->bufferReplaced = false;

            if (listener != nullptr) {
                // 设置死亡通知，以便当远程 producer 死亡时，我们可以自动断开连接
                if (IInterface::asBinder(listener)->remoteBinder() != nullptr) {
                    status = IInterface::asBinder(listener)->linkToDeath(
                            static_cast<IBinder::DeathRecipient*>(this));
                    if (status != NO_ERROR) {
                        BQ_LOGE("connect: linkToDeath failed: %s (%d)",
                                strerror(-status), status);
                    }
                    mCore->mLinkedToDeath = listener;
                }
                if (listener->needsReleaseNotify()) {
                    mCore->mConnectedProducerListener = listener;
                }
            }
            break;
        default:
            BQ_LOGE("connect: unknown API %d", api);
            status = BAD_VALUE;
            break;
    }
    mCore->mConnectedPid = BufferQueueThreadState::getCallingPid();
    mCore->mBufferHasBeenQueued = false;
    mCore->mDequeueBufferCannotBlock = false;
    mCore->mQueueBufferCanDrop = false;
    mCore->mLegacyBufferDrop = true;
    if (mCore->mConsumerControlledByApp && producerControlledByApp) {
        mCore->mDequeueBufferCannotBlock = mDequeueTimeout < 0;
        mCore->mQueueBufferCanDrop = mDequeueTimeout <= 0;
    }

    mCore->mAllowAllocation = true;
    VALIDATE_CONSISTENCY();
    return status;
}

status_t BufferQueueProducer::disconnect(int api, DisconnectMode mode) {
    ATRACE_CALL();
    BQ_LOGV("disconnect: api %d", api);

    int status = NO_ERROR;
    sp<IConsumerListener> listener;
    { // Autolock scope
        std::unique_lock<std::mutex> lock(mCore->mMutex);

        if (mode == DisconnectMode::AllLocal) {
            if (BufferQueueThreadState::getCallingPid() != mCore->mConnectedPid) {
                return NO_ERROR;
            }
            api = BufferQueueCore::CURRENTLY_CONNECTED_API;
        }

        mCore->waitWhileAllocatingLocked(lock);

        if (mCore->mIsAbandoned) {
            // It's not really an error to disconnect after the surface has
            // been abandoned; it should just be a no-op.
            return NO_ERROR;
        }

        if (api == BufferQueueCore::CURRENTLY_CONNECTED_API) {
            if (mCore->mConnectedApi == NATIVE_WINDOW_API_MEDIA) {
                ALOGD("About to force-disconnect API_MEDIA, mode=%d", mode);
            }
            api = mCore->mConnectedApi;
            // If we're asked to disconnect the currently connected api but
            // nobody is connected, it's not really an error.
            if (api == BufferQueueCore::NO_CONNECTED_API) {
                return NO_ERROR;
            }
        }

        switch (api) {
            case NATIVE_WINDOW_API_EGL:
            case NATIVE_WINDOW_API_CPU:
            case NATIVE_WINDOW_API_MEDIA:
            case NATIVE_WINDOW_API_CAMERA:
                if (mCore->mConnectedApi == api) {
                    mCore->freeAllBuffersLocked();

                    //删除死亡通知回调
                    if (mCore->mLinkedToDeath != nullptr) {
                        sp<IBinder> token =
                                IInterface::asBinder(mCore->mLinkedToDeath);
                        // This can fail if we're here because of the death
                        // notification, but we just ignore it
                        token->unlinkToDeath(
                                static_cast<IBinder::DeathRecipient*>(this));
                    }
                    mCore->mSharedBufferSlot =
                            BufferQueueCore::INVALID_BUFFER_SLOT;
                    mCore->mLinkedToDeath = nullptr;
                    mCore->mConnectedProducerListener = nullptr;
                    mCore->mConnectedApi = BufferQueueCore::NO_CONNECTED_API;
                    mCore->mConnectedPid = -1;
                    mCore->mSidebandStream.clear();
                    mCore->mDequeueCondition.notify_all();
                    listener = mCore->mConsumerListener;
                } else if (mCore->mConnectedApi == BufferQueueCore::NO_CONNECTED_API) {
                    BQ_LOGE("disconnect: not connected (req=%d)", api);
                    status = NO_INIT;
                } else {
                    BQ_LOGE("disconnect: still connected to another API "
                            "(cur=%d req=%d)", mCore->mConnectedApi, api);
                    status = BAD_VALUE;
                }
                break;
            default:
                BQ_LOGE("disconnect: unknown API %d", api);
                status = BAD_VALUE;
                break;
        }
    } // Autolock scope

    // Call back without lock held
    if (listener != nullptr) {
        listener->onBuffersReleased();
        listener->onDisconnect();
    }

    return status;
}
```

下面的这些函数涉及对 Slot（槽）的处理，理解它们之前先来熟悉一下各种字段的含义。

mQueue —— 同步模式下使用的排队缓冲区的 FIFO。

mMaxAcquiredBufferCount —— 使用者一次可以获取的缓冲区数。它默认为1，并且可以由消费者通过 setMaxAcquiredBufferCount 来更改，但这可能只有在没有生产者连接到 BufferQueue 时才能完成。

mMaxDequeuedBufferCount —— 生产者一次可以出列的缓冲区数。它默认为1，可以由生产者通过 setMaxDequeuedBufferCount 更改。

mAsyncMode —— 表示是否开启异步模式。在异步模式中将分配一个额外的缓冲区，以允许生产者在不阻塞的情况下将缓冲区放入队列。

mDequeueBufferCannotBlock —— 表示是否允许 dequeueBuffer 阻塞。当生产者和消费者都由应用程序控制时，此标志在连接期间设置。

mSlots —— 一个缓冲区槽[数组](https://so.csdn.net/so/search?q=%E6%95%B0%E7%BB%84&spm=1001.2101.3001.7020)，必须在生产者端进行镜像。这允许缓冲区所有权在生产者和消费者之间转移，而不需要通过 Binder 发送 GraphicBuffer。整个数组在构造时被初始化为 NULL，并且在使用槽的索引调用 requestBuffer 时为槽分配缓冲区。

mFreeSlots —— 包含所有空闲的槽，这些槽目前没有一个 attach 了缓冲区，类型为 std::set。

mFreeBuffers —— 包含所有空闲的槽，并且当前有一个 attach 了缓冲区，类型为 std::list。

mActiveBuffers —— 包含所有 attach 了的非空闲缓冲区的槽，类型为 std::set。

mLastQueuedSlot —— 最后一个排队的缓冲区槽。

mUnusedSlots —— 包含当前未使用的所有槽位，类型为 std::list。

adjustAvailableSlotsLocked(…) 函数实现非常简单，如果需要分配额外的 Buffer，那么从 mUnusedSlots 中取出对应槽位插入到 mFreeSlots 中。如果需要减少分配，则首先遍历 mFreeSlots 取出槽位插入到 mUnusedSlots 中，如果 mFreeSlots 为空了，就去 mFreeBuffers 中取出槽位插入到 mUnusedSlots 中。

freeAllBuffersLocked() 则遍历 mFreeSlots 并调用 clearBufferSlotLocked(int slot) 清空每个槽位，然后清空 mFreeBuffers 中的槽位，并归还到 mFreeSlots 中，对 mActiveBuffers 同样清空其每一项的槽位，并归还到 mFreeSlots 中，最后给同步模式下的 mQueue mIsStale 和 mAcquireCalled 分别赋值 true 和 false。

getMaxBufferCountLocked() 返回简单计算最大缓冲区数，具体计算方法可以参考代码实现。

frameworks/native/libs/gui/BufferQueueCore.cpp

```cpp
int BufferQueueCore::getMaxBufferCountLocked(bool asyncMode,
        bool dequeueBufferCannotBlock, int maxBufferCount) const {
    int maxCount = mMaxAcquiredBufferCount + mMaxDequeuedBufferCount +
            ((asyncMode || dequeueBufferCannotBlock) ? 1 : 0);
    maxCount = std::min(maxBufferCount, maxCount);
    return maxCount;
}

int BufferQueueCore::getMaxBufferCountLocked() const {
    int maxBufferCount = mMaxAcquiredBufferCount + mMaxDequeuedBufferCount +
            ((mAsyncMode || mDequeueBufferCannotBlock) ? 1 : 0);

    // limit maxBufferCount by mMaxBufferCount always
    maxBufferCount = std::min(mMaxBufferCount, maxBufferCount);

    return maxBufferCount;
}

void BufferQueueCore::clearBufferSlotLocked(int slot) {
    BQ_LOGV("clearBufferSlotLocked: slot %d", slot);

    mSlots[slot].mGraphicBuffer.clear();
    mSlots[slot].mBufferState.reset();
    mSlots[slot].mRequestBufferCalled = false;
    mSlots[slot].mFrameNumber = 0;
    mSlots[slot].mAcquireCalled = false;
    mSlots[slot].mNeedsReallocation = true;

    // Destroy fence as BufferQueue now takes ownership
    // 销毁 fence，因为 BufferQueue 现在拥有所有权
    if (mSlots[slot].mEglFence != EGL_NO_SYNC_KHR) {
        eglDestroySyncKHR(mSlots[slot].mEglDisplay, mSlots[slot].mEglFence);
        mSlots[slot].mEglFence = EGL_NO_SYNC_KHR;
    }
    mSlots[slot].mFence = Fence::NO_FENCE;
    mSlots[slot].mEglDisplay = EGL_NO_DISPLAY;

    if (mLastQueuedSlot == slot) {
        mLastQueuedSlot = INVALID_BUFFER_SLOT;
    }
}

void BufferQueueCore::freeAllBuffersLocked() {
    for (int s : mFreeSlots) {
        clearBufferSlotLocked(s);
    }

    for (int s : mFreeBuffers) {
        mFreeSlots.insert(s);
        clearBufferSlotLocked(s);
    }
    mFreeBuffers.clear();

    for (int s : mActiveBuffers) {
        mFreeSlots.insert(s);
        clearBufferSlotLocked(s);
    }
    mActiveBuffers.clear();

    for (auto& b : mQueue) {
        b.mIsStale = true;

        // We set this to false to force the BufferQueue to resend the buffer
        // handle upon acquire, since if we're here due to a producer
        // disconnect, the consumer will have been told to purge its cache of
        // slot-to-buffer-handle mappings and will not be able to otherwise
        // obtain a valid buffer handle.
        b.mAcquireCalled = false;
    }

    VALIDATE_CONSISTENCY();
}
......
bool BufferQueueCore::adjustAvailableSlotsLocked(int delta) {
    if (delta >= 0) {
        // If we're going to fail, do so before modifying anything
        if (delta > static_cast<int>(mUnusedSlots.size())) {
            return false;
        }
        while (delta > 0) {
            if (mUnusedSlots.empty()) {
                return false;
            }
            int slot = mUnusedSlots.back();
            mUnusedSlots.pop_back();
            mFreeSlots.insert(slot);
            delta--;
        }
    } else {
        // If we're going to fail, do so before modifying anything
        if (-delta > static_cast<int>(mFreeSlots.size() +
                mFreeBuffers.size())) {
            return false;
        }
        while (delta < 0) {
            if (!mFreeSlots.empty()) {
                auto slot = mFreeSlots.begin();
                clearBufferSlotLocked(*slot);
                mUnusedSlots.push_back(*slot);
                mFreeSlots.erase(slot);
            } else if (!mFreeBuffers.empty()) {
                int slot = mFreeBuffers.back();
                clearBufferSlotLocked(slot);
                mUnusedSlots.push_back(slot);
                mFreeBuffers.pop_back();
            } else {
                return false;
            }
            delta++;
        }
    }
    return true;
}
```

回到主线上，调用 MediaCodec initiateConfigureComponent(…) 初始化组件。从前面的文章不难得出 mCodec 实际指向 ACodec。

设置了消息 what id 为 kWhatConfigureComponent，调用 post() 发出消息。

frameworks/av/media/libstagefright/ACodec.cpp

```cpp
void ACodec::initiateConfigureComponent(const sp<AMessage> &msg) {
    msg->setWhat(kWhatConfigureComponent);
    msg->setTarget(this);
    msg->post();
}
```

经过 native MediaCodec 的 init 流程后，ACodec 状态已经被更改为 LoadedState。所以消息处理就在 ACodec::LoadedState::onMessageReceived(…) 中。kWhatConfigureComponent 消息处理只是调用了 onConfigureComponent(…) 进一步处理。

frameworks/av/media/libstagefright/ACodec.cpp

```cpp
bool ACodec::LoadedState::onMessageReceived(const sp<AMessage> &msg) {
    bool handled = false;

    switch (msg->what()) {
        case ACodec::kWhatConfigureComponent:
        {
            onConfigureComponent(msg);
            handled = true;
            break;
        }
        ......
    }
}
```

onConfigureComponent(…) 中主要调用 ACodec configureCodec(…) 进行配置 codec。

frameworks/av/media/libstagefright/ACodec.cpp

```cpp
bool ACodec::LoadedState::onConfigureComponent(
        const sp<AMessage> &msg) {
    ALOGV("onConfigureComponent");

    CHECK(mCodec->mOMXNode != NULL);

    status_t err = OK;
    AString mime;
    if (!msg->findString("mime", &mime)) {
        err = BAD_VALUE;
    } else {
        err = mCodec->configureCodec(mime.c_str(), msg);
    }
    if (err != OK) {
        ALOGE("[%s] configureCodec returning error %d",
              mCodec->mComponentName.c_str(), err);

        mCodec->signalError(OMX_ErrorUndefined, makeNoSideEffectStatus(err));
        return false;
    }

    mCodec->mCallback->onComponentConfigured(mCodec->mInputFormat, mCodec->mOutputFormat);

    return true;
}
```

1. 调用 setComponentRole(…) 设置组件角色；
2. 查找是否配置了 surface，此流程已配置，所以 haveNativeWindow 为 true，调用 native_window_set_sideband_stream(…) 显式重设非隧道视频窗口的 sideband 句柄，以防窗口以前被用于隧道视频回放。调用 setPortMode(…) 设置 Port 模式为 IOMX::kPortModeDynamicANWBuffer，设置失败；
3. 再次调用 setPortMode(…) 设置 Port 模式，这次设置为 IOMX::kPortModePresetANWBuffer；
4. 调用 setupVideoDecoder(…) 设置视频解码器；
5. 从 App 配置下来的格式为 MediaCodecInfo.CodecCapabilities.COLOR_FormatYUV420Flexible，所以此处 requestedColorFormat 就等于 OMX_COLOR_FormatYUV420Flexible，但调用 getPortFormat(…) 获取的输出格式却不是，因此会打印 Log：ACodec: [OMX.rk.video_decoder.avc] Requested output format 0x7f420888 and got 0x15.
6. 调用 setVendorParameters(…) 设置供应商参数；
7. 最后调用 getPortFormat(…) 获取输入、输出格式。

frameworks/av/media/libstagefright/ACodec.cpp

```cpp
status_t ACodec::configureCodec(
        const char *mime, const sp<AMessage> &msg) {
    int32_t encoder;
    if (!msg->findInt32("encoder", &encoder)) {
        encoder = false;
    }

    sp<AMessage> inputFormat = new AMessage;
    sp<AMessage> outputFormat = new AMessage;
    mConfigFormat = msg;

    mIsEncoder = encoder;
    mIsVideo = !strncasecmp(mime, "video/", 6);
    mIsImage = !strncasecmp(mime, "image/", 6);

    mPortMode[kPortIndexInput] = IOMX::kPortModePresetByteBuffer;
    mPortMode[kPortIndexOutput] = IOMX::kPortModePresetByteBuffer;

    status_t err = setComponentRole(encoder /* isEncoder */, mime);

    if (err != OK) {
        return err;
    }
    ......
    // NOTE: we only use native window for video decoders
    sp<RefBase> obj;
    bool haveNativeWindow = msg->findObject("native-window", &obj)
            && obj != NULL && mIsVideo && !encoder;
    mUsingNativeWindow = haveNativeWindow;
    if (mIsVideo && !encoder) {
        inputFormat->setInt32("adaptive-playback", false);
        ......
    }
    ......
    if (haveNativeWindow) {
        sp<ANativeWindow> nativeWindow =
            static_cast<ANativeWindow *>(static_cast<Surface *>(obj.get()));
        ......
        int32_t tunneled;
        if (msg->findInt32("feature-tunneled-playback", &tunneled) &&
            tunneled != 0) {
            ......
        } else {
            // 配置 CPU 控制视频回放
            ALOGV("Configuring CPU controlled video playback.");
            mTunneled = false;

            // Explicity reset the sideband handle of the window for
            // non-tunneled video in case the window was previously used
            // for a tunneled video playback.
            // 显式重设非隧道视频窗口的 sideband 句柄，以防窗口以前被用于隧道视频回放。
            err = native_window_set_sideband_stream(nativeWindow.get(), NULL);
            if (err != OK) {
                ALOGE("set_sideband_stream(NULL) failed! (err %d).", err);
                return err;
            }

            err = setPortMode(kPortIndexOutput, IOMX::kPortModeDynamicANWBuffer);
            if (err != OK) {
                // we will not do adaptive playback on software accessed
                // surfaces as they never had to respond to changes in the
                // crop window, and we don't trust that they will be able to.
                int usageBits = 0;
                bool canDoAdaptivePlayback;

                if (nativeWindow->query(
                        nativeWindow.get(),
                        NATIVE_WINDOW_CONSUMER_USAGE_BITS,
                        &usageBits) != OK) {
                    ......
                } else {
                    canDoAdaptivePlayback =
                        (usageBits &
                                (GRALLOC_USAGE_SW_READ_MASK |
                                 GRALLOC_USAGE_SW_WRITE_MASK)) == 0;
                }
                ......
                // allow failure
                err = OK;
            } else {
                ......
            }
            ......
        }

        int32_t rotationDegrees;
        if (msg->findInt32("rotation-degrees", &rotationDegrees)) {
            mRotationDegrees = rotationDegrees;
        } else {
            mRotationDegrees = 0;
        }
    }
    ......
    if (mIsVideo || mIsImage) {
        // determine need for software renderer
        bool usingSwRenderer = false;
        if (haveNativeWindow && mComponentName.startsWith("OMX.google.")) {
            ......
        } else if (haveNativeWindow && !storingMetadataInDecodedBuffers()) {
            err = setPortMode(kPortIndexOutput, IOMX::kPortModePresetANWBuffer);
            if (err != OK) {
                return err;
            }
        }

        if (encoder) {
            err = setupVideoEncoder(mime, msg, outputFormat, inputFormat);
        } else {
            err = setupVideoDecoder(mime, msg, haveNativeWindow, usingSwRenderer, outputFormat);
        }

        if (err != OK) {
            return err;
        }

        if (haveNativeWindow) {
            mNativeWindow = static_cast<Surface *>(obj.get());
            // fallback for devices that do not handle flex-YUV for native buffers
            int32_t requestedColorFormat = OMX_COLOR_FormatUnused;
            if (msg->findInt32("color-format", &requestedColorFormat) &&
                    requestedColorFormat == OMX_COLOR_FormatYUV420Flexible) {
                status_t err = getPortFormat(kPortIndexOutput, outputFormat);
                if (err != OK) {
                    return err;
                }
                int32_t colorFormat = OMX_COLOR_FormatUnused;
                OMX_U32 flexibleEquivalent = OMX_COLOR_FormatUnused;
                if (!outputFormat->findInt32("color-format", &colorFormat)) {
                    ALOGE("ouptut port did not have a color format (wrong domain?)");
                    return BAD_VALUE;
                }
                ALOGD("[%s] Requested output format %#x and got %#x.",
                        mComponentName.c_str(), requestedColorFormat, colorFormat);
                if (!IsFlexibleColorFormat(
                        mOMXNode, colorFormat, haveNativeWindow, &flexibleEquivalent)
                        || flexibleEquivalent != (OMX_U32)requestedColorFormat) {
                    // device did not handle flex-YUV request for native window, fall back
                    // to SW renderer
                    ALOGI("[%s] Falling back to software renderer", mComponentName.c_str());
                    ......
                }
            }
        }
        ......
    }
    ......

    if (err != OK) {
        return err;
    }
    ......
    if (err == OK) {
        err = setVendorParameters(msg);
        if (err != OK) {
            return err;
        }
    }
    ......
    // NOTE: both mBaseOutputFormat and mOutputFormat are outputFormat to signal first frame.
    mBaseOutputFormat = outputFormat;
    mLastOutputFormat.clear();

    err = getPortFormat(kPortIndexInput, inputFormat);
    if (err == OK) {
        err = getPortFormat(kPortIndexOutput, outputFormat);
        if (err == OK) {
            mInputFormat = inputFormat;
            mOutputFormat = outputFormat;
        }
    }
    ......
    return err;
}
```

调用 setComponentRole(…) 设置组件角色，此方法仅仅内部先调用了 GetComponentRole(…) 获取组件角色，然后再次调用 SetComponentRole(…) 将组件角色设置下去。mOMXNode 是在 ACodec::UninitializedState::onAllocateComponent(…) 赋值的，实际指向 LWOmxNode 对象。

frameworks/av/media/libstagefright/ACodec.cpp

```cpp
status_t ACodec::setComponentRole(
        bool isEncoder, const char *mime) {
    const char *role = GetComponentRole(isEncoder, mime);
    if (role == NULL) {
        return BAD_VALUE;
    }
    status_t err = SetComponentRole(mOMXNode, role);
    if (err != OK) {
        ALOGW("[%s] Failed to set standard component role '%s'.",
             mComponentName.c_str(), role);
    }
    return err;
}
```

GetComponentRole(…) 方法入参 isEncoder 为 false，mime 为 video/avc，所以会返回 **video_decoder.avc**。

SetComponentRole(…) 中首先将 OMX_PARAM_COMPONENTROLETYPE 结构中 cRole 字段填充 video_decoder.avc 字符串，然后调用 LWOmxNode setParameter(…) 设置组件角色。

frameworks/av/media/libstagefright/omx/OMXUtils.cpp

```cpp
const char *GetComponentRole(bool isEncoder, const char *mime) {
    struct MimeToRole {
        const char *mime;
        const char *decoderRole;
        const char *encoderRole;
    };

    static const MimeToRole kMimeToRole[] = {
        { MEDIA_MIMETYPE_AUDIO_MPEG,
            "audio_decoder.mp3", "audio_encoder.mp3" },
        { MEDIA_MIMETYPE_AUDIO_MPEG_LAYER_I,
            "audio_decoder.mp1", "audio_encoder.mp1" },
        { MEDIA_MIMETYPE_AUDIO_MPEG_LAYER_II,
            "audio_decoder.mp2", "audio_encoder.mp2" },
        { MEDIA_MIMETYPE_AUDIO_AMR_NB,
            "audio_decoder.amrnb", "audio_encoder.amrnb" },
        { MEDIA_MIMETYPE_AUDIO_AMR_WB,
            "audio_decoder.amrwb", "audio_encoder.amrwb" },
        { MEDIA_MIMETYPE_AUDIO_AAC,
            "audio_decoder.aac", "audio_encoder.aac" },
        { MEDIA_MIMETYPE_AUDIO_VORBIS,
            "audio_decoder.vorbis", "audio_encoder.vorbis" },
        { MEDIA_MIMETYPE_AUDIO_OPUS,
            "audio_decoder.opus", "audio_encoder.opus" },
        { MEDIA_MIMETYPE_AUDIO_G711_MLAW,
            "audio_decoder.g711mlaw", "audio_encoder.g711mlaw" },
        { MEDIA_MIMETYPE_AUDIO_G711_ALAW,
            "audio_decoder.g711alaw", "audio_encoder.g711alaw" },
        { MEDIA_MIMETYPE_VIDEO_AVC,
            "video_decoder.avc", "video_encoder.avc" },
        { MEDIA_MIMETYPE_VIDEO_HEVC,
            "video_decoder.hevc", "video_encoder.hevc" },
        { MEDIA_MIMETYPE_VIDEO_MPEG4,
            "video_decoder.mpeg4", "video_encoder.mpeg4" },
        { MEDIA_MIMETYPE_VIDEO_H263,
            "video_decoder.h263", "video_encoder.h263" },
        { MEDIA_MIMETYPE_VIDEO_VP8,
            "video_decoder.vp8", "video_encoder.vp8" },
        { MEDIA_MIMETYPE_VIDEO_VP9,
            "video_decoder.vp9", "video_encoder.vp9" },
        { MEDIA_MIMETYPE_VIDEO_AV1,
            "video_decoder.av1", "video_encoder.av1" },
        { MEDIA_MIMETYPE_AUDIO_RAW,
            "audio_decoder.raw", "audio_encoder.raw" },
        { MEDIA_MIMETYPE_VIDEO_DOLBY_VISION,
            "video_decoder.dolby-vision", "video_encoder.dolby-vision" },
        { MEDIA_MIMETYPE_AUDIO_FLAC,
            "audio_decoder.flac", "audio_encoder.flac" },
        { MEDIA_MIMETYPE_AUDIO_MSGSM,
            "audio_decoder.gsm", "audio_encoder.gsm" },
        { MEDIA_MIMETYPE_VIDEO_MPEG2,
            "video_decoder.mpeg2", "video_encoder.mpeg2" },
        { MEDIA_MIMETYPE_AUDIO_AC3,
            "audio_decoder.ac3", "audio_encoder.ac3" },
        { MEDIA_MIMETYPE_AUDIO_EAC3,
            "audio_decoder.eac3", "audio_encoder.eac3" },
        { MEDIA_MIMETYPE_AUDIO_EAC3_JOC,
            "audio_decoder.eac3_joc", "audio_encoder.eac3_joc" },
        { MEDIA_MIMETYPE_AUDIO_AC4,
            "audio_decoder.ac4", "audio_encoder.ac4" },
        { MEDIA_MIMETYPE_IMAGE_ANDROID_HEIC,
            "image_decoder.heic", "image_encoder.heic" },
    };

    static const size_t kNumMimeToRole =
        sizeof(kMimeToRole) / sizeof(kMimeToRole[0]);

    size_t i;
    for (i = 0; i < kNumMimeToRole; ++i) {
        if (!strcasecmp(mime, kMimeToRole[i].mime)) {
            break;
        }
    }

    if (i == kNumMimeToRole) {
        return NULL;
    }

    return isEncoder ? kMimeToRole[i].encoderRole
                  : kMimeToRole[i].decoderRole;
}

status_t SetComponentRole(const sp<IOMXNode> &omxNode, const char *role) {
    OMX_PARAM_COMPONENTROLETYPE roleParams;
    InitOMXParams(&roleParams);

    strncpy((char *)roleParams.cRole,
            role, OMX_MAX_STRINGNAME_SIZE - 1);

    roleParams.cRole[OMX_MAX_STRINGNAME_SIZE - 1] = '\0';

    return omxNode->setParameter(
            OMX_IndexParamStandardComponentRole,
            &roleParams, sizeof(roleParams));
}
```

LWOmxNode::setParameter(…) 调用 TWOmxNode（mBase 指向 TWOmxNode 结构体）setParameter(…) 进行参数设置。

frameworks/av/media/libstagefright/omx/1.0/WOmxNode.cpp

```cpp
status_t LWOmxNode::setParameter(
        OMX_INDEXTYPE index, const void *params, size_t size) {
    hidl_vec<uint8_t> tParams = inHidlBytes(params, size);
    return toStatusT(mBase->setParameter(
            toRawIndexType(index), tParams));
}
```

TWOmxNode::setParameter(…) 调用 OMXNodeInstance setParameter(…) 进行参数设置。

frameworks/av/media/libstagefright/omx/1.0/WOmxNode.cpp

```cpp
Return<Status> TWOmxNode::setParameter(
        uint32_t index, hidl_vec<uint8_t> const& inParams) {
    hidl_vec<uint8_t> params(inParams);
    return toStatus(mBase->setParameter(
            toEnumIndexType(index),
            static_cast<void const*>(params.data()),
            params.size()));
}
```

现在转到了实际干活的方法，mHandle 是谁？实际是指向 OMX_COMPONENTTYPE 结构体的指针。在瑞芯微实现中结构体 OMX_COMPONENTTYPE 定义在 hardware/rockchip/omx_il/include/khronos/OMX_Component.h 中。它在 Rockchip_OMX_ComponentConstructor(…) 和 Rockchip_OMX_BaseComponent_Constructor(…) 中进行了初始化。

OMXNodeInstance::setParameter(…) 实际调用了 OMX_SetParameter(…) 进行参数设置。

frameworks/av/media/libstagefright/omx/OMXNodeInstance.cpp

```cpp
status_t OMXNodeInstance::setParameter(
        OMX_INDEXTYPE index, const void *params, size_t size) {
    Mutex::Autolock autoLock(mLock);
    if (mHandle == NULL) {
        return DEAD_OBJECT;
    }

    OMX_INDEXEXTTYPE extIndex = (OMX_INDEXEXTTYPE)index;
    CLOG_CONFIG(setParameter, "%s(%#x), %zu@%p)", asString(extIndex), index, size, params);

    if (extIndex == OMX_IndexParamMaxFrameDurationForBitrateControl) {
        return setMaxPtsGapUs(params, size);
    }

    if (isProhibitedIndex_l(index)) {
        android_errorWriteLog(0x534e4554, "29422020");
        return BAD_INDEX;
    }

    OMX_ERRORTYPE err = OMX_SetParameter(
            mHandle, index, const_cast<void *>(params));
    CLOG_IF_ERROR(setParameter, err, "%s(%#x)", asString(extIndex), index);
    return StatusFromOMXError(err);
}
```

mHandle 是个 OMX_HANDLETYPE 类型，OMX_HANDLETYPE 又是什么？

frameworks/av/media/libstagefright/omx/include/media/stagefright/omx/OMXNodeInstance.h

```cpp
struct OMXNodeInstance : public BnOMXNode {
    ......
private:
    ......
    OMX_HANDLETYPE mHandle;
    ......
}
```

OMX_HANDLETYPE 是 OMX_PTR 的别名。结合注释理解一下：定义 OMX 句柄的公共接口，OMX 核心不会在内部使用这个值，但是应用程序应该只使用这个值。

编译选项 OMX_ANDROID_COMPILE_AS_32BIT_ON_64BIT_PLATFORMS 从定义不难看出 64 位平台上使用 32 位才会打开。所以实际上 OMX_PTR 就是个 void* 指针。

```cpp
/*
 * Temporary Android 64 bit modification
 *
 * #define OMX_ANDROID_COMPILE_AS_32BIT_ON_64BIT_PLATFORMS
 * overrides all OMX pointer types to be uint32_t.
 *
 * After this change, OMX codecs will work in 32 bit only, so 64 bit processes
 * must communicate to a remote 32 bit process for OMX to work.
 */

#ifdef OMX_ANDROID_COMPILE_AS_32BIT_ON_64BIT_PLATFORMS

typedef uint32_t OMX_PTR;
typedef OMX_PTR OMX_STRING;
typedef OMX_PTR OMX_BYTE;

#else /* OMX_ANDROID_COMPILE_AS_32BIT_ON_64BIT_PLATFORMS */

/** The OMX_PTR type is intended to be used to pass pointers between the OMX
    applications and the OMX Core and components.  This is a 32 bit pointer and
    is aligned on a 32 bit boundary.
 */
typedef void* OMX_PTR;
......
/** Define the public interface for the OMX Handle.  The core will not use
    this value internally, but the application should only use this value.
 */
typedef OMX_PTR OMX_HANDLETYPE;
```

OMX_SetParameter(…) 是一个宏。在此场景下展开以后可以看到实际会调用 OMX_COMPONENTTYPE 结构体内部的 SetParameter(…) 函数指针。

frameworks/native/headers/media_plugin/media/openmax/OMX_Core.h

根据前面组件分配一节的分析，此处实际会调用瑞芯微的具体实现 Rkvpu_OMX_SetParameter(…)。此方法内部将 input Port portDefinition.format.video.eCompressionFormat 改为了 OMX_VIDEO_CodingAVC 枚举。

hardware/rockchip/omx_il/component/video/dec/Rkvpu_OMX_VdecControl.c

```c
OMX_ERRORTYPE Rkvpu_OMX_SetParameter(
    OMX_IN OMX_HANDLETYPE hComponent,
    OMX_IN OMX_INDEXTYPE  nIndex,
    OMX_IN OMX_PTR        ComponentParameterStructure)
{
    OMX_ERRORTYPE          ret = OMX_ErrorNone;
    OMX_COMPONENTTYPE     *pOMXComponent = NULL;
    ROCKCHIP_OMX_BASECOMPONENT *pRockchipComponent = NULL;

    FunctionIn();

    if (hComponent == NULL) {
        ret = OMX_ErrorBadParameter;
        goto EXIT;
    }
    pOMXComponent = (OMX_COMPONENTTYPE *)hComponent;
    ......
    pRockchipComponent = (ROCKCHIP_OMX_BASECOMPONENT *)pOMXComponent->pComponentPrivate;
    ......
    switch ((OMX_U32)nIndex) {
    ......
    case OMX_IndexParamStandardComponentRole: {
        OMX_PARAM_COMPONENTROLETYPE *pComponentRole = (OMX_PARAM_COMPONENTROLETYPE*)ComponentParameterStructure;

        ret = Rockchip_OMX_Check_SizeVersion(pComponentRole, sizeof(OMX_PARAM_COMPONENTROLETYPE));
        if (ret != OMX_ErrorNone) {
            goto EXIT;
        }

        if ((pRockchipComponent->currentState != OMX_StateLoaded) && (pRockchipComponent->currentState != OMX_StateWaitForResources)) {
            ret = OMX_ErrorIncorrectStateOperation;
            goto EXIT;
        }

        if (!Rockchip_OSAL_Strcmp((char*)pComponentRole->cRole, RK_OMX_COMPONENT_H264_DEC_ROLE)) {
            pRockchipComponent->pRockchipPort[INPUT_PORT_INDEX].portDefinition.format.video.eCompressionFormat = OMX_VIDEO_CodingAVC;
        }
        ......
    }
    break;
    ......
    default: {
        ret = Rockchip_OMX_SetParameter(hComponent, nIndex, ComponentParameterStructure);
    }
    break;
    }

EXIT:
    FunctionOut();

    return ret;
}
```

调用 setPortMode(…) 设置 Port 模式为 IOMX::kPortModeDynamicANWBuffer，第二次设置为 IOMX::kPortModePresetANWBuffer。第一次设置出错，所以打印 Log：ACodec: [OMX.rk.video_decoder.avc] setPortMode on output to DynamicANWBuffer failed w/ err -1010。

frameworks/av/media/libstagefright/ACodec.cpp

```cpp
status_t ACodec::setPortMode(int32_t portIndex, IOMX::PortMode mode) {
    status_t err = mOMXNode->setPortMode(portIndex, mode);
    if (err != OK) {
        ALOGE("[%s] setPortMode on %s to %s failed w/ err %d",
                mComponentName.c_str(),
                portIndex == kPortIndexInput ? "input" : "output",
                asString(mode),
                err);
        return err;
    }

    mPortMode[portIndex] = mode;
    return OK;
}
```

根据上面的分析不难知道 setPortMode(…) 最终会调到 OMXNodeInstance.cpp 中实现。

1. 第一次设置，PortMode 为 IOMX::kPortModeDynamicANWBuffer，接着调用 enableNativeBuffers_l(…) 返回 err 直接退出；
2. 第二次设置，PortMode 为 IOMX::kPortModePresetANWBuffer，接着调用了两次 enableNativeBuffers_l(…) 禁用安全缓冲区并启用图形缓冲区，最后调用 storeMetaDataInBuffers_l(…) 禁用元数据模式，使用 legacy 模式。

frameworks/av/media/libstagefright/omx/OMXNodeInstance.cpp

```cpp
status_t OMXNodeInstance::setPortMode(OMX_U32 portIndex, IOMX::PortMode mode) {
    ......
    status_t err = OK;
    switch (mode) {
    case IOMX::kPortModeDynamicANWBuffer:
    {
        if (portIndex == kPortIndexOutput) {
            ......
            err = enableNativeBuffers_l(
                    portIndex, OMX_TRUE /*graphic*/, OMX_TRUE);
            if (err != OK) {
                break;
            }
        }
        (void)enableNativeBuffers_l(portIndex, OMX_FALSE /*graphic*/, OMX_FALSE);
        err = storeMetaDataInBuffers_l(portIndex, OMX_TRUE, NULL);
        break;
    }
    ......
    case IOMX::kPortModePresetANWBuffer:
    {
        if (portIndex != kPortIndexOutput) {
            CLOG_ERROR(setPortMode, BAD_VALUE,
                    "%s(%d) mode is only supported on output port", asString(mode), mode);
            err = BAD_VALUE;
            break;
        }
        ......
        // Disable secure buffer and enable graphic buffer
        (void)enableNativeBuffers_l(portIndex, OMX_FALSE /*graphic*/, OMX_FALSE);
        err = enableNativeBuffers_l(portIndex, OMX_TRUE /*graphic*/, OMX_TRUE);
        if (err != OK) {
            break;
        }

        // Not running experiment, or metadata is not supported.
        // Disable metadata mode and use legacy mode.
        (void)storeMetaDataInBuffers_l(portIndex, OMX_FALSE, NULL);
        break;
    }
    ......
    default:
        CLOG_ERROR(setPortMode, BAD_VALUE, "invalid port mode %d", mode);
        err = BAD_VALUE;
        break;
    }

    if (err == OK) {
        mPortMode[portIndex] = mode;
    }
    return err;
}
```

1. 调用 OMX_GetExtensionIndex(…) 获取 index；
2. 构造 EnableAndroidNativeBuffersParams 结构（这个结构用于支持 Android 本地缓冲区用于图形缓冲区或安全缓冲区），并初始化其 nPortIndex 和 enable 字段；
3. 调用 OMX_SetParameter(…) 真正去设置。

frameworks/av/media/libstagefright/omx/OMXNodeInstance.cpp

```cpp
status_t OMXNodeInstance::enableNativeBuffers_l(
        OMX_U32 portIndex, OMX_BOOL graphic, OMX_BOOL enable) {
    ......
    OMX_STRING name = const_cast<OMX_STRING>(
            graphic ? "OMX.google.android.index.enableAndroidNativeBuffers"
                    : "OMX.google.android.index.allocateNativeHandle");

    OMX_INDEXTYPE index;
    OMX_ERRORTYPE err = OMX_GetExtensionIndex(mHandle, name, &index);

    if (err == OMX_ErrorNone) {
        EnableAndroidNativeBuffersParams params;
        InitOMXParams(&params);
        params.nPortIndex = portIndex;
        params.enable = enable;

        err = OMX_SetParameter(mHandle, index, &params);
        CLOG_IF_ERROR(setParameter, err, "%s(%#x): %s:%u en=%d", name, index,
                      portString(portIndex), portIndex, enable);
        if (!graphic) {
            if (err == OMX_ErrorNone) {
                mSecureBufferType[portIndex] =
                    enable ? kSecureBufferTypeNativeHandle : kSecureBufferTypeOpaque;
            } else if (mSecureBufferType[portIndex] == kSecureBufferTypeUnknown) {
                mSecureBufferType[portIndex] = kSecureBufferTypeOpaque;
            }
        } else {
            if (err == OMX_ErrorNone) {
                mGraphicBufferEnabled[portIndex] = enable;
            } else if (enable) {
                mGraphicBufferEnabled[portIndex] = false;
            }
        }
    } else {
        ......
    }

    return StatusFromOMXError(err);
}
```

OMX_GetExtensionIndex(…) 的作用是将供应商特定的配置或参数字符串转换为 OMX 结构索引。

frameworks/native/headers/media_plugin/media/openmax/OMX_Core.h

瑞芯微的对应实现函数是 Rkvpu_OMX_GetExtensionIndex。在《【Android 10 源码】深入理解 Omx 初始化》一节中得知 rk3399 android 10 上 AVS80、HAVE_L1_SVP_MODE 编译选项其实都没开。当传递的 cParameterName 为 OMX.google.android.index.allocateNativeHandle 就会找不到匹配项，接着会调用到 Rockchip_OMX_GetExtensionIndex(…)。当传递的 cParameterName 为 OMX.google.android.index.enableAndroidNativeBuffers 找到了匹配项，并给 *pIndexType 赋值为 OMX_IndexParamEnableAndroidBuffers（0x7F000011）。

hardware/rockchip/omx_il/component/video/dec/Rkvpu_OMX_VdecControl.c

```c
OMX_ERRORTYPE Rkvpu_OMX_GetExtensionIndex(
    OMX_IN OMX_HANDLETYPE  hComponent,
    OMX_IN OMX_STRING      cParameterName,
    OMX_OUT OMX_INDEXTYPE *pIndexType)
{
    OMX_ERRORTYPE           ret = OMX_ErrorNone;
    OMX_COMPONENTTYPE     *pOMXComponent = NULL;
    ROCKCHIP_OMX_BASECOMPONENT *pRockchipComponent = NULL;
    ......
    pOMXComponent = (OMX_COMPONENTTYPE *)hComponent;
    ......
    pRockchipComponent = (ROCKCHIP_OMX_BASECOMPONENT *)pOMXComponent->pComponentPrivate;
    ......
#ifdef USE_ANB
    if (Rockchip_OSAL_Strcmp(cParameterName, ROCKCHIP_INDEX_PARAM_ENABLE_ANB) == 0) {
        *pIndexType = (OMX_INDEXTYPE) OMX_IndexParamEnableAndroidBuffers;
        goto EXIT;
    }
    ......
#endif
    ......
#ifdef AVS80
#ifdef HAVE_L1_SVP_MODE
    if (Rockchip_OSAL_Strcmp(cParameterName, ROCKCHIP_INDEX_PARAM_ALLOCATENATIVEHANDLE) == 0) {
        *pIndexType = (OMX_INDEXTYPE)OMX_IndexParamAllocateNativeHandle;
        goto EXIT;
    }
#endif
#endif
    ......
    ret = Rockchip_OMX_GetExtensionIndex(hComponent, cParameterName, pIndexType);

EXIT:
    FunctionOut();

    return ret;
}
```

Rockchip_OMX_GetExtensionIndex(…) 检查了一系列参数如果都没问题，就返回 OMX_ErrorBadParameter，这个值等于 0x80001005。

hardware/rockchip/omx_il/component/common/Rockchip_OMX_Basecomponent.c

```c
OMX_ERRORTYPE Rockchip_OMX_GetExtensionIndex(
    OMX_IN OMX_HANDLETYPE  hComponent,
    OMX_IN OMX_STRING      cParameterName,
    OMX_OUT OMX_INDEXTYPE *pIndexType)
{
    OMX_ERRORTYPE             ret = OMX_ErrorNone;
    OMX_COMPONENTTYPE        *pOMXComponent = NULL;
    ROCKCHIP_OMX_BASECOMPONENT *pRockchipComponent = NULL;
    FunctionIn();
    ......
    pOMXComponent = (OMX_COMPONENTTYPE *)hComponent;
    .....
    pRockchipComponent = (ROCKCHIP_OMX_BASECOMPONENT *)pOMXComponent->pComponentPrivate;
    ......
    ret = OMX_ErrorBadParameter;

EXIT:
    FunctionOut();

    return ret;
}
```

现在来分析调用 OMX_SetParameter(…) 真正去设置 EnableAndroidNativeBuffersParams 结构用于本地缓冲区用作图形缓冲区或安全缓冲区，最终使用图形缓冲区。这里会调用 Rockchip_OSAL_SetANBParameter(…) 进一步设置。

hardware/rockchip/omx_il/component/video/dec/Rkvpu_OMX_VdecControl.c

```c
OMX_ERRORTYPE Rkvpu_OMX_SetParameter(
    OMX_IN OMX_HANDLETYPE hComponent,
    OMX_IN OMX_INDEXTYPE  nIndex,
    OMX_IN OMX_PTR        ComponentParameterStructure)
{
    ......
    switch ((OMX_U32)nIndex) {
    ......
#ifdef USE_ANB
    case OMX_IndexParamEnableAndroidBuffers:
    case OMX_IndexParamUseAndroidNativeBuffer:
    case OMX_IndexParamStoreMetaDataBuffer:
    case OMX_IndexParamprepareForAdaptivePlayback:
    case OMX_IndexParamAllocateNativeHandle: {
        omx_trace("Rockchip_OSAL_SetANBParameter!!");
        ret = Rockchip_OSAL_SetANBParameter(hComponent, nIndex, ComponentParameterStructure);
    }
    break;
#endif
    ......
    }
    ......
}
```

具体业务逻辑不再展开，看注释和 ANB 和 DPB 缓冲区共享有关。

hardware/rockchip/omx_il/osal/Rockchip_OSAL_Android.cpp

```cpp
OMX_ERRORTYPE Rockchip_OSAL_SetANBParameter(
    OMX_IN OMX_HANDLETYPE hComponent,
    OMX_IN OMX_INDEXTYPE  nIndex,
    OMX_IN OMX_PTR        ComponentParameterStructure)
{
    OMX_ERRORTYPE          ret = OMX_ErrorNone;
    OMX_COMPONENTTYPE     *pOMXComponent = NULL;
    ROCKCHIP_OMX_BASECOMPONENT *pRockchipComponent = NULL;

    FunctionIn();
    ......
    pOMXComponent = (OMX_COMPONENTTYPE *)hComponent;
    ......
    pRockchipComponent = (ROCKCHIP_OMX_BASECOMPONENT *)pOMXComponent->pComponentPrivate;
    ......
    switch (nIndex) {
    case OMX_IndexParamEnableAndroidBuffers: {
        RKVPU_OMX_VIDEODEC_COMPONENT *pVideoDec = (RKVPU_OMX_VIDEODEC_COMPONENT *)pRockchipComponent->hComponentHandle;
        EnableAndroidNativeBuffersParams *pANBParams = (EnableAndroidNativeBuffersParams *) ComponentParameterStructure;
        OMX_U32 portIndex = pANBParams->nPortIndex;
        ROCKCHIP_OMX_BASEPORT *pRockchipPort = NULL;
        omx_trace("%s: OMX_IndexParamEnableAndroidNativeBuffers", __func__);
        ......
        pRockchipPort = &pRockchipComponent->pRockchipPort[portIndex];
        ......
        /* ANB and DPB Buffer Sharing */
        if (pVideoDec->bStoreMetaData != OMX_TRUE) {
            pVideoDec->bIsANBEnabled = pANBParams->enable;
            if (portIndex == OUTPUT_PORT_INDEX)
                pRockchipPort->portDefinition.format.video.eColorFormat = (OMX_COLOR_FORMATTYPE)HAL_PIXEL_FORMAT_YCrCb_NV12;
            omx_trace("OMX_IndexParamEnableAndroidBuffers set buffcount %d", pRockchipPort->portDefinition.nBufferCountActual);
            /*
                this is temp way to avoid android.media.cts.ImageReaderDecoderTest rk decoder test

            */
            if (pRockchipPort->bufferProcessType == BUFFER_COPY) {
                if ((pVideoDec->codecId != OMX_VIDEO_CodingH263) && (pRockchipPort->portDefinition.format.video.nFrameWidth >= 176)) {
                    pRockchipPort->bufferProcessType = BUFFER_ANBSHARE;
                }
            }
        }

        omx_trace("portIndex = %d,pRockchipPort->bufferProcessType =0x%x", portIndex, pRockchipPort->bufferProcessType);
        if ((portIndex == OUTPUT_PORT_INDEX) &&
            ((pRockchipPort->bufferProcessType & BUFFER_ANBSHARE) == BUFFER_ANBSHARE)) {
            if (pVideoDec->bIsANBEnabled == OMX_TRUE) {
                pRockchipPort->bufferProcessType = BUFFER_SHARE;
                if (portIndex == OUTPUT_PORT_INDEX)
                    pRockchipPort->portDefinition.format.video.eColorFormat = (OMX_COLOR_FORMATTYPE)HAL_PIXEL_FORMAT_YCrCb_NV12;
                omx_trace("OMX_IndexParamEnableAndroidBuffers & bufferProcessType change to BUFFER_SHARE");
            }
            Rockchip_OSAL_Openvpumempool(pRockchipComponent, portIndex);
        }

        if ((portIndex == OUTPUT_PORT_INDEX) && !pVideoDec->bIsANBEnabled) {
            pRockchipPort->bufferProcessType = BUFFER_COPY;
            Rockchip_OSAL_Openvpumempool(pRockchipComponent, portIndex);
        }
    }
    break;
    ......
    }

EXIT:
    FunctionOut();

    return ret;
}
```

回到主线继续分析调用 setupVideoDecoder(…) 设置视频解码器。

1. 调用 GetVideoCodingTypeFromMime(…) 获取 OMX_VIDEO_CODINGTYPE 结构；
2. 第一次调用 setVideoPortFormatType(…) 将端口输入颜色空间配置为 OMX_COLOR_FormatUnused；
3. 根据实际颜色空间再次调用 setVideoPortFormatType(…) 进行端口输出配置；
4. 调用 setVideoFormatOnPort(…) 在端口上配置宽、高、颜色空间和帧率等参数；
5. 调用 setColorAspectsForVideoDecoder(…) 进行配置；
6. 调用 setHDRStaticInfoForVideoCodec(…) 进行配置。

frameworks/av/media/libstagefright/ACodec.cpp

```cpp
status_t ACodec::setupVideoDecoder(
        const char *mime, const sp<AMessage> &msg, bool haveNativeWindow,
        bool usingSwRenderer, sp<AMessage> &outputFormat) {
    int32_t width, height;
    if (!msg->findInt32("width", &width)
            || !msg->findInt32("height", &height)) {
        return INVALID_OPERATION;
    }

    OMX_VIDEO_CODINGTYPE compressionFormat;
    status_t err = GetVideoCodingTypeFromMime(mime, &compressionFormat);

    if (err != OK) {
        return err;
    }
    ......
    err = setVideoPortFormatType(
            kPortIndexInput, compressionFormat, OMX_COLOR_FormatUnused);

    if (err != OK) {
        return err;
    }

    int32_t tmp;
    if (msg->findInt32("color-format", &tmp)) {
        OMX_COLOR_FORMATTYPE colorFormat =
            static_cast<OMX_COLOR_FORMATTYPE>(tmp);
        err = setVideoPortFormatType(
                kPortIndexOutput, OMX_VIDEO_CodingUnused, colorFormat, haveNativeWindow);
        ......
    } else {
        ......
    }

    if (err != OK) {
        return err;
    }
    ......
    int32_t frameRateInt;
    float frameRateFloat;
    if (!msg->findFloat("frame-rate", &frameRateFloat)) {
        if (!msg->findInt32("frame-rate", &frameRateInt)) {
            frameRateInt = -1;
        }
        frameRateFloat = (float)frameRateInt;
    }

    err = setVideoFormatOnPort(
            kPortIndexInput, width, height, compressionFormat, frameRateFloat);

    if (err != OK) {
        return err;
    }

    err = setVideoFormatOnPort(
            kPortIndexOutput, width, height, OMX_VIDEO_CodingUnused);

    if (err != OK) {
        return err;
    }

    err = setColorAspectsForVideoDecoder(
            width, height, haveNativeWindow | usingSwRenderer, msg, outputFormat);
    if (err == ERROR_UNSUPPORTED) { // support is optional
        err = OK;
    }

    if (err != OK) {
        return err;
    }

    err = setHDRStaticInfoForVideoCodec(kPortIndexOutput, msg, outputFormat);
    if (err == ERROR_UNSUPPORTED) { // support is optional
        err = OK;
    }
    return err;
}
```

GetVideoCodingTypeFromMime(…) 直接从 kVideoCodingMapEntry 数组中查找匹配项，此处查到的是 OMX_VIDEO_CodingAVC。

frameworks/av/media/libstagefright/ACodec.cpp

```cpp
static const struct VideoCodingMapEntry {
    const char *mMime;
    OMX_VIDEO_CODINGTYPE mVideoCodingType;
} kVideoCodingMapEntry[] = {
    { MEDIA_MIMETYPE_VIDEO_AVC, OMX_VIDEO_CodingAVC },
    { MEDIA_MIMETYPE_VIDEO_HEVC, OMX_VIDEO_CodingHEVC },
    { MEDIA_MIMETYPE_VIDEO_MPEG4, OMX_VIDEO_CodingMPEG4 },
    { MEDIA_MIMETYPE_VIDEO_H263, OMX_VIDEO_CodingH263 },
    { MEDIA_MIMETYPE_VIDEO_MPEG2, OMX_VIDEO_CodingMPEG2 },
    { MEDIA_MIMETYPE_VIDEO_VP8, OMX_VIDEO_CodingVP8 },
    { MEDIA_MIMETYPE_VIDEO_VP9, OMX_VIDEO_CodingVP9 },
    { MEDIA_MIMETYPE_VIDEO_DOLBY_VISION, OMX_VIDEO_CodingDolbyVision },
    { MEDIA_MIMETYPE_IMAGE_ANDROID_HEIC, OMX_VIDEO_CodingImageHEIC },
};

static status_t GetVideoCodingTypeFromMime(
        const char *mime, OMX_VIDEO_CODINGTYPE *codingType) {
    for (size_t i = 0;
         i < sizeof(kVideoCodingMapEntry) / sizeof(kVideoCodingMapEntry[0]);
         ++i) {
        if (!strcasecmp(mime, kVideoCodingMapEntry[i].mMime)) {
            *codingType = kVideoCodingMapEntry[i].mVideoCodingType;
            return OK;
        }
    }

    *codingType = OMX_VIDEO_CodingUnused;

    return ERROR_UNSUPPORTED;
}
```

1. 调用 LWOmxNode getParameter(…) 获取以 OMX_IndexParamVideoPortFormat 索引的 OMX_VIDEO_PARAM_PORTFORMATTYPE 结构，最终会调用 Rkvpu_OMX_GetParameter(…)；
2. 将灵活的颜色格式替换为编解码器支持的格式，此处端口输出配置颜色空间时会打印 Log：ACodec: [OMX.rk.video_decoder.avc] using color format 0x15 in place of 0x7f420888。
3. 调用 LWOmxNode setParameter(…) 进行 Video Port Format 配置，最终会调到 Rkvpu_OMX_SetParameter(…)。

frameworks/av/media/libstagefright/ACodec.cpp

```cpp
status_t ACodec::setVideoPortFormatType(
        OMX_U32 portIndex,
        OMX_VIDEO_CODINGTYPE compressionFormat,
        OMX_COLOR_FORMATTYPE colorFormat,
        bool usingNativeBuffers) {
    OMX_VIDEO_PARAM_PORTFORMATTYPE format;
    InitOMXParams(&format);
    format.nPortIndex = portIndex;
    format.nIndex = 0;
    bool found = false;

    for (OMX_U32 index = 0; index <= kMaxIndicesToCheck; ++index) {
        format.nIndex = index;
        status_t err = mOMXNode->getParameter(
                OMX_IndexParamVideoPortFormat,
                &format, sizeof(format));

        if (err != OK) {
            return err;
        }

        // substitute back flexible color format to codec supported format
        // 将灵活的颜色格式替换为编解码器支持的格式
        OMX_U32 flexibleEquivalent;
        if (compressionFormat == OMX_VIDEO_CodingUnused
                && IsFlexibleColorFormat(
                        mOMXNode, format.eColorFormat, usingNativeBuffers, &flexibleEquivalent)
                && colorFormat == flexibleEquivalent) {
            ALOGI("[%s] using color format %#x in place of %#x",
                    mComponentName.c_str(), format.eColorFormat, colorFormat);
            colorFormat = format.eColorFormat;
        }
        ......
        if (format.eCompressionFormat == compressionFormat
            && format.eColorFormat == colorFormat) {
            found = true;
            break;
        }
        ......
    }

    if (!found) {
        return UNKNOWN_ERROR;
    }

    status_t err = mOMXNode->setParameter(
            OMX_IndexParamVideoPortFormat, &format, sizeof(format));

    return err;
}
```

setVideoFormatOnPort(…) 重点调用了 LWOmxNode getParameter(…) 用来查找端口定义结构 OMX_PARAM_PORTDEFINITIONTYPE，然后进行了必要的赋值，再调用 LWOmxNode setParameter(…) 进行配置。

frameworks/av/media/libstagefright/ACodec.cpp

```cpp
status_t ACodec::setVideoFormatOnPort(
        OMX_U32 portIndex,
        int32_t width, int32_t height, OMX_VIDEO_CODINGTYPE compressionFormat,
        float frameRate) {
    OMX_PARAM_PORTDEFINITIONTYPE def;
    InitOMXParams(&def);
    def.nPortIndex = portIndex;

    OMX_VIDEO_PORTDEFINITIONTYPE *video_def = &def.format.video;

    status_t err = mOMXNode->getParameter(
            OMX_IndexParamPortDefinition, &def, sizeof(def));
    if (err != OK) {
        return err;
    }

    if (portIndex == kPortIndexInput) {
        // XXX Need a (much) better heuristic to compute input buffer sizes.
        // 需要一个更好的启发式来计算输入缓冲区大小。
        const size_t X = 64 * 1024;
        if (def.nBufferSize < X) {
            def.nBufferSize = X;
        }
    }

    if (def.eDomain != OMX_PortDomainVideo) {
        ALOGE("expected video port, got %s(%d)", asString(def.eDomain), def.eDomain);
        return FAILED_TRANSACTION;
    }

    video_def->nFrameWidth = width;
    video_def->nFrameHeight = height;

    if (portIndex == kPortIndexInput) {
        video_def->eCompressionFormat = compressionFormat;
        video_def->eColorFormat = OMX_COLOR_FormatUnused;
        if (frameRate >= 0) {
            video_def->xFramerate = (OMX_U32)(frameRate * 65536.0f);
        }
    }

    err = mOMXNode->setParameter(
            OMX_IndexParamPortDefinition, &def, sizeof(def));

    return err;
}
```

DescribeColorAspectsParams 结构体代表颜色空间描述参数。当“OMX.google.android.index.describeColorAspects”扩展被给出时，通过 OMX_SetConfig 或 OMX_GetConfig 传递给视频编码器或解码器。

1. 调用 getColorAspectsFromFormat(…) 查询 ColorAspects（表征颜色 Aspect 的结构体）；
2. 调用 setDefaultCodecColorAspectsIfNeeded(…) 设置默认 ColorAspects；
3. 调用 setColorAspectsIntoFormat(…) 设置 ColorAspects；
4. 调用 initDescribeColorAspectsIndex() 初始化 Describe ColorAspects Index（OMX.google.android.index.describeColorAspects）；
5. 将 ColorAspects 传达给编解码器。

frameworks/av/media/libstagefright/ACodec.cpp

```cpp
status_t ACodec::setColorAspectsForVideoDecoder(
        int32_t width, int32_t height, bool usingNativeWindow,
        const sp<AMessage> &configFormat, sp<AMessage> &outputFormat) {
    DescribeColorAspectsParams params;
    InitOMXParams(&params);
    params.nPortIndex = kPortIndexOutput;

    getColorAspectsFromFormat(configFormat, params.sAspects);
    if (usingNativeWindow) {
        setDefaultCodecColorAspectsIfNeeded(params.sAspects, width, height);
        // The default aspects will be set back to the output format during the
        // getFormat phase of configure(). Set non-Unspecified values back into the
        // format, in case component does not support this enumeration.
        setColorAspectsIntoFormat(params.sAspects, outputFormat);
    }

    (void)initDescribeColorAspectsIndex();

    // communicate color aspects to codec
    return setCodecColorAspects(params);
}
```

setCodecColorAspects(…) 方法中 verify 参数默认为 false，最终会调用 Rkvpu_OMX_SetConfig(…) 进一步配置参数。

frameworks/av/media/libstagefright/ACodec.cpp

```cpp
status_t ACodec::setCodecColorAspects(DescribeColorAspectsParams &params, bool verify) {
    status_t err = ERROR_UNSUPPORTED;
    if (mDescribeColorAspectsIndex) {
        err = mOMXNode->setConfig(mDescribeColorAspectsIndex, &params, sizeof(params));
    }
    ALOGV("[%s] setting color aspects (R:%d(%s), P:%d(%s), M:%d(%s), T:%d(%s)) err=%d(%s)",
            mComponentName.c_str(),
            params.sAspects.mRange, asString(params.sAspects.mRange),
            params.sAspects.mPrimaries, asString(params.sAspects.mPrimaries),
            params.sAspects.mMatrixCoeffs, asString(params.sAspects.mMatrixCoeffs),
            params.sAspects.mTransfer, asString(params.sAspects.mTransfer),
            err, asString(err));

    if (verify && err == OK) {
        err = getCodecColorAspects(params);
    }

    ALOGW_IF(err == ERROR_UNSUPPORTED && mDescribeColorAspectsIndex,
            "[%s] setting color aspects failed even though codec advertises support",
            mComponentName.c_str());
    return err;
}
```

DescribeHDRStaticInfoParams 结构体代表 HDR 颜色描述参数。当“OMX.google.android.index.describeHDRStaticInfo”扩展被给出并检测到 HDR 流时，通过 OMX_SetConfig 或 OMX_GetConfig 传递给视频编码器或解码器。

frameworks/av/media/libstagefright/ACodec.cpp

```cpp
status_t ACodec::setHDRStaticInfoForVideoCodec(
        OMX_U32 portIndex, const sp<AMessage> &configFormat, sp<AMessage> &outputFormat) {
    CHECK(portIndex == kPortIndexInput || portIndex == kPortIndexOutput);

    DescribeHDRStaticInfoParams params;
    InitOMXParams(&params);
    params.nPortIndex = portIndex;

    HDRStaticInfo *info = &params.sInfo;
    if (getHDRStaticInfoFromFormat(configFormat, info)) {
        setHDRStaticInfoIntoFormat(params.sInfo, outputFormat);
    }

    (void)initDescribeHDRStaticInfoIndex();

    // communicate HDR static Info to codec
    return setHDRStaticInfo(params);
}
```

1. 调用 LWOmxNode getParameter(…) 获取以 OMX_IndexParamPortDefinition 为索引的 OMX_PARAM_PORTDEFINITIONTYPE 结构体；
2. 根据 OMX_PARAM_PORTDEFINITIONTYPE 结构体 eDomain 字段（端口的域）case 到不同分支，这里会分支到 OMX_PortDomainVideo；
3. 给入参 notify 指向的 AMessage 对象写入各种参数；
4. 调用 getVendorParameters(…) 进一步获取供应商参数。

frameworks/av/media/libstagefright/ACodec.cpp

```cpp
status_t ACodec::getPortFormat(OMX_U32 portIndex, sp<AMessage> &notify) {
    const char *niceIndex = portIndex == kPortIndexInput ? "input" : "output";
    OMX_PARAM_PORTDEFINITIONTYPE def;
    InitOMXParams(&def);
    def.nPortIndex = portIndex;

    status_t err = mOMXNode->getParameter(OMX_IndexParamPortDefinition, &def, sizeof(def));
    if (err != OK) {
        return err;
    }

    if (def.eDir != (portIndex == kPortIndexOutput ? OMX_DirOutput : OMX_DirInput)) {
        ALOGE("unexpected dir: %s(%d) on %s port", asString(def.eDir), def.eDir, niceIndex);
        return BAD_VALUE;
    }

    switch (def.eDomain) {
        case OMX_PortDomainVideo:
        {
            OMX_VIDEO_PORTDEFINITIONTYPE *videoDef = &def.format.video;
            switch ((int)videoDef->eCompressionFormat) {
                ......
                default:
                {
                    if (mIsEncoder ^ (portIndex == kPortIndexOutput)) {
                        // should be CodingUnused
                        ALOGE("Raw port video compression format is %s(%d)",
                                asString(videoDef->eCompressionFormat),
                                videoDef->eCompressionFormat);
                        return BAD_VALUE;
                    }
                    AString mime;
                    if (GetMimeTypeForVideoCoding(
                        videoDef->eCompressionFormat, &mime) != OK) {
                        notify->setString("mime", "application/octet-stream");
                    } else {
                        notify->setString("mime", mime.c_str());
                    }
                    uint32_t intraRefreshPeriod = 0;
                    if (mIsEncoder && !mIsImage &&
                            getIntraRefreshPeriod(&intraRefreshPeriod) == OK
                            && intraRefreshPeriod > 0) {
                        notify->setInt32("intra-refresh-period", intraRefreshPeriod);
                    }
                    break;
                }
            }
            notify->setInt32("width", videoDef->nFrameWidth);
            notify->setInt32("height", videoDef->nFrameHeight);
            ALOGV("[%s] %s format is %s", mComponentName.c_str(),
                    portIndex == kPortIndexInput ? "input" : "output",
                    notify->debugString().c_str());

            break;
        }
        ......
        default:
            ALOGE("Unsupported domain: %s(%d)", asString(def.eDomain), def.eDomain);
            return BAD_TYPE;
    }

    return getVendorParameters(portIndex, notify);
}
```

