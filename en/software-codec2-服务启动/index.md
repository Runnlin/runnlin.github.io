# 

想要支持 Codec2 [框架](https://so.csdn.net/so/search?q=%E6%A1%86%E6%9E%B6&spm=1001.2101.3001.7020)，需要实现其要求的 hal 接口，并实现相应的支撑服务。这里以源码中实现 software Codec2 服务启动为例进行分析。

software Codec2 服务启动位于 CodecServiceRegistrant.cpp 中。在 rk3399 Android 10 平台上可以看到启动 Log：

CodecServiceRegistrant: Creating software Codec2 service…

CodecServiceRegistrant: Software Codec2 service created.

可是 RegisterCodecServices() 并不是 main 函数，说明这并不是代码起点。

1. 调用 android::GetCodec2PlatformComponentStore() 创建 C2PlatformComponentStore 对象；
2. 将 C2PlatformComponentStore 对象作为入参构建 utils::ComponentStore 对象。

frameworks/av/services/mediacodec/registrant/CodecServiceRegistrant.cpp

```
extern "C" void RegisterCodecServices() {
    using namespace ::android::hardware::media::c2::V1_0;
    LOG(INFO) << "Creating software Codec2 service...";
    android::sp<IComponentStore> store =
        new utils::ComponentStore(
                android::GetCodec2PlatformComponentStore());
    if (store == nullptr) {
        LOG(ERROR) <<
                "Cannot create software Codec2 service.";
    } else {
        if (store->registerAsService("software") != android::OK) {
            LOG(ERROR) <<
                    "Cannot register software Codec2 service.";
        } else {
            LOG(INFO) <<
                    "Software Codec2 service created.";
        }
    }
}
12345678910111213141516171819
```

重点来了，main_swcodecservice.cpp main 函数中发现了启动 software Codec2 service 的调用，启动就是从这里开始的。

frameworks/av/services/mediacodec/main_swcodecservice.cpp

```
extern "C" void RegisterCodecServices();

int main(int argc __unused, char** argv)
{
    LOG(INFO) << "media swcodec service starting";
    signal(SIGPIPE, SIG_IGN);
    SetUpMinijail(kSystemSeccompPolicyPath, kVendorSeccompPolicyPath);
    strcpy(argv[0], "media.swcodec");

    ::android::hardware::configureRpcThreadpool(64, false);

    RegisterCodecServices();

    ::android::hardware::joinRpcThreadpool();
}

12345678910111213141516
```

android::GetCodec2PlatformComponentStore() 函数定义在 C2PlatformSupport.h 中。注释解释这个函数的作用是返回平台组件仓库（store），如果返回为空说明获取失败了。

frameworks/av/media/codec2/vndk/include/C2PlatformSupport.h

```
namespace android {
/**
 * Returns the platform component store.
 * \retval nullptr if the platform component store could not be obtained
 */
std::shared_ptr<C2ComponentStore> GetCodec2PlatformComponentStore();
}

```

1. 创建一个指向 C2ComponentStore 的“弱”指针；
2. “弱”指针上调用 C2ComponentStore lock() 获取对象；
3. 如果上一步无法获取，调用 C2PlatformComponentStore 构造函数构造 C2PlatformComponentStore 对象。

第一次启动务必会调用到 C2PlatformComponentStore 无参构造器。

frameworks/av/media/codec2/vndk/C2Store.cpp

```
std::shared_ptr<C2ComponentStore> GetCodec2PlatformComponentStore() {
    static std::mutex mutex;
    static std::weak_ptr<C2ComponentStore> platformStore;
    std::lock_guard<std::mutex> lock(mutex);
    std::shared_ptr<C2ComponentStore> store = platformStore.lock();
    if (store == nullptr) {
        store = std::make_shared<C2PlatformComponentStore>();
        platformStore = store;
    }
    return store;
}
12345678910
```

初始化了三个变量 mVisited、mReflector 和 mInterface。然后将各种 so 路径添加到 std::map<C2String, ComponentLoader>。不难看出这些 so 就是各种软编解码具体实现。

frameworks/av/media/codec2/vndk/C2Store.cpp

```
C2PlatformComponentStore::C2PlatformComponentStore()
    : mVisited(false),
      mReflector(std::make_shared<C2ReflectorHelper>()),
      mInterface(mReflector) {

    auto emplace = [this](const char *libPath) {
        mComponents.emplace(libPath, libPath);
    };

    // TODO: move this also into a .so so it can be updated
    emplace("libcodec2_soft_aacdec.so");
    emplace("libcodec2_soft_aacenc.so");
    emplace("libcodec2_soft_amrnbdec.so");
    emplace("libcodec2_soft_amrnbenc.so");
    emplace("libcodec2_soft_amrwbdec.so");
    emplace("libcodec2_soft_amrwbenc.so");
    emplace("libcodec2_soft_av1dec.so");
    emplace("libcodec2_soft_avcdec.so");
    emplace("libcodec2_soft_avcenc.so");
    emplace("libcodec2_soft_flacdec.so");
    emplace("libcodec2_soft_flacenc.so");
    emplace("libcodec2_soft_g711alawdec.so");
    emplace("libcodec2_soft_g711mlawdec.so");
    emplace("libcodec2_soft_gsmdec.so");
    emplace("libcodec2_soft_h263dec.so");
    emplace("libcodec2_soft_h263enc.so");
    emplace("libcodec2_soft_hevcdec.so");
    emplace("libcodec2_soft_hevcenc.so");
    emplace("libcodec2_soft_mp3dec.so");
    emplace("libcodec2_soft_mpeg2dec.so");
    emplace("libcodec2_soft_mpeg4dec.so");
    emplace("libcodec2_soft_mpeg4enc.so");
    emplace("libcodec2_soft_opusdec.so");
    emplace("libcodec2_soft_opusenc.so");
    emplace("libcodec2_soft_rawdec.so");
    emplace("libcodec2_soft_vorbisdec.so");
    emplace("libcodec2_soft_vp8dec.so");
    emplace("libcodec2_soft_vp8enc.so");
    emplace("libcodec2_soft_vp9dec.so");
    emplace("libcodec2_soft_vp9enc.so");
}
12345678910111213141516171819202122232425262728293031323334353637383940
```

1. 构造 CachedConfigurable 对象赋值给 mConfigurable；
2. 调用 android::GetCodec2PlatformComponentStore() 获取指向 C2ComponentStore 的共享指针；
3. 调用 SetPreferredCodec2ComponentStore(…) 如果平台分配器 store 是活的使用 C2ComponentStore 共享指针进行更新；
4. 调用 C2ComponentStore getParamReflector() 检索结构描述符；
5. CachedConfigurable 对象调用其 init(…) 函数初始化。

frameworks/av/media/codec2/hidl/1.0/utils/ComponentStore.cpp

```
ComponentStore::ComponentStore(const std::shared_ptr<C2ComponentStore>& store)
      : mConfigurable{new CachedConfigurable(std::make_unique<StoreIntf>(store))},
        mStore{store} {

    std::shared_ptr<C2ComponentStore> platformStore = android::GetCodec2PlatformComponentStore();
    SetPreferredCodec2ComponentStore(store);

    // Retrieve struct descriptors
    mParamReflector = mStore->getParamReflector();

    // Retrieve supported parameters from store
    mInit = mConfigurable->init(this);
}
123456789101112
```

CachedConfigurable 构造函数非常简单，仅仅将入参 ConfigurableC2Intf（实际为指向 StoreIntf 的结构体）（通用 Codec 2.0 接口包装器） 指针用来初始化其 mIntf 字段。

frameworks/av/media/codec2/hidl/1.0/utils/Configurable.cpp

```
CachedConfigurable::CachedConfigurable(
        std::unique_ptr<ConfigurableC2Intf>&& intf)
      : mIntf{std::move(intf)} {
}

```

1. 使用入参更新全局变量 gPreferredComponentStore，实际是一个 std::shared_ptr 指针；
2. 如果平台分配器 store 是活的，使用 C2ComponentStore 共享指针进行更新。

frameworks/av/media/codec2/vndk/C2Store.cpp

```
namespace {
    std::mutex gPreferredComponentStoreMutex;
    std::shared_ptr<C2ComponentStore> gPreferredComponentStore;

    std::mutex gPlatformAllocatorStoreMutex;
    std::weak_ptr<C2PlatformAllocatorStoreImpl> gPlatformAllocatorStore;
}
......
void SetPreferredCodec2ComponentStore(std::shared_ptr<C2ComponentStore> componentStore) {
    static std::mutex mutex;
    std::lock_guard<std::mutex> lock(mutex); // don't interleve set-s

    // update preferred store
    {
        std::lock_guard<std::mutex> lock(gPreferredComponentStoreMutex);
        gPreferredComponentStore = componentStore;
    }

    // update platform allocator's store as well if it is alive
    std::shared_ptr<C2PlatformAllocatorStoreImpl> allocatorStore;
    {
        std::lock_guard<std::mutex> lock(gPlatformAllocatorStoreMutex);
        allocatorStore = gPlatformAllocatorStore.lock();
    }
    if (allocatorStore) {
        allocatorStore->setComponentStore(componentStore);
    }
}
123456789101112131415161718192021222324252627
```

1. 使用入参更新 _mComponentStore 字段；
2. 调用 UseComponentStoreForIonAllocator() 函数，给 Ion 分配器使用组件 store。

frameworks/av/media/codec2/vndk/C2Store.cpp

```
void C2PlatformAllocatorStoreImpl::setComponentStore(std::shared_ptr<C2ComponentStore> store) {
    // technically this set lock is not needed, but is here for safety in case we add more
    // getter orders
    std::lock_guard<std::mutex> lock(_mComponentStoreSetLock);
    {
        std::lock_guard<std::mutex> lock(_mComponentStoreReadLock);
        _mComponentStore = store;
    }
    std::shared_ptr<C2AllocatorIon> allocator;
    {
        std::lock_guard<std::mutex> lock(gIonAllocatorMutex);
        allocator = gIonAllocator.lock();
    }
    if (allocator) {
        UseComponentStoreForIonAllocator(allocator, store);
    }
}
110111213141516
```

1. minUsage 赋初值等于 0，maxUsage 初值则是结合了 C2MemoryUsage::CPU_READ 和 C2MemoryUsage::CPU_WRITE 的期望值；
2. 调用 getpagesize() 获取页面大小赋值给 blockSize；
3. 构建一个查询条件容器（包括 usage 和 capacity） std::vector，调用 C2ComponentStore querySupportedValues_sm(…) 进行查询；
4. 根据查询结果更新 minUsage、maxUsage 和 blockSize；
5. 构造一个函数指针 C2AllocatorIon::UsageMapperFn mapper 指向匿名函数，匿名函数的作用是调用 C2ComponentStore config_sm(…) 进行配置 C2StoreIonUsageInfo（此结构内部包含了最小和最大使用量以及块大小 ），配置成功后更新匿名函数入参 align、heapMask 和 flags；
6. 调用 C2AllocatorIon setUsageMapper(…) 设置 Mapper。

frameworks/av/media/codec2/vndk/C2Store.cpp

```
void UseComponentStoreForIonAllocator(
        const std::shared_ptr<C2AllocatorIon> allocator,
        std::shared_ptr<C2ComponentStore> store) {
    C2AllocatorIon::UsageMapperFn mapper;
    uint64_t minUsage = 0;
    uint64_t maxUsage = C2MemoryUsage(C2MemoryUsage::CPU_READ, C2MemoryUsage::CPU_WRITE).expected;
    size_t blockSize = getpagesize();

    // query min and max usage as well as block size via supported values
    C2StoreIonUsageInfo usageInfo;
    std::vector<C2FieldSupportedValuesQuery> query = {
        C2FieldSupportedValuesQuery::Possible(C2ParamField::Make(usageInfo, usageInfo.usage)),
        C2FieldSupportedValuesQuery::Possible(C2ParamField::Make(usageInfo, usageInfo.capacity)),
    };
    c2_status_t res = store->querySupportedValues_sm(query);
    if (res == C2_OK) {
        if (query[0].status == C2_OK) {
            const C2FieldSupportedValues &fsv = query[0].values;
            if (fsv.type == C2FieldSupportedValues::FLAGS && !fsv.values.empty()) {
                minUsage = fsv.values[0].u64;
                maxUsage = 0;
                for (C2Value::Primitive v : fsv.values) {
                    maxUsage |= v.u64;
                }
            }
        }
        if (query[1].status == C2_OK) {
            const C2FieldSupportedValues &fsv = query[1].values;
            if (fsv.type == C2FieldSupportedValues::RANGE && fsv.range.step.u32 > 0) {
                blockSize = fsv.range.step.u32;
            }
        }

        mapper = [store](C2MemoryUsage usage, size_t capacity,
                         size_t *align, unsigned *heapMask, unsigned *flags) -> c2_status_t {
            if (capacity > UINT32_MAX) {
                return C2_BAD_VALUE;
            }
            C2StoreIonUsageInfo usageInfo = { usage.expected, capacity };
            std::vector<std::unique_ptr<C2SettingResult>> failures; // TODO: remove
            c2_status_t res = store->config_sm({&usageInfo}, &failures);
            if (res == C2_OK) {
                *align = usageInfo.minAlignment;
                *heapMask = usageInfo.heapMask;
                *flags = usageInfo.allocFlags;
            }
            return res;
        };
    }

    allocator->setUsageMapper(mapper, minUsage, maxUsage, blockSize);
}
123456789101112131415161718192021222324252627282930313233343536373839404142434445464748495051
```

根据注释不难理解 UsageMapperFn 的作用，分配器使用的 usage mapper 函数，函数入参：(usage, capacity) ，函数返回值 ：(align, heapMask, flags)。容量与默认块大小（默认页大小）对齐，以减少缓存开销。

setUsageMapper(…) 的作用是更新用于后续新分配的 usage mapper，以及支持的最小和最大 usage 掩码和用于 mapper 的默认块大小。

frameworks/av/media/codec2/vndk/include/C2AllocatorIon.h

```
namespace android {

class C2AllocatorIon : public C2Allocator {
public:
    // Usage mapper function used by the allocator
    //   (usage, capacity) => (align, heapMask, flags)
    //
    // capacity is aligned to the default block-size (defaults to page size) to reduce caching
    // overhead
    typedef std::function<c2_status_t(C2MemoryUsage, size_t,
                      /* => */ size_t*, unsigned*, unsigned*)> UsageMapperFn;
    ......
    /**
     * Updates the usage mapper for subsequent new allocations, as well as the supported
     * minimum and maximum usage masks and default block-size to use for the mapper.
     *
     * \param mapper this method is called to map Codec 2.0 buffer usage to ion flags
     *        required by the ion device
     * \param minUsage minimum buffer usage required for supported allocations (defaults to 0)
     * \param maxUsage maximum buffer usage supported by the ion allocator (defaults to SW_READ
     *        | SW_WRITE)
     * \param blockSize alignment used prior to calling |mapper| for the buffer capacity.
     *        This also helps reduce the size of cache required for caching mapper results.
     *        (defaults to the page size)
     */
    void setUsageMapper(
            const UsageMapperFn &mapper, uint64_t minUsage, uint64_t maxUsage, uint64_t blockSize);
    ......
}
12345678910111213141516171819202122232425262728
```

不难得出 maxUsage = C2MemoryUsage(C2MemoryUsage::CPU_READ, C2MemoryUsage::CPU_WRITE).expected，实际为 **0x001 | 0x100 = 0x101**。

frameworks/av/media/codec2/core/include/C2BufferBase.h

```
struct C2MemoryUsage {
// public:
    /**
     * Buffer read usage.
     */
    enum read_t : uint64_t {
        /** Buffer is read by the CPU. */
        CPU_READ        = 1 << 0,
        ......
    };

    /**
     * Buffer write usage.
     */
    enum write_t : uint64_t {
        /** Buffer is writted to by the CPU. */
        CPU_WRITE        = 1 << 2,
        ......
    };
    ......
    /** Create a usage from separate consumer and producer usage mask. \deprecated */
    inline C2MemoryUsage(uint64_t consumer, uint64_t producer)
        : expected(consumer | producer) { }
    ......
    uint64_t expected; // expected buffer usage
};
134567891011121314151617181920212223242526
```

由于是 android 10 系统所以实际一定满足 >= **ANDROID_API_L** 的条件。

bionic/libc/include/unistd.h

直接返回了 PAGE_SIZE，PAGE_SIZE 定义在 /sys/user.h 下，其值等于 4096，也就是 4KB 页面大小。至于这个地方为啥没使用 sysconf(3) 系统调用，注释中做了说明是为了防止静态库变的臃肿，因为会把 stdio 相关代码合进来。

bionic/libc/bionic/getpagesize.cpp

bionic/libc/include/sys/user.h

根据前面的分析，C2ComponentStore 实际指向 C2PlatformComponentStore。

frameworks/av/media/codec2/vndk/C2Store.cpp

```
c2_status_t C2PlatformComponentStore::querySupportedValues_sm(
        std::vector<C2FieldSupportedValuesQuery> &fields) const {
    return mInterface.querySupportedValues(fields, C2_MAY_BLOCK);
}

```

mInterface 初始化传入了指向 C2ReflectorHelper 的共享指针。具体 Interface 结构体构造过程不再展开。

frameworks/av/media/codec2/vndk/C2Store.cpp

```
class C2PlatformComponentStore : public C2ComponentStore {
    ......
    private:
    struct Interface : public C2InterfaceHelper {
        std::shared_ptr<C2StoreIonUsageInfo> mIonUsageInfo;

        Interface(std::shared_ptr<C2ReflectorHelper> reflector)
            : C2InterfaceHelper(reflector) {
            setDerivedInstance(this);

            struct Setter {
                static C2R setIonUsage(bool /* mayBlock */, C2P<C2StoreIonUsageInfo> &me) {
                    me.set().heapMask = ~0;
                    me.set().allocFlags = 0;
                    me.set().minAlignment = 0;
                    return C2R::Ok();
                }
            };

            addParameter(
                DefineParam(mIonUsageInfo, "ion-usage")
                .withDefault(new C2StoreIonUsageInfo())
                .withFields({
                    C2F(mIonUsageInfo, usage).flags({C2MemoryUsage::CPU_READ | C2MemoryUsage::CPU_WRITE}),
                    C2F(mIonUsageInfo, capacity).inRange(0, UINT32_MAX, 1024),
                    C2F(mIonUsageInfo, heapMask).any(),
                    C2F(mIonUsageInfo, allocFlags).flags({}),
                    C2F(mIonUsageInfo, minAlignment).equalTo(0)
                })
                .withSetter(Setter::setIonUsage)
                .build());
        }
    };
    ......
}
12345678910111213141516171819202122232425262728293031323334
```

回到主题，Interface querySupportedValues(…) 实际是调用父类 C2InterfaceHelper querySupportedValues(…) 实现的。同样 C2PlatformComponentStore::config_sm(…) 内部调用 Interface config(…) 实现，其实也是调用父类 C2InterfaceHelper config(…) 。

```
c2_status_t C2PlatformComponentStore::config_sm(
        const std::vector<C2Param*> &params,
        std::vector<std::unique_ptr<C2SettingResult>> *const failures) {
    return mInterface.config(params, C2_MAY_BLOCK, failures);
}

```

此处直接返回了 mReflector，这个字段是在 C2PlatformComponentStore 构造器中初始化的，其为指向 C2ReflectorHelper（实现参数反射，这个类是动态的，设计成由多个接口共享。这允许接口根据需要添加结构描述符） 的指针。

最后来分析一番 CachedConfigurable 对象调用其 init(…) 函数初始化。

主要功能为从存储区检索支持的参数。前面分析道 mIntf 指向 StoreIntf 结构体，所以此处调用实际为调用 StoreIntf querySupportedParams(…) 函数，最后调用 ComponentStore validateSupportedParams(…) 校验支持参数。

frameworks/av/media/codec2/hidl/1.0/utils/Configurable.cpp

```
c2_status_t CachedConfigurable::init(ComponentStore* store) {
    // 从存储区检索支持的参数
    c2_status_t init = mIntf->querySupportedParams(&mSupportedParams);
    c2_status_t validate = store->validateSupportedParams(mSupportedParams);
    return init == C2_OK ? C2_OK : validate;
}

```

StoreIntf querySupportedParams(…) 分发给了 C2ComponentStore querySupportedParams_nb(…) 方法。

frameworks/av/media/codec2/hidl/1.0/utils/ComponentStore.cpp

```
struct StoreIntf : public ConfigurableC2Intf {
    StoreIntf(const std::shared_ptr<C2ComponentStore>& store)
          : ConfigurableC2Intf{store ? store->getName() : "", 0},
            mStore{store} {
    }
    ......
    virtual c2_status_t querySupportedParams(
            std::vector<std::shared_ptr<C2ParamDescriptor>> *const params
            ) const override {
        return mStore->querySupportedParams_nb(params);
    }
    ......
protected:
    std::shared_ptr<C2ComponentStore> mStore;
};
}
123456789101112131415
```

mInterface 是在 C2PlatformComponentStore 构造器中初始化的，mInterface 是一个 Interface 结构（继承自 C2InterfaceHelper），所以调用其 querySupportedParams(…) 函数实际实现在其父类 C2InterfaceHelper 中。

frameworks/av/media/codec2/vndk/C2Store.cpp

```
c2_status_t C2PlatformComponentStore::querySupportedParams_nb(
        std::vector<std::shared_ptr<C2ParamDescriptor>> *const params) const {
    return mInterface.querySupportedParams(params);
}

```

C2InterfaceHelper::querySupportedParams(…) 内部仅仅调用了 C2InterfaceHelper::FactoryImpl querySupportedParams(…) 方法。

frameworks/av/media/codec2/vndk/util/C2InterfaceHelper.cpp

```
c2_status_t C2InterfaceHelper::querySupportedParams(
        std::vector<std::shared_ptr<C2ParamDescriptor>> *const params) const {
    std::lock_guard<std::mutex> lock(mMutex);
    return _mFactory->querySupportedParams(params);
}

```

遍历 _mParams（std::map<ParamRef, std::shared_ptr>）将其 value 转化为 C2ParamDescriptor 添加到入参提供的容器。

frameworks/av/media/codec2/vndk/util/C2InterfaceHelper.cpp

```
struct C2InterfaceHelper::FactoryImpl : public C2InterfaceHelper::Factory {
    ......
public:
    ......
    c2_status_t querySupportedParams(
            std::vector<std::shared_ptr<C2ParamDescriptor>> *const params) const {
        for (const auto &it : _mParams) {
            // TODO: change querySupportedParams signature?
            params->push_back(
                    std::const_pointer_cast<C2ParamDescriptor>(it.second->getDescriptor()));
        }
        // TODO: handle errors
        return C2_OK;
    }
    ......
}
1101112131415
```

逐个遍历入参容器中的 C2ParamDescriptor，将 map 中不存在的 CoreIndex-C2ParamDescriptor 结构对插入到 map（std::map<C2Param::CoreIndex, std::shared_ptr>）中。

frameworks/av/media/codec2/hidl/1.0/utils/ComponentStore.cpp

```
c2_status_t ComponentStore::validateSupportedParams(
        const std::vector<std::shared_ptr<C2ParamDescriptor>>& params) {
    c2_status_t res = C2_OK;

    for (const std::shared_ptr<C2ParamDescriptor> &desc : params) {
        if (!desc) {
            // All descriptors should be valid
            res = res ? res : C2_BAD_VALUE;
            continue;
        }
        C2Param::CoreIndex coreIndex = desc->index().coreIndex();
        std::lock_guard<std::mutex> lock(mStructDescriptorsMutex);
        auto it = mStructDescriptors.find(coreIndex);
        if (it == mStructDescriptors.end()) {
            std::shared_ptr<C2StructDescriptor> structDesc =
                    mParamReflector->describe(coreIndex);
            if (!structDesc) {
                // All supported params must be described
                res = C2_BAD_INDEX;
            }
            mStructDescriptors.insert({ coreIndex, structDesc });
        }
    }
    return res;
}
110111213141516171819202122232425
```
