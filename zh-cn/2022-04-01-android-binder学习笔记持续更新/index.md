# Android-AIDL学习笔记(持续更新)


# **Android-AIDL学习笔记(持续更新)**

通过对Android中的Proxy设计模式的学习，可以发现AIDL就是Proxy模式的很好的实现

用户通过AIDL定义一系列接口，编译时AIDL会自动生成相应的源码，随后我们可以分别在Service和Client中调用

当Service与Client在同一个进程中时，使用本地Proxy模式自动调用；
当Service与Client不在同个进程中时，使用Binder进行进程间通信，本质上是aidl实现了远程的Proxy接口，当Client进行调用时，通过Binder进行转发到Service中，这里的Binder就实现了Proxy的功能

计算机系统中，设备驱动分为内核态和用户态

1. 保护厂商利益（出发点）
2. 内核空间主要负责硬件访问逻辑（GPL）
3. 用户空间主要负责参数和访问流程控制（Apache License）

处于内核态中的程序可以访问并操作所有的硬件设备，Binder的思想就是通过一个一直运行在内核态的Binder驱动转发相关信息，来对想要跨进程的程序进行联通。其次，通过Binder驱动也可以对用户发出的指令进行过滤和识别，以防止对设备造成破坏

## MediaPlayerService

以下以 Android 2.3.5 的 MediaPlayerService 为例

mian：

```  java
int main(int argc, char** argv)

{

    sp<ProcessState> proc(ProcessState::self());

    sp<IServiceManager> sm = defaultServiceManager();   

MediaPlayerService::instantiate();

---》该函数内部调用 addService，把 MediaPlayerService 信息 add 到 ServiceManager 中

    ProcessState::self()->startThreadPool();

    IPCThreadState::self()->joinThreadPool();

}
```

在 MediaPlayerService::instantiate() 中，使用了 ServiceManager 的 addService 方法来加入服务。在Android系统中Service信息都应该先add到ServiceManager[SM]中，由ServiceManager来集中管理，这样可以查询当前系统有哪些服务。而且，Android 系统中某个服务例如 MediaPlayerService 的客户端想要与其通讯的话，必须先向 ServiceManager 查询 MediaPlayerService 的信息，然后通过SerivceManager 返回的东西再来和 MediaPlayerService 交互。

1. MediaPlayerService 向 SM 注册
2. MediaPlayerClient 查询当前注册在 SM 中的MediaPlayerService 的信息
3. 根据这个信息，MediaPlaerClient 和 MediaPlayerService 交互

另外，ServiceManager 的handle 标识是0， 所以只要往 handle 是 0 的服务发送信息了，最终都会传递到 ServiceManger 中去

### MediaService 的运行

1. defaultServiceManager 得到了 PbServiceManager，然后 MediaPlayerServcie 实例化后，调用 BpServiceManger 的 addService 函数；

2. 在这个过程中，是 service_manager 收到 addService 的请求，然后把对应信息放到自己保存的一个服务 list 中
3. service_manager有一个binder_looper 函数，专门等着从binder中接受请求。虽然 service_manager 没有中 BnServiceManager 中派生，但是他肯定完成了 BnServiceManager 的功能。

同理，和 socket 的工作方式类似的，一个BnMediaPlayerService应该会：

1. 打开binder 设备
2. 整一个looper循环，坐等请求

BnMediaPlayerService 从 BBinder 中派生，所以会调用到它的 onTransact 函数，在其中所有 IMediaPlayerService 提供的函数都会通过命令类型来区分。
也就是说，BnXXX类的onTransact函数收取命令，然后派发到派生类的函数，由他们完成实际的工作。

在 startThreadPool 和 joinThreadPool 结束后会有两个线程，分别是主线程和工作线程，而且都在做消息循环。

### MediaPlayerClietn

关于 MediaPlayerClient[Client] 和 MediaPlayerService[Service]的交互

使用 Service 的时候，要先创建它的 BpMediaPlayerSerive，例子：

``` java
IMediaDeathNotifier::getMediaPlayerService() {

    sp<IServiceManager> sm = defaultServiceManager();

    sp<IBinder> binder;
    
    do {
        // 向 SM 查询对应服务的信息，返回 binder
        binder = sm->getService(String16("media.player"));

        if (binder != 0) {
            break;
        }
        usleep(500000); // 0.5 s
    } while(true);

    // 通过 interface_cast，将这个 binder 转化成 BpMediaPlayerService

    // 注意，这个 binder 只是用来和 binder 设备通讯用的，

    // 实际上和 IMediaPlayerService 的功能一点关系都没有。

    // BpMediaPlayerService 用这个 binder 和  BnMediaPlayerService 通讯。

    sMediaPlayerService = interface_cast<IMediaPlayerService>(binder);

}

return sMediaPlayerService;
```

Binder 其实就是一个和 binder 设备打交道的接口，而上层 IMediaPlayerService 只不过把它当作一个类似 socket 来使用罢了。

