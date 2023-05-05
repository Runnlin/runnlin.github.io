# Android.bp 文件解析


# **Android.bp 文件解析**

## Android.bp文件是什么？

Android.bp文件首先是Android系统的一种编译配置文件，是用来代替原来的Android.mk文件的。在Android7.0以前，Android都是使用make来组织各模块的编译，对应的编译配置文件就是Android.mk。在Android7.0开始，Google引入了ninja和kati来编译，为啥引入ninja？因为随着Android越来越庞大，module越来越多，编译时间也越来越久，而使用ninja在编译的并发处理上较make有很大的提升。Ninja的配置文件就是Android.bp，Android系统使用Blueprint和Soong工具来解析Android.bp转换生成ninja文件。为了兼容老的mk配置文件，Android当初也开发了Kati 工具来转换mk文件生成ninja，目前Android Q里边，还是支持Android.mk方式的。相信在将来的版本中，会彻底让mk文件废弃，同时Kati也就淘汰了，只保留bp配置方式，所以我们要提前学习bp。Blueprint和Soong工具的源码在Android/build/目录下，我们可以通过查阅相关代码来学习！

## Android.bp文件配置规则

### 【模块和属性】

Android.bp描述的编译对象都是以模块为组织单位的，定义一个模块从模块的类型开始，模块有不同的类型，模块包含一些属性，下面举一个例子来具体说明：

```  
cc_binary {
        name: ”avbctl”,
        defaults: [“avb_defaults”],
        static_libs: [
                  “libavb_user”,
                  “libfs_mgr”,
        ],
        shared_libs: [“libbase”],
        srcs: [“tools/avbctl/avbctl.cc”],
}
```

上面例子中的cc_binary就是模块类型，表示该模块目标为二进制可执行文件。
如果要编译一个APP，那么就使用android_app模块，要编译一个动态库，那么就使用cc_library_shared.soong工具支持的模块类型在android/build/soong/androidmk/cmd/androidmk/android.go 中可以查到，有以下：

● 模块类型后面用大括号“{}”将模块的所有属性包裹起来。

● 每个属性的名字和值用中间用冒号连接起来,属性值要用双引号””包裹起来(如果属性值是变量，变量不需要加双引号)：

name: ”avbctl”表示模块name属性的值为avbctl，就是类比Android.mk中的LOCAL_MODULE := avbctl。

模块的name属性是必须的，内容必须是独一无二的。如果属性被定义为数组，需要用中括号“[]”将数组的各元素包裹起来，每个元素中间用逗号“,”连接，一般常用的属性有name,srcs,cflags, cppflags, shared_libs,static_libs。

● 查看全部支持的模块和各个模块支持的属性定义，请查看这个网址：<https://ci.android.com/builds/submitted/6504066/linux/latest/view/soong_build.html>。

● cc_defaults模块比较特殊，它表示该模块的属性可以被其他模块重复引用，类似于我们的头文件被其他cpp文件引用，举例：

```
cc_defaults {
   name: "gzip_defaults",
   shared_libs: ["libz"],
   stl: "none",
}

cc_binary {
   name: "gzip",
   defaults: ["gzip_defaults"],
   /*这里等价于
   shared_libs: ["libz"],
    stl: "none",*/
   srcs: ["src/test/minigzip.c"],
}
```

● 属性可以使用列表数组的形式，也可以使用unix通配符，例如：”*.java”

● 每一条完整的属性定义语句加上逗号“，”表示结束

● 注释包括单行注释//和多行注释/**/

● 最重要的一点：目前android编译系统同时支持mk和bp两种，但是这两种是彼此单独运行的，所以bp中依赖的目标，例如动态库静态库目标，如果库是源代码的形式存在的，那么库的编译脚本必须也是通过bp文件编译才能被找到，否则用mk文件编译库，bp会提示找不到依赖的库目标。（paxdroid代码中因为要将framework相关内容编译到android的framework中，而android已经全部转还成bp，所以这一部分paxdroid也是使用的bp来编写的，不然会提示找不到）

更多的说明可以在android/build/soong/README.md文件中查找。
如果找不到想要的，可以在官网找：<https://ci.android.com/builds/submitted/6504066/linux/latest/view/soong_build.html>

### 【变量】

变量可以直接定义，使用“=”号赋值，例如：

```
avbctl_srcs = [“tools/avbctl/avbctl.cc”],
cc_binary {
        name: ”avbctl”,
        defaults: [“avb_defaults”],
        static_libs: [
                  “libavb_user”,
                  “libfs_mgr”,
        ],
        shared_libs: [“libbase”],
        srcs: avbctl_srcs,
}
```

### 【条件编译】

例如我们的mk文件中包括条件判断：

```
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)

LOCAL_MODULE := fs_mgr

ifeq ($(ENABLE_USER2ENG),true)
LOCAL_CFLAGS += -DALLOW_ADBD_DISABLE_VERITY=1
LOCAL_CFLAGS += -DENABLE_USER2ENG=1
endif

LOCAL_CFLAGS += -Wno-error=implicit-function-declaration

ifeq ($(shell if [ -d $(TOPDIR)paxdroid/external/libethtool ]; then echo "exist"; else echo "notexist"; fi;), exist)

LOCAL_SHARED_LIBRARIES +=libethtool
endif

include $(BUILD_EXECUTABLE)
```

我们先分析下上面两个条件的意思，如果变量ENABLE_USER2ENG的值为true，那么追加这两个编译参数，否则不追加。第二个条件是说如果存在paxdroid/external/libethtool这个目录，那么就添加libethtool这个动态库，否则不添加。

要转换为bp写法，就需要我们通过go语言写一个新文件，新建一个自定义类型的模块，来判断这些条件，比如我的修改是在system/core/fs_mgr/Android.bp,那么要在添加 system/core/fs_mgr/fs_mgr.go，

``` go
//这是申明当前的包名，就是你在哪个文件夹下，就写啥

package fs_mgr

//导入android的soong工具目录
import (
    "android/soong/android"
    "android/soong/cc"
   "fmt"
    //一般情况下不需要引用os包，这里要判断文件夹是否存在需要引用
    "os"

)

//初始化入口函数
func init() {
   // for DEBUG
   fmt.Println("init start")
    //注册模块名：fs_mgr_condition，模块名的方法入口是：fs_mgrDefaultsFactory方法
   android.RegisterModuleType("fs_mgr_condition", fs_mgrDefaultsFactory)
}

//实现fs_mgrDefaultsFactory方法：
func fs_mgrDefaultsFactory() (android.Module) {
    //如果我们编译的目标是bin文件，这里就是调用cc.DefaultsFactory()
    //如果编译目标是so库，那么调用的是cc.LibrarySharedFactory()。这里我们举例目标是bin文件：
   module := cc.DefaultsFactory()
    //添加装载时的钩子函数fs_mgrDefaults
   android.AddLoadHook(module, fs_mgrDefaults)
   return module
}

//实现钩子函数
func fs_mgrDefaults(ctx android.LoadHookContext) {
    //这里定义我们所有需要受到条件控制的变量，比如我们这里需要根据条件来控制Cflags和依赖
    //的动态库两个变量，所以我们只需要定义这两个即可，按照实际需求定义：
   type props struct {
       Cflags []string
        Shared_libs []string
   }
   p := &props{}
   p.Cflags = getCflags(ctx)
   p.Shared_libs = getShared_libs(ctx)
   ctx.AppendProperties(p)
}

//实现getCflags(ctx)方法
func getCflags(ctx android.BaseContext) ([]string) {
    var cppflags []string
    fmt.Println("ENABLE_USER2ENG:",
    ctx.AConfig().IsEnvTrue("ENABLE_USER2ENG"))
    if ctx.AConfig().IsEnvTrue("ENABLE_USER2ENG")
    {
         cppflags = append(cppflags, "-DALLOW_ADBD_DISABLE_VERITY=1", "-DENABLE_USER2ENG=1")
    }
    return cppflags
}

//实现getShared_libs(ctx)方法
func getShared_libs(ctx android.BaseContext) ([]string) {
    var shared_libs []string
    //判断文件是否存在，该方法返回两个参数，一个isExists，一个error
    isExists,error := PathExists ("paxdroid/external/libethtool")
    if(isExists && error == nil){
        //路径存在
        shared_libs = append(shared_libs, “libethtool”)
    }
    return shared_libs
}

//实现判断文件夹是否存在的工具方法：
func PathExists(path string) (bool, error) {
    _, err := os.Stat(path)
    if err == nil {
        return true, nil
    }
    if os.IsNotExist(err) {
        return false, nil
    }
    return false, err
}
```

go脚本写完了，相应的再Android.bp文件引用，如下：

``` go
// 这些个必须添加，编译刚刚写的那个go脚本需要的一些依赖
bootstrap_go_package {
   // 名字和包路径和刚刚写的go文件一致
   name: "soong-fs_mgr",
   pkgPath: "android/soong/fs_mgr",
   deps: [
       "blueprint",
        "blueprint-pathtools",
       "soong",
        "soong-android",
       "soong-cc",
        "soong-genrule",
   ],

//这里的srcs就写我们刚刚写的go脚本
   srcs: [
          "fs_mgr.go",
   ],
   pluginFor: ["soong_build"],
}
// fs_mgr_condition 是我们在go语言中自定义的模块类型，模块的类型是fs_mgr_condition，
// 这个模块名字叫做：fs_mgr_defaults
fs_mgr_condition {
   name: "fs_mgr_defaults",
}
//-----------------------------------------------------------------------
//以上所有代码是标准的添加自定义模块来实现条件控制的代码，我们可以记下来作为参考，
//下面的代码才是我们编译具体模块的：
cc_binary {
   name: "fs_mgr",

//这里引用上面定义的模块
   defaults: ["fs_mgr_defaults"],
   cppflags: ["-Wno-error=implicit-function-declaration "],
}
```

### 【操作符】

String类型、字符串列表类型和Map类型支持操作符“+”，例如：

``` c
binder_src_files = ["lib/libsystool_client/binder.c"],
cc_library_static {
   srcs: ["lib/libsystool_client/systool_client_static.c",
       "ipc/pipe_client.c",
   ] + binder_src_files,
}
```

