# Android UEvent分析




# Android UEvent分析

>在项目中我们有时会需要接收由底层发出的UEvent信息，来对连接或断开的设备在挂载、卸载之前做出一些逻辑，本文通过一个 Android 9.0 项目中使用到的接收UEvent的流程来一步步分析其实现逻辑

{{< admonition tip >}}
RxBus有很多实现，如：

1. 在Linux系统下一切皆是文件，设备也不例外；/dev目录下包含了所有可能出现的设备的设备文件（设备节点），通过Linux内核驱动程序来与硬件设备通信（读写），但是用户很难在大量设备文件中找到匹配的设备文件，于是Linux系统开始使用udev来在运行时管理/dev下的设备文件，如udev将在设备被检测到时创建设备文件并在设备移除时删除这些文件（设备拔插消息由内核通知出），包括热插拔设备。
   
2. udev是一种新的管理/dev目录的方法；udev在/etc/udev/rules.d目录下有一个强大的脚本接口，终端用户和驱动发布者通常使用脚本文件编写udev规则来定制设备节点文件的创建。
   可定制的属性包括文件权限、在文件系统中的路径和符号链接。

3. udev处理的所有设备信息都存储在udev数据库中，并且会发送给可能的设备事件的订阅者。可以通过libudev库访问udev数据库以及设备事件源。
   
4. udev以守护进程的方式运行于Linux系统，并监听在新设备初始化或设备从系统中移除时，内核（通过netlinksocket）所发出的uevent。调用libudev接口从sys文件系统中读取设备的信息。

{{< /admonition >}}


- ## Netlink与UEvent

  1. Netlink是Linux系统中一种使用socket来与Kernel交互的机制，使得APP层可以接收来自kernel的消息，也能发送一些控制指令
  2. Uevent实际上是一组字符串，可以告知APP层发生了什么事情，插入u盘时，内核通过Netlink发送一个消息给Vold，vold根据接收到的消息进行处理，例如挂载这个U盘，这个消息就是UEvent

  ```uevent
  add@/devices/platform/msm_sdcc.2/mmc_host/mmc1/mmc1:c9f2/block/mmcblk0
  ACTION=add  // add \ remove \ change
  // means this device's path of /sys 
  DEVPATH=/devices/platform/msm_sdcc.2/mmc_host/mmc1/mmc1:c9f2/block/mmcblk0   
  SUBSYSTEM=block  // what kind of this device belong, such as character... 块设备
  MAJOR=179  // MAJOR和MINOR分别表示该设备的主次设备号，二者联合起来可以标识一个设备
  MINOR=0
  DEVNAME=设备名字
  DEVTYPE=disk // 设备类型
  NPARTS=3 // 该SD卡上的分区
  SEQNUM=1357  序号
  
  ```

- 上面可以看到该SD卡上有3个分区，所以还会接收到分区相关的UEvent

     ```
  add@/devices/platform/msm_sdcc.2/mmc_host/mmc1/mmc1:c9f2/block/mmcblk0/mmcblk0p1
  DEVPATH=/devices/platform/msm_sdcc.2/mmc_host/mmc1/mmc1:c9f2/block/mmcblk0/mmcblk0p1
  DEVTYPE=partition   // 表示分区
  ```

其他消息都一样，就没有赘述


### 那么有哪些情况Kernel会发出UEvent消息呢？

1. 设备发生变化时，如插入和拔出。如果vold在设备发生变化之前已经建立了Netlink IPC通信，那么vold可以接收到这些Uevent消息
2. 设备一般在 /sys 对应的目录下有一个叫uevent的文件，往该文件中写入指定的数据，也会触发Kernel发送和该设备相关的Uevent消息，这是由应用层触发的。例如vold启动时，会往这些uevent文件中写入数据，来促使内核发送Uevent消息，这样vold就能得到这些设备的当前信息了


## app层

### UEventReceiver

APP在此处通过一个receiver进行UEvent信息的接收

涉案人员:

> APP/myReciever/UEventReceiver.java

``` java
public class UEventReceiver extends UEventObserver {
  // 实现了UeventObserver接口
  // ...

  @Override
  public void onUEvent(UEvent uEvent) {
      Log.d(TAG, "onUEvent: uEvent = " + uEvent);
      // ...
  }

  public void init(Context context) {
      Log.d(TAG, "init: Observing = " + USB_STATE_MATCH);
      startObserving(USB_STATE_MATCH);
  }

  public void destroy() {
      stopObserving();
  }
}
```

UEventObserver 提供了 onUEvent 回调，底层的UEvent消息UEventObserver发送到APP层，同时提供了 `startObserving()` 和 `stopObserving()` 接口来注册回调，其中 `startObserving(USB_STATE_MATCH)` 使用的 `USB_STATE_MATCH` 是指我们想要关注的根路径，这里是：

  `private static final String USB_STATE_MATCH = "DEVPATH=/";`

- ### UEventObserver
  
  这里提供了给APP层的各种Observing方法，先看看 startObserving：
  
  ``` java
  public final void startObserving(String match) {
    if (match == null || match.isEmpty()) {
        throw new IllegalArgumentException("match g must be non-empty");
    }
  
     final UEventThread t = getThread();
     t.addObserver(match, this);
  }
  
  先进入getThread()看看
  
  private static UEventThread getThread() {
    synchronized (UEventObserver.class) {
        if (sThread == null) {
            sThread = new UEventThread();
            sThread.start();
        }
        return sThread;
    }
  }
  
  在getThread中直接启动了 UEventThrad 线程
  
  private static final class UEventThread extends Thread {
    /** Many to many mapping of string match to observer.
     *  Multimap would be better, but not available in android, so use
     *  an ArrayList where even elements are the String match and odd
     *  elements the corresponding UEventObserver observer */
     /** 这里有个官方的注释，我来尝试翻译一下：
     * 这里使用了多对多的数据结构，如果采用 multimap会性能更好，但在Android上不适用，
     * 因为map不允许重复的key，而此处是会有重复的，所以使用两个 ArrayList 来相互配对
     */
    private final ArrayList<Object> mKeysAndObservers = new ArrayList<Object>();
  
    private final ArrayList<UEventObserver> mTempObserversToSignal =
            new ArrayList<UEventObserver>();
  
    // 这个UEventThread是UEventObserver的内部类
    public UEventThread() {
        super("UEventObserver");
    }
  
    @Override
    public void run() {
        nativeSetup();
        // 可以看到这是一个死循环，不断地监听UEvent
        while (true) {
            // 更底层的Uevent来源于jni，此处暂时不进行跟进
            String message = nativeWaitForNextEvent();
            if (message != null) {
                if (DEBUG) {
                    Log.d(TAG, message);
                }
                sendEvent(message);
            }
        }
    }
    // 像上层（APP）发送Event消息，调用他们注册的回调
    private void sendEvent(String message) {
        synchronized (mKeysAndObservers) {
            final int N = mKeysAndObservers.size();
            for (int i = 0; i < N; i += 2) {
                final String key = (String)mKeysAndObservers.get(i);
                if (message.contains(key)) {
                    final UEventObserver observer =
                            (UEventObserver)mKeysAndObservers.get(i + 1);
                    mTempObserversToSignal.add(observer);
                }
            }
        }
  
        if (!mTempObserversToSignal.isEmpty()) {
            final UEvent event = new UEvent(message);
            final int N = mTempObserversToSignal.size();
            for (int i = 0; i < N; i++) {
                final UEventObserver observer = mTempObserversToSignal.get(i);
                observer.onUEvent(event);
            }
            mTempObserversToSignal.clear();
        }
    }
    // 提供注册回调
    public void addObserver(String match, UEventObserver observer) {
        synchronized (mKeysAndObservers) {
            mKeysAndObservers.add(match);
            mKeysAndObservers.add(observer);
            nativeAddMatch(match);
        }
    }
  
    /** Removes every key/value pair where value=observer from mObservers */
    public void removeObserver(UEventObserver observer) {
        synchronized (mKeysAndObservers) {
            for (int i = 0; i < mKeysAndObservers.size(); ) {
                if (mKeysAndObservers.get(i + 1) == observer) {
                    mKeysAndObservers.remove(i + 1);
                    final String match = (String)mKeysAndObservers.remove(i);
                    nativeRemoveMatch(match);
                } else {
                    i += 2;
                }
            }
        }
    }
  }
  
  UEvent 对象是这样的：
  public static final class UEvent {
    // collection of key=value pairs parsed from the uevent message
    private final HashMap<String,String> mMap = new HashMap<String,;
  
    // Uevent对象在创建时会进行message的解析，
    public UEvent(String message) {
        int offset = 0;
        int length = message.length();
  
        while (offset < length) {
            int equals = message.indexOf('=', offset);
            int at = message.indexOf('\0', offset);
            if (at < 0) break;
  
            if (equals > offset && equals < at) {
                // key is before the equals sign, and value is after
                mMap.put(message.substring(offset, equals),
                        message.substring(equals + 1, at));
            }
  
            offset = at + 1;
        }
    }
  
    public String get(String key) {
        return mMap.get(key);
    }
  
    public String get(String key, String defaultValue) {
        String result = mMap.get(key);
        return (result == null ? defaultValue : result);
    }
  
    public String toString() {
        return mMap.toString();
    }
  }
  
  ```
  
  底层的用来封装UEvent的message来源是 nativeWaitForNextEvent，他在：
  
>  /frameworks/base/core/jni/android_os_UEventObserver.cpp
  
  ``` CPP
  static jstring nativeWaitForNextEvent(JNIEnv *env, jclass clazz) {
    char buffer[1024];
  
    for (;;) {
        // uevent_next_event 从硬件层获取到信息， 放到bufer里
        int length = uevent_next_event(buffer, sizeof(buffer) - 1);
        if (length <= 0) {
            return NULL;
        }
        buffer[length] = '\0';
  
        ALOGV("Received uevent message: %s", buffer);
  
        if (isMatch(buffer, length)) {
            // Assume the message is ASCII.
            jchar message[length];
            for (int i = 0; i < length; i++) {
                message[i] = buffer[i];
            }
            return env->NewString(message, length);
        }
    }
  }
  ```
  
  ``` CPP
  int uevent_next_event(char* buffer, int buffer_length)
  {
    while (1) {
        struct pollfd fds;
        int nr;
    
        fds.fd = fd;
        fds.events = POLLIN;
        fds.revents = 0;
        nr = poll(&fds, 1, -1);
     
        if(nr > 0 && (fds.revents & POLLIN)) {
            // fd 从 socket 传输过来的数据
            int count = recv(fd, buffer, buffer_length, 0);
            if (count > 0) {
                struct uevent_handler *h;
                pthread_mutex_lock(&uevent_handler_list_lock);
                // 向前遍历数据，解析，放到一个双向链表里
                LIST_FOREACH(h, &uevent_handler_list, list)
                    h->handler(h->handler_data, buffer, buffer_length);
                pthread_mutex_unlock(&uevent_handler_list_lock);
  
                return count;
            } 
        }
    }
    
    // won't get here
    return 0;
  }
  
  int uevent_add_native_handler(void (*handler)(void *data, const char *msg, int msg_len), 
                             void *handler_data)
  {
    struct uevent_handler *h;
  
    h = malloc(sizeof(struct uevent_handler));
    if (h == NULL)
        return -1;
    h->handler = handler;
    h->handler_data = handler_data;
  
    pthread_mutex_lock(&uevent_handler_list_lock);
    LIST_INSERT_HEAD(&uevent_handler_list, h, list);
    pthread_mutex_unlock(&uevent_handler_list_lock);
  
    return 0;
  }
  
  int uevent_remove_native_handler(void (*handler)(void *data, const char *msg, int msg_len))
  {
    struct uevent_handler *h;
    int err = -1;
  
    pthread_mutex_lock(&uevent_handler_list_lock);
    LIST_FOREACH(h, &uevent_handler_list, list) {
        if (h->handler == handler) {
            LIST_REMOVE(h, list);
            err = 0;
            break;
       }
    }
    pthread_mutex_unlock(&uevent_handler_list_lock);
  
    return err;
  }
  ```
  
  c驱动层这块知识后续再进行跟进，现在我们回到实际项目中看看APP中怎么利用Uevent
  
  先截取一下APP中收到 onUEvent 回调时获取到的 UEvent 内容：
  
  ``` json
  {
    SUBSYSTEM=usb, // 设备类型
    MAJOR=189, // MAJOR和MINOR分别表示该设备的主次设备号，二者联合起来可以标识一个设备
    MINOR=4, 
    BUSNUM=001, // 设备BUS序号
    SEQNUM=1927, // 消息区块下标
    ACTION=add, // 动作类型，如 add、remove 等
    DEVNAME=bus/usb/001/005, // 分配名字，这里最后的005也代表 DEVNUM
    DEVTYPE=usb_device, // 类型
    PRODUCT=951/1665/100, // 用作设备标识识别 key
    INTERFACE= 8/6/80, // 识别类型，如hub、hid、printer等
    DEVPATH=/devices/platform/soc/ee080100.usb/usb1/1-1, // 准备挂载路径
    TYPE=0/0/0, //  设备类型序号
    DEVNUM=005 // 开机后累计插入的设备计数，通过设备反复拔插也会增加
  }
  ```
  
  APP中先通过确定DEVPATH在对的位置上，然后通过action分别处理各种事件
  
  通过INTERFACE分辨USB设备类型，分别处理u盘、ipod等设备，如果是U盘，则获取对应的封装设备信息进行封装，然后进行数量记录和 Toast显示。 在本APP中UEventReceiver只用于即时的toast提示用户已插入，和USBNotification相同。
  
  后续操作是等待挂载完成广播，挂载部分在Vold分析中再细表，且听下回分解。
-
- [[2022-04-25-Android Vold分析]]

