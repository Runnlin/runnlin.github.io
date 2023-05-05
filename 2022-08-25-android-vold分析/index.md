# Android Vold分析




# Android Vold分析

  > 当接收到UEvent消息后，除了APP层可能会进行额外的逻辑以外，其中一个很重要的组件Vold会进行重要的挂载操作，可以说没有进行挂载的设备在Android中很难进行使用

- Android中的Vold是Volume Daemon，用于管理和控制Android平台外部存储设备的守护进程，是个Native程序。
- 在Java世界中，与Vold交互的模块是MountService，他一边接收vold发出的消息，一边根据信息触发发送ACTION_MEDIA_MOUNT/ACTION_MEDIA_EJECT等广播，同时，他还能向vold发送控制命令，如挂载等。
- Vold架构可用图9.1表示

  <img src=../assets/image_1656588456012_0.png />

- 当Linux内核检测到外部设备插入时，会发送uevent事件到NetlinkManager（NM）
- NM将这些消息转发给VolumeManager（VM），VM会做一些操作，然后通过CommandListener（CL）发送相关信息给MountService，其会根据收到的消息发送相关的处理指令给VM进行进一步处理。例如，当U盘插入后，VM会将来自NM的“Disk insert“消息发给MountService，然后MountService则发送”Mount“指令给Vold，指示他挂载这个U盘
- CL模块内部封装了一个Socket用于跨进程通信。来与VM、NM、MountService相互接收和发送消息。不使用Binder机制的原因是这样代码量和派生关系等都会简单不少
-

- ## 初识Vold

  1. 创建VolumeManager对象
  2. 创建NetlinkManager对象
  3. 创建CommandListener对象（该对象在Android6及以上版本不存在，需进一步验证）
  4. 启动VM
  5. 根据配置文件来初始化VM
  6. 启动NM
  7. 通过往/sys/block 目录下对应的uevent文件写"add\n"来触发内核发送Uevent消息
  8. VM通过CL向感兴趣的模块（如MountService）通知UMS（USB mass storage）的状态
  9. 启动CL
- 下面一个个来介绍各个模块

- #### NetlinkManager

- 在vold中，使用NM的流程是：
  1. 调用instance创建一个NM对象
  2. 调用setBroadcaster设置CL对象
  3. 调用start启动NM
- NM的start分为两个步骤：
  1. 创建地址簇为PF_NETLINK类型的socket并做一些设置，这样NM就能和Kernel通信了。
  2. 创建NetlinkHandler对象，并调用他的start。后续工作都是由NetlinkHandler完成。

- NetlinkHandler分析：
  1. 创建NLH：是在多线程环境中使用
  2. start：

