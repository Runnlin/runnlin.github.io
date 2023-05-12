# Android广播原理分析


一、 Boradcast 前题概要
-----------------

### 1、广播分为前台广播和后台广播

发送前台广播 (Intent.FLAG_RECEIVER_FOREGROUND 标志)

```  java
val intent = Intent(Intent.ACTION_SHUTDOWN)
intent.addFlags(Intent.FLAG_RECEIVER_FOREGROUND)
sendBroadcast(intent)
```

默认发送的是后台广播

```  java
val intent = Intent(Intent.ACTION_SHUTDOWN)
sendBroadcast(intent)
```

在 ActivityManagerService 中，前台广播和后台广播各自分别有一个广播队列，互不干扰。

```  java
mFgBroadcastQueue = new BroadcastQueue(this, mHandler,
        "foreground", BROADCAST_FG_TIMEOUT, false);
mBgBroadcastQueue = new BroadcastQueue(this, mHandler,
        "background", BROADCAST_BG_TIMEOUT, true);
mBroadcastQueues[0] = mFgBroadcastQueue;
mBroadcastQueues[1] = mBgBroadcastQueue;
```

* 前台队列相对比较空闲，处理广播会相对快一些
* 前台队列的超时时间是 10s，而后台是 60s. 后台广播的设计思想就是当前应用优先，尽可能多让收到广播的应用有充足的时间把事件做完。而前台广播的目的是紧急通知，设计上就倾向于当前应用赶快处理完，尽快传给下一个。
* 前台队列不等后台服务，而后台队列要多等后台服务一定的时间

### 2、有序广播 (串行广播) 和无序广播(并发广播)

#### 2.1、发送广播的时候，可以指定发送的广播是并发广播还是串行广播

```
//并发广播
sendBroadcast(intent)
//串行广播
sendOrderedBroadcast(intent)
```

串行和并发是相对于订阅了广播的多个接收者而言的，比如 A、B、C 都注册了同一个广播。

如果发送并发广播, 则 A、B、C 会 " 并发” 地收到消息, A、B、C 没有明确的先后顺序。  
如果发送串行广播, 则 A、B、C 三个接收者会按照 priority 优先级排序，顺序地接收广播消息, 假设优先级顺序为 A>B>C, 则 A 首先接收到广播, 处理完之后, B 才接收广播, 然后是 C 接收广播。

串行广播的特性:

> * 先接收的广播接收者可以对广播进行截断（abortBroadcast()），即后接收的广播接收者不再接收到此广播；
> * 先接收的广播接收者可以对广播进行修改，再使用 setResult() 函数来结果传给下一个广播接收器接收，那么后接收的广播接收者将通过 getResult() 函数来取得上个广播接收器接收返回的结果

#### 2.2、静态注册的广播, 无论发送时是否执行串行广播, AMS 处理的时候都会按照串行广播来处理

> 由于静态广播 可以启动新的进程, 如果采用串行广播处理, 可能会出现统一时间启动大量进程的场景, 造成系统资源进程。因此静态注册的广播, 统一按串行广播处理。

#### 2.3、只有串行广播才有有超时时间处理

* 串行广播才有超时时间限制: 前台广播 10s 处理时间, 后台广播 60s 处理事件；
* 并发广播没有处理时间的限制。

### 3、应用内广播 LocalBroadCastManager

context 发送广播存在两个问题:

* context 发送的广播, 都需要 ActivityManagerService 统一管理，中间需要经过多次 IPC 的处理，另外 ActivityManagerService 中分发广播消息时, 都统一加了锁, 当 Android 系统中安装了众多 App, 同时发送广播, 有可能会造成 AMS 处理繁忙, 造成发送广播效率低下
* context 发送的广播, 可以跨 app 接收，这就造成了一定的安全隐患。

针对上面两点问题, 对于只需要在 app 内部发送和接收消息的场景, 推荐用 LocalBroadCastManager。LocalBroadCastManager 非常简单，采用订阅者模式, 利用 Handler 机制 (而非 Binder) 实现广播消息的分发和处理。

二、源码解析
------

### 2.1、几个重要的类

#### 2.2.1、BroadcastRecord

sendBroadCast 发送出的广播, 在 SystemServer 进程会映射成一个 BroadCastRecord 对象，代表一条广播消息。  
其中重要的属性:

* ordered 是否是有序广播
* sticky 是否是粘性广播
* receivers 改广播的接收者数组
* callerApp 发送该广播消息的进程

```  java
final class BroadcastRecord extends Binder {
    final Intent intent;    // the original intent that generated us
    final ComponentName targetComp; // original component name set on the intent
    final ProcessRecord callerApp; // process that sent this //代表发送广播的进程
    final String callerPackage; // who sent this //发送广播的包名
    final int callingPid;   // the pid of who sent this //发送者的进程ID
    final int callingUid;   // the uid of who sent this //发送这的userID
    final boolean callerInstantApp; // caller is an Instant App? //发送这是否是即时应用
    final boolean ordered;  // serialize the send to receivers? //是否是优先广播
    final boolean sticky;   // originated from existing sticky data? //是否是粘性广播

    final List receivers;   // contains BroadcastFilter and ResolveInfo //该条广播的接收这


 
    long enqueueClockTime;  // the clock time the broadcast was enqueued //广播的入队列时间
    long dispatchTime;      // when dispatch started on this set of receivers//广播的分发时间
    long dispatchClockTime; // the clock time the dispatch started //广播的分发时间
    long receiverTime;      // when current receiver started for timeouts. //广播分发完成的时间
    long finishTime;        // when we finished the broadcast. //接收者处理完广播的时间


    int nextReceiver;       // next receiver to be executed. //下一个接收广播消息的接受者
   
    BroadcastQueue queue;   // the outbound queue handling this broadcast //该广播所属的广播队列
}
```

#### 2.1.2、BroadcastQueue

BroadcastQueue: 广播队列:  
在 AMS 中定义了两个队列 mFgBroadcastQueue 和 mBroadcastQueues, 处理广播时根据 Intent 中的 Intent.FLAG_RECEIVER_FOREGROUND 来区分是放在前台广播队列处理，还是后台广播队列处理。

```  java
public class ActivityManagerService extends IActivityManager.Stub{
    BroadcastQueue mFgBroadcastQueue;
    BroadcastQueue mBgBroadcastQueue;
    // Convenient for easy iteration over the queues. Foreground is first
    // so that dispatch of foreground broadcasts gets precedence.
    final BroadcastQueue[] mBroadcastQueues = new BroadcastQueue[2];
    
     public ActivityManagerService(Context systemContext) {
            mFgBroadcastQueue = new BroadcastQueue(this, mHandler,
                    "foreground", BROADCAST_FG_TIMEOUT, false);
            mBgBroadcastQueue = new BroadcastQueue(this, mHandler,
                    "background", BROADCAST_BG_TIMEOUT, true);
     }
     
      BroadcastQueue broadcastQueueForIntent(Intent intent) {
        final boolean isFg = (intent.getFlags() & Intent.FLAG_RECEIVER_FOREGROUND) != 0;
        if (DEBUG_BROADCAST_BACKGROUND) Slog.i(TAG_BROADCAST,
                "Broadcast intent " + intent + " on "
                + (isFg ? "foreground" : "background") + " queue");
        return (isFg) ? mFgBroadcastQueue : mBgBroadcastQueue;
    }
 }
```

广播消息的核心处理操作都由在 BroadCastQueue 完成的

* 持有两个 BroadCastRecord 的数组, mParallelBroadcasts 用于保存待处理的并发广播记录, mOrderedBroadcasts 用于保存待处理串行广播记录
* mBroadcastHistory 用于保存已经处理过的历史消息
* BroadcastHandler 用处将 processNextBroadcast 处理广播消息，broadcastTimeoutLocked 广播消息超时终止操作, 组织到一个单线程队列中。
* scheduleBroadcastsLocked() 中发送 BROADCAST_INTENT_MSG 消息, 触发队里中广播的一次分发。
* processNextBroadcast 处理下一条广播, 该方法利用 mServices 对广播的分发的整个过程做了线程同步处理。也就是说无论前台广播还是后台广播 消息的处理 受制于同步锁, 必须是串行执行的。
* processNextBroadcastLocked() 方法是广播的具体分发操作
* processCurBroadcastLocked：处理静态注册的 BroadCastReceiver 会调用此方法，最终会 new 出一个 BroadCastReceiver 对象，调用其 onReceive()
* deliverToRegisteredReceiverLocked(): 处理动态注册的 BroadCastReceiver, 会调用此方法，最终会索引到最初注册的 BroadCastReceiver 对象实例, 调用其 onReceive()

```  java
public final class BroadcastQueue {
 
 //并行消息队列
 final ArrayList<BroadcastRecord> mParallelBroadcasts = new ArrayList<>();
 //串行消息队列
 final ArrayList<BroadcastRecord> mOrderedBroadcasts = new ArrayList<>();
 //已处理消息历史
 final BroadcastRecord[] mBroadcastHistory = new BroadcastRecord[MAX_BROADCAST_HISTORY];
 // AMS 对象实例
 final ActivityManagerService mService;


private final class BroadcastHandler extends Handler {
        public BroadcastHandler(Looper looper) {
            super(looper, null, true);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case BROADCAST_INTENT_MSG: {
                    if (DEBUG_BROADCAST) Slog.v(
                            TAG_BROADCAST, "Received BROADCAST_INTENT_MSG");
                    processNextBroadcast(true);
                } break;
                case BROADCAST_TIMEOUT_MSG: {
                    synchronized (mService) {
                        broadcastTimeoutLocked(true);
                    }
                } break;
            }
        }
    }

   //触发消息分发
  public void scheduleBroadcastsLocked() {
        if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Schedule broadcasts ["
                + mQueueName + "]: current="
                + mBroadcastsScheduled);

        if (mBroadcastsScheduled) {
            return;
        }
        mHandler.sendMessage(mHandler.obtainMessage(BROADCAST_INTENT_MSG, this));
        mBroadcastsScheduled = true;
    }

    //处理广播超时
   final void broadcastTimeoutLocked(boolean fromMsg){

   }

   //消息分发的处理函数,整个处理过程都用mService加了锁。
   final void processNextBroadcast(boolean fromMsg) {
        synchronized (mService) {
            processNextBroadcastLocked(fromMsg, false);
        }
    }

    final void processNextBroadcastLocked(boolean fromMsg, boolean skipOomAdj) {
        //1、处理所有的并发队列中广播
        //2、判断有无正在处理的串行消息(等待新启线程),有 则等待新线程启动完成
        //3、串行队列中队首消息的超时处理
        //4、取出串行队列中的第一个BroadCastRecord,分发给下一个Receiver，如果是动态注册的广播,则调用deliverToRegisteredReceiverLocked()
        // 如果是静态注册的广播，则调用processCurBroadcastLocked，若广播的进程未启动,则首先启动新的进程。

    }

    


    // 处理静态注册的BroadCastReceiver ->最终会new 出一个BroadCastReceiver对象，调用其onReceive()
    private final void processCurBroadcastLocked(BroadcastRecord r,
            ProcessRecord app, boolean skipOomAdj){

            }

     //处理动态注册的BroadCastReceiver -> 会取出注册的BroadCastReceiver对象,然后调用其onReceive()
     private void deliverToRegisteredReceiverLocked(BroadcastRecord r,
    BroadcastFilter filter, boolean ordered, int index) {

    }

}
```

#### 2.1.3、ReceiverDispatcher Receiver 分发器

ReceiverDispatcher 是 LoadedApk 的内部类, 用于接收 AMS 的 IPC 消息, 进行 BroadCast 的分发，适用于动态注册的 BroadCastReceiver。

* ReceiverDispatcher 持有注册 BroadcastReceiver 实例
* ReceiverDispatcher 持有一个 IIntentReceiver.Stub 实例, 用于接收 AMS 的 IPC 消息, 分发广播消息。
* ReceiverDispatcher.performReceive 用于接收广播后的处理操作。

```  java
tatic final class ReceiverDispatcher {

        final IIntentReceiver.Stub mIIntentReceiver; //用于接收AMS IPC 消息的Binder
        final BroadcastReceiver mReceiver; //自定义的广播接收者
        final Context mContext;
        final Handler mActivityThread;
            
        ReceiverDispatcher(BroadcastReceiver receiver, Context context,
                        Handler activityThread, Instrumentation instrumentation,
                        boolean registered) {

                    mIIntentReceiver = new InnerReceiver(this, !registered);
                    mReceiver = receiver;
                    mContext = context;
                    mActivityThread = activityThread;
              
                }
         public void performReceive(Intent intent, int resultCode, String data,
                        Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
                    final Args args = new Args(intent, resultCode, data, extras, ordered,
                            sticky, sendingUser);
                    i
                    if (intent == null || !mActivityThread.post(args.getRunnable())) {
                        if (mRegistered && ordered) {
                            IActivityManager mgr = ActivityManager.getService();
                            args.sendFinished(mgr);
                        }
                    }
                }
                
         IIntentReceiver getIIntentReceiver() {
            return mIIntentReceiver;
        }
    
}
```

创建一个 Args 实例, 在主线程中执行 args.getRunnable()

```  java
public void performReceive(Intent intent, int resultCode, String data,
                        Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
                    final Args args = new Args(intent, resultCode, data, extras, ordered,
                            sticky, sendingUser);
                    i
                    if (intent == null || !mActivityThread.post(args.getRunnable())) {
                        if (mRegistered && ordered) {
                            IActivityManager mgr = ActivityManager.getService();
                            args.sendFinished(mgr);
                        }
                    }
                }
```

* Args.getRunnable() 中 完成对 Intent 和 Receiver 的一些参数设置, 然后调用 receiver.onReceive() 方法。
* 最后调用 finish(): 针对串行广播, 需要通知 AMS 广播处理完成。

```  java
//Args是ReceiverDispatcher的一个内部类,持有可以方便的持有 ReceiverDispatcher中的mReceiver属性
//代表一个未处理完的结果,getRunnable()返回出现广播的真正的操作，处理完成之后,需要调用finish 通知AMS
final class Args extends BroadcastReceiver.PendingResult {
        private Intent mCurIntent;
        private final boolean mOrdered;
        private boolean mDispatched;
   

        public Args(Intent intent, int resultCode, String resultData, Bundle resultExtras,
                boolean ordered, boolean sticky, int sendingUser) {
            super(resultCode, resultData, resultExtras,
                    mRegistered ? TYPE_REGISTERED : TYPE_UNREGISTERED, ordered,
                    sticky, mIIntentReceiver.asBinder(), sendingUser, intent.getFlags());
            mCurIntent = intent;
            mOrdered = ordered;
        }

        public final Runnable getRunnable() {
            return () -> {
                final BroadcastReceiver receiver = mReceiver;
                final boolean ordered = mOrdered;

                final IActivityManager mgr = ActivityManager.getService();
          
                try {
                    //提取ClassLoader
                    ClassLoader cl = mReceiver.getClass().getClassLoader();
                    intent.setExtrasClassLoader(cl);
                    intent.prepareToEnterProcess();
                    setExtrasClassLoader(cl);
                    //设置receiver的pendingResult() 
                    receiver.setPendingResult(this);
                    //调用onReceive() 
                    receiver.onReceive(mContext, intent);
                } catch (Exception e) {
                    if (mRegistered && ordered) {
                        sendFinished(mgr);
                    }
                    
                }

                //对于有序广播,消息接收完毕,需要发送am.finishReceiver()消息，通知AMS 消息处理完毕,可以将分发下一个串行消息了。
                if (receiver.getPendingResult() != null) {
                    finish();
                }
         
            };
        }
    }
```

我们再看一下 InnerReceiver 类, 它继承自 IIntentReceiver.Stub，属于 Binder 的一个服务端类，可以被 AMS 调用, 通过 IPC 进行广播的分发。

* InnerReceiver 持有 ReceiverDispatcher 的一个弱引用
* AMS 分发广播时, 会调用 InnerReceiver 的 performReceive。
* InnerReceiver.performReceive() 直接会调用 ReceiverDispatcher 的 performReceive() 方法

```  java
//InnerReceiver 是专门用于接收AMS IPC 消息的Binder,AMS 向动态注册的BroadCastReceiver分发广播消息时,会调用InnerReceiver.performReceive() 
        final static class InnerReceiver extends IIntentReceiver.Stub {
                    final WeakReference<LoadedApk.ReceiverDispatcher> mDispatcher;
                    final LoadedApk.ReceiverDispatcher mStrongRef;

                    InnerReceiver(LoadedApk.ReceiverDispatcher rd, boolean strong) {
                        mDispatcher = new WeakReference<LoadedApk.ReceiverDispatcher>(rd);
                        mStrongRef = strong ? rd : null;
                    }

                    @Override
                    public void performReceive(Intent intent, int resultCode, String data,
                            Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
                        final LoadedApk.ReceiverDispatcher rd;
                        if (intent == null) {
                            Log.wtf(TAG, "Null intent received");
                            rd = null;
                        } else {
                            rd = mDispatcher.get();
                        }
                    
                        if (rd != null) {
                            rd.performReceive(intent, resultCode, data, extras,
                                    ordered, sticky, sendingUser);
                        } else {
                  
                            IActivityManager mgr = ActivityManager.getService();
                            try {
                                if (extras != null) {
                                    extras.setAllowFds(false);
                                }
                                mgr.finishReceiver(this, resultCode, data, extras, false, intent.getFlags());
                            } catch (RemoteException e) {
                                throw e.rethrowFromSystemServer();
                            }
                        }
                    }
                }

         }
```

总结一下 动态注册 BroadCastReceiver 时的广播分发流程:

    AMS 分发广播->InnerReceiver.performReceive()->BroadcastReceiver.performReceive()->Args().getRunnable()->BroadcastReceiver.onReceive()

### 2.2、广播的注册

发送广播是 Context 的行为能力，具体调用过程如下:

* Context.registerReceiver()
* ContextImpl.registerReceiver()
* ContextImpl.registerReceiverInternal()
* IActivityManager.registerReceiver()
* ActivityManagerService.registerReceiver()

```  java
## ContextImpl.java

 private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
            IntentFilter filter, String broadcastPermission,
            Handler scheduler, Context context, int flags) {
        IIntentReceiver rd = null;
    if (receiver != null) {
        
        //(1)LoadedApk.getReceiverDispatcher()创建一个IIntentReceiver对象
        rd = mPackageInfo.getReceiverDispatcher(
                    receiver, context, scheduler,
                    mMainThread.getInstrumentation(), true);
           
       //(2) IActivityManager.registerReceiver(),向AMS发送IPC消息,并将applicationThread和IIntentReceiver传递给AMS
            final Intent intent = ActivityManager.getService().registerReceiver(
                    mMainThread.getApplicationThread(), mBasePackageName, rd, filter,
                    broadcastPermission, userId, flags);
            if (intent != null) {
                intent.setExtrasClassLoader(getClassLoader());
                intent.prepareToEnterProcess();
            }
            return intent;
      
    }
```

我们再看一下 IIntentReceiver 实例是如何创建的:

* 首先创建了一个 ReceiverDispatcher() 实例, ReceiverDispatcher 内部持有一个 IIntentReceiver 实例.
* ReceiverDispatcher.getIIntentReceiver() 将 ReceiverDispatcher 内部的 IIntentReceiver 返回。

调用 IActivityManager.registerReceiver() 之后, 我们梳理一下, 对象的持有关系

* ActivtivityMangerService 通过 Binder 持有 IIntentReceiver 实例
* IIntentReceiver 持有一个 ReceiverDispatcher 的 WeakReference 弱引用。
* ReceiverDispatcher 持有注册的 BroadCastReceiver 实例。

```  java
## LoadedApk.java
public IIntentReceiver getReceiverDispatcher(BroadcastReceiver r,
            Context context, Handler handler,
            Instrumentation instrumentation, boolean registered) {
        synchronized (mReceivers) {
            LoadedApk.ReceiverDispatcher rd = null;
            ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher> map = null;
            if (registered) {
                map = mReceivers.get(context);
                if (map != null) {
                    rd = map.get(r);
                }
            }
            if (rd == null) {
                rd = new ReceiverDispatcher(r, context, handler,
                        instrumentation, registered);
                if (registered) {
                    if (map == null) {
                        map = new ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher>();
                        mReceivers.put(context, map);
                    }
                    map.put(r, rd);
                }
            } else {
                rd.validate(context, handler);
            }
            rd.mForgotten = false;
            return rd.getIIntentReceiver();
        }
    }
```

我们看一下 ActivityManagerService.registerReceiver() 方法

```  java
## ActivityManagerService.java
final HashMap<IBinder, ReceiverList> mRegisteredReceivers = new HashMap<>();
    
public Intent registerReceiver(IApplicationThread caller, String callerPackage,
        IIntentReceiver receiver, IntentFilter filter, String permission, int userId,
        int flags) {


    // 创建ReceiverList对象,以receiver为key，以eceiverList为value, 保存到mRegisteredReceivers.
    ReceiverList rl = mRegisteredReceivers.get(receiver.asBinder());
     if (rl == null) {
                rl = new ReceiverList(this, callerApp, callingPid, callingUid,
                        userId, receiver);
                mRegisteredReceivers.put(receiver.asBinder(), rl);
            } 

    // 创建BroadcastFilter,保存到mReceiverResolver中
     BroadcastFilter bf = new BroadcastFilter(filter, rl, callerPackage,
            permission, callingUid, userId, instantApp, visibleToInstantApps);


     rl.add(bf);
     mReceiverResolver.addFilter(bf);

}
```

mReceiverResolver 记录着所有已经注册的广播，是以 receiver IBinder 为 key， ReceiverList 为 value 的 ArrayMap。

### 2.3、广播的发送

发送广播是 Context 的行为, 所以调用的起点也是 Context

* Context.sendBroadcast()
* ContextImpl.sendBroadcast()
* ActivityManager.getService().broadcastIntent()
* ActivityManagerService.broadcastIntent()
* ActivityManagerService.broadcastIntentLocked()

broadcastIntentLocked 是具体发送广播地方, 代码较长, 主要做了以下 8 步操作

```  java
final int broadcastIntentLocked(ProcessRecord callerApp,
            String callerPackage, Intent intent, String resolvedType,
            IIntentReceiver resultTo, int resultCode, String resultData,
            Bundle resultExtras, String[] requiredPermissions, int appOp, Bundle bOptions,
            boolean ordered, boolean sticky, int callingPid, int callingUid, int userId) {
        intent = new Intent(intent);

        //setp1：设置广播flags
         //setp2：广播权限验证
         //setp3：处理系统相关广播
         //setp4：增加sticky广播
         //setp5：查询receivers和registeredReceivers
         //setp6：处理并行广播
         //setp7：合并registeredReceivers到receivers
         //setp8：处理串行广播


}
```

我们只看重要的操作 step5-step8

#### 2.3.1、查询有哪些 Receiver 会接收该广播

```  java
// Figure out who all will receive this broadcast.
        List receivers = null;
        List<BroadcastFilter> registeredReceivers = null;
        // Need to resolve the intent to interested receivers...
         //当允许静态接收者处理广播时，则通过PKMS根据intent查询静态receivers
        if ((intent.getFlags()&Intent.FLAG_RECEIVER_REGISTERED_ONLY)
                 == 0) {
            receivers = collectReceiverComponents(intent, resolvedType, callingUid, users);
        }
        if (intent.getComponent() == null) {
            if (userId == UserHandle.USER_ALL && callingUid == SHELL_UID) {
                // Query one target user at a time, excluding shell-restricted users
                for (int i = 0; i < users.length; i++) {
                    if (mUserController.hasUserRestriction(
                            UserManager.DISALLOW_DEBUGGING_FEATURES, users[i])) {
                        continue;
                    }

                     //查询匹配的动态注册广播Receiver
                    List<BroadcastFilter> registeredReceiversForUser =
                            mReceiverResolver.queryIntent(intent,
                                    resolvedType, false /*defaultOnly*/, users[i]);
                    if (registeredReceivers == null) {
                        registeredReceivers = registeredReceiversForUser;
                    } else if (registeredReceiversForUser != null) {
                        registeredReceivers.addAll(registeredReceiversForUser);
                    }
                }
            } else {
                registeredReceivers = mReceiverResolver.queryIntent(intent,
                        resolvedType, false /*defaultOnly*/, userId);
            }
        }
```

1. 根据 userId 判断发送的是全部的接收者还是指定的 userId

2. 查询广播，并将其放入到两个列表：

registeredReceivers：来匹配当前 intent 的所有动态注册的广播接收者（mReceiverResolver 见 2.4 节）

receivers：记录当前 intent 的所有静态注册的广播接收者

```  java
private List<ResolveInfo> collectReceiverComponents(Intent intent, String resolvedType,
            int callingUid, int[] users) {
       ...
      //调用PKMS的queryIntentReceivers，可以获取AndroidManifeset中注册的接收信息
       List<ResolveInfo> newReceivers = AppGlobals.getPackageManager()
                        .queryIntentReceivers(intent, resolvedType, pmFlags, user).getList();
       ...        
       return receivers;
    }
```

#### 2.3.2、处理并行广播

```  java
int NR = registeredReceivers != null ? registeredReceivers.size() : 0;
        if (!ordered && NR > 0) {
            // If we are not serializing this broadcast, then send the
            // registered receivers separately so they don't wait for the
            // components to be launched.
         
            //(1)判断是前台广播队列还是后台广播队列
            final BroadcastQueue queue = broadcastQueueForIntent(intent);
            //(2)创建BroadcastRecord实例,指定接收者为registeredReceivers
            BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
                    callerPackage, callingPid, callingUid, callerInstantApp, resolvedType,
                    requiredPermissions, appOp, brOptions, registeredReceivers, resultTo,
                    resultCode, resultData, resultExtras, ordered, sticky, false, userId);
           
            final boolean replaced = replacePending
                    && (queue.replaceParallelBroadcastLocked(r) != null);
            // Note: We assume resultTo is null for non-ordered broadcasts.
            if (!replaced) {
                //(3)将BroadcastRecord加入到并发队列
                queue.enqueueParallelBroadcastLocked(r);
                //(4)处理该BroadcastRecord
                queue.scheduleBroadcastsLocked();
            }
            
            //(5)处理完成后,将registeredReceivers置空
            registeredReceivers = null;
            NR = 0;
        }
```

广播队列中有一个 mParallelBroadcasts 变量，类型为 ArrayList，记录所有的并行广播

```  java
public void enqueueParallelBroadcastLocked(BroadcastRecord r) {
        mParallelBroadcasts.add(r);
        enqueueBroadcastHelper(r);
  }
```

#### 2.3.3、合并 registeredReceivers 到 receivers

```  java
// Merge into one list.
        int ir = 0;
        if (receivers != null) {
            // A special case for PACKAGE_ADDED: do not allow the package
            // being added to see this broadcast.  This prevents them from
            // using this as a back door to get run as soon as they are
            // installed.  Maybe in the future we want to have a special install
            // broadcast or such for apps, but we'd like to deliberately make
            // this decision.
            String skipPackages[] = null;
            if (Intent.ACTION_PACKAGE_ADDED.equals(intent.getAction())
                    || Intent.ACTION_PACKAGE_RESTARTED.equals(intent.getAction())
                    || Intent.ACTION_PACKAGE_DATA_CLEARED.equals(intent.getAction())) {
                Uri data = intent.getData();
                if (data != null) {
                    String pkgName = data.getSchemeSpecificPart();
                    if (pkgName != null) {
                        skipPackages = new String[] { pkgName };
                    }
                }
            } else if (Intent.ACTION_EXTERNAL_APPLICATIONS_AVAILABLE.equals(intent.getAction())) {
                skipPackages = intent.getStringArrayExtra(Intent.EXTRA_CHANGED_PACKAGE_LIST);
            }
            if (skipPackages != null && (skipPackages.length > 0)) {
                for (String skipPackage : skipPackages) {
                    if (skipPackage != null) {
                        int NT = receivers.size();
                        for (int it=0; it<NT; it++) {
                            ResolveInfo curt = (ResolveInfo)receivers.get(it);
                            if (curt.activityInfo.packageName.equals(skipPackage)) {
                                receivers.remove(it);
                                it--;
                                NT--;
                            }
                        }
                    }
                }
            }

            int NT = receivers != null ? receivers.size() : 0;
            int it = 0;
            ResolveInfo curt = null;
            BroadcastFilter curr = null;
            while (it < NT && ir < NR) {
                if (curt == null) {
                    curt = (ResolveInfo)receivers.get(it);
                }
                if (curr == null) {
                    curr = registeredReceivers.get(ir);
                }
                if (curr.getPriority() >= curt.priority) {
                    // Insert this broadcast record into the final list.
                    receivers.add(it, curr);
                    ir++;
                    curr = null;
                    it++;
                    NT++;
                } else {
                    // Skip to the next ResolveInfo in the final list.
                    it++;
                    curt = null;
                }
            }
        }
        while (ir < NR) {
            if (receivers == null) {
                receivers = new ArrayList();
            }
            receivers.add(registeredReceivers.get(ir));
            ir++;
        }
```

registeredReceivers 代表动态注册的 BroadCastReceiver, 它可以接收串行广播，也可以接收并行广播。

* 如果当前的广播是并发的, 则 在 2.3.2 节中, registeredReceivers 中的 receiver 就已经被消费, registeredReceivers 会置空, registeredReceivers 合并到 receiver 等于没有合并。
* 如果当前的广播是串行的, 则 registeredReceivers 会被保留, 而 receivers 保存的是静态注册的 BroadReceiver, 静态注册的 Receiver 都会被按照串行来处理
* 所以 registeredReceivers 合并到 receiver 的最终结果, 是一个需要被串行处理的 recevier 的集合。
* 静态广播保存在 receivers 中的是 ResolveInfo, 动态广播保存在 receivers 中的是 BroadcastFilter

#### 2.3.4、处理串行广播

```  java
if ((receivers != null && receivers.size() > 0)
                || resultTo != null) {
            
            //(1)选择BroadcastQueue 区分前台广播还是后台广播
            BroadcastQueue queue = broadcastQueueForIntent(intent);

            //(2)生成BroadcastRecord对象,指定接收者为receivers
            BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
                    callerPackage, callingPid, callingUid, callerInstantApp, resolvedType,
                    requiredPermissions, appOp, brOptions, receivers, resultTo, resultCode,
                    resultData, resultExtras, ordered, sticky, false, userId);

   
            final BroadcastRecord oldRecord =
                    replacePending ? queue.replaceOrderedBroadcastLocked(r) : null;
            if (oldRecord != null) {
                // Replaced, fire the result-to receiver.
                if (oldRecord.resultTo != null) {
                    final BroadcastQueue oldQueue = broadcastQueueForIntent(oldRecord.intent);
                    try {
                        oldQueue.performReceiveLocked(oldRecord.callerApp, oldRecord.resultTo,
                                oldRecord.intent,
                                Activity.RESULT_CANCELED, null, null,
                                false, false, oldRecord.userId);
                    } catch (RemoteException e) {
                        Slog.w(TAG, "Failure ["
                                + queue.mQueueName + "] sending broadcast result of "
                                + intent, e);

                    }
                }
            } else {
                //(3)将BroadcastRecord 加入到串行队列
                queue.enqueueOrderedBroadcastLocked(r);
                //(4)BroadcastQueue处理该广播
                queue.scheduleBroadcastsLocked();
            }
        } else {
            // There was nobody interested in the broadcast, but we still want to record
            // that it happened.
            if (intent.getComponent() == null && intent.getPackage() == null
                    && (intent.getFlags()&Intent.FLAG_RECEIVER_REGISTERED_ONLY) == 0) {
                // This was an implicit broadcast... let's record it for posterity.
                addBroadcastStatLocked(intent.getAction(), callerPackage, 0, 0, 0);
            }
        }
```

广播队列中有一个 mOrderedBroadcasts 变量，类型为 ArrayList，记录所有的有序广播

```  java
/串行广播加入到mOrderedBroadcasts队列
public void enqueueOrderedBroadcastLocked(BroadcastRecord r) {
        mOrderedBroadcasts.add(r);
        enqueueBroadcastHelper(r);
  }
```

#### 2.3.5、小结

发送广播的过程：

1. 默认不发送给已停止的（FLAG_EXCLUDE_STOPPED_PACKAGES）应用和即时应用（需要添加该 FLAG_RECEIVER_VISIBLE_TO_INSTANT_APPS 标记才可以）
2. 对广播进行权限验证，是否是受保护的广播，是否允许后台接收广播，是否允许后台发送广播
3. 处理系统广播，主要是 package、时间、网络相关的广播
4. 当为粘性广播时，将 sticky 广播增加到 list，并放入 mStickyBroadcasts 队列
5. 当广播的 intent 没有设置 FLAG_RECEIVER_REGISTERED_ONLY，则允许静态广播接收者来处理该广播
6. 创建 BroadcastRecord 对象，并将该对象加入到相应的广播队列，然后调用 BroadcastQueue 的 scheduleBroadcastsLocked 方法来完成不同广播的处理。

不同广播的处理方式：

1. sticky 广播：广播注册过程中处理 AMS.registerReceiver，开始处理粘性广播，

* 创建 BroadcastRecord 对象
* 添加到 mParallelBroadcasts 队列
* 然后执行 queue.scheduleBroadcastsLocked()

2. 并行广播

* 只有动态注册的 registeredReceivers 才会进行并行处理
* 创建 BroadcastRecord 对象
* 添加到 mParallelBroadcasts 队列
* 然后执行 queue.scheduleBroadcastsLocked()

3. 串行广播

* 所有静态注册的 receivers 以及动态注册的 registeredReceivers（当发送的广播是有序广播时）合并到一张表处理
* 创建 BroadcastRecord 对象
* 添加到 mOrderedBroadcasts 队列
* 然后执行 queue.scheduleBroadcastsLocked()

### 2.4、广播的接收

上一节讲到创建的 BroadcastRecord 添加到 BroadcastQue 中，都会调用 queue.scheduleBroadcastsLocked() 来对 BroadcastRecord 向 BroadcastReceier 进行分发。

scheduleBroadcastsLocked 实际上是发送了一调 Handler 消息 BROADCAST_INTENT_MSG

```  java
BroadcastQueue.java
  public void scheduleBroadcastsLocked() {
        if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Schedule broadcasts ["
                + mQueueName + "]: current="
                + mBroadcastsScheduled);

        if (mBroadcastsScheduled) {
            return;
        }
        mHandler.sendMessage(mHandler.obtainMessage(BROADCAST_INTENT_MSG, this));
        mBroadcastsScheduled = true;
    }
```

参照 2.1.2 节, 调用过程如下

* BroadcastQueue.scheduleBroadcastsLocked
* 发送 BROADCAST_INTENT_MSG
* BroadcastQueue.processNextBroadcast()
* BroadcastQueue.processNextBroadcastLocked()

```  java
final void processNextBroadcastLocked(boolean fromMsg, boolean skipOomAdj) {
        BroadcastRecord r;

        // First, deliver any non-serialized broadcasts right away.
        // (1) 遍历mParallelBroadcasts中的所有BroadRecord,依次分发给BroadRecord指定的receivers
        while (mParallelBroadcasts.size() > 0) {
            r = mParallelBroadcasts.remove(0);
            r.dispatchTime = SystemClock.uptimeMillis();
            r.dispatchClockTime = System.currentTimeMillis();

          System.identityHashCode(r));
            }

            final int N = r.receivers.size();
        
            for (int i=0; i<N; i++) {
                Object target = r.receivers.get(i);
                //调用deliverToRegisteredReceiverLocked 向BroadcastReceiver进行广播分发
                deliverToRegisteredReceiverLocked(r, (BroadcastFilter)target, false, i);
            }
            //将处理过的BroadcastReceiver,保存到mBroadcastHistory当中
            addBroadcastToHistoryLocked(r);
           
        }


        //(2) mPendingBroadcast != null, 如果当前正在等待新的进程启动，来处理下一个串行广播,则继续等待;如果等待的进程已死,则mPendingBroadcast置空

        // If we are waiting for a process to come up to handle the next
        // broadcast, then do nothing at this point.  Just in case, we
        // check that the process we're waiting for still exists.
        if (mPendingBroadcast != null) {
            if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG_BROADCAST,
                    "processNextBroadcast [" + mQueueName + "]: waiting for "
                    + mPendingBroadcast.curApp);

            boolean isDead;
            if (mPendingBroadcast.curApp.pid > 0) {
                synchronized (mService.mPidsSelfLocked) {
                    ProcessRecord proc = mService.mPidsSelfLocked.get(
                            mPendingBroadcast.curApp.pid);
                    isDead = proc == null || proc.crashing;
                }
            } else {
                final ProcessRecord proc = mService.mProcessNames.get(
                        mPendingBroadcast.curApp.processName, mPendingBroadcast.curApp.uid);
                isDead = proc == null || !proc.pendingStart;
            }
            if (!isDead) {
                // It's still alive, so keep waiting
                return;
            } else {
                Slog.w(TAG, "pending app  ["
                        + mQueueName + "]" + mPendingBroadcast.curApp
                        + " died before responding to broadcast");
                mPendingBroadcast.state = BroadcastRecord.IDLE;
                mPendingBroadcast.nextReceiver = mPendingBroadcastRecvIndex;
                mPendingBroadcast = null;
            }
        }



        //(3) 对mOrderedBroadcasts 串行中的BroadcastRecord做 超时判断,若已超时,则broadcastTimeoutLocked()做超时处理,直到mOrderedBroadcasts队首的BroadcastRecord 未超时
        do {
            r = mOrderedBroadcasts.get(0);
            boolean forceReceive = false;

  
            int numReceivers = (r.receivers != null) ? r.receivers.size() : 0;
            if (mService.mProcessesReady && r.dispatchTime > 0) {
                long now = SystemClock.uptimeMillis();
                if ((numReceivers > 0) &&
                        (now > r.dispatchTime + (2*mTimeoutPeriod*numReceivers))) {
                   
                    broadcastTimeoutLocked(false); // forcibly finish this broadcast
                    forceReceive = true;
                    r.state = BroadcastRecord.IDLE;
                }
            }

            if (r.receivers == null || r.nextReceiver >= numReceivers
                    || r.resultAbort || forceReceive) {
                mOrderedBroadcasts.remove(0);
                r = null;
                looped = true;
                continue;
            }
        } while (r == null);


        //(4) 取出当前BroadcastRecord的 下一个接收者Receiver, 仅完成对该单个Receiver的广播分发。
        //若接收者的进程已经存在,则直接分发,若接收者的进程不存在,则首先创建接收者进程。

      
        int recIdx = r.nextReceiver++;
 
        final Object nextReceiver = r.receivers.get(recIdx);

        //如果接收者是动态注册的BroadcastReceiver,直接调用deliverToRegisteredReceiverLocked() 来向receiver分发广播
        if (nextReceiver instanceof BroadcastFilter) {
            // Simple case: this is a registered receiver who gets
            // a direct call.
            BroadcastFilter filter = (BroadcastFilter)nextReceiver;
    
            deliverToRegisteredReceiverLocked(r, filter, r.ordered, recIdx);
           
            return;
        }


        //如果接收者是静态注册广播,
         ResolveInfo info = (ResolveInfo)nextReceiver;
           // 接收者的进程存在,则调用processCurBroadcastLocked，想receiver进行广播分发
        if (app != null && app.thread != null && !app.killed) {
            try {
                app.addPackage(info.activityInfo.packageName,
                        info.activityInfo.applicationInfo.versionCode, mService.mProcessStats);
                processCurBroadcastLocked(r, app, skipOomAdj);
                return;
            }

            // 接收者的进程不存在,则调用startProcessLocked 创建接收者进程
            if ((r.curApp=mService.startProcessLocked(targetProcess,
                info.activityInfo.applicationInfo, true,
                r.intent.getFlags() | Intent.FLAG_FROM_BACKGROUND,
                "broadcast", r.curComponent,
                (r.intent.getFlags()&Intent.FLAG_RECEIVER_BOOT_UPGRADE) != 0, false, false))
                        == null) {

             }
            mPendingBroadcast = r;

}
```

processNextBroadcastLocked 代码很长, 大概分为四个主要操作

* (1) 遍历 mParallelBroadcasts 中的所有 BroadcastRecord, 针对 BroadcastRecord 的所有接收者 receiver 并发进行分发。所有并发 BroadcastRecord 的所有接收者 Recevier 在此 都派发完成。
* (2) mPendingBroadcast != null, 当前正在等待新的进程启动, 来处理下一个串行广播, 则继续等待; 如果等待的进程已死, 则 mPendingBroadcast 置空。
* 对 mOrderedBroadcasts 串行中的 BroadcastRecord 做 超时判断, 若已超时, 则 broadcastTimeoutLocked() 做超时处理, 直到 mOrderedBroadcasts 队首的 BroadcastRecord 未超时
* (4) 取出队首 BroadcastRecord 的 下一个接收者 Receiver, 仅完成对该单个 Receiver 的广播分发。

```
若接收者的进程已经存在,则直接调用deliverToRegisteredReceiverLocked进行分发
若接收者的进程不存在,则首先创建接收者进程。
```

备注:  
一次 processNextBroadcastLocked 调用

* 所有 mParallelBroadcasts 中的 BroadcasrRecord 都会被分发完成。
* 对于 mOrderedBroadcasts, 最多会处理队首串行 BroadcastRecord 向其中一个 receiver 的广播分发。比如队首串行 BroadcastRecord 有 10 个接收者 receiver, 那么需要 10 次 processNextBroadcastLocked 才能完成 BroadcastRecord 的分发。

#### 2.4.1、deliverToRegisteredReceiverLocked

deliverToRegisteredReceiverLocked 完成动态注册 BroadcastReceiver 的分发

* deliverToRegisteredReceiverLocked
* performReceiveLocked

```  java
void performReceiveLocked(ProcessRecord app, IIntentReceiver receiver,
            Intent intent, int resultCode, String data, Bundle extras,
            boolean ordered, boolean sticky, int sendingUser) throws RemoteException {
        // Send the intent to the receiver asynchronously using one-way binder calls.
        if (app != null) {
            if (app.thread != null) {
                // If we have an app thread, do the call through that so it is
                // correctly ordered with other one-way calls.
              
                app.thread.scheduleRegisteredReceiver(receiver, intent, resultCode,
                        data, extras, ordered, sticky, sendingUser, app.repProcState);
            
                
            } else {
                // Application has died. Receiver doesn't exist.
                throw new RemoteException("app.thread must not be null");
            }
        } else {
            receiver.performReceive(intent, resultCode, data, extras, ordered,
                    sticky, sendingUser);
        }
    }
```

ApplicationThread.scheduleRegisteredReceive() 方法内部会调用

IIntentReceiver.performReceive(), 参照 2.1.3 节, 可知最终会调用 BroadcastReceiver.onReceive() 方法。

```  java
public void scheduleRegisteredReceiver(IIntentReceiver receiver, Intent intent,
                int resultCode, String dataStr, Bundle extras, boolean ordered,
                boolean sticky, int sendingUser, int processState) throws RemoteException {
            updateProcessState(processState, false);
            receiver.performReceive(intent, resultCode, dataStr, extras, ordered,
                    sticky, sendingUser);
        }
```

#### 2.4.2、processCurBroadcastLocked()

processCurBroadcastLocked 完成静态注册广播的分发

```  java
private final void processCurBroadcastLocked(BroadcastRecord r,
            ProcessRecord app, boolean skipOomAdj) throws RemoteException {
     
        ...
        app.thread.scheduleReceiver(new Intent(r.intent), r.curReceiver,mService.compatibilityInfoForPackageLocked(r.curReceiver.applicationInfo),
                r.resultCode, r.resultData, r.resultExtras, r.ordered, r.userId,
                app.repProcState);
        ...
       
       
    }
```

processCurBroadcastLocked 内部会调用 ApplicationThread.scheduleReceiver()

* ApplicationThread.scheduleReceiver()
* sendMessage(H.RECEIVER, r);
* ActivityThread.handleReceiver()

```  java
private void handleReceiver(ReceiverData data) {
  
      
        LoadedApk packageInfo = getPackageInfoNoCheck(
                data.info.applicationInfo, data.compatInfo);

        IActivityManager mgr = ActivityManager.getService();

         //(1) 调用通过loadedApk 创建一个receiver实例
        app = packageInfo.makeApplication(false, mInstrumentation);
        context = (ContextImpl) app.getBaseContext();
        if (data.info.splitName != null) {
            context = (ContextImpl) context.createContextForSplit(data.info.splitName);
        }
        java.lang.ClassLoader cl = context.getClassLoader();
        data.intent.setExtrasClassLoader(cl);
        data.intent.prepareToEnterProcess();
        data.setExtrasClassLoader(cl);
     
        receiver = packageInfo.getAppFactory()
                .instantiateReceiver(cl, data.info.name, data.intent);
     

      
        //(2)调用receiver.onReceive
        sCurrentBroadcastIntent.set(data.intent);
        receiver.setPendingResult(data);
        receiver.onReceive(context.getReceiverRestrictedContext(),
                data.intent);
       
       //(3)广播接收完毕,需要调用finish,通知AMS,触发下一次广播分发
        if (receiver.getPendingResult() != null) {
            data.finish();
        }
    }
```

### 2.5 总结

广播机制

1. 当发送串行广播（order= true）时

* 静态注册的广播接收者（receivers），采用串行处理
* 动态注册的广播接收者（registeredReceivers），采用串行处理

2. 当发送并行广播（order= false）时

* 静态注册的广播接收者（receivers），采用串行处理
* 动态注册的广播接收者（registeredReceivers），采用并行处理

> 静态注册的 receiver 都是采用串行处理；动态注册的 registeredReceivers 处理方式是串行还是并行，取决于广播的发送方式（processNextBroadcast）；静态注册的广播由于其所在的进程没有创建，而进程的创建需要耗费系统的资源比较多，所以让静态注册的广播串行化，防止瞬间启动大量的进程。

> 广播 ANR 只有在串行广播时才需要考虑，因为接收者是串行处理的，前一个 receiver 处理慢，会影响后一个 receiver；并行广播通过一个循环一次性将所有的 receiver 分发完，不存在彼此影响的问题，没有广播超时。

串行超时情况：
1. 某个广播处理时间 >  mTimeoutPeriod，其中 mTimeoutPeriod在前台队列为 10s, 后台队列为 60s；
2. 某个 receiver 的执行时间超过 mTimeoutPeriod。

