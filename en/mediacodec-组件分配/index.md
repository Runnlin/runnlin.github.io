# 

d组件分配的起点位于 native MediaCodec 的 init 流程。init() 函数中构建 kWhatInit 消息后等待处理结束，消息是由 MediaCodec 本身处理的。

frameworks/av/media/libstagefright/MediaCodec.cpp

```
status_t MediaCodec::init(const AString &name) {
    ......
    sp<AMessage> msg = new AMessage(kWhatInit, this);
    msg->setObject("codecInfo", mCodecInfo);
    // name may be different from mCodecInfo->getCodecName() if we stripped
    // ".secure"
    msg->setString("name", name);
    ......
    for (int i = 0; i <= kMaxRetry; ++i) {
        ......
        sp<AMessage> response;
        err = PostAndAwaitResponse(msg, &response);
        if (!isResourceError(err)) {
            break;
        }
    }
    return err;
}
```

1. 调用 setState(…) 将状态更改为 INITIALIZING；
2. 从 msg 入参中重新构建消息，并调用 ACodec initiateAllocateComponent(…) 分配组件。

frameworks/av/media/libstagefright/MediaCodec.cpp

```
void MediaCodec::onMessageReceived(const sp<AMessage> &msg) {
    switch (msg->what()) {
        ......
        case kWhatInit:
        {
            sp<AReplyToken> replyID;
            CHECK(msg->senderAwaitsResponse(&replyID));

            if (mState != UNINITIALIZED) {
                PostReplyWithError(replyID, INVALID_OPERATION);
                break;
            }

            mReplyID = replyID;
            setState(INITIALIZING);

            sp<RefBase> codecInfo;
            CHECK(msg->findObject("codecInfo", &codecInfo));
            AString name;
            CHECK(msg->findString("name", &name));

            sp<AMessage> format = new AMessage;
            format->setObject("codecInfo", codecInfo);
            format->setString("componentName", name);

            mCodec->initiateAllocateComponent(format);
            break;
        }
        ......
    }
}
12345678910111213141516171819202122232425262728293031
```

ACodec::initiateAllocateComponent(…) 中将消息真正发出，消息 what 标志设置为 kWhatAllocateComponent。处理者设置为 this 意味着就是 ACodec 自己。

frameworks/av/media/libstagefright/ACodec.cpp

```
void ACodec::initiateAllocateComponent(const sp<AMessage> &msg) {
    msg->setWhat(kWhatAllocateComponent);
    msg->setTarget(this);
    msg->post();
}

```

由于 MediaCodec 初始化期间构造了 ACodec，ACodec 此时的状态设置为 mUninitializedState，也就是未初始化（调用 changeState(…) 实现）。changeState(…) 实现在 AHierarchicalStateMachine.cpp 中，ACodec 继承了 AHierarchicalStateMachine。

frameworks/av/media/libstagefright/include/media/stagefright/ACodec.h

AState 结构体定义了纯虚函数 onMessageReceived(…) 这肯定是和消息处理相关的，后面揭晓答案。

AHierarchicalStateMachine 结构体里定义了 changeState(…) 方法；另外还有 handleMessage(…) 虚函数，和消息处理相关。

frameworks/av/media/libstagefright/include/media/stagefright/AHierarchicalStateMachine.h

```
#ifndef A_HIERARCHICAL_STATE_MACHINE_H_

#define A_HIERARCHICAL_STATE_MACHINE_H_

#include <media/stagefright/foundation/AHandler.h>

namespace android {

struct AState : public RefBase {
    AState(const sp<AState> &parentState = NULL);

    sp<AState> parentState();

protected:
    virtual ~AState();

    virtual void stateEntered();
    virtual void stateExited();

    virtual bool onMessageReceived(const sp<AMessage> &msg) = 0;

private:
    friend struct AHierarchicalStateMachine;

    sp<AState> mParentState;

    DISALLOW_EVIL_CONSTRUCTORS(AState);
};

struct AHierarchicalStateMachine {
    AHierarchicalStateMachine();

protected:
    virtual ~AHierarchicalStateMachine();

    virtual void handleMessage(const sp<AMessage> &msg);

    // Only to be called in response to a message.
    void changeState(const sp<AState> &state);

private:
    sp<AState> mState;

    DISALLOW_EVIL_CONSTRUCTORS(AHierarchicalStateMachine);
};

}  // namespace android

#endif  // A_HIERARCHICAL_STATE_MACHINE_H_
12345678910111213141516171819202122232425262728293031323334353637383940414243444546474849
```

现在来摸清楚 changeState(…) 改变状态都干了什么？

注意 AState 构造器中初始化了 mParentState，并且 parentState() 方法返回就是 mParentState。AHierarchicalStateMachine::changeState(const sp &state) 入参是一个指向 AState 的指针，所有前面提到的 ACodec 九种状态当然就需要全部继承自 AState。

AHierarchicalStateMachine::changeState(…) 实现：

1. 如果设置状态和当前保存在 mState 中的状态一样，直接返回；
2. 将旧的 State 链 push 到容器中；
3. 将新的 State 链 push 到容器中；
4. 去掉两个容器中相同的 top；
5. 旧的 State 链调用 stateExited()；
6. 新的 State 链调用 stateEntered()。

AHierarchicalStateMachine::handleMessage(…) 实际调用了 State 链去处理消息，如果 onMessageReceived(…) 不能返回 true，就沿着父 State 链上升，直到可以处理消息返回 true。

frameworks/av/media/libstagefright/AHierarchicalStateMachine.cpp

```
namespace android {

AState::AState(const sp<AState> &parentState)
    : mParentState(parentState) {
}
......
sp<AState> AState::parentState() {
    return mParentState;
}
......
void AHierarchicalStateMachine::handleMessage(const sp<AMessage> &msg) {
    sp<AState> save = mState;

    sp<AState> cur = mState;
    while (cur != NULL && !cur->onMessageReceived(msg)) {
        // If you claim not to have handled the message you shouldn't
        // have called setState...
        CHECK(save == mState);

        cur = cur->parentState();
    }

    if (cur != NULL) {
        return;
    }

    ALOGW("Warning message %s unhandled in root state.",
         msg->debugString().c_str());
}

void AHierarchicalStateMachine::changeState(const sp<AState> &state) {
    if (state == mState) {
        // Quick exit for the easy case.
        return;
    }

    Vector<sp<AState> > A;
    sp<AState> cur = mState;
    for (;;) {
        A.push(cur);
        if (cur == NULL) {
            break;
        }
        cur = cur->parentState();
    }

    Vector<sp<AState> > B;
    cur = state;
    for (;;) {
        B.push(cur);
        if (cur == NULL) {
            break;
        }
        cur = cur->parentState();
    }

    // Remove the common tail.
    while (A.size() > 0 && B.size() > 0 && A.top() == B.top()) {
        A.pop();
        B.pop();
    }

    mState = state;

    for (size_t i = 0; i < A.size(); ++i) {
        A.editItemAt(i)->stateExited();
    }

    for (size_t i = B.size(); i > 0;) {
        i--;
        B.editItemAt(i)->stateEntered();
    }
}

}  // namespace android
1234567891011121314151617181920212223242526272829303132333435363738394041424344454647484950515253545556575859606162636465666768697071727374
```

UninitializedState 继承自 ACodec::BaseState，看来 BaseState 可能继承自 AState。

frameworks/av/media/libstagefright/ACodec.cpp

```
struct ACodec::UninitializedState : public ACodec::BaseState {
    explicit UninitializedState(ACodec *codec);

protected:
    virtual bool onMessageReceived(const sp<AMessage> &msg);
    virtual void stateEntered();

private:
    void onSetup(const sp<AMessage> &msg);
    bool onAllocateComponent(const sp<AMessage> &msg);

    sp<DeathNotifier> mDeathNotifier;

    DISALLOW_EVIL_CONSTRUCTORS(UninitializedState);
};

123456789101112131415
```

BaseState 的确继承自 AState。

frameworks/av/media/libstagefright/ACodec.cpp

![](content/assets/images/ACodec组件分配流程.jpg)

回到主线，kWhatAllocateComponent 这个标志的 message 必定是发给了 ACodec 去处理， ACodec 将 onMessageReceived(…) 定义为一个虚函数，其实现内部则调用了 handleMessage(…) ，如此消息处理就明确了，最终落在了 BaseState 的子类实现上，就是前面提到的九种状态之一。

frameworks/av/media/libstagefright/include/media/stagefright/ACodec.h

```
struct ACodec : public AHierarchicalStateMachine, public CodecBase {
    ......
    // AHierarchicalStateMachine implements the message handling
    virtual void onMessageReceived(const sp<AMessage> &msg) {
        handleMessage(msg);
    }
    ......
}

```

由于 ACodec 此时的状态设置为 mUninitializedState，所以消息处理就落在它的头上。注意它的 onMessageReceived(…) 返回了 true，意味着仅仅由它处理消息，消息不再上升。kWhatAllocateComponent 这个消息仅仅调用了 onAllocateComponent(…) 做进一步处理。

frameworks/av/media/libstagefright/ACodec.cpp

```
bool ACodec::UninitializedState::onMessageReceived(const sp<AMessage> &msg) {
    bool handled = false;

    switch (msg->what()) {
        ......
        case ACodec::kWhatAllocateComponent:
        {
            onAllocateComponent(msg);
            handled = true;
            break;
        }
        ......
        default:
            return BaseState::onMessageReceived(msg);
    }

    return handled;
}
1234567891011121314151617
```

mCodec 指向 ACodec，ACodec 对象是在 ACodec 无参构造器中传给 UninitializedState 构造器的，实际上九种状态都将 ACodec 对象作为实参传入进行了构造工作。

1. 构建 kWhatOMXMessageList message，将 generation 的值设为 1，因为 ACodec 之前在无参构造器中将 mNodeGeneration 字段初始化 0；
2. 从入参 msg 消息中获取 MediaCodecInfo、owner（属主）和 componentName（组件名称）；
3. 创建 CodecObserver 对象；
4. 调用 OMXClient connect(…) 方法连接；
5. 调用 OMXClient interface() 获取 OMX 代理对象；
6. 修改当前线程优先级为 ANDROID_PRIORITY_FOREGROUND，接着调用 OMX allocateNode(…) 分配 omx 节点，还原线程优先级；
7. 调用 OMXNode getHalInterface(…) 获取 hal 接口，并设置“死亡通知” DeathNotifier；
8. mNodeGeneration 自增加 1；
9. 给 mCodec 指向的 ACodec 对象设置 mComponentName、mOMX 等字段的值；
10. 调用 changeState(…) 将 ACodec 对象的状态更改为加载状态（mLoadedState）。

frameworks/av/media/libstagefright/ACodec.cpp

```
bool ACodec::UninitializedState::onAllocateComponent(const sp<AMessage> &msg) {
    ALOGV("onAllocateComponent");

    CHECK(mCodec->mOMXNode == NULL);

    sp<AMessage> notify = new AMessage(kWhatOMXMessageList, mCodec);
    notify->setInt32("generation", mCodec->mNodeGeneration + 1);

    sp<RefBase> obj;
    CHECK(msg->findObject("codecInfo", &obj));
    sp<MediaCodecInfo> info = (MediaCodecInfo *)obj.get();
    if (info == nullptr) {
        ALOGE("Unexpected nullptr for codec information");
        mCodec->signalError(OMX_ErrorUndefined, UNKNOWN_ERROR);
        return false;
    }
    AString owner = (info->getOwnerName() == nullptr) ? "default" : info->getOwnerName();

    AString componentName;
    CHECK(msg->findString("componentName", &componentName));

    sp<CodecObserver> observer = new CodecObserver(notify);
    sp<IOMX> omx;
    sp<IOMXNode> omxNode;

    status_t err = NAME_NOT_FOUND;
    OMXClient client;
    if (client.connect(owner.c_str()) != OK) {
        mCodec->signalError(OMX_ErrorUndefined, NO_INIT);
        return false;
    }
    omx = client.interface();

    pid_t tid = gettid();
    int prevPriority = androidGetThreadPriority(tid);
    androidSetThreadPriority(tid, ANDROID_PRIORITY_FOREGROUND);
    err = omx->allocateNode(componentName.c_str(), observer, &omxNode);
    androidSetThreadPriority(tid, prevPriority);

    if (err != OK) {
        ALOGE("Unable to instantiate codec '%s' with err %#x.", componentName.c_str(), err);

        mCodec->signalError((OMX_ERRORTYPE)err, makeNoSideEffectStatus(err));
        return false;
    }

    mDeathNotifier = new DeathNotifier(new AMessage(kWhatOMXDied, mCodec));
    auto tOmxNode = omxNode->getHalInterface<IOmxNode>();
    if (tOmxNode && !tOmxNode->linkToDeath(mDeathNotifier, 0)) {
        mDeathNotifier.clear();
    }

    ++mCodec->mNodeGeneration;

    mCodec->mComponentName = componentName;
    mCodec->mRenderTracker.setComponentName(componentName);
    mCodec->mFlags = 0;

    if (componentName.endsWith(".secure")) {
        mCodec->mFlags |= kFlagIsSecure;
        mCodec->mFlags |= kFlagIsGrallocUsageProtected;
        mCodec->mFlags |= kFlagPushBlankBuffersToNativeWindowOnShutdown;
    }

    mCodec->mOMX = omx;
    mCodec->mOMXNode = omxNode;
    mCodec->mCallback->onComponentAllocated(mCodec->mComponentName.c_str());
    mCodec->changeState(mCodec->mLoadedState);

    return true;
}
1234567891011121314151617181920212223242526272829303132333435363738394041424344454647484950515253545556575859606162636465666768697071
```

CodecObserver 的创建非常简单，仅仅给 mNotify 字段进行了赋值而已。

frameworks/av/media/libstagefright/ACodec.cpp

```
struct CodecObserver : public BnOMXObserver {
    explicit CodecObserver(const sp<AMessage> &msg) : mNotify(msg) {}
    ......
private:
    const sp<AMessage> mNotify;

    DISALLOW_EVIL_CONSTRUCTORS(CodecObserver);
};

```

OMXClient::connect(const char* name) 内部首先调用 IOmx::getService(…) 获取 Omx 服务的代理，然后将这个代理对象传入 LWOmx 构造器初始化了 LWOmx 对象。

frameworks/av/media/libstagefright/OMXClient.cpp

```
namespace android {

OMXClient::OMXClient() {
}

status_t OMXClient::connect() {
    return connect("default");
}

status_t OMXClient::connect(const char* name) {
    using namespace ::android::hardware::media::omx::V1_0;
    if (name == nullptr) {
        name = "default";
    }
    sp<IOmx> tOmx = IOmx::getService(name);
    if (tOmx.get() == nullptr) {
        ALOGE("Cannot obtain IOmx service.");
        return NO_INIT;
    }
    if (!tOmx->isRemote()) {
        ALOGE("IOmx service running in passthrough mode.");
        return NO_INIT;
    }
    mOMX = new utils::LWOmx(tOmx);
    ALOGI("IOmx service obtained");
    return OK;
}

void OMXClient::disconnect() {
    mOMX.clear();
}

sp<IOMX> OMXClient::interface() {
    return mOMX;
}

}  // namespace android
123456789101112131415161718192021222324252627282930313233343536
```

用于转换的包装器类。LW = Legacy Wrapper， 它将 Treble 对象包装在 Legacy 对象中。

结合上面的代码，不难看出调用 OMXClient interface() 获取 OMX 代理对象实际上是一个包装对象 LWOmx（确切的说是结构体）。

frameworks/av/media/libmedia/include/media/omx/1.0/WOmx.h

```
/**
 * Wrapper classes for conversion
 * ==============================
 *
 * Naming convention:
 * - LW = Legacy Wrapper --- It wraps a Treble object inside a legacy object.
 * - TW = Treble Wrapper --- It wraps a legacy object inside a Treble object.
 */

struct LWOmx : public IOMX {
    sp<IOmx> mBase;
    LWOmx(sp<IOmx> const& base);
    status_t listNodes(List<IOMX::ComponentInfo>* list) override;
    status_t allocateNode(
            char const* name,
            sp<IOMXObserver> const& observer,
            sp<IOMXNode>* omxNode) override;
    status_t createInputSurface(
            sp<::android::IGraphicBufferProducer>* bufferProducer,
            sp<::android::IGraphicBufferSource>* bufferSource) override;
};
1234567891011121314151617181920
```

调用 OMX allocateNode(…) 分配 omx 节点，实际是调用 LWOmx allocateNode(…) 分配 omx 节点。

mBase 指向 Omx 服务的代理对象，所以实际还是调用了 Omx 服务的 allocateNode(…)。我们看到返回指向 IOmxNode 的指针，实际指向 LWOmxNode 对象。

frameworks/av/media/libmedia/omx/1.0/WOmx.cpp

```
status_t LWOmx::allocateNode(
        char const* name,
        sp<IOMXObserver> const& observer,
        sp<IOMXNode>* omxNode) {
    status_t fnStatus;
    status_t transStatus = toStatusT(mBase->allocateNode(
            name, new TWOmxObserver(observer),
            [&fnStatus, omxNode](Status status, sp<IOmxNode> const& node) {
                fnStatus = toStatusT(status);
                *omxNode = new LWOmxNode(node);
            }));
    return transStatus == NO_ERROR ? fnStatus : transStatus;
}
123456789101112
```

1. 如果 mLiveNodes 容器的 size 已经达到了 2^16，返回 NO_MEMORY；
2. 新建 OMXNodeInstance 对象；
3. 调用 OMXMaster makeComponentInstance(…) 真正的创建组件实例；
4. 从 mParser 中查找 quirk，调用 OMXNodeInstance setQuirks(…) 设置 quirk；
5. mLiveNodes 容器和 mNode2Observer 容器中添加新条目；
6. hal 返回 OK 状态和 TWOmxNode 对象。

frameworks/av/media/libstagefright/omx/1.0/Omx.cpp

```

Return<void> Omx::allocateNode(
        const hidl_string& name,
        const sp<IOmxObserver>& observer,
        allocateNode_cb _hidl_cb) {

    using ::android::IOMXNode;
    using ::android::IOMXObserver;

    sp<OMXNodeInstance> instance;
    {
        Mutex::Autolock autoLock(mLock);
        if (mLiveNodes.size() == kMaxNodeInstances) {
            _hidl_cb(toStatus(NO_MEMORY), nullptr);
            return Void();
        }

        instance = new OMXNodeInstance(
                this, new LWOmxObserver(observer), name.c_str());

        OMX_COMPONENTTYPE *handle;
        OMX_ERRORTYPE err = mMaster->makeComponentInstance(
                name.c_str(), &OMXNodeInstance::kCallbacks,
                instance.get(), &handle);

        if (err != OMX_ErrorNone) {
            LOG(ERROR) << "Failed to allocate omx component "
                    "'" << name.c_str() << "' "
                    " err=" << asString(err) <<
                    "(0x" << std::hex << unsigned(err) << ")";
            _hidl_cb(toStatus(StatusFromOMXError(err)), nullptr);
            return Void();
        }
        instance->setHandle(handle);

        // Find quirks from mParser
        const auto& codec = mParser.getCodecMap().find(name.c_str());
        if (codec == mParser.getCodecMap().cend()) {
            LOG(WARNING) << "Failed to obtain quirks for omx component "
                    "'" << name.c_str() << "' "
                    "from XML files";
        } else {
            uint32_t quirks = 0;
            for (const auto& quirk : codec->second.quirkSet) {
                if (quirk == "quirk::requires-allocate-on-input-ports") {
                    quirks |= OMXNodeInstance::
                            kRequiresAllocateBufferOnInputPorts;
                }
                if (quirk == "quirk::requires-allocate-on-output-ports") {
                    quirks |= OMXNodeInstance::
                            kRequiresAllocateBufferOnOutputPorts;
                }
            }
            instance->setQuirks(quirks);
        }

        mLiveNodes.add(observer.get(), instance);
        mNode2Observer.add(instance.get(), observer.get());
    }
    observer->linkToDeath(this, 0);

    _hidl_cb(toStatus(OK), new TWOmxNode(instance));
    return Void();
}
123456789101112131415161718192021222324252627282930313233343536373839404142434445464748495051525354555657585960616263
```

先来分析新建 OMXNodeInstance 对象。给一系列字段进行了初始化，赋初值。

frameworks/av/media/libstagefright/omx/OMXNodeInstance.cpp

```
OMXNodeInstance::OMXNodeInstance(
        Omx *owner, const sp<IOMXObserver> &observer, const char *name)
    : mOwner(owner),
      mHandle(NULL),
      mObserver(observer),
      mDying(false),
      mSailed(false),
      mQueriedProhibitedExtensions(false),
      mQuirks(0),
      mBufferIDCount(0),
      mRestorePtsFailed(false),
      mMaxTimestampGapUs(0LL),
      mPrevOriginalTimeUs(-1LL),
      mPrevModifiedTimeUs(-1LL)
{
    mName = ADebug::GetDebugName(name);
    DEBUG = ADebug::GetDebugLevelFromProperty(name, "debug.stagefright.omx-debug");
    ALOGV("debug level for %s is %d", name, DEBUG);
    DEBUG_BUMP = DEBUG;
    mNumPortBuffers[0] = 0;
    mNumPortBuffers[1] = 0;
    mDebugLevelBumpPendingBuffers[0] = 0;
    mDebugLevelBumpPendingBuffers[1] = 0;
    mMetadataType[0] = kMetadataBufferTypeInvalid;
    mMetadataType[1] = kMetadataBufferTypeInvalid;
    mPortMode[0] = IOMX::kPortModePresetByteBuffer;
    mPortMode[1] = IOMX::kPortModePresetByteBuffer;
    mSecureBufferType[0] = kSecureBufferTypeUnknown;
    mSecureBufferType[1] = kSecureBufferTypeUnknown;
    mGraphicBufferEnabled[0] = false;
    mGraphicBufferEnabled[1] = false;
    mIsSecure = AString(name).endsWith(".secure");
    mLegacyAdaptiveExperiment = ADebug::isExperimentEnabled("legacy-adaptive");
}
123456789101112131415161718192021222324252627282930313233
```

再来调用 OMXMaster makeComponentInstance(…) 真正的创建组件实例。以 rk3399 android10 平台为例，mine 类型为 video/avc，此处打印如下 Log：

OMXMaster: makeComponentInstance(OMX.rk.video_decoder.avc) in android.hardwar process

首先从 mPluginByComponentName 中查找插件，这里必然是 RK OMX Plugin。然后调用插件中的 makeComponentInstance(…) 实现。

frameworks/av/media/libstagefright/omx/OMXMaster.cpp

```
OMX_ERRORTYPE OMXMaster::makeComponentInstance(
        const char *name,
        const OMX_CALLBACKTYPE *callbacks,
        OMX_PTR appData,
        OMX_COMPONENTTYPE **component) {
    ALOGI("makeComponentInstance(%s) in %s process", name, mProcessName);
    Mutex::Autolock autoLock(mLock);

    *component = NULL;

    ssize_t index = mPluginByComponentName.indexOfKey(String8(name));

    if (index < 0) {
        return OMX_ErrorInvalidComponentName;
    }

    OMXPluginBase *plugin = mPluginByComponentName.valueAt(index);
    OMX_ERRORTYPE err =
        plugin->makeComponentInstance(name, callbacks, appData, component);

    if (err != OMX_ErrorNone) {
        return err;
    }

    mPluginByInstance.add(*component, plugin);

    return err;
}
1101112131415161718192021222324252627
```

此处主要会调用 libOMX_Core.so 中的 RKOMX_GetHandle(…) 方法。

hardware/rockchip/librkvpu/libstagefrighthw/RKOMXPlugin.cpp

```
OMX_ERRORTYPE RKOMXPlugin::makeComponentInstance(
        const char *name,
        const OMX_CALLBACKTYPE *callbacks,
        OMX_PTR appData,
        OMX_COMPONENTTYPE **component) {
    for (OMX_U32 i = 0; i < mCores.size(); i++) {
        if (mCores[i] != NULL) {
            if (mCores[i]->mLibHandle == NULL) {
                continue;
            }

            OMX_ERRORTYPE omx_res = (*(mCores[i]->mGetHandle))(
                reinterpret_cast<OMX_HANDLETYPE *>(component),
                const_cast<char *>(name),
                appData, const_cast<OMX_CALLBACKTYPE *>(callbacks));
            if(omx_res == OMX_ErrorNone) {
                Mutex::Autolock autoLock(mMutex);
                RKOMXComponent comp;

                comp.mComponent = *component;
                comp.mCore = mCores[i];

                mComponents.push_back(comp);
                return OMX_ErrorNone;
            } else if (omx_res == OMX_ErrorInsufficientResources) {
                return omx_res;
            }
        }
    }
    return OMX_ErrorInvalidComponentName;
}
1101112131415161718192021222324252627282930
```

1. 检查入参是否存在 NULL，如果存在直接退出；
2. 从 gComponentList 列表中查找组件名称为 OMX.rk.video_decoder.avc 的组件，然后根据找到的条目给 ROCKCHIP_OMX_COMPONENT 结构各字段赋值；
3. 调用 Rockchip_OMX_ComponentLoad(…) 进行组件加载；
4. 调用 ROCKCHIP_OMX_COMPONENT 结构内的 pOMXComponent 字段 SetCallbacks(…) 方法，pOMXComponent 指向 OMX_HANDLETYPE 结构，这个结构体定义了组件句柄，组件句柄用于访问组件的所有公共方法，还包含指向组件私有数据区域的指针；
5. 调用 Rockchip_OMX_Check_Resource(…) 检查资源；
6. 修改 gLoadComponentList 全局变量，如果 gLoadComponentList 为 NULL，直接将第一步分配的 ROCKCHIP_OMX_COMPONENT 结构赋给它，否则将ROCKCHIP_OMX_COMPONENT 结构挂在以 gLoadComponentList 为头的单链表尾部；

hardware/rockchip/omx_il/core/Rockchip_OMX_Core.c

```
OMX_API OMX_ERRORTYPE OMX_APIENTRY RKOMX_GetHandle(
    OMX_OUT OMX_HANDLETYPE *pHandle,
    OMX_IN  OMX_STRING cComponentName,
    OMX_IN  OMX_PTR pAppData,
    OMX_IN  OMX_CALLBACKTYPE *pCallBacks)
{
    OMX_ERRORTYPE         ret = OMX_ErrorNone;
    ROCKCHIP_OMX_COMPONENT *loadComponent;
    ROCKCHIP_OMX_COMPONENT *currentComponent;
    unsigned int i = 0;

    FunctionIn();

    if (gInitialized != 1) {
        ret = OMX_ErrorNotReady;
        goto EXIT;
    }

    if ((pHandle == NULL) || (cComponentName == NULL) || (pCallBacks == NULL)) {
        ret = OMX_ErrorBadParameter;
        goto EXIT;
    }
    omx_trace("ComponentName : %s", cComponentName);

    for (i = 0; i < gComponentNum; i++) {
        if (Rockchip_OSAL_Strcmp(cComponentName, gComponentList[i].component.componentName) == 0) {
            loadComponent = Rockchip_OSAL_Malloc(sizeof(ROCKCHIP_OMX_COMPONENT));
            Rockchip_OSAL_Memset(loadComponent, 0, sizeof(ROCKCHIP_OMX_COMPONENT));

            Rockchip_OSAL_Strcpy(loadComponent->libName, gComponentList[i].libName);
            Rockchip_OSAL_Strcpy(loadComponent->componentName, gComponentList[i].component.componentName);
            ret = Rockchip_OMX_ComponentLoad(loadComponent);
            if (ret != OMX_ErrorNone) {
                Rockchip_OSAL_Free(loadComponent);
                omx_err("OMX_Error, Line:%d", __LINE__);
                goto EXIT;
            }

            ret = loadComponent->pOMXComponent->SetCallbacks(loadComponent->pOMXComponent, pCallBacks, pAppData);
            if (ret != OMX_ErrorNone) {
                Rockchip_OMX_ComponentUnload(loadComponent);
                Rockchip_OSAL_Free(loadComponent);
                omx_err("OMX_Error 0x%x, Line:%d", ret, __LINE__);
                goto EXIT;
            }

            ret = Rockchip_OMX_Check_Resource(loadComponent->pOMXComponent);
            if (ret != OMX_ErrorNone) {
                Rockchip_OMX_ComponentUnload(loadComponent);
                Rockchip_OSAL_Free(loadComponent);
                omx_err("OMX_Error 0x%x, Line:%d", ret, __LINE__);

                goto EXIT;
            }
            Rockchip_OSAL_MutexLock(ghLoadComponentListMutex);
            if (gLoadComponentList == NULL) {
                gLoadComponentList = loadComponent;
            } else {
                currentComponent = gLoadComponentList;
                while (currentComponent->nextOMXComp != NULL) {
                    currentComponent = currentComponent->nextOMXComp;
                }
                currentComponent->nextOMXComp = loadComponent;
            }
            Rockchip_OSAL_MutexUnlock(ghLoadComponentListMutex);

            *pHandle = loadComponent->pOMXComponent;
            ret = OMX_ErrorNone;
            omx_trace("Rockchip_OMX_GetHandle : %s", "OMX_ErrorNone");
            goto EXIT;
        }
    }

    ret = OMX_ErrorComponentNotFound;

EXIT:
    FunctionOut();

    return ret;
}
1101112131415161718192021222324252627282930313233343536373839404142434445464748
```

1. 由于整体流程在分析硬解码所以打开的 so 库实际是 libomxvpu_dec.so，获取到 Rockchip_OMX_ComponentConstructor 函数符号；
2. 调用 Rockchip_OSAL_Malloc(…) 给 OMX_COMPONENTTYPE 结构分配内存，然后调用 libomxvpu_dec.so 中的 Rockchip_OMX_ComponentConstructor(…) 组件构造方法；
3. 调用 Rockchip_OMX_ComponentAPICheck(…) 检查 OMX_COMPONENTTYPE 结构中的方法是否为 NULL；
4. 给入参 rockchip_component 其他字段赋值。

hardware/rockchip/omx_il/core/Rockchip_OMX_Component_Register.c

```
OMX_ERRORTYPE Rockchip_OMX_ComponentLoad(ROCKCHIP_OMX_COMPONENT *rockchip_component)
{
    OMX_ERRORTYPE      ret = OMX_ErrorNone;
    OMX_HANDLETYPE     libHandle;
    OMX_COMPONENTTYPE *pOMXComponent;

    FunctionIn();

    OMX_ERRORTYPE (*Rockchip_OMX_ComponentConstructor)(OMX_HANDLETYPE hComponent, OMX_STRING componentName);

    libHandle = Rockchip_OSAL_dlopen((OMX_STRING)rockchip_component->libName, RTLD_NOW);
    if (!libHandle) {
        ret = OMX_ErrorInvalidComponentName;
        omx_err("OMX_ErrorInvalidComponentName, Line:%d", __LINE__);
        goto EXIT;
    }

    Rockchip_OMX_ComponentConstructor = Rockchip_OSAL_dlsym(libHandle, "Rockchip_OMX_ComponentConstructor");
    if (!Rockchip_OMX_ComponentConstructor) {
        Rockchip_OSAL_dlclose(libHandle);
        ret = OMX_ErrorInvalidComponent;
        omx_err("OMX_ErrorInvalidComponent, Line:%d", __LINE__);
        goto EXIT;
    }

    pOMXComponent = (OMX_COMPONENTTYPE *)Rockchip_OSAL_Malloc(sizeof(OMX_COMPONENTTYPE));
    INIT_SET_SIZE_VERSION(pOMXComponent, OMX_COMPONENTTYPE);
    ret = (*Rockchip_OMX_ComponentConstructor)((OMX_HANDLETYPE)pOMXComponent, (OMX_STRING)rockchip_component->componentName);
    if (ret != OMX_ErrorNone) {
        Rockchip_OSAL_Free(pOMXComponent);
        Rockchip_OSAL_dlclose(libHandle);
        ret = OMX_ErrorInvalidComponent;
        omx_err("OMX_ErrorInvalidComponent, Line:%d", __LINE__);
        goto EXIT;
    } else {
        if (Rockchip_OMX_ComponentAPICheck(pOMXComponent) != OMX_ErrorNone) {
            if (NULL != pOMXComponent->ComponentDeInit)
                pOMXComponent->ComponentDeInit(pOMXComponent);
            Rockchip_OSAL_Free(pOMXComponent);
            Rockchip_OSAL_dlclose(libHandle);
            ret = OMX_ErrorInvalidComponent;
            omx_err("OMX_ErrorInvalidComponent, Line:%d", __LINE__);
            goto EXIT;
        }
        rockchip_component->libHandle = libHandle;
        rockchip_component->pOMXComponent = pOMXComponent;
        rockchip_component->rkversion = OMX_COMPILE_INFO;
        ret = OMX_ErrorNone;
    }

EXIT:
    FunctionOut();

    return ret;
}

```

这个组件构造方法非常长。

1. 检查入参 hComponent 或 componentName 是否为 NULL；
2. 调用 Rockchip_OMX_Check_SizeVersion(…) 检查结构体大小和版本是否正常；
3. 调用 Rockchip_OMX_BaseComponent_Constructor(…) 构造 ROCKCHIP_OMX_BASECOMPONENT 结构；
4. 调用 Rockchip_OMX_Port_Constructor(…) 构造 Input 和 Output port；
5. 调用 Rockchip_OSAL_Malloc(…) 给 RKVPU_OMX_VIDEODEC_COMPONENT 结构分配内存；
6. 调用 Rockchip_OSAL_SharedMemory_Open() 开启共享内存；
7. pVideoDec 指向 RKVPU_OMX_VIDEODEC_COMPONENT 结构，对它进行各种字段赋值；
8. pRockchipComponent 指向 ROCKCHIP_OMX_BASECOMPONENT 结构，对它进行各种字段赋值；
9. pOMXComponent 指向 OMX_COMPONENTTYPE 对它的进行各种字段赋值。

hardware/rockchip/omx_il/component/video/dec/Rkvpu_OMX_Vdec.c

```
OMX_ERRORTYPE Rockchip_OMX_ComponentConstructor(OMX_HANDLETYPE hComponent, OMX_STRING componentName)
{
    OMX_ERRORTYPE          ret = OMX_ErrorNone;
    OMX_COMPONENTTYPE     *pOMXComponent = NULL;
    ROCKCHIP_OMX_BASECOMPONENT *pRockchipComponent = NULL;
    ROCKCHIP_OMX_BASEPORT      *pRockchipPort = NULL;
    RKVPU_OMX_VIDEODEC_COMPONENT *pVideoDec = NULL;

    FunctionIn();

    if ((hComponent == NULL) || (componentName == NULL)) {
        ret = OMX_ErrorBadParameter;
        omx_err("OMX_ErrorBadParameter, Line:%d", __LINE__);
        goto EXIT;
    }
    pOMXComponent = (OMX_COMPONENTTYPE *)hComponent;
    ret = Rockchip_OMX_Check_SizeVersion(pOMXComponent, sizeof(OMX_COMPONENTTYPE));
    if (ret != OMX_ErrorNone) {
        omx_err("OMX_Error, Line:%d", __LINE__);
        goto EXIT;
    }

    ret = Rockchip_OMX_BaseComponent_Constructor(pOMXComponent);
    if (ret != OMX_ErrorNone) {
        omx_err("OMX_Error, Line:%d", __LINE__);
        goto EXIT;
    }

    ret = Rockchip_OMX_Port_Constructor(pOMXComponent);
    if (ret != OMX_ErrorNone) {
        Rockchip_OMX_BaseComponent_Destructor(pOMXComponent);
        omx_err("OMX_Error, Line:%d", __LINE__);
        goto EXIT;
    }

    pRockchipComponent = (ROCKCHIP_OMX_BASECOMPONENT *)pOMXComponent->pComponentPrivate;

    pVideoDec = Rockchip_OSAL_Malloc(sizeof(RKVPU_OMX_VIDEODEC_COMPONENT));
    if (pVideoDec == NULL) {
        Rockchip_OMX_BaseComponent_Destructor(pOMXComponent);
        ret = OMX_ErrorInsufficientResources;
        omx_err("OMX_ErrorInsufficientResources, Line:%d", __LINE__);
        goto EXIT;
    }

    Rockchip_OSAL_Memset(pVideoDec, 0, sizeof(RKVPU_OMX_VIDEODEC_COMPONENT));
    pVideoDec->hSharedMemory = Rockchip_OSAL_SharedMemory_Open();
    if (pVideoDec->hSharedMemory == NULL) {
        omx_err("Rockchip_OSAL_SharedMemory_Open open fail");
    }

    pRockchipComponent->componentName = (OMX_STRING)Rockchip_OSAL_Malloc(MAX_OMX_COMPONENT_NAME_SIZE);
    if (pRockchipComponent->componentName == NULL) {
        Rockchip_OMX_ComponentDeInit(hComponent);
        ret = OMX_ErrorInsufficientResources;
        omx_err("OMX_ErrorInsufficientResources, Line:%d", __LINE__);
        goto EXIT;
    }
    Rockchip_OSAL_Memset(pRockchipComponent->componentName, 0, MAX_OMX_COMPONENT_NAME_SIZE);
    pVideoDec->nDpbSize = 0;
    pRockchipComponent->hComponentHandle = (OMX_HANDLETYPE)pVideoDec;

    pRockchipComponent->bSaveFlagEOS = OMX_FALSE;
    pRockchipComponent->nRkFlags = 0;
    pRockchipComponent->bBehaviorEOS = OMX_FALSE;
    pVideoDec->bDecSendEOS = OMX_FALSE;
    pVideoDec->bPvr_Flag = OMX_FALSE;
    pVideoDec->bFastMode = OMX_FALSE;

    pVideoDec->fp_in = NULL;
    pVideoDec->fp_out = NULL;
    pVideoDec->b4K_flags = OMX_FALSE;
    pVideoDec->codecProfile = 0;
    pVideoDec->power_fd = -1;
    pVideoDec->bIsPowerControl = OMX_FALSE;
    pVideoDec->bIsHevc = 0;
    pVideoDec->bIs10bit = OMX_FALSE;
    pRockchipComponent->bMultiThreadProcess = OMX_TRUE;
    pRockchipComponent->codecType = HW_VIDEO_DEC_CODEC;

    /* for debug */
    pVideoDec->bPrintFps = OMX_FALSE;
    pVideoDec->bPrintBufferPosition = OMX_FALSE;
    pVideoDec->bGtsMediaTest = OMX_FALSE;
    pVideoDec->nVdecDebug = 0;

    pVideoDec->bFirstFrame = OMX_TRUE;

    pVideoDec->vpumem_handle = NULL;

    /* Set componentVersion */
    pRockchipComponent->componentVersion.s.nVersionMajor = VERSIONMAJOR_NUMBER;
    pRockchipComponent->componentVersion.s.nVersionMinor = VERSIONMINOR_NUMBER;
    pRockchipComponent->componentVersion.s.nRevision     = REVISION_NUMBER;
    pRockchipComponent->componentVersion.s.nStep         = STEP_NUMBER;
    /* Set specVersion */
    pRockchipComponent->specVersion.s.nVersionMajor = VERSIONMAJOR_NUMBER;
    pRockchipComponent->specVersion.s.nVersionMinor = VERSIONMINOR_NUMBER;
    pRockchipComponent->specVersion.s.nRevision     = REVISION_NUMBER;
    pRockchipComponent->specVersion.s.nStep         = STEP_NUMBER;
    /* Input port */
    pRockchipPort = &pRockchipComponent->pRockchipPort[INPUT_PORT_INDEX];
    pRockchipPort->portDefinition.nBufferCountActual = MAX_VIDEO_INPUTBUFFER_NUM;
    pRockchipPort->portDefinition.nBufferCountMin = MAX_VIDEO_INPUTBUFFER_NUM;
    pRockchipPort->portDefinition.nBufferSize = 0;
    pRockchipPort->portDefinition.eDomain = OMX_PortDomainVideo;
    pRockchipPort->portDefinition.format.video.nFrameWidth = DEFAULT_FRAME_WIDTH;
    pRockchipPort->portDefinition.format.video.nFrameHeight = DEFAULT_FRAME_HEIGHT;
    pRockchipPort->portDefinition.format.video.nStride = 0; /*DEFAULT_FRAME_WIDTH;*/
    pRockchipPort->portDefinition.format.video.nSliceHeight = 0;
    pRockchipPort->portDefinition.nBufferSize = DEFAULT_VIDEO_INPUT_BUFFER_SIZE;
    pRockchipPort->portDefinition.format.video.eCompressionFormat = OMX_VIDEO_CodingUnused;

    pRockchipPort->portDefinition.format.video.cMIMEType =  Rockchip_OSAL_Malloc(MAX_OMX_MIMETYPE_SIZE);
    Rockchip_OSAL_Memset(pRockchipPort->portDefinition.format.video.cMIMEType, 0, MAX_OMX_MIMETYPE_SIZE);
    pRockchipPort->portDefinition.format.video.pNativeRender = 0;
    pRockchipPort->portDefinition.format.video.bFlagErrorConcealment = OMX_FALSE;
    pRockchipPort->portDefinition.format.video.eColorFormat = OMX_COLOR_FormatUnused;
    pRockchipPort->portDefinition.bEnabled = OMX_TRUE;
    pRockchipPort->portWayType = WAY2_PORT;
    /* Output port */
    pRockchipPort = &pRockchipComponent->pRockchipPort[OUTPUT_PORT_INDEX];
    pRockchipPort->portDefinition.nBufferCountActual = MAX_VIDEO_OUTPUTBUFFER_NUM;
    pRockchipPort->portDefinition.nBufferCountMin = MAX_VIDEO_OUTPUTBUFFER_NUM;
    pRockchipPort->portDefinition.nBufferSize = DEFAULT_VIDEO_OUTPUT_BUFFER_SIZE;
    pRockchipPort->portDefinition.eDomain = OMX_PortDomainVideo;
    pRockchipPort->portDefinition.format.video.nFrameWidth = DEFAULT_FRAME_WIDTH;
    pRockchipPort->portDefinition.format.video.nFrameHeight = DEFAULT_FRAME_HEIGHT;
    pRockchipPort->portDefinition.format.video.nStride = 0; /*DEFAULT_FRAME_WIDTH;*/
    pRockchipPort->portDefinition.format.video.nSliceHeight = 0;
    pRockchipPort->portDefinition.nBufferSize = DEFAULT_VIDEO_OUTPUT_BUFFER_SIZE;
    pRockchipPort->portDefinition.format.video.eCompressionFormat = OMX_VIDEO_CodingUnused;

    pRockchipPort->portDefinition.format.video.cMIMEType = Rockchip_OSAL_Malloc(MAX_OMX_MIMETYPE_SIZE);
    Rockchip_OSAL_Strcpy(pRockchipPort->portDefinition.format.video.cMIMEType, "raw/video");
    pRockchipPort->portDefinition.format.video.pNativeRender = 0;
    pRockchipPort->portDefinition.format.video.bFlagErrorConcealment = OMX_FALSE;
    pRockchipPort->portDefinition.format.video.eColorFormat = OMX_COLOR_FormatYUV420SemiPlanar;
    pRockchipPort->portDefinition.bEnabled = OMX_TRUE;
    pRockchipPort->portWayType = WAY2_PORT;
    pRockchipPort->portDefinition.eDomain = OMX_PortDomainVideo;
    pRockchipPort->bufferProcessType = BUFFER_COPY | BUFFER_ANBSHARE;

    pRockchipPort->processData.extInfo = (OMX_PTR)Rockchip_OSAL_Malloc(sizeof(DECODE_CODEC_EXTRA_BUFFERINFO));
    Rockchip_OSAL_Memset(((char *)pRockchipPort->processData.extInfo), 0, sizeof(DECODE_CODEC_EXTRA_BUFFERINFO));
    {
        DECODE_CODEC_EXTRA_BUFFERINFO *pBufferInfo = NULL;
        pBufferInfo = (DECODE_CODEC_EXTRA_BUFFERINFO *)(pRockchipPort->processData.extInfo);
    }
    pOMXComponent->UseBuffer              = &Rkvpu_OMX_UseBuffer;
    pOMXComponent->AllocateBuffer         = &Rkvpu_OMX_AllocateBuffer;
    pOMXComponent->FreeBuffer             = &Rkvpu_OMX_FreeBuffer;
    pOMXComponent->ComponentTunnelRequest = &Rkvpu_OMX_ComponentTunnelRequest;
    pOMXComponent->GetParameter           = &Rkvpu_OMX_GetParameter;
    pOMXComponent->SetParameter           = &Rkvpu_OMX_SetParameter;
    pOMXComponent->GetConfig              = &Rkvpu_OMX_GetConfig;
    pOMXComponent->SetConfig              = &Rkvpu_OMX_SetConfig;
    pOMXComponent->GetExtensionIndex      = &Rkvpu_OMX_GetExtensionIndex;
    pOMXComponent->ComponentRoleEnum      = &Rkvpu_OMX_ComponentRoleEnum;
    pOMXComponent->ComponentDeInit        = &Rockchip_OMX_ComponentDeInit;

    pRockchipComponent->rockchip_codec_componentInit      = &Rkvpu_Dec_ComponentInit;
    pRockchipComponent->rockchip_codec_componentTerminate = &Rkvpu_Dec_Terminate;

    pRockchipComponent->rockchip_AllocateTunnelBuffer = &Rkvpu_OMX_AllocateTunnelBuffer;
    pRockchipComponent->rockchip_FreeTunnelBuffer     = &Rkvpu_OMX_FreeTunnelBuffer;
    pRockchipComponent->rockchip_BufferProcessCreate    = &Rkvpu_OMX_BufferProcess_Create;
    pRockchipComponent->rockchip_BufferProcessTerminate = &Rkvpu_OMX_BufferProcess_Terminate;
    pRockchipComponent->rockchip_BufferFlush          = &Rkvpu_OMX_BufferFlush;

    pRockchipPort = &pRockchipComponent->pRockchipPort[INPUT_PORT_INDEX];
    if (!strcmp(componentName, RK_OMX_COMPONENT_H264_DEC)) {
        Rockchip_OSAL_Memset(pRockchipPort->portDefinition.format.video.cMIMEType, 0, MAX_OMX_MIMETYPE_SIZE);
        Rockchip_OSAL_Strcpy(pRockchipPort->portDefinition.format.video.cMIMEType, "video/avc");
        pVideoDec->codecId = OMX_VIDEO_CodingAVC;
        pRockchipPort->portDefinition.format.video.eCompressionFormat = OMX_VIDEO_CodingAVC;
    }
    ......
    {
        int gpu_fd = -1;
        gpu_fd = open("/dev/pvrsrvkm", O_RDWR, 0);
        if (gpu_fd > 0) {
            pVideoDec->bPvr_Flag = OMX_TRUE;
            close(gpu_fd);
        }
    }

    strcpy(pRockchipComponent->componentName, componentName);

    Rkvpu_OMX_DebugSwitchfromPropget(pRockchipComponent);

    pRockchipComponent->currentState = OMX_StateLoaded;
EXIT:
    FunctionOut();

    return ret;
}
100101102103104105106107108109110111112113114115116117118119120121122123124125126127128129130131132133134135136137138139140141142143144145146147148149150151152153154155156157158159160161162163164165166167168169170171172173174175176177178179180181182183184185186187188189190191192193194195196
```

Rockchip_OMX_Check_SizeVersion(…) 实现在 Rockchip_OMX_Basecomponent.c 中，首先从结构体头部取出 32 位长度做比较，然后再去检查版本是否匹配。

hardware/rockchip/omx_il/component/common/Rockchip_OMX_Basecomponent.c

```
OMX_ERRORTYPE Rockchip_OMX_Check_SizeVersion(OMX_PTR header, OMX_U32 size)
{
    OMX_ERRORTYPE ret = OMX_ErrorNone;

    OMX_VERSIONTYPE* version = NULL;
    if (header == NULL) {
        ret = OMX_ErrorBadParameter;
        goto EXIT;
    }
    version = (OMX_VERSIONTYPE*)((char*)header + sizeof(OMX_U32));
    if (*((OMX_U32*)header) != size) {
        ret = OMX_ErrorBadParameter;
        goto EXIT;
    }
    if (version->s.nVersionMajor != VERSIONMAJOR_NUMBER ||
        version->s.nVersionMinor != VERSIONMINOR_NUMBER) {
        ret = OMX_ErrorVersionMismatch;
        goto EXIT;
    }
    ret = OMX_ErrorNone;
EXIT:
    return ret;
}

```

Rockchip_OMX_BaseComponent_Constructor(…) 方法内部分配了 ROCKCHIP_OMX_BASECOMPONENT 结构内存，并调用 Rockchip_OSAL_Memset(…) 对结构体进行清零，然后对其进行了初始化。注意 pOMXComponent->SetCallbacks 赋值为 Rockchip_OMX_SetCallbacks 函数的地址，后面分析会用到。

hardware/rockchip/omx_il/component/common/Rockchip_OMX_Basecomponent.c

```
OMX_ERRORTYPE Rockchip_OMX_BaseComponent_Constructor(
    OMX_IN OMX_HANDLETYPE hComponent)
{
    OMX_ERRORTYPE             ret = OMX_ErrorNone;
    OMX_COMPONENTTYPE        *pOMXComponent;
    ROCKCHIP_OMX_BASECOMPONENT *pRockchipComponent = NULL;

    FunctionIn();

    if (hComponent == NULL) {
        ret = OMX_ErrorBadParameter;
        omx_err("OMX_ErrorBadParameter, Line:%d", __LINE__);
        goto EXIT;
    }
    pOMXComponent = (OMX_COMPONENTTYPE *)hComponent;
    pRockchipComponent = Rockchip_OSAL_Malloc(sizeof(ROCKCHIP_OMX_BASECOMPONENT));
    if (pRockchipComponent == NULL) {
        ret = OMX_ErrorInsufficientResources;
        omx_err("OMX_ErrorInsufficientResources, Line:%d", __LINE__);
        goto EXIT;
    }
    Rockchip_OSAL_Memset(pRockchipComponent, 0, sizeof(ROCKCHIP_OMX_BASECOMPONENT));
    pRockchipComponent->rkversion = OMX_COMPILE_INFO;
    pOMXComponent->pComponentPrivate = (OMX_PTR)pRockchipComponent;

    ret = Rockchip_OSAL_SemaphoreCreate(&pRockchipComponent->msgSemaphoreHandle);
    if (ret != OMX_ErrorNone) {
        ret = OMX_ErrorInsufficientResources;
        omx_err("OMX_ErrorInsufficientResources, Line:%d", __LINE__);
        goto EXIT;
    }
    ret = Rockchip_OSAL_MutexCreate(&pRockchipComponent->compMutex);
    if (ret != OMX_ErrorNone) {
        ret = OMX_ErrorInsufficientResources;
        omx_err("OMX_ErrorInsufficientResources, Line:%d", __LINE__);
        goto EXIT;
    }
    ret = Rockchip_OSAL_SignalCreate(&pRockchipComponent->abendStateEvent);
    if (ret != OMX_ErrorNone) {
        ret = OMX_ErrorInsufficientResources;
        omx_err("OMX_ErrorInsufficientResources, Line:%d", __LINE__);
        goto EXIT;
    }

    pRockchipComponent->bExitMessageHandlerThread = OMX_FALSE;
    Rockchip_OSAL_QueueCreate(&pRockchipComponent->messageQ, MAX_QUEUE_ELEMENTS);
    ret = Rockchip_OSAL_ThreadCreate(&pRockchipComponent->hMessageHandler,
                                     Rockchip_OMX_MessageHandlerThread,
                                     pOMXComponent,
                                     "omx_msg_hdl");
    if (ret != OMX_ErrorNone) {
        ret = OMX_ErrorInsufficientResources;
        omx_err("OMX_ErrorInsufficientResources, Line:%d", __LINE__);
        goto EXIT;
    }

    pRockchipComponent->bMultiThreadProcess = OMX_FALSE;

    pOMXComponent->GetComponentVersion = &Rockchip_OMX_GetComponentVersion;
    pOMXComponent->SendCommand         = &Rockchip_OMX_SendCommand;
    pOMXComponent->GetState            = &Rockchip_OMX_GetState;
    pOMXComponent->SetCallbacks        = &Rockchip_OMX_SetCallbacks;
    pOMXComponent->UseEGLImage         = &Rockchip_OMX_UseEGLImage;

EXIT:
    FunctionOut();

    return ret;
}

```

调用 Rockchip_OMX_Port_Constructor(…) 构造 Input 和 Output port，可以看到主要对 ROCKCHIP_OMX_BASEPORT 结构进行了内存分配，ALL_PORT_NUM = 2，所以一共分配了两个元素。然后对 Input 和 Output port 进行了初始化。INPUT_PORT_INDEX 定义为 0，而 OUTPUT_PORT_INDEX 定义为 1。

hardware/rockchip/omx_il/component/common/Rockchip_OMX_Baseport.c

```
OMX_ERRORTYPE Rockchip_OMX_Port_Constructor(OMX_HANDLETYPE hComponent)
{
    OMX_ERRORTYPE          ret = OMX_ErrorNone;
    OMX_COMPONENTTYPE     *pOMXComponent = NULL;
    ROCKCHIP_OMX_BASECOMPONENT *pRockchipComponent = NULL;
    ROCKCHIP_OMX_BASEPORT      *pRockchipPort = NULL;
    ROCKCHIP_OMX_BASEPORT      *pRockchipInputPort = NULL;
    ROCKCHIP_OMX_BASEPORT      *pRockchipOutputPort = NULL;

    FunctionIn();

    if (hComponent == NULL) {
        ret = OMX_ErrorBadParameter;
        omx_err("OMX_ErrorBadParameter, Line:%d", __LINE__);
        goto EXIT;
    }
    pOMXComponent = (OMX_COMPONENTTYPE *)hComponent;
    ret = Rockchip_OMX_Check_SizeVersion(pOMXComponent, sizeof(OMX_COMPONENTTYPE));
    if (ret != OMX_ErrorNone) {
        goto EXIT;
    }

    if (pOMXComponent->pComponentPrivate == NULL) {
        ret = OMX_ErrorBadParameter;
        omx_err("OMX_ErrorBadParameter, Line:%d", __LINE__);
        goto EXIT;
    }
    pRockchipComponent = (ROCKCHIP_OMX_BASECOMPONENT *)pOMXComponent->pComponentPrivate;

    INIT_SET_SIZE_VERSION(&pRockchipComponent->portParam, OMX_PORT_PARAM_TYPE);
    pRockchipComponent->portParam.nPorts = ALL_PORT_NUM;
    pRockchipComponent->portParam.nStartPortNumber = INPUT_PORT_INDEX;

    pRockchipPort = Rockchip_OSAL_Malloc(sizeof(ROCKCHIP_OMX_BASEPORT) * ALL_PORT_NUM);
    if (pRockchipPort == NULL) {
        ......
    }
    Rockchip_OSAL_Memset(pRockchipPort, 0, sizeof(ROCKCHIP_OMX_BASEPORT) * ALL_PORT_NUM);
    pRockchipComponent->pRockchipPort = pRockchipPort;

    /* Input Port */
    pRockchipInputPort = &pRockchipPort[INPUT_PORT_INDEX];

    Rockchip_OSAL_QueueCreate(&pRockchipInputPort->bufferQ, MAX_QUEUE_ELEMENTS);
    Rockchip_OSAL_QueueCreate(&pRockchipInputPort->securebufferQ, MAX_QUEUE_ELEMENTS);

    pRockchipInputPort->extendBufferHeader = Rockchip_OSAL_Malloc(sizeof(ROCKCHIP_OMX_BUFFERHEADERTYPE) * MAX_BUFFER_NUM);
    if (pRockchipInputPort->extendBufferHeader == NULL) {
        ......
    }
    Rockchip_OSAL_Memset(pRockchipInputPort->extendBufferHeader, 0, sizeof(ROCKCHIP_OMX_BUFFERHEADERTYPE) * MAX_BUFFER_NUM);

    pRockchipInputPort->bufferStateAllocate = Rockchip_OSAL_Malloc(sizeof(OMX_U32) * MAX_BUFFER_NUM);
    if (pRockchipInputPort->bufferStateAllocate == NULL) {
        ......
    }
    Rockchip_OSAL_Memset(pRockchipInputPort->bufferStateAllocate, 0, sizeof(OMX_U32) * MAX_BUFFER_NUM);

    pRockchipInputPort->bufferSemID = NULL;
    pRockchipInputPort->assignedBufferNum = 0;
    pRockchipInputPort->portState = OMX_StateMax;
    pRockchipInputPort->bIsPortFlushed = OMX_FALSE;
    pRockchipInputPort->bIsPortDisabled = OMX_FALSE;
    pRockchipInputPort->tunneledComponent = NULL;
    pRockchipInputPort->tunneledPort = 0;
    pRockchipInputPort->tunnelBufferNum = 0;
    pRockchipInputPort->bufferSupplier = OMX_BufferSupplyUnspecified;
    pRockchipInputPort->tunnelFlags = 0;
    ret = Rockchip_OSAL_SemaphoreCreate(&pRockchipInputPort->loadedResource);
    if (ret != OMX_ErrorNone) {
        ......
    }
    ret = Rockchip_OSAL_SemaphoreCreate(&pRockchipInputPort->unloadedResource);
    if (ret != OMX_ErrorNone) {
        ......
    }

    INIT_SET_SIZE_VERSION(&pRockchipInputPort->portDefinition, OMX_PARAM_PORTDEFINITIONTYPE);
    pRockchipInputPort->portDefinition.nPortIndex = INPUT_PORT_INDEX;
    pRockchipInputPort->portDefinition.eDir = OMX_DirInput;
    pRockchipInputPort->portDefinition.nBufferCountActual = 0;
    pRockchipInputPort->portDefinition.nBufferCountMin = 0;
    pRockchipInputPort->portDefinition.nBufferSize = 0;
    pRockchipInputPort->portDefinition.bEnabled = OMX_FALSE;
    pRockchipInputPort->portDefinition.bPopulated = OMX_FALSE;
    pRockchipInputPort->portDefinition.eDomain = OMX_PortDomainMax;
    pRockchipInputPort->portDefinition.bBuffersContiguous = OMX_FALSE;
    pRockchipInputPort->portDefinition.nBufferAlignment = 0;
    pRockchipInputPort->markType.hMarkTargetComponent = NULL;
    pRockchipInputPort->markType.pMarkData = NULL;
    pRockchipInputPort->exceptionFlag = GENERAL_STATE;

    /* Output Port */
    pRockchipOutputPort = &pRockchipPort[OUTPUT_PORT_INDEX];

    Rockchip_OSAL_QueueCreate(&pRockchipOutputPort->bufferQ, MAX_QUEUE_ELEMENTS); /* For in case of "Output Buffer Share", MAX ELEMENTS(DPB + EDPB) */

    pRockchipOutputPort->extendBufferHeader = Rockchip_OSAL_Malloc(sizeof(ROCKCHIP_OMX_BUFFERHEADERTYPE) * MAX_BUFFER_NUM);
    if (pRockchipOutputPort->extendBufferHeader == NULL) {
        ......
    }
    Rockchip_OSAL_Memset(pRockchipOutputPort->extendBufferHeader, 0, sizeof(ROCKCHIP_OMX_BUFFERHEADERTYPE) * MAX_BUFFER_NUM);

    pRockchipOutputPort->bufferStateAllocate = Rockchip_OSAL_Malloc(sizeof(OMX_U32) * MAX_BUFFER_NUM);
    if (pRockchipOutputPort->bufferStateAllocate == NULL) {
        ......
    }
    Rockchip_OSAL_Memset(pRockchipOutputPort->bufferStateAllocate, 0, sizeof(OMX_U32) * MAX_BUFFER_NUM);

    pRockchipOutputPort->bufferSemID = NULL;
    pRockchipOutputPort->assignedBufferNum = 0;
    pRockchipOutputPort->portState = OMX_StateMax;
    pRockchipOutputPort->bIsPortFlushed = OMX_FALSE;
    pRockchipOutputPort->bIsPortDisabled = OMX_FALSE;
    pRockchipOutputPort->tunneledComponent = NULL;
    pRockchipOutputPort->tunneledPort = 0;
    pRockchipOutputPort->tunnelBufferNum = 0;
    pRockchipOutputPort->bufferSupplier = OMX_BufferSupplyUnspecified;
    pRockchipOutputPort->tunnelFlags = 0;
    ret = Rockchip_OSAL_SemaphoreCreate(&pRockchipOutputPort->loadedResource);
    if (ret != OMX_ErrorNone) {
        ......
    }
    ret = Rockchip_OSAL_SignalCreate(&pRockchipOutputPort->unloadedResource);
    if (ret != OMX_ErrorNone) {
        ......
    }

    INIT_SET_SIZE_VERSION(&pRockchipOutputPort->portDefinition, OMX_PARAM_PORTDEFINITIONTYPE);
    pRockchipOutputPort->portDefinition.nPortIndex = OUTPUT_PORT_INDEX;
    pRockchipOutputPort->portDefinition.eDir = OMX_DirOutput;
    pRockchipOutputPort->portDefinition.nBufferCountActual = 0;
    pRockchipOutputPort->portDefinition.nBufferCountMin = 0;
    pRockchipOutputPort->portDefinition.nBufferSize = 0;
    pRockchipOutputPort->portDefinition.bEnabled = OMX_FALSE;
    pRockchipOutputPort->portDefinition.bPopulated = OMX_FALSE;
    pRockchipOutputPort->portDefinition.eDomain = OMX_PortDomainMax;
    pRockchipOutputPort->portDefinition.bBuffersContiguous = OMX_FALSE;
    pRockchipOutputPort->portDefinition.nBufferAlignment = 0;
    pRockchipOutputPort->markType.hMarkTargetComponent = NULL;
    pRockchipOutputPort->markType.pMarkData = NULL;
    pRockchipOutputPort->exceptionFlag = GENERAL_STATE;

    pRockchipComponent->checkTimeStamp.needSetStartTimeStamp = OMX_FALSE;
    pRockchipComponent->checkTimeStamp.needCheckStartTimeStamp = OMX_FALSE;
    pRockchipComponent->checkTimeStamp.startTimeStamp = 0;
    pRockchipComponent->checkTimeStamp.nStartFlags = 0x0;

    pOMXComponent->EmptyThisBuffer = &Rockchip_OMX_EmptyThisBuffer;
    pOMXComponent->FillThisBuffer  = &Rockchip_OMX_FillThisBuffer;

    ret = OMX_ErrorNone;
EXIT:
    FunctionOut();

    return ret;
}
100101102103104105106107108109110111112113114115116117118119120121122123124125126127128129130131132133134135136137138139140141142143144145146147148149150151152153154155156
```

最后来分析一波 Rockchip_OSAL_SharedMemory_Open()，调用 access(…) 检查是否可以打开 /dev/dri/card0 设备，否则打开 /dev/ion 设备。可以看到前一种属于 DRM，而后一种属于 ION。

hardware/rockchip/omx_il/osal/Rockchip_OSAL_SharedMemory.c

```
OMX_HANDLETYPE Rockchip_OSAL_SharedMemory_Open()
{
    ROCKCHIP_SHARED_MEMORY *pHandle = NULL;
    int  IONClient = 0;

    pHandle = (ROCKCHIP_SHARED_MEMORY *)malloc(sizeof(ROCKCHIP_SHARED_MEMORY));
    Rockchip_OSAL_Memset(pHandle, 0, sizeof(ROCKCHIP_SHARED_MEMORY));
    if (pHandle == NULL)
        goto EXIT;
    if (!access("/dev/dri/card0", F_OK)) {
        IONClient = open("/dev/dri/card0", O_RDWR);
        mem_type = MEMORY_TYPE_DRM;
    } else {
        IONClient = open("/dev/ion", O_RDWR);
        mem_type = MEMORY_TYPE_ION;
    }

    if (IONClient <= 0) {
        omx_err("ion_client_create Error: %d", IONClient);
        Rockchip_OSAL_Free((void *)pHandle);
        pHandle = NULL;
        goto EXIT;
    }

    pHandle->fd = IONClient;

    Rockchip_OSAL_MutexCreate(&pHandle->hSMMutex);

EXIT:
    return (OMX_HANDLETYPE)pHandle;
}

```

回到 Rockchip_OMX_ComponentLoad(…) 函数， Rockchip_OMX_ComponentAPICheck(…) 实现非常简单，检查 OMX_COMPONENTTYPE 一系列字段是否赋值，只要有一个为 NULL，就返回 OMX_ErrorInvalidComponent 错误。

hardware/rockchip/omx_il/core/Rockchip_OMX_Component_Register.c

```
OMX_ERRORTYPE Rockchip_OMX_ComponentAPICheck(OMX_COMPONENTTYPE *component)
{
    OMX_ERRORTYPE ret = OMX_ErrorNone;

    if ((NULL == component->GetComponentVersion)    ||
        (NULL == component->SendCommand)            ||
        (NULL == component->GetParameter)           ||
        (NULL == component->SetParameter)           ||
        (NULL == component->GetConfig)              ||
        (NULL == component->SetConfig)              ||
        (NULL == component->GetExtensionIndex)      ||
        (NULL == component->GetState)               ||
        (NULL == component->ComponentTunnelRequest) ||
        (NULL == component->UseBuffer)              ||
        (NULL == component->AllocateBuffer)         ||
        (NULL == component->FreeBuffer)             ||
        (NULL == component->EmptyThisBuffer)        ||
        (NULL == component->FillThisBuffer)         ||
        (NULL == component->SetCallbacks)           ||
        (NULL == component->ComponentDeInit)        ||
        (NULL == component->UseEGLImage)            ||
        (NULL == component->ComponentRoleEnum))
        ret = OMX_ErrorInvalidComponent;
    else
        ret = OMX_ErrorNone;

    return ret;
}

```

回到 RKOMX_GetHandle(…) 方法，现在来分析调用 ROCKCHIP_OMX_COMPONENT 结构内的 pOMXComponent 字段 SetCallbacks(…) 方法，由上面的分析可以得出实际会调到 Rockchip_OMX_Basecomponent.c 中的 Rockchip_OMX_SetCallbacks 函数。

只是做了一些参数检查，然后给 ROCKCHIP_OMX_BASECOMPONENT pCallbacks 和 callbackData 字段进行了赋值。

hardware/rockchip/omx_il/component/common/Rockchip_OMX_Basecomponent.c

```
OMX_ERRORTYPE Rockchip_OMX_SetCallbacks (
    OMX_IN OMX_HANDLETYPE    hComponent,
    OMX_IN OMX_CALLBACKTYPE* pCallbacks,
    OMX_IN OMX_PTR           pAppData)
{
    OMX_ERRORTYPE             ret = OMX_ErrorNone;
    OMX_COMPONENTTYPE        *pOMXComponent = NULL;
    ROCKCHIP_OMX_BASECOMPONENT *pRockchipComponent = NULL;

    FunctionIn();

    if (hComponent == NULL) {
        ret = OMX_ErrorBadParameter;

        omx_err("OMX_ErrorBadParameter :%d", __LINE__);
        goto EXIT;
    }
    pOMXComponent = (OMX_COMPONENTTYPE *)hComponent;
    ret = Rockchip_OMX_Check_SizeVersion(pOMXComponent, sizeof(OMX_COMPONENTTYPE));
    if (ret != OMX_ErrorNone) {

        omx_err("OMX_ErrorNone :%d", __LINE__);
        goto EXIT;
    }

    if (pOMXComponent->pComponentPrivate == NULL) {

        omx_err("OMX_ErrorBadParameter :%d", __LINE__);
        ret = OMX_ErrorBadParameter;
        goto EXIT;
    }
    pRockchipComponent = (ROCKCHIP_OMX_BASECOMPONENT *)pOMXComponent->pComponentPrivate;

    if (pCallbacks == NULL) {
        omx_err("OMX_ErrorBadParameter :%d", __LINE__);
        ret = OMX_ErrorBadParameter;
        goto EXIT;
    }
    if (pRockchipComponent->currentState == OMX_StateInvalid) {

        omx_err("OMX_ErrorInvalidState :%d", __LINE__);
        ret = OMX_ErrorInvalidState;
        goto EXIT;
    }
    if (pRockchipComponent->currentState != OMX_StateLoaded) {
        omx_err("OMX_StateLoaded :%d", __LINE__);
        ret = OMX_ErrorIncorrectStateOperation;
        goto EXIT;
    }

    pRockchipComponent->pCallbacks = pCallbacks;
    pRockchipComponent->callbackData = pAppData;

    ret = OMX_ErrorNone;

EXIT:
    FunctionOut();

    return ret;
}

```

