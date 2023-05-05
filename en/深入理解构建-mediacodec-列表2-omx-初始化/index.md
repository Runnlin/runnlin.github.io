# 深入理解构建 MediaCodec 列表：2-Omx 初始化



Omx 初始化分为四步，其中最为重要的一点在 Omx 初始化过程中会加载 libstagefrighthw.so。这个库是由供应商来实现的，实现多媒体编[解码](https://so.csdn.net/so/search?q=%E8%A7%A3%E7%A0%81&spm=1001.2101.3001.7020)芯片级支持。

![](content/assets/images/加载so.png)

Omx 初始化主要从其[构造函数](https://so.csdn.net/so/search?q=%E6%9E%84%E9%80%A0%E5%87%BD%E6%95%B0&spm=1001.2101.3001.7020)开始分析。

1. 创建 OMXMaster 对象，并初始化 mMaster 变量；
2. 初始化 MediaCodecsXmlParser 对象；
3. 调用 parseXmlFilesInSearchDirs() 方法从一组搜索目录解析顶级 XML 文件；
4. 调用 parseXmlPath(…) 方法解析顶级 XML 文件。

frameworks/av/media/libstagefright/omx/1.0/Omx.cpp

```
Omx::Omx() :
    mMaster(new OMXMaster()),
    mParser() {
    (void)mParser.parseXmlFilesInSearchDirs();
    (void)mParser.parseXmlPath(mParser.defaultProfilingResultsXmlPath);
}

```

1. 打开 /proc/**pid**/comm 路径获取进程名称；
2. 调用 addVendorPlugin() 添加供应商插件；
3. 调用 addPlatformPlugin() 添加平台插件。

frameworks/av/media/libstagefright/omx/OMXMaster.cpp

```
OMXMaster::OMXMaster() {

    pid_t pid = getpid();
    char filename[20];
    snprintf(filename, sizeof(filename), "/proc/%d/comm", pid);
    int fd = open(filename, O_RDONLY);
    if (fd < 0) {
      ALOGW("couldn't determine process name");
      strlcpy(mProcessName, "<unknown>", sizeof(mProcessName));
    } else {
      ssize_t len = read(fd, mProcessName, sizeof(mProcessName));
      if (len < 2) {
        ALOGW("couldn't determine process name");
        strlcpy(mProcessName, "<unknown>", sizeof(mProcessName));
      } else {
        // the name is newline terminated, so erase the newline
        mProcessName[len - 1] = 0;
      }
      close(fd);
    }

    addVendorPlugin();
    addPlatformPlugin();
}
123456789101112131415161718192021222324
```

addVendorPlugin() 和 addPlatformPlugin() 内部实际都是调用了 addPlugin(const char *libname)，只是传递的实参不一样，前一个为 libstagefrighthw.so，后一个为 libstagefright_softomx_plugin.so 字符串。第一个 so 是硬件厂家实现的，第二个是软件实现的。

addPlugin(const char *libname) 方法主要步骤如下：

1. 调用 android_load_sphal_library(…) 打开 so 库获取句柄；
2. 调用 dlsym(…) 解析 createOMXPlugin 方法符号，如果此符号不存在就去解析 _ZN7android15createOMXPluginEv 这个符号；
3. 调用 so 库中的 createOMXPlugin 函数；
4. 如果 CreateOMXPluginFunc 函数指针存在将其添加到 List 列表，并调用 addPlugin(OMXPluginBase *plugin) 重载方法进一步处理。

addPlugin(OMXPluginBase *plugin) 方法主要步骤如下：

1. while 循环调用 OMXPluginBase 子类具体实现 enumerateComponents(…) 方法枚举组件，如果组件已存在就跳过，不存在就添加到容器 KeyedVector<String8, OMXPluginBase *> 中；
2. 如果遍历期间发生错误则打印 Log：

OMX plugin failed w/ error 0x%08x after registering %zu components

frameworks/av/media/libstagefright/omx/OMXMaster.cpp

```
void OMXMaster::addVendorPlugin() {
    addPlugin("libstagefrighthw.so");
}

void OMXMaster::addPlatformPlugin() {
    addPlugin("libstagefright_softomx_plugin.so");
}

void OMXMaster::addPlugin(const char *libname) {
    void *libHandle = android_load_sphal_library(libname, RTLD_NOW);

    if (libHandle == NULL) {
        return;
    }

    typedef OMXPluginBase *(*CreateOMXPluginFunc)();
    CreateOMXPluginFunc createOMXPlugin =
        (CreateOMXPluginFunc)dlsym(
                libHandle, "createOMXPlugin");
    if (!createOMXPlugin)
        createOMXPlugin = (CreateOMXPluginFunc)dlsym(
                libHandle, "_ZN7android15createOMXPluginEv");

    OMXPluginBase *plugin = nullptr;
    if (createOMXPlugin) {
        plugin = (*createOMXPlugin)();
    }

    if (plugin) {
        mPlugins.push_back({ plugin, libHandle });
        addPlugin(plugin);
    } else {
        android_unload_sphal_library(libHandle);
    }
}

void OMXMaster::addPlugin(OMXPluginBase *plugin) {
    Mutex::Autolock autoLock(mLock);

    OMX_U32 index = 0;

    char name[128];
    OMX_ERRORTYPE err;
    while ((err = plugin->enumerateComponents(
                    name, sizeof(name), index++)) == OMX_ErrorNone) {
        String8 name8(name);

        if (mPluginByComponentName.indexOfKey(name8) >= 0) {
            ALOGE("A component of name '%s' already exists, ignoring this one.",
                 name8.string());

            continue;
        }

        mPluginByComponentName.add(name8, plugin);
    }

    if (err != OMX_ErrorNoMore) {
        ALOGE("OMX plugin failed w/ error 0x%08x after registering %zu "
             "components", err, mPluginByComponentName.size());
    }
}
1234567891011121314151617181920212223242526272829303132333435363738394041424344454647484950515253545556575859606162
```

我们以瑞芯微 rk3399 android 10 [源码](https://so.csdn.net/so/search?q=%E6%BA%90%E7%A0%81&spm=1001.2101.3001.7020)为例进行分析，当加载的是 libstagefrighthw.so，调用其 createOMXPlugin 函数。

createOMXPlugin() 函数内内部仅仅创建了 RKOMXPlugin 对象，RKOMXPlugin 构造函数内根据不同的宏定义进行分支。USE_ROCKCHIP_OMX 在 mk 文件中指定为 true，所以实际会调用 AddCore(“libOMX_Core.so”) 这行代码。

hardware/rockchip/librkvpu/libstagefrighthw/RKOMXPlugin.cpp

```
OMXPluginBase *createOMXPlugin() {
    return new RKOMXPlugin;
}

RKOMXPlugin::RKOMXPlugin()
{
#if defined(USE_ROCKCHIP_OMX)
   AddCore("libOMX_Core.so");
#endif
#if defined(USE_INTEL_MDP)
   AddCore("libmdp_omx_core.so");
#endif
}
12345678910111213
```

libstagefrighthw 在 rk3399 android 10 源码中编译脚本 mk 内容如下。可以看到在 USE_ROCKCHIP_OMX 为 true 时候会在编译 CFLAGS 上增加 -DUSE_ROCKCHIP_OMX 参数，代表使用 ROCKCHIP OMX。

hardware/rockchip/librkvpu/libstagefrighthw/Android.mk

```
......
USE_ROCKCHIP_OMX:=true

ifeq ($(USE_ROCKCHIP_OMX),true)
    LOCAL_CFLAGS += -DUSE_ROCKCHIP_OMX
endif

......

ifeq ($(USE_INTEL_MDP),true)
    LOCAL_CFLAGS += -DUSE_INTEL_MDP
endif
......
LOCAL_MODULE := libstagefrighthw
......
123456789101112131415
```

1. 判断入参是否为 libOMX_Core.so 这个字符串，如果是则置位 isRKCore；
2. 调用 dlopen(…) 打开 so 库获取句柄；
3. 调用 calloc(…) 给 RKOMXCore 结构分配空间，如果由于系统原因无法分配则关闭上一步打开的句柄；
4. 调用 dlsym(…) 解析插件库各种方法符号；
5. 如果 RKOMX_Init 函数符号解析不为空，对其进行方法调用；
6. 计算在给定 OMX 核心内注册的组件数量；
7. 向容器 Vector<RKOMXCore*> 中添加插件（RKOMXCore* 指针）。

hardware/rockchip/librkvpu/libstagefrighthw/RKOMXPlugin.cpp

```
OMX_ERRORTYPE RKOMXPlugin::AddCore(const char* coreName)
{
   bool isRKCore = false;
   if (!strcmp(coreName, "libOMX_Core.so")) {
       isRKCore = true;
   }
   void* libHandle = dlopen(coreName, RTLD_NOW);

   if (libHandle != NULL) {
        RKOMXCore* core = (RKOMXCore*)calloc(1,sizeof(RKOMXCore));

        if (!core) {
            dlclose(libHandle);
            return OMX_ErrorUndefined;
        }
        // set plugin lib handle and methods
        core->mLibHandle = libHandle;
		if (isRKCore) {
            core->mInit = (RKOMXCore::InitFunc)dlsym(libHandle, "RKOMX_Init");
            core->mDeinit = (RKOMXCore::DeinitFunc)dlsym(libHandle, "RKOMX_DeInit");

            core->mComponentNameEnum =
            (RKOMXCore::ComponentNameEnumFunc)dlsym(libHandle, "RKOMX_ComponentNameEnum");

            core->mGetHandle = (RKOMXCore::GetHandleFunc)dlsym(libHandle, "RKOMX_GetHandle");
            core->mFreeHandle = (RKOMXCore::FreeHandleFunc)dlsym(libHandle, "RKOMX_FreeHandle");

            core->mGetRolesOfComponentHandle =
                (RKOMXCore::GetRolesOfComponentFunc)dlsym(
                        libHandle, "RKOMX_GetRolesOfComponent");

		} else {
            core->mInit = (RKOMXCore::InitFunc)dlsym(libHandle, "OMX_Init");
            core->mDeinit = (RKOMXCore::DeinitFunc)dlsym(libHandle, "OMX_Deinit");

            core->mComponentNameEnum =
            (RKOMXCore::ComponentNameEnumFunc)dlsym(libHandle, "OMX_ComponentNameEnum");

            core->mGetHandle = (RKOMXCore::GetHandleFunc)dlsym(libHandle, "OMX_GetHandle");
            core->mFreeHandle = (RKOMXCore::FreeHandleFunc)dlsym(libHandle, "OMX_FreeHandle");

            core->mGetRolesOfComponentHandle =
                (RKOMXCore::GetRolesOfComponentFunc)dlsym(
                        libHandle, "OMX_GetRolesOfComponent");
		}
        if (core->mInit != NULL) {
            (*(core->mInit))();
        }
        if (core->mComponentNameEnum != NULL) {
            // calculating number of components registered inside given OMX core
            char tmpComponentName[OMX_MAX_STRINGNAME_SIZE] = { 0 };
            OMX_U32 tmpIndex = 0;
            while (OMX_ErrorNone == ((*(core->mComponentNameEnum))(tmpComponentName, OMX_MAX_STRINGNAME_SIZE, tmpIndex))) {
                tmpIndex++;
            ALOGI("OMX IL core %s: declares component %s", coreName, tmpComponentName);
            }
            core->mNumComponents = tmpIndex;
            ALOGI("OMX IL core %s: contains %d components", coreName, core->mNumComponents);
        }
        // add plugin to the vector
        mCores.push_back(core);
    }
    else {
        ALOGW("OMX IL core %s not found", coreName);
        return OMX_ErrorUndefined; // Do we need to return error message
    }
    return OMX_ErrorNone;
}
1234567891011121314151617181920212223242526272829303132333435363738394041424344454647484950515253545556575859606162636465666768
```

libOMX_Core.so 源码位于 /hardware/rockchip/omx_il/core 路径下。重点来看 RKOMX_Init 和 RKOMX_ComponentNameEnum 函数实现。

RKOMX_Init 函数实现主要步骤：

1. 调用 Rockchip_OMX_Component_Register(…) 注册瑞芯微 OMX 组件；
2. 调用 Rockchip_OMX_ResourceManager_Init(…) 进行资源管理初始化；
3. 调用 Rockchip_OSAL_MutexCreate(…) 创建 OSAL 互斥锁。

RKOMX_ComponentNameEnum 函数实现简单地多，在 gComponentList 全局组件列表中根据 index 返回组件名称。

/hardware/rockchip/omx_il/core/Rockchip_OMX_Core.c

```
OMX_API OMX_ERRORTYPE OMX_APIENTRY RKOMX_Init(void)
{
    OMX_ERRORTYPE ret = OMX_ErrorNone;

    FunctionIn();
    Rockchip_OSAL_MutexLock(&gMutex);
    gCount++;
    if (gInitialized == 0) {
        if (Rockchip_OMX_Component_Register(&gComponentList, &gComponentNum)) {
            ret = OMX_ErrorInsufficientResources;
            omx_err("Rockchip_OMX_Init : %s", "OMX_ErrorInsufficientResources");
            goto EXIT;
        }

        ret = Rockchip_OMX_ResourceManager_Init();
        if (OMX_ErrorNone != ret) {
            omx_err("Rockchip_OMX_Init : Rockchip_OMX_ResourceManager_Init failed");
            goto EXIT;
        }

        ret = Rockchip_OSAL_MutexCreate(&ghLoadComponentListMutex);
        if (OMX_ErrorNone != ret) {
            omx_err("Rockchip_OMX_Init : Rockchip_OSAL_MutexCreate(&ghLoadComponentListMutex) failed");
            goto EXIT;
        }

        gInitialized = 1;
        omx_trace("1Rockchip_OMX_Init : %s", "OMX_ErrorNone");
    }

EXIT:

    Rockchip_OSAL_MutexUnlock(&gMutex);
    FunctionOut();

    return ret;
}
......
OMX_API OMX_ERRORTYPE OMX_APIENTRY RKOMX_ComponentNameEnum(
    OMX_OUT OMX_STRING cComponentName,
    OMX_IN  OMX_U32 nNameLength,
    OMX_IN  OMX_U32 nIndex)
{
    OMX_ERRORTYPE ret = OMX_ErrorNone;

    FunctionIn();

    if (nIndex >= gComponentNum) {
        ret = OMX_ErrorNoMore;
        goto EXIT;
    }

    snprintf(cComponentName, nNameLength, "%s", gComponentList[nIndex].component.componentName);
    ret = OMX_ErrorNone;

EXIT:
    FunctionOut();

    return ret;
}
123456789101112131415161718192021222324252627282930313233343536373839404142434445464748495051525354555657585960
```

打开 libomxvpu_dec.so 和 libomxvpu_enc.so，获取符号 Rockchip_OMX_COMPONENT_Library_Register，调用 Rockchip_OMX_COMPONENT_Library_Register 函数获取 so 内部的组件数量和详细信息（包括组件名称、角色、角色数量和 lib 名称），最后将组件 list 和组件数量通过函数入参返回给调用者。

/hardware/rockchip/omx_il/core/Rockchip_OMX_Core.c

```
static const ROCKCHIP_COMPONENT_INFO kCompInfo[] = {
    { "rk.omx_dec", "libomxvpu_dec.so" },
    { "rk.omx_enc", "libomxvpu_enc.so" },
};

OMX_ERRORTYPE Rockchip_OMX_Component_Register(ROCKCHIP_OMX_COMPONENT_REGLIST **compList, OMX_U32 *compNum)
{
    OMX_ERRORTYPE  ret = OMX_ErrorNone;
    int            componentNum = 0, totalCompNum = 0;
    OMX_U32        i = 0;
    const char    *errorMsg;
    omx_err_f("in");

    int (*Rockchip_OMX_COMPONENT_Library_Register)(RockchipRegisterComponentType **rockchipComponents);
    RockchipRegisterComponentType **rockchipComponentsTemp;
    ROCKCHIP_OMX_COMPONENT_REGLIST *componentList;

    FunctionIn();

    componentList = (ROCKCHIP_OMX_COMPONENT_REGLIST *)Rockchip_OSAL_Malloc(sizeof(ROCKCHIP_OMX_COMPONENT_REGLIST) * MAX_OMX_COMPONENT_NUM);
    Rockchip_OSAL_Memset(componentList, 0, sizeof(ROCKCHIP_OMX_COMPONENT_REGLIST) * MAX_OMX_COMPONENT_NUM);

    for (i = 0; i < ARRAY_SIZE(kCompInfo); i++) {
        ROCKCHIP_COMPONENT_INFO com_inf = kCompInfo[i];
        OMX_PTR soHandle = NULL;
        omx_err_f("in");
        if ((soHandle = Rockchip_OSAL_dlopen(com_inf.lib_name, RTLD_NOW)) != NULL) {
            omx_err_f("in so name: %s", com_inf.lib_name);
            Rockchip_OSAL_dlerror();    /* clear error*/
            if ((Rockchip_OMX_COMPONENT_Library_Register = Rockchip_OSAL_dlsym(soHandle, "Rockchip_OMX_COMPONENT_Library_Register")) != NULL) {
                int i = 0;
                unsigned int j = 0;
                componentNum = (*Rockchip_OMX_COMPONENT_Library_Register)(NULL);
                omx_err_f("in num: %d", componentNum);
                rockchipComponentsTemp = (RockchipRegisterComponentType **)Rockchip_OSAL_Malloc(sizeof(RockchipRegisterComponentType*) * componentNum);
                for (i = 0; i < componentNum; i++) {
                    rockchipComponentsTemp[i] = Rockchip_OSAL_Malloc(sizeof(RockchipRegisterComponentType));
                    Rockchip_OSAL_Memset(rockchipComponentsTemp[i], 0, sizeof(RockchipRegisterComponentType));
                }
                (*Rockchip_OMX_COMPONENT_Library_Register)(rockchipComponentsTemp);

                for (i = 0; i < componentNum; i++) {
                    Rockchip_OSAL_Strcpy(componentList[totalCompNum].component.componentName, rockchipComponentsTemp[i]->componentName);
                    for (j = 0; j < rockchipComponentsTemp[i]->totalRoleNum; j++)
                        Rockchip_OSAL_Strcpy(componentList[totalCompNum].component.roles[j], rockchipComponentsTemp[i]->roles[j]);
                    componentList[totalCompNum].component.totalRoleNum = rockchipComponentsTemp[i]->totalRoleNum;

                    Rockchip_OSAL_Strcpy(componentList[totalCompNum].libName, com_inf.lib_name);

                    totalCompNum++;
                }
                for (i = 0; i < componentNum; i++) {
                    Rockchip_OSAL_Free(rockchipComponentsTemp[i]);
                }

                Rockchip_OSAL_Free(rockchipComponentsTemp);
            } else {
                if ((errorMsg = Rockchip_OSAL_dlerror()) != NULL)
                    omx_warn("dlsym failed: %s", errorMsg);
            }
            Rockchip_OSAL_dlclose(soHandle);
        }
        omx_err_f("Rockchip_OSAL_dlerror: %s", Rockchip_OSAL_dlerror());
    }
    *compList = componentList;
    *compNum = totalCompNum;

    FunctionOut();

    return ret;
}
1234567891011121314151617181920212223242526272829303132333435363738394041424344454647484950515253545556575859606162636465666768697071
```

Rockchip_OMX_Component_Register(…) 方法中进行了两次调用 Rockchip_OMX_COMPONENT_Library_Register(…) 方法，一次入参为 NULL，另一次入参是 RockchipRegisterComponentType 数组。

Rockchip_OMX_Component_Register(…) 方法中首先打开的是 libomxvpu_dec.so，因此先分析它的 Rockchip_OMX_COMPONENT_Library_Register(…) 函数。

1. 不难看出当入参为 NULL 时，直接 goto 到了 EXIT 标签处，也就是直接返回 SIZE_OF_DEC_CORE。这个值定义在 RkOMX_Core.h 中；
2. 当入参不为 NULL 时，for 循环遍历 dec_core 数组，对 RockchipRegisterComponentType 具体项进行 copy 赋值。dec_core 数组也定义在 RkOMX_Core.h 中 ；

hardware/rockchip/omx_il/[component](https://so.csdn.net/so/search?q=component&spm=1001.2101.3001.7020)/video/dec/library_register.c

```
OSCL_EXPORT_REF int Rockchip_OMX_COMPONENT_Library_Register(RockchipRegisterComponentType **rockchipComponents)
{
    FunctionIn();

    if (rockchipComponents == NULL)
        goto EXIT;

    OMX_U32 i = 0;
    for (i = 0; i < SIZE_OF_DEC_CORE; i++) {
        Rockchip_OSAL_Strcpy(rockchipComponents[i]->componentName, (OMX_PTR)dec_core[i].compName);
        Rockchip_OSAL_Strcpy(rockchipComponents[i]->roles[0], (OMX_PTR)dec_core[i].roles);
        rockchipComponents[i]->totalRoleNum = MAX_COMPONENT_ROLE_NUM;
        omx_err_f("in");
    }

EXIT:
    FunctionOut();

    return SIZE_OF_DEC_CORE;
}
1234567891011121314151617181920
```

dec_core 数组非常直观直接进行了定义，但是有些项条件编译进行了结合。确切地说，在 rk3399 android 10 平台上有些项就会被排除在外。现在再来看 SIZE_OF_DEC_CORE 的值就非常明确了。

hardware/rockchip/omx_il/component/video/dec/RkOMX_Core.h

```
static const omx_core_cb_type dec_core[] = {
    {
        "OMX.rk.video_decoder.avc",
        "video_decoder.avc"
    },

    {
        "OMX.rk.video_decoder.m4v",
        "video_decoder.mpeg4"
    },

    {
        "OMX.rk.video_decoder.h263",
        "video_decoder.h263"
    },

    {
        "OMX.rk.video_decoder.flv1",
        "video_decoder.flv1"
    },

    {
        "OMX.rk.video_decoder.m2v",
        "video_decoder.mpeg2"
    },
#ifndef AVS80
    {
        "OMX.rk.video_decoder.rv",
        "video_decoder.rv"
    },
#endif

#ifdef SUPPORT_VP6
    {
        "OMX.rk.video_decoder.vp6",
        "video_decoder.vp6"
    },
#endif

    {
        "OMX.rk.video_decoder.vp8",
        "video_decoder.vp8"
    },
#ifdef SUPPORT_VP9
    {
        "OMX.rk.video_decoder.vp9",
        "video_decoder.vp9"
    },
#endif

    {
        "OMX.rk.video_decoder.vc1",
        "video_decoder.vc1"
    },

    {
        "OMX.rk.video_decoder.wmv3",
        "video_decoder.wmv3"
    },
#ifdef SUPPORT_HEVC
    {
        "OMX.rk.video_decoder.hevc",
        "video_decoder.hevc"
    },
#endif
    {
        "OMX.rk.video_decoder.mjpeg",
        "video_decoder.mjpeg"
    },
#ifdef HAVE_L1_SVP_MODE
    {
        "OMX.rk.video_decoder.avc.secure",
        "video_decoder.avc"
    },

    {
        "OMX.rk.video_decoder.hevc.secure",
        "video_decoder.hevc"
    },

    {
        "OMX.rk.video_decoder.m2v.secure",
        "video_decoder.mpeg2"
    },

    {
        "OMX.rk.video_decoder.m4v.secure",
        "video_decoder.mpeg4"
    },

    {
        "OMX.rk.video_decoder.vp8.secure",
        "video_decoder.vp8"
    },

    {
        "OMX.rk.video_decoder.vp9.secure",
        "video_decoder.vp9"
    },
#endif
};

const unsigned int SIZE_OF_DEC_CORE = sizeof(dec_core) / sizeof(dec_core[0]);
123456789101112131415161718192021222324252627282930313233343536373839404142434445464748495051525354555657585960616263646566676869707172737475767778798081828384858687888990919293949596979899100101102103
```

具体哪些项没有开启需要结合 mk 脚本去分析。当 PLATFORM_SDK_VERSION >= 26 时，编译选项会添加 AVS80，android 10 为 29，所以 AVS80 被定义了，也就是说 AVS80 预编译这一项不再包含在内。BOARD_WIDEVINE_OEMCRYPTO_LEVEL、BOARD_SUPPORT_HEVC、BOARD_SUPPORT_VP6 和 BOARD_SUPPORT_VP9 从这个脚本不能看出，需进一步分析，实际上它们被定义在 device/rockchip/rk3399/BoardConfig.mk。

hardware/rockchip/omx_il/component/video/dec/Android.mk

```
......
ifeq (1,$(strip $(shell expr $(PLATFORM_SDK_VERSION) \>= 26)))
LOCAL_CFLAGS += -DAVS80
endif
......
LOCAL_SRC_FILES := \
	Rkvpu_OMX_VdecControl.c \
	Rkvpu_OMX_Vdec.c \
	library_register.c

LOCAL_MODULE := libomxvpu_dec
......
ifeq ($(BOARD_WIDEVINE_OEMCRYPTO_LEVEL), 1)
LOCAL_CFLAGS += -DHAVE_L1_SVP_MODE=ON
endif
......
ifeq (1,$(strip $(shell expr $(PLATFORM_SDK_VERSION) \>= 26)))
LOCAL_CFLAGS += -DAVS80
endif

ifeq ($(filter %false, $(BOARD_SUPPORT_HEVC)), )
LOCAL_CFLAGS += -DSUPPORT_HEVC=1
endif

ifeq ($(filter %false, $(BOARD_SUPPORT_VP6)), )
LOCAL_CFLAGS += -DSUPPORT_VP6=1
endif

ifeq ($(filter %false, $(BOARD_SUPPORT_VP9)), )
LOCAL_CFLAGS += -DSUPPORT_VP9=1
endif
......
1234567891011121314151617181920212223242526272829303132
```

对于 rk3399 android 10 平台 BOARD_WIDEVINE_OEMCRYPTO_LEVEL 等于 3，3 不等于 1 也就是不满足上面的 ifeq 语句，编译选项不再添加 -DHAVE_L1_SVP_MODE=ON。BOARD_SUPPORT_VP9 等于 true，表示支持 VP9。但不支持 VP6，BOARD_SUPPORT_HEVC 没有定义，但经过 ifeq 语句 filter 函数的过滤就会打开添加 -DSUPPORT_HEVC=1 编译选项。

filter：过滤语句，过滤掉不符合指定的模式的内容，仅保留符合指定模式的内容。

个人猜测这里之所以这样写，就是为了在不定义某个变量时，默认其开启。

device/rockchip/rk3399/BoardConfig.mk

```
......
# Add widevine L3 support
BOARD_WIDEVINE_OEMCRYPTO_LEVEL := 3
......
#Config omx to support codec type.
BOARD_SUPPORT_VP9 := true
BOARD_SUPPORT_VP6 := false

```

再来看打开 libomxvpu_enc.so。

1. 同样当入参为 NULL 时，直接 goto 到了 EXIT 标签处，也就是直接返回 SIZE_OF_ENC_CORE。这个值定义在 RkOMX_Core.h 中；
2. 当入参不为 NULL 时，for 循环遍历 enc_core 数组，对 RockchipRegisterComponentType 具体项进行 copy 赋值。enc_core 数组也定义在 RkOMX_Core.h 中 ；

hardware/rockchip/omx_il/component/video/enc/library_register.c

```
OSCL_EXPORT_REF int Rockchip_OMX_COMPONENT_Library_Register(RockchipRegisterComponentType **rockchipComponents)
{
    FunctionIn();

    if (rockchipComponents == NULL)
        goto EXIT;

    unsigned int i = 0;
    for (i = 0; i < SIZE_OF_ENC_CORE; i++) {
        Rockchip_OSAL_Strcpy(rockchipComponents[i]->componentName, (OMX_PTR)enc_core[i].compName);
        Rockchip_OSAL_Strcpy(rockchipComponents[i]->roles[0], (OMX_PTR)enc_core[i].roles);
        rockchipComponents[i]->totalRoleNum = MAX_COMPONENT_ROLE_NUM;
    }

EXIT:
    FunctionOut();

    return SIZE_OF_ENC_CORE;
}
123456789101112131415161718
```

SUPPORT_HEVC_ENC 和 SUPPORT_VP8_ENC 都没有被定义，因此编译时不支持它们，也就是 rk3399 只支持两种编码器 OMX.rk.video_encoder.avc 和 OMX.IMG.rk_encoder.avc。

hardware/rockchip/omx_il/component/video/enc/RkOMX_Core.h

```
static const omx_core_cb_type enc_core[] = {
    {
        "OMX.rk.video_encoder.avc",
        "video_encoder.avc"
    },
#ifdef SUPPORT_HEVC_ENC
    {
        "OMX.rk.video_encoder.hevc",
        "video_encoder.hevc"
    },
#endif

#ifdef SUPPORT_VP8_ENC
    {
        "OMX.rk.video_encoder.vp8",
        "video_encoder.vp8"
    },
#endif

    {
        "OMX.IMG.rk_encoder.avc",
        "video_encoder.avc"
    },
};

const unsigned int SIZE_OF_ENC_CORE = sizeof(enc_core) / sizeof(enc_core[0]);
12345678910111213141516171819202122232425
```

BOARD_SUPPORT_HEVC_ENC 和 BOARD_SUPPORT_VP8_ENC 没有定义，filter 过滤 true 就会过滤不到，ifneq 此时就不满足条件，所以 -DSUPPORT_HEVC_ENC=1 和 -DSUPPORT_VP8_ENC=1 编译选项不再添加到编译过程。

hardware/rockchip/omx_il/component/video/enc/Android.mk

```
......
ifneq ($(filter %true, $(BOARD_SUPPORT_HEVC_ENC)), )
LOCAL_CFLAGS += -DSUPPORT_HEVC_ENC=1
endif

ifneq ($(filter %true, $(BOARD_SUPPORT_VP8_ENC)), )
LOCAL_CFLAGS += -DSUPPORT_VP8_ENC=1
endif
......
12345678
```

现在继续分析 RKOMX_Init(void) 函数中的 Rockchip_OMX_ResourceManager_Init() 方法具体调用做了些什么？我们看到只是创建了一把互斥锁 ghVideoRMComponentListMutex。

hardware/rockchip/omx_il/component/common/Rockchip_OMX_Resourcemanager.c

```
OMX_ERRORTYPE Rockchip_OMX_ResourceManager_Init()
{
    OMX_ERRORTYPE ret = OMX_ErrorNone;

    FunctionIn();
    ret = Rockchip_OSAL_MutexCreate(&ghVideoRMComponentListMutex);
    omx_trace("Rockchip_OSAL_MutexCreate ghVideoRMComponentListMutex 0x%x", ghVideoRMComponentListMutex);
    FunctionOut();

    return ret;
}
12345678910
```

Rockchip_OSAL_MutexCreate(…) 函数实现非常简单，内部有 pthread 函数进行支持。首先给 mutex 指针变量分配内存，然后调用 pthread_mutex_init(…) 进行初始化。

hardware/rockchip/omx_il/osal/Rockchip_OSAL_Mutex.c

```
OMX_ERRORTYPE Rockchip_OSAL_MutexCreate(OMX_HANDLETYPE *mutexHandle)
{
    pthread_mutex_t *mutex;

    mutex = (pthread_mutex_t *)Rockchip_OSAL_Malloc(sizeof(pthread_mutex_t));
    if (!mutex)
        return OMX_ErrorInsufficientResources;

    if (pthread_mutex_init(mutex, NULL) != 0) {
        Rockchip_OSAL_Free(mutex);
        return OMX_ErrorUndefined;
    }

    *mutexHandle = (OMX_HANDLETYPE)mutex;
    return OMX_ErrorNone;
}
123456789101112131415
```

OMXMaster() 构造函数中最后一行调用了 addPlatformPlugin()，也就是添加软编码插件库。软编码插件库编译 bp 定义在 frameworks/av/media/libstagefright/omx/Android.bp 中，编译源文件只使用到了 SoftOMXPlugin.cpp。OMXMaster::addPlugin(const char *libname) 方法中对 createOMXPlugin 函数指针的调用实际上会调用到 frameworks/av/media/libstagefright/omx/SoftOMXPlugin.cpp 类中的 createOMXPlugin() 函数。

![addPlatformPlugin.jpg](content/assets/images/addPlatformPlugin.jpg)

frameworks/av/media/libstagefright/omx/Android.bp

```
cc_library_shared {
    name: "libstagefright_softomx_plugin",
    vendor_available: true,

    srcs: [
        "SoftOMXPlugin.cpp",
    ],

    export_include_dirs: [
        "include",
    ],

    header_libs: [
        "media_plugin_headers",
    ],

    export_header_lib_headers: [
        "media_plugin_headers",
    ],

    shared_libs: [
        "libstagefright_softomx",
        "libstagefright_foundation",
        "liblog",
        "libutils",
    ],

    cflags: [
        "-Werror",
        "-Wall",
        "-Wno-unused-parameter",
        "-Wno-documentation",
    ],

    sanitize: {
        misc_undefined: [
            "signed-integer-overflow",
            "unsigned-integer-overflow",
        ],
        cfi: true,
    },
}
11011121314151617181920212223242526272829303132333435363738394041
```

此方法中仅仅新建了一个 SoftOMXPlugin 对象，并返回。SoftOMXPlugin 构造函数啥也没干。

frameworks/av/media/libstagefright/omx/SoftOMXPlugin.cpp

```
extern "C" OMXPluginBase* createOMXPlugin() {
    ALOGI("createOMXPlugin");
    return new SoftOMXPlugin();
}
......
SoftOMXPlugin::SoftOMXPlugin() {
}

```

再来分析 SoftOMXPlugin::enumerateComponents(…) 函数。实现也非常简单，仅仅将 index 处的 kComponents 数组具体项中的 name 拷贝给入参 name。

frameworks/av/media/libstagefright/omx/SoftOMXPlugin.cpp

```
OMX_ERRORTYPE SoftOMXPlugin::enumerateComponents(
        OMX_STRING name,
        size_t /* size */,
        OMX_U32 index) {
    if (index >= kNumComponents) {
        return OMX_ErrorNoMore;
    }

    strcpy(name, kComponents[index].mName);

    return OMX_ErrorNone;
}
11011
```

现在看看 kComponents 变量，它的每一项是一个结构体（包含组件名称、库前缀名称和角色）。

frameworks/av/media/libstagefright/omx/SoftOMXPlugin.cpp

```
static const struct {
    const char *mName;
    const char *mLibNameSuffix;
    const char *mRole;

} kComponents[] = {
    // two choices for aac decoding.
    // configurable in media/libstagefright/data/media_codecs_google_audio.xml
    // default implementation
    { "OMX.google.aac.decoder", "aacdec", "audio_decoder.aac" },
    // alternate implementation
    { "OMX.google.xaac.decoder", "xaacdec", "audio_decoder.aac" },
    { "OMX.google.aac.encoder", "aacenc", "audio_encoder.aac" },
    { "OMX.google.amrnb.decoder", "amrdec", "audio_decoder.amrnb" },
    { "OMX.google.amrnb.encoder", "amrnbenc", "audio_encoder.amrnb" },
    { "OMX.google.amrwb.decoder", "amrdec", "audio_decoder.amrwb" },
    { "OMX.google.amrwb.encoder", "amrwbenc", "audio_encoder.amrwb" },
    { "OMX.google.h264.decoder", "avcdec", "video_decoder.avc" },
    { "OMX.google.h264.encoder", "avcenc", "video_encoder.avc" },
    { "OMX.google.hevc.decoder", "hevcdec", "video_decoder.hevc" },
    { "OMX.google.g711.alaw.decoder", "g711dec", "audio_decoder.g711alaw" },
    { "OMX.google.g711.mlaw.decoder", "g711dec", "audio_decoder.g711mlaw" },
    { "OMX.google.mpeg2.decoder", "mpeg2dec", "video_decoder.mpeg2" },
    { "OMX.google.h263.decoder", "mpeg4dec", "video_decoder.h263" },
    { "OMX.google.h263.encoder", "mpeg4enc", "video_encoder.h263" },
    { "OMX.google.mpeg4.decoder", "mpeg4dec", "video_decoder.mpeg4" },
    { "OMX.google.mpeg4.encoder", "mpeg4enc", "video_encoder.mpeg4" },
    { "OMX.google.mp3.decoder", "mp3dec", "audio_decoder.mp3" },
    { "OMX.google.vorbis.decoder", "vorbisdec", "audio_decoder.vorbis" },
    { "OMX.google.opus.decoder", "opusdec", "audio_decoder.opus" },
    { "OMX.google.vp8.decoder", "vpxdec", "video_decoder.vp8" },
    { "OMX.google.vp9.decoder", "vpxdec", "video_decoder.vp9" },
    { "OMX.google.vp8.encoder", "vpxenc", "video_encoder.vp8" },
    { "OMX.google.vp9.encoder", "vpxenc", "video_encoder.vp9" },
    { "OMX.google.raw.decoder", "rawdec", "audio_decoder.raw" },
    { "OMX.google.flac.decoder", "flacdec", "audio_decoder.flac" },
    { "OMX.google.flac.encoder", "flacenc", "audio_encoder.flac" },
    { "OMX.google.gsm.decoder", "gsmdec", "audio_decoder.gsm" },
};
167891011121314151617181920212223242526272829303132333435363738
```

Omx 构造函数中，还初始化了 mParser 变量，mParser 指向 MediaCodecsXmlParser 对象。MediaCodecsXmlParser 构造器中新建了 Impl 对象，并将其用来初始化 mImpl 变量。

frameworks/av/media/libstagefright/xmlparser/MediaCodecsXmlParser.cpp

Impl 无参构造函数中初始化了 mState 和 mParsingStatus（NO_INIT 未初始化）变量，和解析 [xml](https://so.csdn.net/so/search?q=xml&spm=1001.2101.3001.7020) 有关。

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
     // Computed longest common prefix
    Data mData;
    State mState;
    ......
    status_t mParsingStatus;

    Impl()
        : mState(&mData),
          mParsingStatus(NO_INIT) {
    }
    ......
}
123456789101112131415161718192021
```

接下来执行代码 Omx 构造函数中的 (void)mParser.parseXmlFilesInSearchDirs()。MediaCodecsXmlParser::parseXmlFilesInSearchDirs(…) 仅仅将任务委托给 Impl 具体实现去处理。我们看到这个函数有两个入参，但调用确是无参的，实际上使用了默认值，MediaCodecsXmlParser.h 文件中给出了答案。

![parseXmlFilesInSearchDirsSequenceDiagram.jpg](content/assets/images/parseXmlFilesInSearchDirsSequenceDiagram.jpg)

MediaCodecsXmlParser::Impl::parseXmlFilesInSearchDirs(…) 具体实现中，首先遍历所有文件名，然后调用 findFileInDirs(…) 在给定的目录中开始查找，找到后调用 parseXmlPath(…) 进行解析，最后调用 combineStatus(…) 确定最终返回值（返回状态）。

frameworks/av/media/libstagefright/xmlparser/MediaCodecsXmlParser.cpp

```
status_t MediaCodecsXmlParser::parseXmlFilesInSearchDirs(
        const std::vector<std::string> &fileNames,
        const std::vector<std::string> &searchDirs) {
    return mImpl->parseXmlFilesInSearchDirs(fileNames, searchDirs);
}
......
status_t MediaCodecsXmlParser::Impl::parseXmlFilesInSearchDirs(
        const std::vector<std::string> &fileNames,
        const std::vector<std::string> &searchDirs) {
    status_t res = NO_INIT;
    for (const std::string fileName : fileNames) {
        status_t err = NO_INIT;
        std::string path;
        if (findFileInDirs(searchDirs, fileName, &path)) {
            err = parseXmlPath(path);
        } else {
            ALOGD("Cannot find %s", path.c_str());
        }
        res = combineStatus(res, err);
    }
    return res;
}
123456789101112131415161718192021
```

搜索路径在 “/odm/etc”、 “/vendor/etc” 和 “/etc” 中进行查找，文件名称是 “media_codecs.xml” 和 “media_codecs_performance.xml”。

frameworks/av/media/libstagefright/xmlparser/include/media/stagefright/xmlparser/MediaCodecsXmlParser.h

```
namespace android {

class MediaCodecsXmlParser {
public:
    // Treblized media codec list will be located in /odm/etc or /vendor/etc.
    static std::vector<std::string> getDefaultSearchDirs() {
            return { "/odm/etc", "/vendor/etc", "/etc" };
    }
    static std::vector<std::string> getDefaultXmlNames() {
            return { "media_codecs.xml", "media_codecs_performance.xml" };
    }
    ......
    status_t parseXmlFilesInSearchDirs(
            const std::vector<std::string> &xmlFiles = getDefaultXmlNames(),
            const std::vector<std::string> &searchDirs = getDefaultSearchDirs());
    ......
} // namespace android
12345678910111213141516
```

在搜索目录列表中搜索文件。对于’ searchDirs ‘中的每个字符串’ searchDir '， ’ searchDir/fileName ‘将被测试是否它是一个有效的文件名。如果它是一个有效的文件名，’ searchDir/fileName '将存储在输出变量’outPath ‘中，函数将返回’ true ‘。否则，继续搜索，直到到达’ searchDirs ‘中的’ nullptr ‘元素，此时函数返回’ false '。

findFileInDirs(…) 内部调用了 fileExists(…) 用于判断文件是否存在。

> 
> 
> 
> stat 用来判断没有打开的文件，而 fstat 用来判断打开的文件。我们使用最多的属性是 st_mode。通过这个属性我们可以判断给定的文件是一个普通文件还是一个目录、连接等等。可以使用下面几个宏来判断：
> 
> **S_ISREG(st_mode) 是否是一个常规文件**
> 

frameworks/av/media/libstagefright/xmlparser/MediaCodecsXmlParser.cpp

```
bool fileExists(const std::string &path) {
    struct stat fileStat;
    return stat(path.c_str(), &fileStat) == 0 && S_ISREG(fileStat.st_mode);
}
......
bool findFileInDirs(
        const std::vector<std::string> &searchDirs,
        const std::string &fileName,
        std::string *outPath) {
    for (const std::string &searchDir : searchDirs) {
        std::string path = searchDir + "/" + fileName;
        if (fileExists(path)) {
            *outPath = path;
            return true;
        }
    }
    return false;
}
1234567891011121314151617
```

MediaCodecsXmlParser::parseXmlPath(…) 实际直接委托其实现 Impl 进行处理。

MediaCodecsXmlParser::Impl::parseXmlPath(…) 具体实现中再次判断文件是否存在，不存在直接返回。然后初始化 Parser 结构体，调用其 parseXmlFile() 方法进行 xml 文件解析。最后调用 combineStatus(…) 处理最终状态返回。

frameworks/av/media/libstagefright/xmlparser/MediaCodecsXmlParser.cpp

```
status_t MediaCodecsXmlParser::parseXmlPath(const std::string &path) {
    return mImpl->parseXmlPath(path);
}
......
status_t MediaCodecsXmlParser::Impl::parseXmlPath(const std::string &path) {
    std::lock_guard<std::mutex> guard(mLock);
    if (!fileExists(path)) {
        ALOGD("Cannot find %s", path.c_str());
        mParsingStatus = combineStatus(mParsingStatus, NAME_NOT_FOUND);
        return NAME_NOT_FOUND;
    }

    // save state (even though we should always be at toplevel here)
    State::RestorePoint rp = mState.createRestorePoint();
    Parser parser(&mState, path);
    parser.parseXmlFile();
    mState.restore(rp);

    if (parser.getStatus() != OK) {
        ALOGD("parseXmlPath(%s) failed with %s", path.c_str(), asString(parser.getStatus()));
    }
    mParsingStatus = combineStatus(mParsingStatus, parser.getStatus());
    return parser.getStatus();
}
1234567891011121314151617181920212223
```

MediaCodecsXmlParser::Impl::Parser::parseXmlFile() 方法中打开了 xml 文件，调用 XML Parser expat 库进行实际的 xml 解析。

> 
> 
> 
> expat 是使用 C 所写的 XML 解释器，采用流的方式来解析 XML 文件，并且基于事件通知型来调用分析到的数据，并不需要把所有 XML 文件全部加载到内存里，这样可以分析非常大的 XML 文件。由于 expat 库是由 XML 的主要负责人 James Clark 来实现的，因此它是符合 W3C 的 XML 标准的。
> 
> **XML_ParserCreate**(const XML_Char *encodingName) 参数一般为 NULL，函数返回一个 XML_Parser 类型指针
> 
> **XML_SetElementHandler**(XML_Parser parser, XML_StartElementHandler start, XML_EndElementHandler end) 第一个参数是 Parser 句柄，第二个和第三个参数则是整个 Parser 的核心，类型为 CallBack 的函数
> 
> typedef void (XMLCALL ***XML_StartElementHandler**) (void *userData, const XML_Char *name, const XML_Char **atts); 处理 xml 标签开始。 其中第一个参数 userData, 可以由函数 XML_SetUserData(XML_Parser parser, void *p) 设置
> 
> typedef void (XMLCALL *XML_EndElementHandler) (void *userData, const XML_Char *name); 处理 xml 标签结束。
> 

frameworks/av/media/libstagefright/xmlparser/MediaCodecsXmlParser.cpp

```
void MediaCodecsXmlParser::Impl::Parser::parseXmlFile() {
    const char *path = mPath.c_str();
    ALOGD("parsing %s...", path);
    FILE *file = fopen(path, "r");

    if (file == nullptr) {
        ALOGD("unable to open media codecs configuration xml file: %s", path);
        mStatus = NAME_NOT_FOUND;
        return;
    }

    mParser = std::shared_ptr<XML_ParserStruct>(
        ::XML_ParserCreate(nullptr),
        [](XML_ParserStruct *parser) { ::XML_ParserFree(parser); });
    LOG_FATAL_IF(!mParser, "XML_MediaCodecsXmlParserCreate() failed.");

    ::XML_SetUserData(mParser.get(), this);
    ::XML_SetElementHandler(mParser.get(), StartElementHandlerWrapper, EndElementHandlerWrapper);

    static constexpr int BUFF_SIZE = 512;
    // updateStatus(OK);
    if (mStatus == NO_INIT) {
        mStatus = OK;
    }
    while (mStatus == OK) {
        void *buff = ::XML_GetBuffer(mParser.get(), BUFF_SIZE);
        if (buff == nullptr) {
            ALOGD("failed in call to XML_GetBuffer()");
            mStatus = UNKNOWN_ERROR;
            break;
        }

        int bytes_read = ::fread(buff, 1, BUFF_SIZE, file);
        if (bytes_read < 0) {
            ALOGD("failed in call to read");
            mStatus = ERROR_IO;
            break;
        }

        XML_Status status = ::XML_ParseBuffer(mParser.get(), bytes_read, bytes_read == 0);
        if (status != XML_STATUS_OK) {
            PLOGD("malformed (%s)", ::XML_ErrorString(::XML_GetErrorCode(mParser.get())));
            mStatus = ERROR_MALFORMED;
            break;
        }

        if (bytes_read == 0) {
            break;
        }
    }

    mParser.reset();

    fclose(file);
    file = nullptr;
}
12345678910111213141516171819202122232425262728293031323334353637383940414243444546474849505152535455
```

StartElementHandlerWrapper(…) 和 EndElementHandlerWrapper(…) 回调函数在上一步进行了设置，我们知道其内部实际最终调用 Parser 的 startElementHandler(…) 和 endElementHandler(…) 方法。

frameworks/av/media/libstagefright/xmlparser/MediaCodecsXmlParser.cpp

```
// static
void MediaCodecsXmlParser::Impl::Parser::StartElementHandlerWrapper(
        void *me, const char *name, const char **attrs) {
    static_cast<MediaCodecsXmlParser::Impl::Parser*>(me)->startElementHandler(name, attrs);
}

// static
void MediaCodecsXmlParser::Impl::Parser::EndElementHandlerWrapper(void *me, const char *name) {
    static_cast<MediaCodecsXmlParser::Impl::Parser*>(me)->endElementHandler(name);
}
123456789
```

解析完成后我们就获得了 CodecMap、RoleMap 和 AttributeMap。

frameworks/av/media/libstagefright/xmlparser/MediaCodecsXmlParser.cpp

```
void MediaCodecsXmlParser::Impl::Parser::startElementHandler(
        const char *name, const char **attrs) {
    bool inType = true;
    Result err = NO_INIT;

    Section section = mState->section();

    // handle include at any level
    if (strEq(name, "Include")) {
        mState->enterSection(SECTION_INCLUDE);
        updateStatus(includeXmlFile(attrs));
        return;
    }

    // handle include section (top level)
    if (section == SECTION_INCLUDE) {
        if (strEq(name, "Included")) {
            return;
        }
        // imitate prior level
        section = mState->lastNonIncludeSection();
    }

    switch (section) {
        case SECTION_TOPLEVEL:
        {
            Section nextSection;
            if (strEq(name, "Decoders")) {
                nextSection = SECTION_DECODERS;
            } else if (strEq(name, "Encoders")) {
                nextSection = SECTION_ENCODERS;
            } else if (strEq(name, "Settings")) {
                nextSection = SECTION_SETTINGS;
            } else if (strEq(name, "MediaCodecs") || strEq(name, "Included")) {
                return;
            } else {
                break;
            }
            mState->enterSection(nextSection);
            return;
        }

        case SECTION_SETTINGS:
        {
            if (strEq(name, "Setting")) {
                err = addSetting(attrs);
            } else if (strEq(name, "Variant")) {
                err = addSetting(attrs, "variant-");
            } else if (strEq(name, "Domain")) {
                err = addSetting(attrs, "domain-");
            } else {
                break;
            }
            updateStatus(err);
            return;
        }

        case SECTION_DECODERS:
        case SECTION_ENCODERS:
        {
            if (strEq(name, "MediaCodec")) {
                err = enterMediaCodec(attrs, section == SECTION_ENCODERS);
                updateStatus(err);
                if (err != OK) { // skip this element on error
                    mState->enterSection(SECTION_UNKNOWN);
                } else {
                    mState->enterVariants(mState->codec().variantSet);
                    mState->enterSection(
                            section == SECTION_DECODERS ? SECTION_DECODER : SECTION_ENCODER);
                }
                return;
            }
            break;
        }

        case SECTION_DECODER:
        case SECTION_ENCODER:
        {
            if (strEq(name, "Quirk")) {
                err = addQuirk(attrs, "quirk::");
            } else if (strEq(name, "Attribute")) {
                err = addQuirk(attrs, "attribute::");
            } else if (strEq(name, "Alias")) {
                err = addAlias(attrs);
            } else if (strEq(name, "Type")) {
                err = enterType(attrs);
                if (err != OK) { // skip this element on error
                    mState->enterSection(SECTION_UNKNOWN);
                } else {
                    mState->enterSection(
                            section == SECTION_DECODER
                                    ? SECTION_DECODER_TYPE : SECTION_ENCODER_TYPE);
                }
            }
        }
        inType = false;
        FALLTHROUGH_INTENDED;

        case SECTION_DECODER_TYPE:
        case SECTION_ENCODER_TYPE:
        case SECTION_VARIANT:
        {
            // ignore limits and features specified outside of type
            if (!mState->inType()
                    && (strEq(name, "Limit") || strEq(name, "Feature") || strEq(name, "Variant"))) {
                PLOGD("ignoring %s specified outside of a Type", name);
                return;
            } else if (strEq(name, "Limit")) {
                err = addLimit(attrs);
            } else if (strEq(name, "Feature")) {
                err = addFeature(attrs);
            } else if (strEq(name, "Variant") && section != SECTION_VARIANT) {
                err = limitVariants(attrs);
                mState->enterSection(err == OK ? SECTION_VARIANT : SECTION_UNKNOWN);
            } else if (inType
                    && (strEq(name, "Alias") || strEq(name, "Attribute") || strEq(name, "Quirk"))) {
                PLOGD("ignoring %s specified not directly in a MediaCodec", name);
                return;
            } else if (err == NO_INIT) {
                break;
            }
            updateStatus(err);
            return;
        }

        default:
            break;
    }

    if (section != SECTION_UNKNOWN) {
        PLOGD("Ignoring unrecognized tag <%s>", name);
    }
    mState->enterSection(SECTION_UNKNOWN);
}

void MediaCodecsXmlParser::Impl::Parser::endElementHandler(const char *name) {
    // XMLParser handles tag matching, so we really just need to handle the section state here
    Section section = mState->section();
    switch (section) {
        case SECTION_INCLUDE:
        {
            // this could also be any of: Included, MediaCodecs
            if (strEq(name, "Include")) {
                mState->exitSection();
                return;
            }
            break;
        }

        case SECTION_SETTINGS:
        {
            // this could also be any of: Domain, Variant, Setting
            if (strEq(name, "Settings")) {
                mState->exitSection();
            }
            break;
        }

        case SECTION_DECODERS:
        case SECTION_ENCODERS:
        case SECTION_UNKNOWN:
        {
            mState->exitSection();
            break;
        }

        case SECTION_DECODER_TYPE:
        case SECTION_ENCODER_TYPE:
        {
            // this could also be any of: Alias, Limit, Feature
            if (strEq(name, "Type")) {
                mState->exitSection();
                mState->exitCodecOrType();
            }
            break;
        }

        case SECTION_DECODER:
        case SECTION_ENCODER:
        {
            // this could also be any of: Alias, Limit, Quirk, Variant
            if (strEq(name, "MediaCodec")) {
                mState->exitSection();
                mState->exitCodecOrType();
                mState->exitVariants();
            }
            break;
        }

        case SECTION_VARIANT:
        {
            // this could also be any of: Alias, Limit, Quirk
            if (strEq(name, "Variant")) {
                mState->exitSection();
                mState->exitVariants();
                return;
            }
            break;
        }

        default:
            break;
    }
}
```

最后再来看执行代码 Omx 构造函数中的 (void)mParser.parseXmlPath(mParser.defaultProfilingResultsXmlPath) 这段代码，defaultProfilingResultsXmlPath 路径等于 “/data/misc/media/media_codecs_profiling_results.xml”，也就是直接解析这个 xml 文件。

