# init 工作流程简述


# **Init 工作流程简述**

init的工作流程可以精简为以下四点：

- 解析两个配置文件，下文分析其中对init.rc 文件的解析
- 执行各个阶段的动作，创建 zygote 的工作就是在其中的某个阶段完成的
- 调用 property_init 初始化属性相关的资源，并且通过 property_start_service 启动属性服务
- init 进入死循环，并且等待一些事情的发生。重点关注 init 如何处理来自 socket 和来自属性服务器的相关事情

> 精简工作流程是分析代码时常用的方法

## 解析配置文件

init会解析两个配置文件，一个是系统配置文件 init.rc， 另一个是与硬件平台相关的配置文件 init.[hardwordName].rc。对这两个配置文件进行解析调用的是同一个 ``parse_config_file`` 函数

  parser.c

里面先找到配置文件的 section，然后针对不同的 section 来使用不同的解析函数来解析。

1. 关键字定义

  keyword.h

- 第一次包含keyword.h时，它声明了一些诸如 do_class_start 的函数，另外定义了一个枚举，枚举值为 K_class、K_mkdir 等关键字
- 第二次包含后，得到一个 keyword_info 结构体数组，这个结构体数组以前面定义的枚举值为索引，存储对应的关键字信息，这些信息包括关键字名称、处理函数、处理函数的参数个数，以及属性

根据keyword.h的定义，其中 symbol 为 on 或 service 的时候表示 section

2. 解析 init.rc

解析属性指的是 persist，就是在 adb 中输入的可以即时生效的配置属性，如 persist.service.adb.enable=1

- 一个section的内容从这个标识 section 的关键字开始，到下一个标识section 的地方结束
- init.rc中出现了名为 boot 和 init 的sction，这里两者就是前面介绍的4个动作执行阶段的boot和init。也就是说，在boot阶段执行的动作都是由 boot 这个 section 定义的

另外还可以发现，zygote被放在了一个 service section 中。下面以zygote 这个section为例，介绍service是如何解析的

## 解析 service

解析service时用到了 parse_service好parse_line_service这两个函数，先看init是如何组织这个service的：

1. service结构体

保存了与 service section 相关的信息


