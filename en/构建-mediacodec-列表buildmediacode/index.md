# 

构建 MediaCodec 列表——buildMediaCodecList(…) ，确切地说是构建了 std::[vector](https://so.csdn.net/so/search?q=vector&spm=1001.2101.3001.7020) 容器，容器内部是指向 MediaCodecInfo 对象的指针。

关于 buildMediaCodecList(…) 在 MediaCodec 中初始化流程可以参考[MediaCodec 硬解码初始化]根据分析得出 builder 调用其 buildMediaCodecList(…)，其一是 OmxInfoBuilder，其二是 Codec2InfoBuilder。现在我们来分析它们的 buildMediaCodecList(…) 具体实现。

1. 获取 OmxStore 服务；

2. 列出角色（Role），通过调用 listRoles(…) 方法实现；

3. 列出服务属性（全局设置），通过 listServiceAttributes(…) 方法实现；

4. 调用 MediaCodecListWriter addGlobalSetting(…) 将服务属性全部写入；

5. 将角色转换为编[解码](https://so.csdn.net/so/search?q=%E8%A7%A3%E7%A0%81&spm=1001.2101.3001.7020)器列表：

    （1）跳过不允许使用输入 surface 的条目；

    （2）通过属性从 OmxStore 服务获得排名（rank）；

    （3）为新节点创建新的 MediaCodecInfo；

    （4）所有 OMX 编解码器都是供应商编解码器（在供应商分区），但处理 OMX.google 编解码器是非硬件加速的和非供应商的，也就是说是软编解码；

![](content/assets/images/OmxInfoBuilderSequenceDiagram.jpg)

frameworks/av/media/libstagefright/OmxInfoBuilder.cpp

```cpp
status_t OmxInfoBuilder::buildMediaCodecList(MediaCodecListWriter* writer) {
    // Obtain IOmxStore
    sp<IOmxStore> omxStore = IOmxStore::getService();
    if (omxStore == nullptr) {
        ALOGE("Cannot find an IOmxStore service.");
        return NO_INIT;
    }

    // List service attributes (global settings)
    Status status;
    hidl_vec<IOmxStore::RoleInfo> roles;
    auto transStatus = omxStore->listRoles(
            [&roles] (
            const hidl_vec<IOmxStore::RoleInfo>& inRoleList) {
                roles = inRoleList;
            });
    if (!transStatus.isOk()) {
        ALOGE("Fail to obtain codec roles from IOmxStore.");
        return NO_INIT;
    }

    hidl_vec<IOmxStore::ServiceAttribute> serviceAttributes;
    transStatus = omxStore->listServiceAttributes(
            [&status, &serviceAttributes] (
            Status inStatus,
            const hidl_vec<IOmxStore::ServiceAttribute>& inAttributes) {
                status = inStatus;
                serviceAttributes = inAttributes;
            });
    if (!transStatus.isOk()) {
        ALOGE("Fail to obtain global settings from IOmxStore.");
        return NO_INIT;
    }
    if (status != Status::OK) {
        ALOGE("IOmxStore reports parsing error.");
        return NO_INIT;
    }
    for (const auto& p : serviceAttributes) {
        writer->addGlobalSetting(
                p.key.c_str(), p.value.c_str());
    }

    // Convert roles to lists of codecs

    // codec name -> index into swCodecs/hwCodecs
    std::map<hidl_string, std::unique_ptr<MediaCodecInfoWriter>> codecName2Info;

    uint32_t defaultRank =
        ::android::base::GetUintProperty("debug.stagefright.omx_default_rank", 0x100u);
    uint32_t defaultSwAudioRank =
        ::android::base::GetUintProperty("debug.stagefright.omx_default_rank.sw-audio", 0x10u);
    uint32_t defaultSwOtherRank =
        ::android::base::GetUintProperty("debug.stagefright.omx_default_rank.sw-other", 0x210u);

    for (const IOmxStore::RoleInfo& role : roles) {
        const hidl_string& typeName = role.type;
        bool isEncoder = role.isEncoder;
        bool isAudio = hasPrefix(role.type, "audio/");
        bool isVideoOrImage = hasPrefix(role.type, "video/") || hasPrefix(role.type, "image/");

        for (const IOmxStore::NodeInfo &node : role.nodes) {
            const hidl_string& nodeName = node.name;

            // currently image and video encoders use surface input
            if (!mAllowSurfaceEncoders && isVideoOrImage && isEncoder) {
                ALOGD("disabling %s for media type %s because we are not using OMX input surface",
                        nodeName.c_str(), role.type.c_str());
                continue;
            }

            bool isSoftware = hasPrefix(nodeName, "OMX.google");
            uint32_t rank = isSoftware
                    ? (isAudio ? defaultSwAudioRank : defaultSwOtherRank)
                    : defaultRank;
            // get rank from IOmxStore via attribute
            for (const IOmxStore::Attribute& attribute : node.attributes) {
                if (attribute.key == "rank") {
                    uint32_t oldRank = rank;
                    char dummy;
                    if (sscanf(attribute.value.c_str(), "%u%c", &rank, &dummy) != 1) {
                        rank = oldRank;
                    }
                    break;
                }
            }

            MediaCodecInfoWriter* info;
            auto c2i = codecName2Info.find(nodeName);
            if (c2i == codecName2Info.end()) {
                // Create a new MediaCodecInfo for a new node.
                c2i = codecName2Info.insert(std::make_pair(
                        nodeName, writer->addMediaCodecInfo())).first;
                info = c2i->second.get();
                info->setName(nodeName.c_str());
                info->setOwner(node.owner.c_str());
                info->setRank(rank);

                typename std::underlying_type<MediaCodecInfo::Attributes>::type attrs = 0;
                // all OMX codecs are vendor codecs (in the vendor partition), but
                // treat OMX.google codecs as non-hardware-accelerated and non-vendor
                if (!isSoftware) {
                    attrs |= MediaCodecInfo::kFlagIsVendor;
                    if (!std::count_if(
                            node.attributes.begin(), node.attributes.end(),
                            [](const IOmxStore::Attribute &i) -> bool {
                                return i.key == "attribute::software-codec";
                                                                      })) {
                        attrs |= MediaCodecInfo::kFlagIsHardwareAccelerated;
                    }
                }
                if (isEncoder) {
                    attrs |= MediaCodecInfo::kFlagIsEncoder;
                }
                info->setAttributes(attrs);
            } else {
                // The node has been seen before. Simply retrieve the
                // existing MediaCodecInfoWriter.
                info = c2i->second.get();
            }
            std::unique_ptr<MediaCodecInfo::CapabilitiesWriter> caps =
                    info->addMediaType(typeName.c_str());
            if (queryCapabilities(
                    node, typeName.c_str(), isEncoder, caps.get()) != OK) {
                ALOGW("Fail to add media type %s to codec %s",
                        typeName.c_str(), nodeName.c_str());
                info->removeMediaType(typeName.c_str());
            }
        }
    }
    return OK;
}
123456789101112131415161718192021222324252627282930313233343536373839404142434445464748495051525354555657585960616263646566676869707172737475767778798081828384858687888990919293949596979899100101102103104105106107108109110111112113114115116117118119120121122123124125126127128129130
```

第一步获取 OmxStore 服务，IOmxStore 接口定义在 hardware/interfaces/media/omx/1.0/IOmxStore.hal 文件中。这个服务在 frameworks/av/services/mediacodec/main_codecservice.cpp 中进行注册（调用了 registerAsService()）。具体可以查看《[【Android 10 源码】深入理解 codecservice 启动](https://blog.csdn.net/tyyj90/article/details/120935509?spm=1001.2014.3001.5501)》一节。

OmxStore listRoles(…) 函数实现非常简单，仅仅将 OmxStore 构造函数中填值的 mRoleList 作为入参送入匿名函数，最终获取 RoleInfo 列表。

frameworks/av/media/libstagefright/omx/1.0/OmxStore.cpp

```
Return<void> OmxStore::listRoles(listRoles_cb _hidl_cb) {
    _hidl_cb(mRoleList);
    return Void();
}

```

OmxStore listServiceAttributes(…) 同样实现非常简单，根据解析状态 _hidl_cb(…) 有选择的送入不同的入参。正常没有解析错误的情况下，直接送入 mServiceAttributeList 作为入参，mServiceAttributeList 也是在 OmxStore 构造函数中初始化的。

frameworks/av/media/libstagefright/omx/1.0/OmxStore.cpp

```
Return<void> OmxStore::listServiceAttributes(listServiceAttributes_cb _hidl_cb) {
    if (mParsingStatus == Status::NO_ERROR) {
        _hidl_cb(Status::NO_ERROR, mServiceAttributeList);
    } else {
        _hidl_cb(mParsingStatus, hidl_vec<ServiceAttribute>());
    }
    return Void();
}

```

调用 MediaCodecListWriter addGlobalSetting(…) 将服务属性全部写入，将服务属性键值对逐个写入 std::vector<std::pair<std::string, std::string>> 容器。

frameworks/av/media/libstagefright/include/media/stagefright/MediaCodecListWriter.h

```
struct MediaCodecListWriter {
    ......
private:
    std::vector<std::pair<std::string, std::string>> mGlobalSettings;
}

```

frameworks/av/media/libstagefright/MediaCodecListWriter.cpp

```
void MediaCodecListWriter::addGlobalSetting(
        const char* key, const char* value) {
    mGlobalSettings.emplace_back(key, value);
}

```

再来分析下将角色转换为编解码器列表中创建新的 MediaCodecInfo 具体流程。内部先新建 MediaCodecInfo 结构体，接着将这个新建的结构体压入 mCodecInfos 容器（std::vector<sp>），最后返回一个新的 MediaCodecInfoWriter 对象。

首先调用 MediaCodecListWriter addMediaCodecInfo()。

frameworks/av/media/libstagefright/MediaCodecListWriter.cpp

```
std::unique_ptr<MediaCodecInfoWriter>
        MediaCodecListWriter::addMediaCodecInfo() {
    sp<MediaCodecInfo> info = new MediaCodecInfo();
    mCodecInfos.push_back(info);
    return std::unique_ptr<MediaCodecInfoWriter>(
            new MediaCodecInfoWriter(info.get()));
}

```

目前，OMX 组件的默认 rank 是 0x100，而所有 Codec2.0 软件组件的默认 rank 是 0x200。

Codec2InfoBuilder buildMediaCodecList(…) 主要工作流程如下：

1. 获取 debug.stagefright.ccodec 的属性值，默认为 4，在 rk3399 android 平台并未修改；

2. 调用 Codec2Client::ListComponents() 获取组件列表，确切地说是返回容器 std::vector；

3. 首先解析 APEX XML，然后解析 vendor XML，也就是先解析”/apex/com.android.media.swcodec/etc/media_codecs.xml“ 和”/apex/com.android.media.swcodec/etc/media_codecs_performance.xml“，再去解析”/odm/etc/media_codecs_c2.xml“、”/odm/etc/media_codecs_performance_c2.xml“、”/vendor/etc/media_codecs_c2.xml“、”/vendor/etc/media_codecs_performance_c2.xml“、”/etc/media_codecs_c2.xml“、”/etc/media_codecs_performance_c2.xml“；

4. 解析默认 XML 文件（”/odm/etc/media_codecs.xml“、”/odm/etc/media_codecs_performance.xml“、”/vendor/etc/media_codecs.xml“、”/vendor/etc/media_codecs_performance.xml“、”/etc/media_codecs.xml“、”/etc/media_codecs_performance.xml“）；

5. 获取解析最终状态如果是 OK，调用 MediaCodecsXmlParser getServiceAttributeMap() 获取 AttributeMap，接着调用 MediaCodecListWriter addGlobalSetting(…) 逐个添加所有属性；

6. 遍历所有组件，将符合条件的先调用 MediaCodecListWriter addMediaCodecInfo() 进行添加，然后通过 addMediaCodecInfo() 返回的 MediaCodecInfoWriter 对象添加各种信息到 MediaCodecInfo。

    (1) 获取别名容器，将本名也添加到容器中；

    (2) 遍历整个别名容器中的元素；

    - 调用 Codec2Client::CreateInterfaceByName(…) 获取 Codec2Client::Interface，如果获取为空，直接 continue 到下一轮循环；

    - 查找名称是否在 CodecMap 中，如果不存在 continue 到下一轮循环，循环之前如果是别名，会在 MediaCodecInfoWriter 中查找是否存在 MediaCodecInfo，存在的话把这个别名也添加到 MediaCodecInfo 的别名容器中；

    - 根据 debug.stagefright.ccodec 属性值的不同，给 rank 赋值为 1，这会改变编解码器在列表中的权重；

    - 通过名称在 CodecMap 中查找 MediaCodecsXmlParser::CodecProperties；

    - 验证是否显式启用了编解码器，或其域之一；

    - 如果编解码器有变体，还要检查至少有一个是启用的；

    - 编解码器没启用或其变体也没启用，直接进行下一轮循环；

    - 添加编解码器条目，这是调用 MediaCodecListWriter addMediaCodecInfo() 实现的；

    - 给 MediaCodecInfo 设置各种属性，包括名称、属主、属性（Attributes）、rank和别名；

    - 遍历 typeMap，如果编解码器条目的媒体类型被禁用 continue 到下一轮循环，调用 MediaCodecInfoWriter addMediaType(…) 获取 CapabilitiesWriter 添加能力，调用 addSupportedProfileLevels(…) 添加 Profile Level，调用 addSupportedColorFormats(…) 添加支持的颜色空间。

![Codec2InfoBuilderSequenceDiagram.jpg](content/assets/images/Codec2InfoBuilderSequenceDiagram.jpg)

frameworks/av/media/codec2/sfplugin/Codec2InfoBuilder.cpp

```

status_t Codec2InfoBuilder::buildMediaCodecList(MediaCodecListWriter* writer) {
    //
    // debug.stagefright.ccodec supports 5 values.
    //   0 - No Codec 2.0 components are available.
    //   1 - Audio decoders and encoders with prefix "c2.android." are available
    //       and ranked first.
    //       All other components with prefix "c2.android." are available with
    //       their normal ranks.
    //       Components with prefix "c2.vda." are available with their normal
    //       ranks.
    //       All other components with suffix ".avc.decoder" or ".avc.encoder"
    //       are available but ranked last.
    //   2 - Components with prefix "c2.android." are available and ranked
    //       first.
    //       Components with prefix "c2.vda." are available with their normal
    //       ranks.
    //       All other components with suffix ".avc.decoder" or ".avc.encoder"
    //       are available but ranked last.
    //   3 - Components with prefix "c2.android." are available and ranked
    //       first.
    //       All other components are available with their normal ranks.
    //   4 - All components are available with their normal ranks.
    //
    // The default value (boot time) is 1.
    //
    // Note: Currently, OMX components have default rank 0x100, while all
    // Codec2.0 software components have default rank 0x200.
    int option = ::android::base::GetIntProperty("debug.stagefright.ccodec", 4);

    // Obtain Codec2Client
    std::vector<Traits> traits = Codec2Client::ListComponents();

    // parse APEX XML first, followed by vendor XML
    MediaCodecsXmlParser parser;
    parser.parseXmlFilesInSearchDirs(
            parser.getDefaultXmlNames(),
            { "/apex/com.android.media.swcodec/etc" });

    // TODO: remove these c2-specific files once product moved to default file names
    parser.parseXmlFilesInSearchDirs(
            { "media_codecs_c2.xml", "media_codecs_performance_c2.xml" });

    // parse default XML files
    parser.parseXmlFilesInSearchDirs();

    if (parser.getParsingStatus() != OK) {
        ALOGD("XML parser no good");
        return OK;
    }

    MediaCodecsXmlParser::AttributeMap settings = parser.getServiceAttributeMap();
    for (const auto &v : settings) {
        if (!hasPrefix(v.first, "media-type-")
                && !hasPrefix(v.first, "domain-")
                && !hasPrefix(v.first, "variant-")) {
            writer->addGlobalSetting(v.first.c_str(), v.second.c_str());
        }
    }

    for (const Traits& trait : traits) {
        C2Component::rank_t rank = trait.rank;

        // Interface must be accessible for us to list the component, and there also
        // must be an XML entry for the codec. Codec aliases listed in the traits
        // allow additional XML entries to be specified for each alias. These will
        // be listed as separate codecs. If no XML entry is specified for an alias,
        // those will be treated as an additional alias specified in the XML entry
        // for the interface name.
        std::vector<std::string> nameAndAliases = trait.aliases;
        nameAndAliases.insert(nameAndAliases.begin(), trait.name);
        for (const std::string &nameOrAlias : nameAndAliases) {
            bool isAlias = trait.name != nameOrAlias;
            std::shared_ptr<Codec2Client::Interface> intf =
                Codec2Client::CreateInterfaceByName(nameOrAlias.c_str());
            if (!intf) {
                ALOGD("could not create interface for %s'%s'",
                        isAlias ? "alias " : "",
                        nameOrAlias.c_str());
                continue;
            }
            if (parser.getCodecMap().count(nameOrAlias) == 0) {
                if (isAlias) {
                    std::unique_ptr<MediaCodecInfoWriter> baseCodecInfo =
                        writer->findMediaCodecInfo(trait.name.c_str());
                    if (!baseCodecInfo) {
                        ALOGD("alias '%s' not found in xml but canonical codec info '%s' missing",
                                nameOrAlias.c_str(),
                                trait.name.c_str());
                    } else {
                        ALOGD("alias '%s' not found in xml; use an XML <Alias> tag for this",
                                nameOrAlias.c_str());
                        // merge alias into existing codec
                        baseCodecInfo->addAlias(nameOrAlias.c_str());
                    }
                } else {
                    ALOGD("component '%s' not found in xml", trait.name.c_str());
                }
                continue;
            }
            std::string canonName = trait.name;

            // TODO: Remove this block once all codecs are enabled by default.
            switch (option) {
            case 0:
                continue;
            case 1:
                if (hasPrefix(canonName, "c2.vda.")) {
                    break;
                }
                if (hasPrefix(canonName, "c2.android.")) {
                    if (trait.domain == C2Component::DOMAIN_AUDIO) {
                        rank = 1;
                        break;
                    }
                    break;
                }
                if (hasSuffix(canonName, ".avc.decoder") ||
                        hasSuffix(canonName, ".avc.encoder")) {
                    rank = std::numeric_limits<decltype(rank)>::max();
                    break;
                }
                continue;
            case 2:
                if (hasPrefix(canonName, "c2.vda.")) {
                    break;
                }
                if (hasPrefix(canonName, "c2.android.")) {
                    rank = 1;
                    break;
                }
                if (hasSuffix(canonName, ".avc.decoder") ||
                        hasSuffix(canonName, ".avc.encoder")) {
                    rank = std::numeric_limits<decltype(rank)>::max();
                    break;
                }
                continue;
            case 3:
                if (hasPrefix(canonName, "c2.android.")) {
                    rank = 1;
                }
                break;
            }

            const MediaCodecsXmlParser::CodecProperties &codec =
                parser.getCodecMap().at(nameOrAlias);

            // verify that either the codec is explicitly enabled, or one of its domains is
            bool codecEnabled = codec.quirkSet.find("attribute::disabled") == codec.quirkSet.end();
            if (!codecEnabled) {
                for (const std::string &domain : codec.domainSet) {
                    const Switch enabled = isDomainEnabled(domain, settings);
                    ALOGV("codec entry '%s' is in domain '%s' that is '%s'",
                            nameOrAlias.c_str(), domain.c_str(), asString(enabled));
                    if (enabled) {
                        codecEnabled = true;
                        break;
                    }
                }
            }
            // if codec has variants, also check that at least one of them is enabled
            bool variantEnabled = codec.variantSet.empty();
            for (const std::string &variant : codec.variantSet) {
                const Switch enabled = isVariantExpressionEnabled(variant, settings);
                ALOGV("codec entry '%s' has a variant '%s' that is '%s'",
                        nameOrAlias.c_str(), variant.c_str(), asString(enabled));
                if (enabled) {
                    variantEnabled = true;
                    break;
                }
            }
            if (!codecEnabled || !variantEnabled) {
                ALOGD("codec entry for '%s' is disabled", nameOrAlias.c_str());
                continue;
            }

            ALOGV("adding codec entry for '%s'", nameOrAlias.c_str());
            std::unique_ptr<MediaCodecInfoWriter> codecInfo = writer->addMediaCodecInfo();
            codecInfo->setName(nameOrAlias.c_str());
            codecInfo->setOwner(("codec2::" + trait.owner).c_str());

            bool encoder = trait.kind == C2Component::KIND_ENCODER;
            typename std::underlying_type<MediaCodecInfo::Attributes>::type attrs = 0;

            if (encoder) {
                attrs |= MediaCodecInfo::kFlagIsEncoder;
            }
            if (trait.owner == "software") {
                attrs |= MediaCodecInfo::kFlagIsSoftwareOnly;
            } else {
                attrs |= MediaCodecInfo::kFlagIsVendor;
                if (trait.owner == "vendor-software") {
                    attrs |= MediaCodecInfo::kFlagIsSoftwareOnly;
                } else if (codec.quirkSet.find("attribute::software-codec")
                        == codec.quirkSet.end()) {
                    attrs |= MediaCodecInfo::kFlagIsHardwareAccelerated;
                }
            }
            codecInfo->setAttributes(attrs);
            if (!codec.rank.empty()) {
                uint32_t xmlRank;
                char dummy;
                if (sscanf(codec.rank.c_str(), "%u%c", &xmlRank, &dummy) == 1) {
                    rank = xmlRank;
                }
            }
            ALOGV("rank: %u", (unsigned)rank);
            codecInfo->setRank(rank);

            for (const std::string &alias : codec.aliases) {
                ALOGV("adding alias '%s'", alias.c_str());
                codecInfo->addAlias(alias.c_str());
            }

            for (auto typeIt = codec.typeMap.begin(); typeIt != codec.typeMap.end(); ++typeIt) {
                const std::string &mediaType = typeIt->first;
                const Switch typeEnabled = isSettingEnabled(
                        "media-type-" + mediaType, settings, Switch::ENABLED_BY_DEFAULT());
                const Switch domainTypeEnabled = isSettingEnabled(
                        "media-type-" + mediaType + (encoder ? "-encoder" : "-decoder"),
                        settings, Switch::ENABLED_BY_DEFAULT());
                ALOGV("type '%s-%s' is '%s/%s'",
                        mediaType.c_str(), (encoder ? "encoder" : "decoder"),
                        asString(typeEnabled), asString(domainTypeEnabled));
                if (!typeEnabled || !domainTypeEnabled) {
                    ALOGD("media type '%s' for codec entry '%s' is disabled", mediaType.c_str(),
                            nameOrAlias.c_str());
                    continue;
                }

                ALOGI("adding type '%s'", typeIt->first.c_str());
                const MediaCodecsXmlParser::AttributeMap &attrMap = typeIt->second;
                std::unique_ptr<MediaCodecInfo::CapabilitiesWriter> caps =
                    codecInfo->addMediaType(mediaType.c_str());
                for (const auto &v : attrMap) {
                    std::string key = v.first;
                    std::string value = v.second;

                    size_t variantSep = key.find(":::");
                    if (variantSep != std::string::npos) {
                        std::string variant = key.substr(0, variantSep);
                        const Switch enabled = isVariantExpressionEnabled(variant, settings);
                        ALOGV("variant '%s' is '%s'", variant.c_str(), asString(enabled));
                        if (!enabled) {
                            continue;
                        }
                        key = key.substr(variantSep + 3);
                    }

                    if (key.find("feature-") == 0 && key.find("feature-bitrate-modes") != 0) {
                        int32_t intValue = 0;
                        // Ignore trailing bad characters and default to 0.
                        (void)sscanf(value.c_str(), "%d", &intValue);
                        caps->addDetail(key.c_str(), intValue);
                    } else {
                        caps->addDetail(key.c_str(), value.c_str());
                    }
                }

                addSupportedProfileLevels(intf, caps.get(), trait, mediaType);
                addSupportedColorFormats(intf, caps.get(), trait, mediaType);
            }
        }
    }
    return OK;
}
101112131415161718192021222324252627282930313233343536373839404142434445464748495051525354555657585960616263646566676869707172737475767778798081828384858687888990919293949596979899100101102103104105106107108109110111112113114115116117118119120121122123124125126127128129130131132133134135136137138139140141142143144145146147148149150151152153154155156157158159160161162163164165166167168169170171172173174175176177178179180181182183184185186187188189190191192193194195196197198199200201202203204205206207208209210211212213214215216217218219220221222223224225226227228229230231232233234235236237238239240241242243244245246247248249250251252253254255256257258259260261262263264265
```

这个函数实现本身不复杂，仅仅将 sList 使用匿名函数进行了初始化，而容器中的项是遍历 Cache::List() 获取后整体插入到 std::vectorC2Component::Traits。

frameworks/av/media/codec2/hidl/client/client.cpp

```
std::vector<C2Component::Traits> const& Codec2Client::ListComponents() {
    static std::vector<C2Component::Traits> sList{[]() {
        std::vector<C2Component::Traits> list;
        for (Cache& cache : Cache::List()) {
            std::vector<C2Component::Traits> const& traits = cache.getTraits();
            list.insert(list.end(), traits.begin(), traits.end());
        }
        return list;
    }()};
    return sList;
}
110
```

Traits 结构体代表组件信息。

frameworks/av/media/codec2/core/include/C2Component.h

```
class C2Component {
public:
    ......
    /**
     * Information about a component.
     */
    struct Traits {
    // public:
        C2String name; ///< name of the component
        domain_t domain; ///< component domain
        kind_t kind; ///< component kind
        rank_t rank; ///< component rank
        C2String mediaType; ///< media type supported by the component
        C2String owner; ///< name of the component store owning this component

        /**
         * name alias(es) for backward compatibility.
         * \note Multiple components can have the same alias as long as their media-type differs.
         */
        std::vector<C2String> aliases; ///< name aliases for backward compatibility
    };
    ......
}
110111213141516171819202122
```

Codec2Client::Cache 这个类缓存一个 Codec2Client 对象及其组件特性（Traits）。客户端将在第一次需要时被创建，如果服务终止（通过调用invalidate()），则被移除。第一次从客户端调用 listComponents() 时，结果将被缓存。

Cache::List() 主要工作流程：

1. 调用 GetServiceNames() 获取服务名 vector；
2. 构建 std::vector ，个数为上一步获取的服务名数量；
3. 遍历 std::vector 中的每个元素，并调用 init(…) 进行初始化;
4. 以上步骤在匿名函数中进行的，因此会初始化 sCaches，初始化后下次再去调用直接返回。

Cache init(…) 方法仅仅将 index 做了一个记录。

Cache::getTraits() 主要工作流程：

1. 调用 getClient() 获取 Codec2Client 客户端；
2. 调用Codec2Client _listComponents(…) 方法获取组件列表（std::vectorC2Component::Traits）。

Cache::getClient() 调用 Codec2Client::_CreateFromIndex(…) 根据记录的 mIndex 获取 Codec2Client 客户端。

frameworks/av/media/codec2/hidl/client/client.cpp

```
class Codec2Client::Cache {
    // Cached client
    std::shared_ptr<Codec2Client> mClient;
    mutable std::mutex mClientMutex;

    // Cached component traits
    std::vector<C2Component::Traits> mTraits;
    std::once_flag mTraitsInitializationFlag;

    // The index of the service. This is based on GetServiceNames().
    size_t mIndex;
    // Called by s() exactly once to initialize the cache. The index must be a
    // valid index into the vector returned by GetServiceNames(). Calling
    // init(index) will associate the cache to the service with name
    // GetServiceNames()[index].
    void init(size_t index) {
        mIndex = index;
    }

public:
    Cache() = default;

    // Initializes mClient if needed, then returns mClient.
    // If the service is unavailable but listed in the manifest, this function
    // will block indefinitely.
    std::shared_ptr<Codec2Client> getClient() {
        std::scoped_lock lock{mClientMutex};
        if (!mClient) {
            mClient = Codec2Client::_CreateFromIndex(mIndex);
        }
        return mClient;
    }

    // Causes a subsequent call to getClient() to create a new client. This
    // function should be called after the service dies.
    //
    // Note: This function is called only by ForAllServices().
    void invalidate() {
        std::scoped_lock lock{mClientMutex};
        mClient = nullptr;
    }

    // Returns a list of traits for components supported by the service. This
    // list is cached.
    std::vector<C2Component::Traits> const& getTraits() {
        std::call_once(mTraitsInitializationFlag, [this]() {
            bool success{false};
            // Spin until _listComponents() is successful.
            while (true) {
                std::shared_ptr<Codec2Client> client = getClient();
                mTraits = client->_listComponents(&success);
                if (success) {
                    break;
                }
                using namespace std::chrono_literals;
                static constexpr auto kServiceRetryPeriod = 5s;
                LOG(INFO) << "Failed to retrieve component traits from service "
                             "\"" << GetServiceNames()[mIndex] << "\". "
                             "Retrying...";
                std::this_thread::sleep_for(kServiceRetryPeriod);
            }
        });
        return mTraits;
    }

    // List() returns the list of all caches.
    static std::vector<Cache>& List() {
        static std::vector<Cache> sCaches{[]() {
            size_t numServices = GetServiceNames().size();
            std::vector<Cache> caches(numServices);
            for (size_t i = 0; i < numServices; ++i) {
                caches[i].init(i);
            }
            return caches;
        }()};
        return sCaches;
    }
};
1011121314151617181920212223242526272829303132333435363738394041424344454647484950515253545556575859606162636465666768697071727374757677
```

以上提交的几个函数需要再次分析下，首先来分析 Codec2Client::GetServiceNames()。

1. 调用 IServiceManager::getService() 获取“大内总管”，调用其 listManifestByInterface(…) 从清单文件中检索 IComponentStore::descriptor 这个描述符对应的服务；
2. 获取到服务名称列表后，在匿名函数内进行筛选（三个分类为 default、vendor 和 other）；
3. 对每个类别中的服务名称进行排序；
4. 按照如下顺序连接三个列表：default、vendor 和 other；
5. 总结 log 输出。

在 rk3399 android 10 平台上可见 Log：Codec2Client: Available Codec2 services: “software”

frameworks/av/media/codec2/hidl/client/client.cpp

```
std::vector<std::string> const& Codec2Client::GetServiceNames() {
    static std::vector<std::string> sServiceNames{[]() {
        using ::android::hardware::media::c2::V1_0::IComponentStore;
        using ::android::hidl::manager::V1_2::IServiceManager;

        while (true) {
            sp<IServiceManager> serviceManager = IServiceManager::getService();
            CHECK(serviceManager) << "Hardware service manager is not running.";

            // There are three categories of services based on names.
            std::vector<std::string> defaultNames; // Prefixed with "default"
            std::vector<std::string> vendorNames;  // Prefixed with "vendor"
            std::vector<std::string> otherNames;   // Others
            Return<void> transResult;
            transResult = serviceManager->listManifestByInterface(
                    IComponentStore::descriptor,
                    [&defaultNames, &vendorNames, &otherNames](
                            hidl_vec<hidl_string> const& instanceNames) {
                        for (hidl_string const& instanceName : instanceNames) {
                            char const* name = instanceName.c_str();
                            if (strncmp(name, "default", 7) == 0) {
                                defaultNames.emplace_back(name);
                            } else if (strncmp(name, "vendor", 6) == 0) {
                                vendorNames.emplace_back(name);
                            } else {
                                otherNames.emplace_back(name);
                            }
                        }
                    });
            if (transResult.isOk()) {
                // Sort service names in each category.
                std::sort(defaultNames.begin(), defaultNames.end());
                std::sort(vendorNames.begin(), vendorNames.end());
                std::sort(otherNames.begin(), otherNames.end());

                // Concatenate the three lists in this order: default, vendor,
                // other.
                std::vector<std::string>& names = defaultNames;
                names.reserve(names.size() + vendorNames.size() + otherNames.size());
                names.insert(names.end(),
                             std::make_move_iterator(vendorNames.begin()),
                             std::make_move_iterator(vendorNames.end()));
                names.insert(names.end(),
                             std::make_move_iterator(otherNames.begin()),
                             std::make_move_iterator(otherNames.end()));

                // Summarize to logcat.
                if (names.empty()) {
                    LOG(INFO) << "No Codec2 services declared in the manifest.";
                } else {
                    std::stringstream stringOutput;
                    stringOutput << "Available Codec2 services:";
                    for (std::string const& name : names) {
                        stringOutput << " \"" << name << "\"";
                    }
                    LOG(INFO) << stringOutput.str();
                }

                return names;
            }
            LOG(ERROR) << "Could not retrieve the list of service instances of "
                       << IComponentStore::descriptor
                       << ". Retrying...";
        }
    }()};
    return sServiceNames;
}
101112131415161718192021222324252627282930313233343536373839404142434445464748495051525354555657585960616263646566
```

再来分析 Codec2Client _listComponents(…) 干了什么？

1. 调用 getServiceName() 获取服务名；
2. mBase 是指向 IComponentStore 的指针，调用其 listComponents(…) 实际由 hal 层服务实现，在匿名函数内将 hidl_vecIComponentStore::ComponentTraits 转化为 std::vectorC2Component::Traits，并将每一项的 owner 设置为服务名。

frameworks/av/media/codec2/hidl/client/client.cpp

```
std::vector<C2Component::Traits> Codec2Client::_listComponents(
        bool* success) const {
    std::vector<C2Component::Traits> traits;
    std::string const& serviceName = getServiceName();
    Return<void> transStatus = mBase->listComponents(
            [&traits, &serviceName](Status s,
                   const hidl_vec<IComponentStore::ComponentTraits>& t) {
                if (s != Status::OK) {
                    LOG(DEBUG) << "_listComponents -- call failed: "
                               << static_cast<c2_status_t>(s) << ".";
                    return;
                }
                traits.resize(t.size());
                for (size_t i = 0; i < t.size(); ++i) {
                    if (!objcpy(&traits[i], t[i])) {
                        LOG(ERROR) << "_listComponents -- corrupted output.";
                        return;
                    }
                    traits[i].owner = serviceName;
                }
            });
    if (!transStatus.isOk()) {
        LOG(ERROR) << "_listComponents -- transaction failed.";
        *success = false;
    } else {
        *success = true;
    }
    return traits;
}
10111213141516171819202122232425262728
```

getServiceName() 方法根据 mServiceIndex 直接从 GetServiceNames() 返回的容器中取出服务名。

frameworks/av/media/codec2/hidl/client/client.cpp

现在再来摸清楚 Codec2Client::_CreateFromIndex(…)，这里的 Base 是个啥？不难看出 Base 就是 IComponentStore 的别名。

Base::getService(…) 获取到 hal 远端服务的代理，最后初始化了 Codec2Client，调用了其构造函数。这里会在 rk3399 android 10 平台上打印以下 Log：

Codec2Client: Creating a Codec2 client to service “software”

Codec2Client: Client to Codec2 service “software” created

frameworks/av/media/codec2/hidl/client/client.cpp

```
std::shared_ptr<Codec2Client> Codec2Client::_CreateFromIndex(size_t index) {
    std::string const& name = GetServiceNames()[index];
    LOG(INFO) << "Creating a Codec2 client to service \"" << name << "\"";
    sp<Base> baseStore = Base::getService(name);
    CHECK(baseStore) << "Codec2 service \"" << name << "\""
                        " inaccessible for unknown reasons.";
    LOG(INFO) << "Client to Codec2 service \"" << name << "\" created";
    return std::make_shared<Codec2Client>(baseStore, index);
}

```

匿名函数内调用远端 ComponentStore getConfigurable() 等初始化 Codec2ConfigurableClient。初始化 mBase 和 mServiceIndex 字段，获取远端 ClientManager 代理。

frameworks/av/media/codec2/hidl/client/client.cpp

```
Codec2Client::Codec2Client(const sp<IComponentStore>& base,
                           size_t serviceIndex)
      : Configurable{
            [base]() -> sp<IConfigurable> {
                Return<sp<IConfigurable>> transResult =
                        base->getConfigurable();
                return transResult.isOk() ?
                        static_cast<sp<IConfigurable>>(transResult) :
                        nullptr;
            }()
        },
        mBase{base},
        mServiceIndex{serviceIndex} {
    Return<sp<IClientManager>> transResult = base->getPoolClientManager();
    if (!transResult.isOk()) {
        LOG(ERROR) << "getPoolClientManager -- transaction failed.";
    } else {
        mHostPoolManager = static_cast<sp<IClientManager>>(transResult);
    }
}
1011121314151617181920
```

