# 

Codecservice 主要负责 HAL 层给 framework 层提供调用音视频编[解码](https://so.csdn.net/so/search?q=%E8%A7%A3%E7%A0%81&spm=1001.2101.3001.7020)接口。它的入口是 main_codecservice.cpp。

首先我们来看下启动它的配置 rc 脚本。可以看到它放在 vendor 目录下，说明和供应商有关。

frameworks/av/services/mediacodec/android.hardware.media.omx@1.0-service.rc

```rc
service vendor.media.omx /vendor/bin/hw/android.hardware.media.omx@1.0-service
    class main
    user mediacodec
    group camera drmrpc mediadrm
    ioprio rt 4
    writepid /dev/cpuset/foreground/tasks
```

1. 新建 implementation::Omx 对象并启动 Omx 服务；
2. 新建 implementation::OmxStore 对象并启动 OmxStore 服务。  frameworks/av/services/mediacodec/main_codecservice.cpp

    ![启动流程.png](启动流程.png)

```cpp
int main(int argc __unused, char** argv)
{
    strcpy(argv[0], "media.codec");
    LOG(INFO) << "mediacodecservice starting";
    signal(SIGPIPE, SIG_IGN);
    SetUpMinijail(kSystemSeccompPolicyPath, kVendorSeccompPolicyPath);

    android::ProcessState::initWithDriver("/dev/vndbinder");
    android::ProcessState::self()->startThreadPool();

    ::android::hardware::configureRpcThreadpool(64, false);

    // Default codec services
    using namespace ::android::hardware::media::omx::V1_0;
    sp<IOmx> omx = new implementation::Omx();
    if (omx == nullptr) {
        LOG(ERROR) << "Cannot create IOmx HAL service.";
    } else if (omx->registerAsService() != OK) {
        LOG(ERROR) << "Cannot register IOmx HAL service.";
    } else {
        LOG(INFO) << "IOmx HAL service created.";
    }
    sp<IOmxStore> omxStore = new implementation::OmxStore(omx);
    if (omxStore == nullptr) {
        LOG(ERROR) << "Cannot create IOmxStore HAL service.";
    } else if (omxStore->registerAsService() != OK) {
        LOG(ERROR) << "Cannot register IOmxStore HAL service.";
    }

    ::android::hardware::joinRpcThreadpool();
}
```

新建 implementation::Omx 对象详细情况可参考《[Android 10 源码 Omx 初始化](https://blog.csdn.net/tyyj90/article/details/120912668?spm=1001.2014.3001.5501)》。

接下来重点看新建 implementation::OmxStore 对象，从 OmxStore 头文件不难看出其构造函数里面很多参数存在默认值，这也就是为什么上一步只传入一个指向 IOmx 指针。

1. 检索 omx 节点列表，最终通过调用 Omx listNodes(…) 完成，获取 node 名称 Set，listNodes(…) 函数形参是一个匿名函数；
2. 解析 xml，这一步和 Omx 构造函数中的部分流程一样，具体可以查看 Omx 初始化一节，个人认为这一步并非必须的，因为 Omx 构造发生在 OmxStore 构造之前，不过这就需要 Omx 中开放这些接口给 OmxStore，目前代码逻辑却并非如此，所以这个步骤没办法直接省略，除非修改代码接口和流程加以适配；
3. 从 MediaCodecsXmlParser 中获取 ServiceAttributeMap 填充 ServiceAttribute List（mServiceAttributeList）；
4. 从 MediaCodecsXmlParser 中获取 RoleMap 填充 Role List（mRoleList），其中构造 RoleInfo，获取 NodeInfo List，这里第二次 resize NodeInfo List 是因为可能存在不匹配第一步中 node 名称 Set 中的情况，这样 List 就需要缩小；
5. 调用 MediaCodecsXmlParser getCommonPrefix() 获取 Codec Map key 中都存在的前缀。

frameworks/av/media/libstagefright/omx/1.0/OmxStore.cpp

```
OmxStore::OmxStore(
        const sp<IOmx> &omx,
        const char* owner,
        const std::vector<std::string> &searchDirs,
        const std::vector<std::string> &xmlNames,
        const char* profilingResultsXmlPath) {
    // retrieve list of omx nodes
    std::set<std::string> nodes;
    if (omx != nullptr) {
        omx->listNodes([&nodes](const Status &status,
                                const hidl_vec<IOmx::ComponentInfo> &nodeList) {
            if (status == Status::OK) {
                for (const IOmx::ComponentInfo& info : nodeList) {
                    nodes.emplace(info.mName.c_str());
                }
            }
        });
    }

    MediaCodecsXmlParser parser;
    parser.parseXmlFilesInSearchDirs(xmlNames, searchDirs);
    if (profilingResultsXmlPath != nullptr) {
        parser.parseXmlPath(profilingResultsXmlPath);
    }
    mParsingStatus = toStatus(parser.getParsingStatus());

    const auto& serviceAttributeMap = parser.getServiceAttributeMap();
    mServiceAttributeList.resize(serviceAttributeMap.size());
    size_t i = 0;
    for (const auto& attributePair : serviceAttributeMap) {
        ServiceAttribute attribute;
        attribute.key = attributePair.first;
        attribute.value = attributePair.second;
        mServiceAttributeList[i] = std::move(attribute);
        ++i;
    }

    const auto& roleMap = parser.getRoleMap();
    mRoleList.resize(roleMap.size());
    i = 0;
    for (const auto& rolePair : roleMap) {
        RoleInfo role;
        role.role = rolePair.first;
        role.type = rolePair.second.type;
        role.isEncoder = rolePair.second.isEncoder;
        role.preferPlatformNodes = false; // deprecated and ignored, using rank instead
        hidl_vec<NodeInfo>& nodeList = role.nodes;
        nodeList.resize(rolePair.second.nodeList.size());
        size_t j = 0;
        for (const auto& nodePair : rolePair.second.nodeList) {
            if (!nodes.count(nodePair.second.name)) {
                // not supported by this OMX instance
                if (!strncasecmp(nodePair.second.name.c_str(), "omx.", 4)) {
                    LOG(INFO) << "node [" << nodePair.second.name.c_str() << "] not found in IOmx";
                }
                continue;
            }
            NodeInfo node;
            node.name = nodePair.second.name;
            node.owner = owner;
            hidl_vec<NodeAttribute>& attributeList = node.attributes;
            attributeList.resize(nodePair.second.attributeList.size());
            size_t k = 0;
            for (const auto& attributePair : nodePair.second.attributeList) {
                NodeAttribute attribute;
                attribute.key = attributePair.first;
                attribute.value = attributePair.second;
                attributeList[k] = std::move(attribute);
                ++k;
            }
            nodeList[j] = std::move(node);
            ++j;
        }
        nodeList.resize(j);
        mRoleList[i] = std::move(role);
        ++i;
    }

    mPrefix = parser.getCommonPrefix();
}
```

为了进一步确认解析 xml 使用的参数，我们需要查看 OmxStore.h 文件来确认。从 OmxStore 构造函数声明中不难发现，其所有参数都存在默认值，也就是在只传入指向 IOmx 的指针时，其他参数都已确定。

owner = “default”

searchDirs = MediaCodecsXmlParser::getDefaultSearchDirs()

xmlFiles = MediaCodecsXmlParser::getDefaultXmlNames()

xmlProfilingResultsPath = MediaCodecsXmlParser::defaultProfilingResultsXmlPath)

以上的三个 xml 解析的相关参数和 Omx 构造函数中解析 xml 流程入参是一样的。

frameworks/av/media/libstagefright/omx/include/media/stagefright/omx/1.0/OmxStore.h

```
namespace implementation {

using ::android::hardware::media::omx::V1_0::IOmxStore;
using ::android::hardware::media::omx::V1_0::IOmx;
using ::android::hardware::media::omx::V1_0::Status;
using ::android::hidl::base::V1_0::IBase;
using ::android::hardware::hidl_string;
using ::android::hardware::hidl_vec;
using ::android::hardware::Return;
using ::android::hardware::Void;
using ::android::sp;
using ::android::wp;

struct OmxStore : public IOmxStore {
    OmxStore(
            const sp<IOmx> &omx = nullptr,
            const char* owner = "default",
            const std::vector<std::string> &searchDirs =
                MediaCodecsXmlParser::getDefaultSearchDirs(),
            const std::vector<std::string> &xmlFiles =
                MediaCodecsXmlParser::getDefaultXmlNames(),
            const char *xmlProfilingResultsPath =
                MediaCodecsXmlParser::defaultProfilingResultsXmlPath);

    virtual ~OmxStore();

    // Methods from IOmxStore
    Return<void> listServiceAttributes(listServiceAttributes_cb) override;
    Return<void> getNodePrefix(getNodePrefix_cb) override;
    Return<void> listRoles(listRoles_cb) override;
    Return<sp<IOmx>> getOmx(hidl_string const&) override;

protected:
    Status mParsingStatus;
    hidl_string mPrefix;
    hidl_vec<ServiceAttribute> mServiceAttributeList;
    hidl_vec<RoleInfo> mRoleList;
};

}  // namespace implementation
}  // namespace V1_0
}  // namespace omx
}  // namespace media
}  // namespace hardware
}  // namespace android
```

listNodes(…) 内部主要调用 OMXMaster enumerateComponents(…) 函数进行枚举组件，具体可以查看 Omx 初始化一节关于 OMXMaster 构造函数的过程。枚举组件名称后，构建 ::android::IOMX::ComponentInfo 并将其推入 list 中，然后给其进行必要的字段初始化，这里会调到 OMXMaster getRolesOfComponent(…) 函数获取角色。最后将 std::list<::android::IOMX::ComponentInfo> 转化为 hidl_vec，接着调用 _hidl_cb(…) 回调前面提到的匿名函数。

frameworks/av/media/libstagefright/omx/1.0/Omx.cpp

```
Return<void> Omx::listNodes(listNodes_cb _hidl_cb) {
    std::list<::android::IOMX::ComponentInfo> list;
    char componentName[256];
    for (OMX_U32 index = 0;
            mMaster->enumerateComponents(
            componentName, sizeof(componentName), index) == OMX_ErrorNone;
            ++index) {
        list.push_back(::android::IOMX::ComponentInfo());
        ::android::IOMX::ComponentInfo& info = list.back();
        info.mName = componentName;
        ::android::Vector<::android::String8> roles;
        OMX_ERRORTYPE err =
                mMaster->getRolesOfComponent(componentName, &roles);
        if (err == OMX_ErrorNone) {
            for (OMX_U32 i = 0; i < roles.size(); ++i) {
                info.mRoles.push_back(roles[i]);
            }
        }
    }

    hidl_vec<ComponentInfo> tList;
    tList.resize(list.size());
    size_t i = 0;
    for (auto const& info : list) {
        convertTo(&(tList[i++]), info);
    }
    _hidl_cb(toStatus(OK), tList);
    return Void();
}
```

1. 先根据 name 在 PluginByComponentName 容器中找到 index；
2. 根据 index 查到对应项，然后获取 plugin；
3. 根据 name 调用 plugin OMXPluginBase 具体实现 getRolesOfComponent(…) 获取所有角色。

frameworks/av/media/libstagefright/omx/OMXMaster.cpp

```
OMX_ERRORTYPE OMXMaster::getRolesOfComponent(
        const char *name,
        Vector<String8> *roles) {
    Mutex::Autolock autoLock(mLock);

    roles->clear();

    ssize_t index = mPluginByComponentName.indexOfKey(String8(name));

    if (index < 0) {
        return OMX_ErrorInvalidComponentName;
    }

    OMXPluginBase *plugin = mPluginByComponentName.valueAt(index);
    return plugin->getRolesOfComponent(name, roles);
}
```

假设我们查找的角色是 rk3399 android 10 实现的，比如角色是 video_decoder.avc 时。这就会调用到 libstagefrighthw so 内的 getRolesOfComponent(…) 具体实现。

1. 调用 RKOMX_GetComponentsOfRole(…) 获取角色数量；
2. 调用 RKOMX_GetComponentsOfRole(…) 获取具体角色数组；
3. 转换角色数组到容器 Vector。

hardware/rockchip/librkvpu/libstagefrighthw/RKOMXPlugin.cpp

```
OMX_ERRORTYPE RKOMXPlugin::getRolesOfComponent(
        const char *name,
        Vector<String8> *roles) {
    roles->clear();
    for (OMX_U32 j = 0; j < mCores.size(); j++) {
        if (mCores[j]->mLibHandle == NULL) {
           continue;
        }

        OMX_U32 numRoles;
        OMX_ERRORTYPE err = (*(mCores[j]->mGetRolesOfComponentHandle))(
                const_cast<OMX_STRING>(name), &numRoles, NULL);

        if (err != OMX_ErrorNone) {
            continue;
        }

        if (numRoles > 0) {
            OMX_U8 **array = new OMX_U8 *[numRoles];
            for (OMX_U32 i = 0; i < numRoles; ++i) {
                array[i] = new OMX_U8[OMX_MAX_STRINGNAME_SIZE];
            }

            OMX_U32 numRoles2 = numRoles;
            err = (*(mCores[j]->mGetRolesOfComponentHandle))(
                    const_cast<OMX_STRING>(name), &numRoles2, array);

            CHECK_EQ(err, OMX_ErrorNone);
            CHECK_EQ(numRoles, numRoles2);

            for (OMX_U32 i = 0; i < numRoles; ++i) {
                String8 s((const char *)array[i]);
                roles->push(s);

                delete[] array[i];
                array[i] = NULL;
            }

            delete[] array;
            array = NULL;
        }
        return OMX_ErrorNone;
    }
    return OMX_ErrorInvalidComponent;
}
```

RKOMX_GetComponentsOfRole(…) 主要遍历所有组件，然后从全局组件列表 gComponentList 中查找角色（role）匹配项，如果入参 compNames 不为 NULL 则将查找到的组件名称写回，否则只返回匹配的组件个数。

hardware/rockchip/omx_il/core/Rockchip_OMX_Core.c

```
OMX_API OMX_ERRORTYPE RKOMX_GetComponentsOfRole (
    OMX_IN    OMX_STRING role,
    OMX_INOUT OMX_U32 *pNumComps,
    OMX_INOUT OMX_U8  **compNames)
{
    OMX_ERRORTYPE ret = OMX_ErrorNone;
    int           max_role_num = 0;
    int i = 0, j = 0;

    FunctionIn();

    if (gInitialized != 1) {
        ret = OMX_ErrorNotReady;
        goto EXIT;
    }

    *pNumComps = 0;

    for (i = 0; i < MAX_OMX_COMPONENT_NUM; i++) {
        max_role_num = gComponentList[i].component.totalRoleNum;

        for (j = 0; j < max_role_num; j++) {
            if (Rockchip_OSAL_Strcmp(gComponentList[i].component.roles[j], role) == 0) {
                if (compNames != NULL) {
                    Rockchip_OSAL_Strcpy((OMX_STRING)compNames[*pNumComps], gComponentList[i].component.componentName);
                }
                *pNumComps = (*pNumComps + 1);
            }
        }
    }

EXIT:
    FunctionOut();

    return ret;
}
```

再来看 MediaCodecsXmlParser 的 getServiceAttributeMap() 具体实现，其内部调用实现 Impl 同名函数，此函数内部只是简单返回 Data 结构体 mServiceAttributeMap 字段，此字段的填充发生在解析 xml 期间。

frameworks/av/media/libstagefright/xmlparser/MediaCodecsXmlParser.cpp

```
struct MediaCodecsXmlParser::Impl {
    ......
    // Parsed data
    struct Data {
        // Service attributes
        AttributeMap mServiceAttributeMap;
        CodecMap mCodecMap;
        Result addGlobal(std::string key, std::string value, bool updating);
    };
    ......
    const AttributeMap& getServiceAttributeMap() const {
        std::lock_guard<std::mutex> guard(mLock);
        return mData.mServiceAttributeMap;
    }
    ......
}
......
const MediaCodecsXmlParser::AttributeMap&
MediaCodecsXmlParser::getServiceAttributeMap() const {
    return mImpl->getServiceAttributeMap();
}
```

MediaCodecsXmlParser 的 getRoleMap() 具体实现中委托给 Impl getRoleMap() 方法的 ，如果 mRoleMap 为空，就调用 generateRoleMap() 先生成，不为空直接返回。

generateRoleMap() 实现并不复杂，主要遍历 mData.mCodecMap，然后在每轮迭代中构建 std::string — RoleProperties 对，如果 mRoleMap 中不存在对应项就插入。中间调用了 GetComponentRole(…) 根据 isEncoder 和 mime 类型字符串作为入参去查找角色名。

frameworks/av/media/libstagefright/xmlparser/MediaCodecsXmlParser.cpp

```
const MediaCodecsXmlParser::RoleMap&
MediaCodecsXmlParser::getRoleMap() const {
    return mImpl->getRoleMap();
}

const MediaCodecsXmlParser::RoleMap&
MediaCodecsXmlParser::Impl::getRoleMap() const {
    std::lock_guard<std::mutex> guard(mLock);
    if (mRoleMap.empty()) {
        generateRoleMap();
    }
    return mRoleMap;
}
......
void MediaCodecsXmlParser::Impl::generateRoleMap() const {
    for (const auto& codec : mData.mCodecMap) {
        const auto &codecName = codec.first;
        if (codecName == "<dummy>") {
            continue;
        }
        bool isEncoder = codec.second.isEncoder;
        size_t order = codec.second.order;
        std::string rank = codec.second.rank;
        const auto& typeMap = codec.second.typeMap;
        for (const auto& type : typeMap) {
            const auto& typeName = type.first;
            const char* roleName = GetComponentRole(isEncoder, typeName.data());
            if (roleName == nullptr) {
                ALOGE("Cannot find the role for %s of type %s",
                        isEncoder ? "an encoder" : "a decoder",
                        typeName.data());
                continue;
            }
            const auto& typeAttributeMap = type.second;

            auto roleIterator = mRoleMap.find(roleName);
            std::multimap<size_t, NodeInfo>* nodeList;
            if (roleIterator == mRoleMap.end()) {
                RoleProperties roleProperties;
                roleProperties.type = typeName;
                roleProperties.isEncoder = isEncoder;
                auto insertResult = mRoleMap.insert(
                        std::make_pair(roleName, roleProperties));
                if (!insertResult.second) {
                    ALOGE("Cannot add role %s", roleName);
                    continue;
                }
                nodeList = &insertResult.first->second.nodeList;
            } else {
                if (roleIterator->second.type != typeName) {
                    ALOGE("Role %s has mismatching types: %s and %s",
                            roleName,
                            roleIterator->second.type.data(),
                            typeName.data());
                    continue;
                }
                if (roleIterator->second.isEncoder != isEncoder) {
                    ALOGE("Role %s cannot be both an encoder and a decoder",
                            roleName);
                    continue;
                }
                nodeList = &roleIterator->second.nodeList;
            }

            NodeInfo nodeInfo;
            nodeInfo.name = codecName;
            // NOTE: no aliases are exposed in role info
            // attribute quirks are exposed as node attributes
            nodeInfo.attributeList.reserve(typeAttributeMap.size());
            for (const auto& attribute : typeAttributeMap) {
                nodeInfo.attributeList.push_back(
                        Attribute{attribute.first, attribute.second});
            }
            for (const std::string &quirk : codec.second.quirkSet) {
                if (strHasPrefix(quirk.c_str(), "attribute::")) {
                    nodeInfo.attributeList.push_back(Attribute{quirk, "present"});
                }
            }
            if (!rank.empty()) {
                nodeInfo.attributeList.push_back(Attribute{"rank", rank});
            }
            nodeList->insert(std::make_pair(
                    std::move(order), std::move(nodeInfo)));
        }
    }
}
```

GetComponentRole(…) 内部遍历定义好的静态常量 MimeToRole 数组，调用 strcasecmp(…) 忽略大小写去比较入参 mime 和 MimeToRole 项 mime 字段是否匹配，匹配后 break 出循环，再根据 isEncoder 入参返回角色名，是那个编码器或解码器就被唯一确定了，也就是角色名查到了。

frameworks/av/media/libstagefright/omx/OMXUtils.cpp

```
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
```

再来看一个常量 MEDIA_MIMETYPE_VIDEO_AVC。回顾一下我们使用 MediaCodec 的时候是不是会传入这个 mime 类型呢？

frameworks/av/media/libstagefright/foundation/MediaDefs.cpp

最后再来看 MediaCodecsXmlParser::Impl::getCommonPrefix() 中调用 generateCommonPrefix() 生成前缀，然后获取指向前缀的 char* 指针后返回。

这里用到了 std::mismatch(…) 标准库的函数，在这里就是找出 CodecMap 中所有元素 key 中共同的前缀部分字符串，然后更新到 mCommonPrefix 这个变量上。

frameworks/av/media/libstagefright/xmlparser/MediaCodecsXmlParser.cpp

```
const char* MediaCodecsXmlParser::Impl::getCommonPrefix() const {
    std::lock_guard<std::mutex> guard(mLock);
    if (mCommonPrefix.empty()) {
        generateCommonPrefix();
    }
    return mCommonPrefix.data();
}
......
void MediaCodecsXmlParser::Impl::generateCommonPrefix() const {
    if (mData.mCodecMap.empty()) {
        return;
    }
    auto i = mData.mCodecMap.cbegin();
    auto first = i->first.cbegin();
    auto last = i->first.cend();
    for (++i; i != mData.mCodecMap.cend(); ++i) {
        last = std::mismatch(
                first, last, i->first.cbegin(), i->first.cend()).first;
    }
    mCommonPrefix.insert(mCommonPrefix.begin(), first, last);
}
```

