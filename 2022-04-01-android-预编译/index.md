# Android 预编译简述


## 常用到的Android.mk编译目标，包括编译包、二进制文件、预编译

### 一. 常用的编译目标

1. BUILD_PACKAGE（既可以编apk，也可以编资源包文件，但是需要指定LOCAL_EXPORT_PACKAGE_RESOURCES:=true)
2. BUILD_JAVA_LIBRARY（java共享库）
3. BUILD_STATIC_JAVA_LIBRARY（java静态库）
4. BUILD_EXECUTABLE（执行文件）
5. BUILD_SHARED_LIBRARY（native共享库）
6. BUILD_STATIC_LIBRARY（native静态库）

### 二. 预编译模块

在实际的开发中，并不会像Android一样将所有的源码集中在一起编译，有很多apk、so、jar等都是预先编译好的，编译系统需要将这些二进制文件复制到生成的image文件中。

常用的方法是通过 *PRODUCT_COPY_FILES* 变量将这些文件直接复制到生成的image文件中；但是有些apk或jar包，需要使用系统的签名系统才能正常运行，这样用复制的方式就不行了。另外一些动态库文件可能是源码中的某些模块所以来的，用复制的方式也无法建立依赖关系，这将导致这些模块编译失败。Android可以通过定义预编译模块的方法来解决上面的问题。

定义一个预编译模块和定义一个普通的编译模块格式相似，不同的是*LOCAL_SRC_FILESS*变成不是源文件了，而是二进制文件。同时可以通过 *LOCAL_MODULE_CLASS* 来指定模块的类型，最后include的是 *BUILD_PREBUILT* 变量定义编译文件。

- 编译apk文件

1. include $(CLEAR_VARS)
2. LOCAL_MODULE := HelloWord.apk
3. LOCAL_SRC_FIELS := app/$(LOCAL_MODULE)
4. LOCAL_MODULE_TAGS := optional
5. xxxxxxxxxx binder_src_files = ["lib/libsystool_client/binder.c"],cc_library_static {   srcs: ["lib/libsystool_client/systool_client_static.c",       "ipc/pipe_client.c",   ] + binder_src_files,}c
6. LOCAL_CERTIFICATE := platform
7. include $ (BUILD_PREBUILT)

- 编译jar包

1. include $(CLEAR_VARS)
2. LOCAL_MODULE := libhelloword.jar
3. LOCAL_SRC_FIELS := app/$(LOCAL_MODULE)
4. LOCAL_MODULE_TAGS := optional
5. LOCAL_MODULE_CLASS := JAVA_LIBRAYIES
6. LOCAL_CERTIFICATE := platform
7. include  $(BUILD_PREBUILT)

- 定义动态库文件目标

1. include $(CLEAR_VARS)
2. LOCAL_MODULE := libhelloword_jni.so
3. LOCAL_MODULE_OWNER :=
4. LOCAL_SRC_FIELS := lib/$(LOCAL_MODULE)
5. LOCAL_MODULE_TAGS := optional
6. LOCAL_MODULE_CLASS := SHARED_LIBRAYIES
7. include $ (BUILD_PREBUILT)

- 编译可执行文件

1. include $(CLEAR_VARS)
2. LOCAL_MODULE := bootanimation
3. LOCAL_MODULE_OWNER :=
4. LOCAL_SRC_FIELS := bin/bootanimation
5. LOCAL_MODULE_TAGS := optional
6. LOCAL_MODULE_CLASS := EXECUTABLES
7. LOCAL_MODULE_PATH := $(TARGET_OUT)/bin
8. include $ (BUILD_PREBUILT)

### 注：BUILD_PREBUILT 作为预编译的主要字段

