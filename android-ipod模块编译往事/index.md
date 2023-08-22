# Android iPod模块编译往事


因项目需要，在Android12的新平台中加入iPod模块功能，由于遵循不重复造轮子的思想，从老项目中复制了代码过来，期间发现了一些问题，在此记录一下

### error: "lib_name" (native:vendor) can not link against other_lib_name (native:platform)
这是由于Google为了解耦，对Android的编译做了个限制：vendor模块不能链接platform模块提供的库，网上查到的修改方案有：
1. 将应用的库复制出来放在vendor里
2. 伪造平台
3. 使用Google内部编译系统

这几个方法都不好，复制出来太不优雅，剩下两个方案太复杂。其实还有一个方法，既然vendor模块不能链接platform模块，那把vendor模块的编译目标改成platform里不就好了

解决方案：在对应mk文件中，加入/修改 `LOCAL_PRODUCT_MODULE := true`

原来可能是 LOCAL_VENDOR_MODULE ，幸好mk文件中可以定义目标位置，这样就能开始编译了


### fatal error: 'JNIHelp.h' file not found
看起来只是很普通的文件找不到问题，但问题在于这个头文件处在原生代码里的 libnativehelper 里，在mk的 LOCAL_SHARED_LIBRARIES 字段中也已经引入了这个模块，却还是找不到。

查询得到Android12 中，libnativehelper 有些改动
https://android.googlesource.com/platform/libnativehelper/+/refs/tags/android-12.0.0_r2/Android.bp

源码定义：
> libnativehelper/include/nativehelper/JNIHelp.h

那么在所有 include JNIHelper 的地方都改成 `#include <nativehelper/JNIHelper.h` 即可


### system/core/include/utils/String16.h:169:9: error: statement not allowed in constexpr function
        for (size_t i = 0; i < N - 1; ++i) r.data[i] = s[i];

不知道为什么，Android12 的源码中该位置会出现因为C++17标准，使用 constexpr 语句而变得非法的for循环
