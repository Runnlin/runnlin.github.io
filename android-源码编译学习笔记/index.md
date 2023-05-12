# Android 源码编译学习笔记


# **Android 源码编译学习笔记**

## 一、目录结构

| 目录              | 解析                            |
| --------------- | ----------------------------- |
| abi             | 应用程序二进制接口，生成libgabi++.so相关库文件 |
| vendor          | 各个厂商定制的相关文件，如驱动等              |
| boinic          | Android的C library，即C库文件       |
| bootable        | Android系统启动引导相关代码             |
| build           | 存放系统编译规则及generic等基础开发包配置      |
| cts             | Android兼容性测试套件标准              |
| tool            | 一些工具的                         |
| developers      | 开发者目录（包含一些事例）                 |
| development     | 应用程序开发相关的工具等                  |
| device          | 设备相关的抽象                       |
| docs            | 指导文档                          |
| external        | android使用到的一些开源的东西            |
| frameworks      | android核心框架，包含了java代码以及c代码    |
| hardware        | 硬件相关的HAL代码                    |
| libcore         | 核心库                           |
| libnativehelper | JNI调用相关的库                     |
| ndk             | ndk开发相关                       |
| out             | 编译后资源输出文件夹                    |
| packges         | 应用程序包                         |
| sdk             | sdk以及模拟器存放目录                  |

## 二、编译过程简介

[Google 官方文档](https://source.android.com/source/building.html)

### 1. 设置环境

使用envsetup.sh脚本初始化环境。请注意：将 source 替换成 . 可以省去一些字符，这种简写形式在文档中更为常用

    source build/envsetup.sh
    或
    . build/envsetup.sh

### 2. 选择目标

使用 lunch 选择要编译的目标。确切的配置可以作为参数进行传递，如

    lunch aosp_arm-eng

该命令表示针对模拟器进行完整编译，并且所有调试功能均处于启用状态。

如果没有提供仍和参数就运行命令，lunch将提示选择一个目标。

所有编译目标都采用 BUILD-BUILDTYPE 形式，其中 BUILD 是表示特定功能组合的代号。

BUILDTYPE 是以下类型之一：

| 编译类型      | 使用情况                                     |
| --------- | ---------------------------------------- |
| user      | 权限受限；适用于生产环境                             |
| userdebug | 与 user 类似，但具有 root 权限和可调式性；是进行调试时的首选编译类型 |
| eng       | 具有额外调试工具的开发配置                            |

### 3. 编译代码

您可以使用 make 编译任何代码。 GNU Make 可以借助 -jN 参数处理并行任务，通常使用的任务数 N 介于编译时所使用的计算机上硬件线程数的 1-2 倍之间

    make -j8

## 三、envsetup.sh 都做了些什么？

在分析envsetup.sh之前，我们先看看两个小问题：

1. 执行该脚本时有些退linux不是很熟悉的朋友经常会碰到envsetup.sh文件无可执行权限问题，使用下面命令加上即可：

    $chmod a+x envsetup.sh

2. 还有一个问题是经常出现的build/envsetup.sh: 1: Syntax error: "(" unexpected，别慌，由于我们使用的是Ubuntu，而Ubuntu默认使用的shell环境是dash，而envsetup.sh默认是指定bash的，所以我们只需要将dash切换到bash即可:

    执行$sudo dpkg-reconfigure dash命令，并选择“否”，最好重新打开终端

3. 我们先看看，执行了source build/envsetup.sh后有什么效果：

![envsetup](https://img-blog.csdn.net/20180621111907813?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L01vbmFMaXNhVGVhcnI=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

我们很直观的看到，直接includ了很多shell脚本，光看这里看不出什么，我们不妨看看源码！

```c
# Execute the contents of any vendorsetup.sh files we can find.
for f in `test -d device && find -L device -maxdepth 4 -name 'vendorsetup.sh' 2> /dev/null | sort` \
         `test -d vendor && find -L vendor -maxdepth 4 -name 'vendorsetup.sh' 2> /dev/null | sort`
do
    echo "including $f"
    . $f
done
unset f
```

上面的脚本代码依次查找{device, vendor, product}目录下的vendorsetup.sh文件，并分别导入到当前环境中来！

4. 我们继续翻envsetup.sh的源码你会发现，一直在定义函数而这些函数有很多我们会是我们后面直接用到的，比如说lunch、mm、mmm等等。

## 四、lunch 命令

1.首先看看lunch命令执行后出现了什么

![lunch](https://img-blog.csdn.net/20180622101109167?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L01vbmFMaXNhVGVhcnI=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

出现了上述信息，就是等待我们选择编译的平台，下表为不同版本的总结

| user                          | userdebug                     | eng                                      |
| ----------------------------- | ----------------------------- | ---------------------------------------- |
| 仅安装标签为 user 的模块               | 安装标签为 user、debug 的模块          | 安装标签为 user、debug、eng 的模块                 |
| 设定属性 ro.secure=1，打开安全检查功能     | 设定属性 ro.secure=1，打开安全检查功能     | 设定属性 ro.secure=0，关闭安全检查功能                |
| 设定属性 ro.debuggable=0，关闭应用调试功能 | 设定属性 ro.debuggable=1，启用应用调试功能 | 设定属性 ro.debuggable=1，启用应用调试功能            |
|                               |                               | ro.kernel.android.checkjni=1，启用 JNI 调用检查 |
| 默认关闭 adb 功能                   | 默认打开 adb 功能                   | 默认打开 adb 功能                              |
| 打开 Proguard 混淆器               | 打开 Proguard 混淆器               | 关闭 Proguard 混淆器                          |
| 打开 DEXPREOPT 预先编译优化           | 打开 DEXPREOPT 预先编译优化           | 关闭 DEXPREOPT 预先编译优化                      |

源码编译的第一个阶段我们知道envsetup.sh干的其中一件事就是定义了很多函数，而这些函数中就有lunch

## 五、lunch源码分析（平台列表如何获取）

1. 首先我们看看执行 lunch 命令之后平台菜单是怎么来的，下面是 lunch 函数的部分源码：

```bash
function lunch()
{
    local answer
    if [ "$1" ] ; then
        answer=$1
    else
        print_lunch_menu
        echo -n "Which would you like? [aosp_arm-eng] "
        read answer
    fi
```

从代码中我们可以看到，当我们执行命令的时候可以有两种方式，第一种就是直接指定编译平台的话可以通过 lunch xxx_xxx_eng 这样的带参数的方式执行，这样会通过 $1 将第一个参数保存到 answer 中，第二中就是不带参数的，这里我们执行的是第二种，那么代码就会走到 print_lunch_menu。

2. 接下来我们看下 print_lunch_menu 源码：

```bash
function print_lunch_menu()
{
    local uname=$(uname)
    echo
    echo "You're building on" $uname
    echo
    echo "Lunch menu... pick a combo:"
    local i=1
    local choice
    for choice in ${LUNCH_MENU_CHOICES[@]}
    do
        echo "     $i. $choice"
        i=$(($i+1))
    done
    echo
}
```

可以看出，这个函数做的一件事就是遍历并打印编译平台列表，而遍历的对象就是 LUNCH_MENU_CHOICES 这个数组，我们接下来看下这个数组的数据是怎么来的

3. LUNCH_MENU_CHOICES 数据来源：

```bash
# Clear this variable.  It will be built up again when the vendorsetup.sh
# files are included at the end of this file.
unset LUNCH_MENU_CHOICES
function add_lunch_combo()
{
    local new_combo=$1
    local c
    for c in ${LUNCH_MENU_CHOICES[@]} ; do
        if [ "$new_combo" = "$c" ] ; then
            return
        fi
    done
    LUNCH_MENU_CHOICES=(${LUNCH_MENU_CHOICES[@]} $new_combo)
}

# add the default one here
add_lunch_combo aosp_arm-eng
add_lunch_combo aosp_arm64-eng
add_lunch_combo aosp_mips-eng
add_lunch_combo aosp_mips64-eng
add_lunch_combo aosp_x86-eng
add_lunch_combo aosp_x86_64-eng
```

在这段代码中，我们看到，程序通过不断的调用 add_lunch_combo 函数，添加默认的编译平台选项到 LUNCH_MENU_CHOICES 中，这就是我们在执行 lunch 命令后出现的平台列表的数据来源了！但是有信息的朋友可能就发现了，这里只不过给我们添加了 6 个默认值，可是我们从前面的打印出来的列表中看一看到有二十多个呢，其它的哪来的呢？

原来 android 源码编译第一个阶段的时候 envsetup.sh 的做的其中一件事，就是通过下面的代码加载了很多的 vendorsetup.sh 脚本：

```bash
# Execute the contents of any vendorsetup.sh files we can find.
for f in `test -d device && find -L device -maxdepth 4 -name 'vendorsetup.sh' 2> /dev/null | sort` \
         `test -d vendor && find -L vendor -maxdepth 4 -name 'vendorsetup.sh' 2> /dev/null | sort` \
         `test -d product && find -L product -maxdepth 4 -name 'vendorsetup.sh' 2> /dev/null | sort`
do
    echo "including $f"
    . $f
done
unset f
```

之前我们并没有说这些脚本是用来干嘛的，那么现在我们看下这些脚本里面都有些什么内容：

```shell
#
# Copyright 2013 The Android Open Source Project
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

add_lunch_combo aosp_deb-userdebug
```

怎么样？很惊喜吧，从代码中我们看到，只有` add_lunch_combo aosp_deb-userdebug` 这句是有效的，有没有似曾相识的感觉？完全就是在通过 add_lunch_combo 函数给 LUNCH_MENU_CHOICES 添加数据嘛，这下知道那么多平台列表是哪来的了吧？

也就是说，当我们定制了自己的系统，我们只需要在 vendor 目录下创建对应的目录，并且创建我们所需的 vendorsetup.sh 脚本调用 add_lunch_combo 函数就可以添加我们对应平台的标识了! 至于这个标识有什么用后面再给大家说明。

## 六、继续lunch剩下的源码

```bash
function lunch()
{
    ......
    local selection=
    if [ -z "$answer" ]
    then
        selection=aosp_arm-eng
    elif (echo -n $answer | grep -q -e "^[0-9][0-9]*$")
    then
        if [ $answer -le ${#LUNCH_MENU_CHOICES[@]} ]
        then
            selection=${LUNCH_MENU_CHOICES[$(($answer-1))]}
        fi
    elif (echo -n $answer | grep -q -e "^[^\-][^\-]*-[^\-][^\-]*$")
    then
        selection=$answer
    fi

    if [ -z "$selection" ]
    then
        echo
        echo "Invalid lunch combo: $answer"
        return 1
    fi

    export TARGET_BUILD_APPS=
    local product=$(echo -n $selection | sed -e "s/-.*$//")
    check_product $product
    if [ $? -ne 0 ]
    then
        echo
        echo "** Don't have a product spec for: '$product'"
        echo "** Do you have the right repo manifest?"
        product=
    fi

    local variant=$(echo -n $selection | sed -e "s/^[^\-]*-//")
    check_variant $variant
    if [ $? -ne 0 ]
    then
        echo
        echo "** Invalid variant: '$variant'"
        echo "** Must be one of ${VARIANT_CHOICES[@]}"
        variant=
    fi

    if [ -z "$product" -o -z "$variant" ]
    then
        echo
        return 1
    fi

    export TARGET_PRODUCT=$product
    export TARGET_BUILD_VARIANT=$variant
    export TARGET_BUILD_TYPE=release
    echo
    set_stuff_for_environment
    printconfig
}
```

上面的代码不算少，但是我们只需要关注这段代码做的下面三件事即可

1. 检查 answer 合法性，并给 selection 赋值

我们首先看第一个判断 if [-z "$answer"]，表示如果用户没有输入任何东西直接回车的话，就会通过 selection=aosp_arm-eng 指定缺省的编译选项。

其实这里我们只需要关注 elif (echo -n \$answer | grep -q -e "^[0-9][0-9]*$") 这个判断即可，走入这个判断就证明我们有手动指定了编译选项，然后通过 selection=\${LUNCH_MENU_CHOICES[\$(($answer-1))]} 给 selection 赋值，做完了一系列的判断赋值之后通过 if [ -z "$selection" ] 判断 selection 是否被赋值，如果没有则退出程序并通知。

2. 按照 product-variant 的格式拆分 selection 并做合法性校验

{{< admonition >}}

这里我们先说明下，什么是 product-variant，这里的 product 指的是我们的产品型号、而 variant 就是编译类型，以 aosp_arm-eng 为列，这里的 product=aosp_arm，variant=eng;

{{< /admonition >}}

那么上面的代码中，我们看到 `local product=\$(echo -n \$selection | sed -e "s/-.\*$//") `提取 product，并通过 check_product 函数进行检验，同样的 `local variant=$(echo -n $selection | sed -e "s/^[^\-]*-//") `对 variant 进行了分离，然后通过 check_variant 函数进行校验, 我们下校验的过程：

```bash
VARIANT_CHOICES=(user userdebug eng)
# check to see if the supplied variant is valid
function check_variant()
{
    for v in ${VARIANT_CHOICES[@]}
    do
        if [ "$v" = "$1" ]
        then
            return 0
        fi
    done
    return 1
}
```

其实很简单，只不过是判断我们分离出来的 variant 是否在 VARIANT_CHOICES 中，如果在就证明是合法的，返回 0, 否则返回 1，紧接着通过` if [\$? -ne 0]` 判断，如果 `\$?`(上一个指令返回的值) 不等于 0 就将 variant 置空并打印提示

3. 设置环境变量

接下来我们看到，程序分别对下面几个环境遍历进行了的赋值处理。

    export TARGET_PRODUCT=$product
    export TARGET_BUILD_VARIANT=$variant
    export TARGET_BUILD_TYPE=release

其实这真个 lunch 的过程还设置了大量的环境变量，涉及到的函数有 `set_stuff_for_environment`、`printconfig` 等，这里就不细细展开，lunch 说白了就是一个设置环境变量的过程，最后你会发现他做了那么多事，只为了最后的那一段环境变量而已，如图：

![设置环境变量](https://img-blog.csdn.net/20180622150608188?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L01vbmFMaXNhVGVhcnI=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

