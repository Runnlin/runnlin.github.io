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

![][img-0]

广播机制

1. 当发送串行广播（order= true）时

* 静态注册的广播接收者（receivers），采用串行处理
* 动态注册的广播接收者（registeredReceivers），采用串行处理

2. 当发送并行广播（order= false）时

* 静态注册的广播接收者（receivers），采用串行处理
* 动态注册的广播接收者（registeredReceivers），采用并行处理

> 静态注册的 receiver 都是采用串行处理；动态注册的 registeredReceivers 处理方式是串行还是并行，取决于广播的发送方式（processNextBroadcast）；静态注册的广播由于其所在的进程没有创建，而进程的创建需要耗费系统的资源比较多，所以让静态注册的广播串行化，防止瞬间启动大量的进程。

> 广播 ANR 只有在串行广播时才需要考虑，因为接收者是串行处理的，前一个 receiver 处理慢，会影响后一个 receiver；并行广播通过一个循环一次性将所有的 receiver 分发完，不存在彼此影响的问题，没有广播超时。
>
> 串行超时情况：某个广播处理时间 > 2receiver 总个数 mTimeoutPeriod，其中 mTimeoutPeriod，前后队列为 10s, 后台队列为 60s；某个 receiver 的执行时间超过 mTimeoutPeriod。

三、参考文章
------

[https://blog.csdn.net/cao861544325/article/details/103846442](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fcao861544325%2Farticle%2Fdetails%2F103846442)  
[https://www.jianshu.com/p/fecc4023abb8](https://www.jianshu.com/p/fecc4023abb8)

[img-0]:data:image/jpeg;base64,/9j/4AAQSkZJRgABAQAAAQABAAD/2wBDAAMCAgICAgMCAgIDAwMDBAYEBAQEBAgGBgUGCQgKCgkICQkKDA8MCgsOCwkJDRENDg8QEBEQCgwSExIQEw8QEBD/2wBDAQMDAwQDBAgEBAgQCwkLEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBAQEBD/wAARCAMqBToDAREAAhEBAxEB/8QAHQABAQEAAgMBAQAAAAAAAAAAAAYHBAUCAwgBCf/EAGYQAAAGAQMBBQUEBAoEBg8AEwABAgMEBQYHERITFBchZ5YIFSIx4xZHd8MYMlaVIzQ3QVFXdrTS0wlhcYEzQliEprEkKDhDSHN1goeRoaWzwcU1NkRSJSZicnSytbZFVWaDotHh/8QAFAEBAAAAAAAAAAAAAAAAAAAAAP/EABQRAQAAAAAAAAAAAAAAAAAAAAD/2gAMAwEAAhEDEQA/AP6pgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAPlDEMei6nX+YXeomv2eUE+TqHc4xR11dmCqqO63EUaWY8aOjiS3CabUoySRqVxUtW58jAaL+i9Xf15a1euZQB+i9Xf15a1euZQB+i9Xf15a1euZQB+i9Xf15a1euZQB+i9Xf15a1euZQDrJGgOHRLyHi8r2j9WGLmwZdlRK5zUN9MqQy0aSdcbaM+a0INaCUoiMi5J323IB2f6L1d/XlrV65lAH6L1d/XlrV65lAOHE9nHG7B2XHg+0FrBJcgPdnlttZ/JWqO7xSvg4RHuhXFaFbHseyiP5GQDmfovV39eWtXrmUAfovV39eWtXrmUA/D9l6tItz1z1qIi//rqUA4tb7OGOXEFizqPaB1hnQpKCcYkRs/kutOoP5KStJmSi/wBZGA5f6L1d/XlrV65lAH6L1d/XlrV65lAH6L1d/XlrV65lAH6L1d/XlrV65lAJHSDSi0yD7aRLHW/VVxuhy2dUQ+WTKWoo7bbKkkpS0GpR7uK8TP8AoAaH3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB7P027cpcrpLnI7O7LH8ts6mHLsnUuyTjNqQpCVuEkuZka1eJ+O2xfIiAaqAAAAAAAD4ooM3ybEsqoKqhntR4uTe0LlNZZoXEZeN6MUeY6SEqcQo2z5tIPkg0q8Nt9jMjDx081m16ks6MZVkuqKbKDqNlNvik6sRSw2UstoRNWxKS6lvn10HFSW3/AAaiPxQZkajCN0/1O1awLRPB6/HsvyzIZ2oOot9XT58NikVYwCbeluKbidrQzEJ15xvkfaDWRbrJCduCCC4wnWP2htQdSNMcCfzhrGW7avypy2dbjVFk/MTWzYrcdxaoy3o7Mg0OKQ6htzglRufARkjiHjU+1FnMvXjEY1LlF3fYfmOVXOPIal1VRBrEtw2JB7REofXZqeQ8wTanX0pZWSt0pSZpIw77TvUHUfUPTQtRrj2kIVPYZVXXqWMTarK0nYD8cnSQiCpaesb7BNkbvWKQk/H4EEAzDR3V3UjTvTzRFFbkEnK2ZujuQ5U7ClQoanHJkZqGphhLrTKXSQ2bjiduXJRK+M1mRGQW8LXDVnHToy73Gc2PNtNbfMHDRWQW00EqPGbcZcj9FCTOOa3Db4yOqo1JL4z8SAUGjGqWsx6oaVUWe6gNZJX6m6eScjfi+6Y0RNdNjdkPkyppJLUTiZJ8iWZkSk7pJJHxIObPucqxm71fzPGs/dhroMzrzOjaixXGbDrQ69BtSDcbU8RrJRpbNlbZkr58/kA7dGZ6mSEKee1CnRmb/UCZiLDrMCCSKeG09J4OoNbJ831m0hklO80fEj+DNW6lB5StQM5Q7Y6dx8yySytYOUOVcKfT1lUq1sorcBqS8k1yiar2ltLe4rWbfilHFKCWfIg4Gn2pmoGpf2WoZuczce51t/Olz2Y1auVPVAsSiNtu7odjls2fN3oERGoy4KSnwMOu0dyHK7nCsYxqBqceHV1BgFfkBWTMSE7HnrdcfStb/aUr2jNkyg1E2tpWzv8AwpeBgO2y7VrM4GZRregya9sKNGWVeNukzAqWKIzecZbfbNTrqrF14uqpaXGdmy+AuKiStSg8bLLdSGH7mHlGb3sZm/Yu2qSZWs08uleJtl9xlEVxCO1tSENNcl9pS40akuJIzM07Br2isKTB0kw2PMuJdm57jhrOTKS0lxRKZSZJMmkIRskjJJfDvsRbmZ7mYUt/f0mKUk7JcjtYtZV1kdcqZMlOk2ywygt1LWo/AiIi+YDNvZ0TZzscyXMptJOqomXZVPvatie10pK4DpNoYdcaP4mjcS3zJCyJaUqSSiSe5EGtgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAMq0C+8f8QLj8oBqoAAAAAAAPmTQq60mbdzxvObbEU2dNqtk0+vTbPRifhPKfUgnmidPk2o0LWnmnY+KlFvsZgNOYtvZvix6mNFsdNmWKGWufUttvQEogSVkslvMER7NOKJxwjWjYz5q3PxMBwpDXsqyqq3opTWlD1bfyjm28Nwq1TFhIP5vSGz+F1z/APKWRn/rAdhXXvs7VUqrm1dxpzCkUcRyBVux5EFtcGM4aTcZYUkyNptRoQakJ2I+Kdy8CAdeTXsrFMl2BN6U9qsLBq2lv7VvUkTmj3bkuK+a3kn4pcPdRfzGQDlVs72aKbI5+Y1EzTKDf2qTTOtYzle1MlEfzJ15Oy1kf/5RmA9VE/7MOKyYs7F3dLqiRBTJTFegKro62CkKSqQSFI2NJOqQg17bcjSk1b7EAUq/Zdxv3unHV6W1RZASityhHXMe8ORGSu0cNuruRmR8999wHOi5F7PkKbV2UO+08Yl0cNddVyGpUFLkGIvhyYYUR7tNn0290J2SfBPh4EA40ux9mqdfs5TNnaZyLqPI7W1YuuV65Tb/ABSnqpdM+ZL4pSnkR77JIv5iAcyVlWgU+om4/PyLT+RV2LjjsyC7MhLjyVuK5LU42Z8VmpXxGZkZmfifiA6+S/7MEzH4eJzHNLn6Ovd68SsdVXKiR3PH422T+BCviV4kRH4n/SA8Zf6Ldgk0WJ6WSU9uOz2eOuWXbDIiOR4/992Ii5/rbEXiA87iT7MWQM1Ua+f0vsmaJfVqm5aq95MBf/3zBK3Jo/AvFOwDxsXvZht7Cwt7V3S6bOtmUsWEqQqucdltpNJpQ6tW5uJI0I2JRmRcS/oIB7K2b7NFJeTMoppemMC5npUiXYRnK9qTISrbkTjqdlrI9i3IzPfYgHLh6j+z/p7jRxqrOMCx6hq23HuhDsIcaNHRua1mSEKJKdzNRnsXiZn/ADmAkKGgu/aHuoOoGoFTKrNPq2Qibi+LTWjbetHkHyatLJpXiREZEqPFUXwfC66XU4IZDdQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABlWgX3j/AIgXH5QDVQAAAAAAAYXo3gWDZDN1Hsb/AAyispas9tkG/MrmXnDSXS2LktJnsQDSe6fS3+rTFf3NG/wAHdPpb/Vpiv7mjf4ADun0t/q0xX9zRv8AAAd0+lv9WmK/uaN/gAO6fS3+rTFf3NG/wAHdPpb/AFaYr+5o3+AA7p9Lf6tMV/c0b/AAd0+lv9WmK/uaN/gAO6fS3+rTFf3NG/wAHdPpb/Vpiv7mjf4ADun0t/q0xX9zRv8AAAd0+lv9WmK/uaN/gAO6fS3+rTFf3NG/wAHdPpb/AFaYr+5o3+AA7p9Lf6tMV/c0b/AAFpRpakyUnTXFSMj3Iypo/h//AIAKsAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAGVaBfeP+IFx+UA1UAAAAAAAGVaBfeP8AiBcflANVAAAAAAAAAfP2E4Rbak2ucWlpqpn9d7uzCzrI0asvFNMNsNqSpBEhSVbbczLYjItiIiIBWdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mA6jK9GJFHjFxdw9Z9UFvwIEiU0lzIjNBrbbUoiURII9tyLfYyAXukdhOttKcLtLOW7KmTMerpEh95ZqW66uM2pS1KPxMzMzMz/pMBXgAD5+wnCLbUm1zi0tNVM/rvd2YWdZGjVl4pphthtSVIIkKSrbbmZbEZFsRERAKzuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wEdrDpXaYRpJm+Z0etOp5WNBjllaQzeyDmjrsRXHEck9PxLkkty/nIBu1Gtb1NAdcWpa1xWlKUo9zUZoLczP+cwHYAAD5+wnCLbUm1zi0tNVM/rvd2YWdZGjVl4pphthtSVIIkKSrbbmZbEZFsRERAKzuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wE65i9rp1rLp3WQtSM0uYd8u2KZGuLhUlpRMwzUjZBEkv1lb+O/ilO22wDdwAAAfP2E4Rbak2ucWlpqpn9d7uzCzrI0asvFNMNsNqSpBEhSVbbczLYjItiIiIBWdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAnTxi1071n09qomo+a3MK9TbnLj3FuqS0fQikpGySJJfNZn4kfiSTLbYBu4AA+fsJwi21Jtc4tLTVTP673dmFnWRo1ZeKaYbYbUlSCJCkq225mWxGRbEREQCs7gvOrVX1H9MA7gvOrVX1H9MA7gvOrVX1H9MA7gvOrVX1H9MA7gvOrVX1H9MA7gvOrVX1H9MA7gvOrVX1H9MA7gvOrVX1H9MA7gvOrVX1H9MA7gvOrVX1H9MA7gvOrVX1H9MA7gvOrVX1H9MA7gvOrVX1H9MA7gvOrVX1H9MA7gvOrVX1H9MBD616bWunmluQ5pRax6lOT6qKTrCZOQKW0ajWlPxElBGfgo/5wH0aAAAD5+wnCLbUm1zi0tNVM/rvd2YWdZGjVl4pphthtSVIIkKSrbbmZbEZFsRERAKzuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wHC0cjW1HqRqNhsvLcgvYdKdP2Ry5nqlOoJ6O44vYz2It1H/MReCU777ANiAAHz9hOEW2pNrnFpaaqZ/Xe7sws6yNGrLxTTDbDakqQRIUlW23My2IyLYiIiAVncF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgI/UzArjTSux3KaPV3UOW+eZ4xXux7C768d6PKuYkd9taOBciU06tPz/nAezCcIttSbXOLS01Uz+u93ZhZ1kaNWXimmG2G1JUgiQpKttuZlsRkWxEREArO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAcTRNu3qM11Nw+wy29vYlBbV7EF24mnJebbdrWH1p5bEW3N1Z+BF4bf0ANgAAAAAZVoF94/wCIFx+UA1UAAAAAAAGVaBfeP+IFx+UA1UAAAAAAAABlWgX3j/iBcflAJf2jbvK7LPtLdGKDLLbFq/OrGeq4tKlZMzTixIxu9mYf+bCnFGndafiJKVcTIz3IPK1kPeybh17klnneW5zT2E+ugY9S3MxUyczYSXUx0MJnvKU4tpx1xs93eRtkSj3UXgAncx9se406h5JUZppK23mmMzcfbepa3IUyo8qJbTCjMvsSlx2zNSVc921tI8UkXIiVyIOo1Y9qnVemwbPoFPpzXY7m+G3VLAcbkXqZUdUOxdQTMhDhRVEazIzQptSPg3NRKXsRKDnZN7dmJ4fqFPwPIYWLRHcdsa2mv2nMuabsUzZaWjP3fBcZS5OYaN5HN3do9iVxQo0mRBYH7SF2rWJeg7Ol7y8vRLVKMjsjKB9n+G6bQ5PQPxNezPZ+PPqbluaS5gMNpteNa9QMc0xyTK5LVCufrXJoTRS3LqkyoDD8xpUV9KWGCU2g2kpLlz6hJStRJUfEg0THPbHzHJXsVOBoNIVGzSzuKKnfLIm/jsYK3fB0jZLpx1oZWrq+KkmRpJtfgag5uP8AteX+ZKwSpw/SFEy+zRq+adhychSwxWTap3pPNOPkws1tm4Si6iUci8P4M9z2DqsO15s9XtZdGrmqXa0MC0r8th3VD25a2CnwHG2FpXx4ofJt1DnBw0l4HuRFuZAPorUX+T7J/wDyNN/+AsBLaIPOnpHgCOsvb7OVRbcj+XZWwGlgADKtAvvH/EC4/KAdL7YGpSdNdFJ7jOWtYzYZJNiY5Bt3JyIZV65bqW1yussyJvpNG45yM/DhuAzfRr2nqXE/ZxsLSdlitSLHC8oPB4k+LbonO3sh2UhquWuWRqSo3W32DW4ZnsZLPx2Ad9qtrb7ReMQsMit6SUeP211m8CifN3KUSocyM4k3DJh5MRThJVwU2tTjDS0GW6CWRkog9uOe0KVJnVpSyMOtDrZ2pb2G2FnLyV2Y3AlKhIdYcaZdb2aYcXs0TSFElKlErx5HsHjF9oeRmmXYRKrcKsGm7i6yetoXSyZ6LFnN17CiKRIYQyaXG3VoWSCUS+mRE4k1GewDhaE+03qBcYppxZawYvVRWNQH7hpi7h2xupYdjmt1hhxnsrSUmppt9JGSj/4BJnyNZ8QzTKNetQsk1i0o1hhTbrGsSOlcsrHHmbdx2FPq5NsmC1NkN8UINZNPNPluk+BeHJRFuYdnpNrDnkXWDWTU25vLi7oJFAzdY9j8++aiV0SK3PkxGltKkrSxGJ1McnVuGe5kv+c9kmFlW+3HDs8RsLyswqoyCyq8wqcTeYxjLGbSvkKnkk23o04mkJcNPLZSFIb2UkyNW3xAKNv2prmJjWeScl06q6HIcBvI1PPiWGXxI1Z05DaHWpa7F9DaUN9Ne60k2pwjSZJSvwMw62j9stOXYrjMrCcGrr3KMpyuZicSDDyRDtSp6K2p1+UmybYVzjE0nklZM8lGok8SMB6L722qzGccTNyPEavHrqTmc/DIkbIMmagVyXYjfUckyZ5tKSw0ZF4bNuKM1IIiM1bEH5jPtpK1CvsGxrTXTePkkvL492t96PkjJRILtZJYZf4vk0pMhlRPGtDqdjURILh8Z8Qg8musswPUi+vvaJstdMfgpyVT9LluLWpvYtGqjdQUZmTDZUtDRbfA4qRGUZmoz5l4GAvX/bowBjVZ/T9S8bRAjZM1iK3XMqjpuXJznFKXmqrga1xScWSDd6hK3IzJBpLcB513tf5TNw++1He0TUxiuN5a9jE6f9oUKXwZszhPTENdDc20J4uqIzT/AMdJGfElKDmZT7Qke11MqcXj4panWV2okLE49tDyJ6G3LmKguPvm4y23xfZa3Sg2lrNKlmZnxNBbhQYR7QeU57LeySl0ldXp0zYWFceR+/GUy2+xqcQ7JcguIRtHNbSkpU2844e5GbRJ8QEzoT7bWE645lUYpWt48z9pq6VZ0yK7KGLGe20waeTdjDQhKoLqkK5pQanC2SojUlRbANQ9pL/udtUv7F3f9xeAUGLvvKqqpKnlmRxmfA1H/wDeEApgABlWgX3j/iBcflAMj9t6+yWvzHRKgp5OoL0C+yWwjWVVhF45V2Nm0ivccS2l1MiOk+K0kvZTqS2SfzPwMJnT3XrJtGMS1Vsb+Dml0WOZBUNVGG5hbG7kFfDmm0ySnZiush5Djilrb2eeLZPFTiD3JIazd+0bn0O/nYVRaNRp+TY5QIyXJoL+TJjs18N1x5LDbL5R1lIkLSw4rgZNoLbY3PkZhM6ke3VimE4ZjupVdUUMjGbvHmclJdxl8OrsH46/mxCgmTj0uSktzNBk2g9tkuGrwAdH7QntG5hkmCaq1el+N2MKswj3dCnZRFujiTWJz/Z3zbZYSgjU2lp5BOOdZJlzMiQvxAalD15y/IM7vcS050oXlNXhlhDqMisTvWoUluU8y28oo0d5HCQTTbqFLNb7R+JkglmWwDqLT2tK+k1zrtHbWkx1tNrde4o3QzKHJuidNlTiZDlW0SltRjNJo5qdJwj23aIj3AcTAfa3lai5bd4nR4hjjUyvTYparpGYNNXLDsXkSe3Vy2EusNuGn4VsnJ2JRGoiIBlWDauaze0D3M6d5pkU3D2M5qLzJbWyxaz6UyfGhutoYjJfJhpUUzN4+ZNfFxaSZOfEewUxaxZR7NOV6t4PYW9zqBQ4bRUmTU5Xlqbk9hM6S5GciOTFIWtxKVNdRKnCWvZWxntsYC+1F9rai0xyHUWqyLF1FDwHHaW8KWmxQk5rti88y2waVoJLJJU0nd1SzLZRmZFx8Q6HHPbdoch6NfDp8atbVOV0+OSzxvLWreuQ1Yko2pDUxtlPUUjgoltG2gyUW3LYyUYczMPbPosMRm6LnHa+C9jGcxsHgu2N+3DhTH3ojMntMiS43xiMoS6oleDh/B4bmriQaNoLrfRa7YlNyKlVWdeps36ieVVaos4RyGuJmqPLQlKX2lJWlSV8Un4mRpIyMgHW6qLUjWvSJSFGk+re+JHt/wDcJANQrXHHOp1FqVtx23Pf+kBzwABlWgX3j/iBcflAMj9t6+yWvzHRKgp5OoL0C+yWwjWVVhF45V2Nm0ivccS2l1MiOk+K0kvZTqS2SfzPwMJnT3XrJtGMS1Vsb+Dml0WOZBUNVGG5hbG7kFfDmm0ySnZiush5Djilrb2eeLZPFTiD3JIazd+0bn0O/nYVRaNRp+TY5QIyXJoL+TJjs18N1x5LDbL5R1lIkLSw4rgZNoLbY3PkZhM6ke3VimE4ZjupVdUUMjGbvHmclJdxl8OrsH46/mxCgmTj0uSktzNBk2g9tkuGrwAdH7QntG5hkmCaq1el+N2MKswj3dCnZRFujiTWJz/Z3zbZYSgjU2lp5BOOdZJlzMiQvxAalD15y/IM7vcS050oXlNXhlhDqMisTvWoUluU8y28oo0d5HCQTTbqFLNb7R+JkglmWwDqLT2tK+k1zrtHbWkx1tNrde4o3QzKHJuidNlTiZDlW0SltRjNJo5qdJwj23aIj3AcTAfa3lai5bd4nR4hjjUyvTYparpGYNNXLDsXkSe3Vy2EusNuGn4VsnJ2JRGoiIBimP8AtL62SrX2f89yvF8kvnMmrsjflY/hk5Eldm2npkw87Hd7Izya3UXEzVxIuRKM1GRBZaV+0dlkRWvWoGQUdxDarMzroUCizjIolO3RsOVsXkTzzrzrEVrkanTSybij57khSjMgFRWe2qV9jeFWuMaew7+wy3NpeDmzV5KzIhNyWIzz3aY8wmiRJYV0k/FxbMkqM9jNPFQc6L7W95PrKelh6Tb6g2uZTcLOgcvSKFHkxWjfffVPSwozZJkiURkzyM1EniR+IDMsE1I1EkaoYcxkGT5HEZkar5fCnV8i0dcbTFZgGtuMoiWaVstq+JCf1S8DJKT8CCzwr2+8HzS3VBgw8cW1YQbeXTR4eWMS7RRwG1uGifCQ3yg9VDalNnyd/oUST2Iw2XQbU/I9YtPa3UO8wE8Ui3caPNrI67NMx16O40lfNfFtHTPkaiJJ7maSSo+JmaEh1OqS1I1v0kUhRpPa/wDEj2/+5GwGn1rjjnU6i1K247bnv/SA54DKtAvvH/EC4/KAdL7YGpSdNdFJ7jOWtYzYZJNiY5Bt3JyIZV65bqW1yussyJvpNG45yM/DhuAy7Sz2oGsL9mO4yVq5TqpO09yf7ILnMX7cly3aXMbbiyTmfwiXHFMPtKMzP4lEZGovmQUea+2Pd6ell9PlOkPHJ8VsMeaTWQr9Lzc2DcSyjR30vqYQSXUqJzkyadt0ERObHzIJfXz2gc5mYjkeGSamZgmXYpmWGofXT3q5CJVbPtGCSpMhDbKi5oJxtxo07bblutJ7gKuP7R7WOX+T41j+H2dxk9pqO/iFPAsMldXGlSUQkyXHuqtpfYYyGkrM220OERp8CM1AKXUD2hcu02waqyDMtO8axq2nTZESSzkuf19ZWME0Rml1E00rW8TuxE2lMcl7n/CJa+YCBqfamzvUXUzTJnDcXr2sJzPALXKpqJFx05hGy6y0ripuOsuTRr2RwcSThOqUZo4JJQc7S/2l7zIMB05o9LtMZl/e3uGIyyTFv8vdPsMDmTaUu2LrDrsqQtXIk8my5cTNS0/MBy4XtjWuYOUzGlmkD+QSLnDZGWpRMvWoBRjYkmw9FdV03CJRKSoiWnkRqIi2IjNRBFa3WVfq5pdpZ7QGHZvqTjJZ7d41BchVeZWVcwiFLe2dbNiO8hrqbKNJuEnkexHv8gHe5FeZ/o1r7DwvTmLk+cwoGnEyyRVXuYSDbceTPIzefkyTeUpzjuhCjQtXiSd0p3Mgp8W9rWVn95gtXgmmD8+JmmIMZk7Mk2yIxVsRT6W3kuJNtXNSCUZp4n8aiItkkZqIOq0w9urB9SsvhY5Cj490r1uyXTMwssiy7XeGSlGmfC4o7D1EIUps1OrL+ZZoMBx6H27MZkV+cTMgoaTq4ZjZ5MtjF8ti5Aam+obXZH1sIShiWTnEjbI3EfFuS1AO400zvVDJfaaKFnVJKxliRp0xYpomLtVhXpdVOMidSrg0RvEgyQs+kW22xKWWxgLj2qj29nvNjL/+Xp/+KgBoUJ55cpCVurMj38DUf9ADtgABlWgX3j/iBcflAIv2kcvyTGdaNAIdPe2UODaZLbN2cOLMcZZntN1MhxLb6UnxcSS0koiURkRkR7bkA4WnfteXeZu4BMvtH3aCk1Hhz11E7363KcKZEQtxbK2UtJMm1IbUaXeW+5bGhPgYD8089sGzytzAbTK9JXcaxzUODYPVtn78bluIkQ21uONrYS0kybU22pSXOW5mWxoT4GA6bT/2/sJz/rLr63HXu247ZZHTRK7LWJs9TUNHUUxYxkNEqvdWg+SUmbpbEojMlFxMITWPWfKdQF6E5FlVHqNhWNZjLnSXqrCLyxftLCIdch1pS0VaUPHxcM/gIlbJIlHx3NKQ7LTD2imtK9OdQdVJl9lV9pqnJYVFhn21t+M9M1SiYltyZMk1PR47b++5yd3EE24fH5EYXdF7bdLlNXFqsUoaHIs2n5QnE4dfSZS3Np5EpUQ5fWTaIY8Y6WErNSuhzJSFJ4Ge24dtqR7Vc3Smtxav1BwqgxTLMokzWWomR5pEgVDLUVPJT52RNr+BfJBNpNlLijVspCNjASzHtbV2SPHqJgWE21+61pxNydEBrKFIiPdmndF+OhlKFsLdI0LNEkvFSSJJbErcgqcy9sbFMTi5DfNY8ufj2O4vV5BItET0tIXJsXOMSGXJPBO6f4RbilkSEmRmk9wE217ddTJxybNp8WosovK3JKageh4rlzNrBdTZKMmXWZqWUpUpJpUlba0I4qLxVtsoB9M49LvJ1FCmZHTsVNo8wlcuCzL7U3HdMviQl7ijqER/8binf+ggGW4a643rzq0SHFJ3Kh+R7f8A3GoBrsJSlxkKWo1Ge/iZ7/zmA5ADKtAvvH/EC4/KAZH7b2SzaXMtEql3INQYFFd5LYR7mLhEuyasZzKK9xaEJRXmT7uziUq2SR7ERmfgRgJbTX2hW9KcH1M1VkXuYXml8C7r6TFkZrZqKyTZrWUeW08/LM5EeOh5SORyviQSXD22IBc0XtvUmTxWaLFsfx/Is3mZK1i8GBRZW3PppL7kRUsn02bbHgylltzmfQ5pUg08D8DMO51I9qubpTW4tX6g4VQYplmUSZrLUTI80iQKhlqKnkp87Im1/Avkgm0mylxRq2UhGxgJ+u9qaFmlvV5lp5h9lezJenNtkcSAnKFswXnYktLTkU2m23GXFmtKiRKIlHx2Ik7KAdvkftpYlR07uVw8eOxx2BgsTNJ85qxQg2FTHUtwoZJUjgZuH1DNxS0Egm99jI/AJ9ft7UScLyK8r8ax7JbrHLSigvwcVzFm1hPs2kpMdpbc1DKS6qDNfJpTafiSRc9lEsB1R02a6ze1tf4jnGY57hDdbp1U2rNPjOay2I8Gc5Oltqd/gTQy+o0IRvzbMj2IjIyIB7tCde8hv8hwrGc4XcZBdwpOZ053ESzXHYsG6l9LaXXYLaSZkPOJ4kSj24LSvj+sYDucO9t5rLcQy3L4+F0CkYzj0y/OsiZiy9ZxjYLfsljCWw2/CeV4luhD7aTSZGvfYjCmx32obqTkDGPZfpYdHIucLczehS3fMPnMjNE31mH1OIZaiupU83sanFIMlbmpOxkQZdlftyZNeab5+rTumw08uxNumeU/UZizdVzLFhJNlO8lqKaO0tqQpK2DQaU7kolrLbkGwO67akzMyscHw7RiPkdjikKufyro5M3GTHfloNZR4JvMJTLWlBcjN04ydjLY9z4gMlh+0RqFplqJrze2WLW2VYnjeaVMeUuTfk0VHCfroRKTEYUSycMnHDcU0k20nyMyUajMgFc/7dGAx9V39P1/Z1uBGyZrEnHHspYRdLnOcUk63VcDcXFJxZIN3qEojIzJBpLcBe6Ca35FrWvIZ7mm/uGko7q0omrBdwmSqZJhTFx1Glom0qShSUEvcz8FGafi25mHv9pgzTgNEZGZGWe4ZsZf2hgAHs8qUqPqGpRmZnn1vuZ//wBoBlHtvX2S1+Y6JUFPJ1BegX2S2EayqsIvHKuxs2kV7jiW0upkR0nxWkl7KdSWyT+Z+BhM6e69ZNoxiWqtjfwc0uixzIKhqow3MLY3cgr4c02mSU7MV1kPIccUtbezzxbJ4qcQe5JDWbv2jc+h387CqLRqNPybHKBGS5NBfyZMdmvhuuPJYbZfKOspEhaWHFcDJtBbbG58jMJnUj26sUwnDMd1KrqihkYzd48zkpLuMvh1dg/HX82IUEycelyUluZoMm0Htslw1eADo/aE9o3MMkwTVWr0vxuxhVmEe7oU7KIt0cSaxOf7O+bbLCUEam0tPIJxzrJMuZkSF+IDUoevOX5Bnd7iWnOlC8pq8MsIdRkVid61CktynmW3lFGjvI4SCabdQpZrfaPxMkEsy2AdRae1pX0muddo7a0mOtptbr3FG6GZQ5N0TpsqcTIcq2iUtqMZpNHNTpOEe27REe4DiYD7W8rUXLbvE6PEMcamV6bFLVdIzBpq5Ydi8iT26uWwl1htw0/Ctk5OxKI1ERAPnqdrNqBqIzoHd6vWOfV9flcDIZc6HpraWy5c9LZt9nccarGmXS4GaiNJJWkiLkaj5HsF/pf7RsjSjSXJdR7q8ur7BrjMY9Jp67mt4iPLJpaEtvHPmyOTjDDb6Hz5Pkp1KUGRpM9iAX1B7aFXmNHVRMFxunyTMbfJn8Uj19ZkqJFOqUzG7S4+Vo2yrlGJk0q5kwa+R8eG4Dkake129pZ9l8dzrCsfxTL8jasJRwcnzWJXVjMeItKVOJsCQ4lw3TcR0mzbQtW580t8VbB0Uv2sqyyeY1LwfCrq+bc0zczBEL7SLZjqYbmpbfZ7MhDjKn0EThk8W5qJHAvBW4Cg1C9szFsDTkdqePpm0FBVUcn3uqzRHYdn2rvGNEUakGlpKWzQ648pWyEKI+J+IDpoftw1lji7lhj+JU2UXcXL63EpETGcsZsYDjk1HNl+NOJpKXU/8VSVIbNKkqI9iIjMPpSilXE2lhS8gqmqyyeYQuXDZldpRHdMviQl3inqER+BK4p3+exAMr0/ddb1m1lJDikl76qPke3/APB4gDX4ilKjIUpRmZl8zP8A1gPeAAADKtAvvH/EC4/KAaqAAAAAAADKtAvvH/EC4/KAaqAAAAAAAAAyrQL7x/xAuPygHdaq6Q4nq9XV0PInbOBPpJqbKmuKmWcWfWSiSaeqw6RGRGaVKSaVEpKkqMlJMgEez7KeASsayCiy7Isvy2fkjkR6Ze3dqTtg25FWTkVbBtoQywbThEtJNtJLl4mSgHGneyLgl3W2rWU5dl19c3lnT2VhfzpUXt7/ALskpkRI+zcdDCGUrSZGlDSTMlrPfkfIB2mb+zLgmfO51Jt7fII7+fFVnNciSGUKhuV58ozsbk0okqJWxn1OokzL5beAAn2bMfZy2RlsHP8ANa921kQ5t9Fgzo8Zm8lxm0NokSVNsJdQtSWkEsmHGkLJOykmXgA/T9mbBvtinUgrrISy9N+d77+KSyUtSDb6Xu8z6XA4XS+Do8f5uXLn8YDrqn2TcBp0VkSNk+Uqr6XNHc6rYLkmMpqLOdU6t1pJ9DqGwpb7izSpalEZlssi8AHa4v7NWDYnHwyNXW164nB7myva435DJm7Imk8TqXtmi5ILtC+JJ4mWydzV47hl832TJ2O6n6eNYBdZTAx2l+1tjNvmLGImbAnWjyX0pQhbfB1BrW6lKVMuJIiLnuZEoBp2E+zNp3p/a4ZbY7KuEv4THtWYvWkNrKa5YrJyVIkn091uqWRr3SaC3UfhtsRBd6i/yfZP/wCRpv8A8BYCT0R/klwD+zlV/dmwGnAADKtAvvH/ABAuPygHfZrpRjGoOVYjleSrlyTw2VImwIG7ZxHZDzJs9R5CkGpSkIUvjspJEaj3IwEtkvsvaZZVMzGRO96R2M0TWOS4kN5phmJMgKNUabG4t8m5CT4GajUpJ9NHw/PcPTa+zVWX9FGrsh1W1FtbSFfRMiiXsuzjLlxZUdJpbJpns5RGm+JqJSUxy5cjM91fEA87r2YMDvcWznE5d1kTaM8vUZJLmsyGESoFgjom27EV0uKDSphCi5pX47/MvAg7Su9n3BalenPu560Za0xiyIlOyT6DS+h6P0HDkGaOS1GndW6TR8RmZ7/IBkmqfsy3EfRur9nvTSPkdxVT75E1u8n2kNn7KRikE670zQlp50jQt5CEklxR8zJaySA1PKvZu06y/qNWRWLMVzDXsHKJGdbQy3XuKQrkkjQZk6k0J4q34lt+qYCfuvY70qvKV6hOdkMOMvHajG2DiymUKjM1r5vxX292jI3icPdXMlNq22NHz3Dyc9kjCpNvKurDOM2nSrC4qL+ecibGWUuxrlbsvqLs/wAHJOyFob4N8SLghB+IDmZV7K+CZVd3mTKyTKKy3ushrsnRNgvxyXAnQmOg0phLrC0Gk0b8kupc3MzMtvDYInP/AGYY2LVMe408h53kdnGzAssZOFkcCPZ1ktxhTUl+GucybLxvEoycZkOEgyWfBSCIkgPZpf7MEydgTqs/fucWyVWa2WYU0qBYsSrOnXJ3bInHXEPR3nFNmrqJNLjW69i3JJGA0ig9n3GKHNMVz9zKMqtrrE620rGH7SxTJOUme606+48akb8iUygkJbNttCTNJI22Ig6fJfZepcumTo+R6tamT8YspfbJeKybxt2tePqE50zWtk5ZM8iL+CKQSNvDbbwAdnB9njGqjN5WX0OXZdUQp9oV3Mx6BPbZrZM8kpSby9mu0ESuKTU0l4mlGW6kH4gOTQez7glBpllOkpLs59Fl824nWHbHW1OkuyecdfShSUJIkpU6rhukzIiLc1H4mHVY77LmBY3jOE4xGusjkowbIVZRHmypTLkuwsFdXm5LX0iJZK6ytyQlHyTsZEWwD3Y57N2MYlkDs+hzTNYtA7Yy7YsTatEN05SpJrN9RoQ2l9xC1OLWbLjymSUe5II9gHK0v0AoNJ5kX7PZnl8qoqorkGnoZk9v3dWR1qI+DbbTSFO8diJCpCnVITuSDSRmA5vtJf8Ac7apf2KvP7i8A73Fv/sZU/8A6Mz/APqEAqgABlWgX3j/AIgXH5QD2ay6DVGs1hiN1KzfKsWtsJnvWVTYY+7EQ8h51hTK+RSY7yFFwWotuJfMBMn7ImBzKPIq/I8xzPILfKptbMtcgspsZVi8UB1LsZkuDCGG2kqSfwoaLfkrx3PcgotRfZ6xrUHJpGXx8ryjFbeyqio7aTj8phlVpXEpSkx3+sy7sSTW5xW3wcTzVsstwEte+xbpFcKsIcSbklJS3OLRsPsqWrnoaiS6+MlSYxKNTankqb5q/UcSle/8IlZbkA9dp7GWB28TIK6VnudIiZZGgt3zDM6I0ixlRENoZmuEmMWz/FpCVEji0oiLds9i2Cmneznjjma2Wa0OcZrjXv6TFmXtbSWaIsW2kR0JQ268omjfQo0IQhXRdbJaUkSiUQDoz9kHBWrJdhBzfNoSG8rVmdfFYsI/QrrNazU8tolMGbiXOSkml83eJKPpm2fiA7uv9nOhazStzLIM7zPJlUK5blLBupsd9muOQhSHODyWEynfgWpJE+86REfh8i2DoHPZCw2HgFHhtFm2YQLDD5MqXiV83MYRPpDeIyNhtaGUodj/ABbG28hw1F8zMyIyCU0U9nW3vtJs4wf2hsav05XlziIuR5Q9ex5T16bRbMyYSmTM4jKNkm2yttBoMz3SrxUYWcb2RNPF/ap/Jsny/JZeY0tbSWky2sGlPGmC665HkNqaaQTbyVO77pIkfwaD4EfI1BzLn2YaHJMSfxnJdTdQbScu1h3MW9es2EToEqKe7CoyG2ExWiT47kTHx7ma+Z+IDiR/ZI09j1uQRk5Jlrtjf5LGy87h2ayudCtmY7TCX2FGzw8UtbqQ4haDNay2JOyUhpuB4ajBaQ6f7S32QPuvuSZFhdSyfkvOLPc/BCUNNpLwIm2kIQkvkkvEBC6r/wAtekX/AI29/uJANQqf++/+b/8AMB2AAAyrQL7x/wAQLj8oB7NZdBqjWawxG6lZvlWLW2Ez3rKpsMfdiIeQ86wplfIpMd5Ci4LUW3EvmAmT9kTA5lHkVfkeY5nkFvlU2tmWuQWU2MqxeKA6l2MyXBhDDbSVJP4UNFvyV47nuQUWovs9Y1qDk0jL4+V5RitvZVRUdtJx+Uwyq0riUpSY7/WZd2JJrc4rb4OJ5q2WW4CWvfYt0iuFWEOJNySkpbnFo2H2VLVz0NRJdfGSpMYlGptTyVN81fqOJSvf+ESstyAeu09jLA7eJkFdKz3OkRMsjQW75hmdEaRYyoiG0MzXCTGLZ/i0hKiRxaURFu2exbBTTvZzxxzNbLNaHOM1xr39JizL2tpLNEWLbSI6EobdeUTRvoUaEIQroutktKSJRKIB0Z+yDgrVkuwg5vm0JDeVqzOvisWEfoV1mtZqeW0SmDNxLnJSTS+bvElH0zbPxAd3X+znQtZpW5lkGd5nkyqFctylg3U2O+zXHIQpDnB5LCZTvwLUkifedIiPw+RbBwsC9lbCtPbXC7WuyrKp/wBgG7Nijjz5EVTTEebx5sK6bCFLQjj8BmrkW58lL8Ng8Mg9lDA8gtchyBOTZTW22QZVCzIpsOTG5wLGLFRGbNhLjC2zQbbZbpdS54qMyMvDYPCq9k3Aqe/qr9jKMufVTZc5m0ePKnMvN+83Irkd41KUybqkOE6pZp5+CyLiaUlwAey09lLArJx2wi5HlFXcFmD+bwraFJjlKr7B5roupZ5sKbNlTe6TQ4he5KPx+Wwe3HPZZ0+xm6pr+JdZLKk0uR2mUNdsmNPE9MnsdF8nTNrkpvYzNJEZGRn8zLYgHnjfsyYrizMyorM3zT7PPQ58KFj6rFpNfXIlkoneklDSXHduauBSFvJb3+EiAaFgOGVenOEUWBUsmU/X49XsVsVyUpKnlNNIJCTWaUpSati8TJJF/qIBA6qfy36Sf7L/APujYDUKn/vv/m//ADAdgAyrQL7x/wAQLj8oB32a6UYxqDlWI5Xkq5ck8NlSJsCBu2cR2Q8ybPUeQpBqUpCFL47KSRGo9yMBj3tFezIrI8RzaTplCsHrbN5mPduqGJceJESmDLQa5LRmSFNu9HlyPqePTRxIlfMOo149k6ZcYJfnh0rIswy7K8kxh6yn2tlGZlN1cCxad6TTiEsIQhlrrKTtu4ozM+S1mAvbD2ScFvIF41kmW5dcWeRXNRczriXLjHMUdZIQ9EjJ4MJaSwlSNjIm+SiUozVyPkA5dt7LGA2bltYN3mRwLewy081iW0SQwmVVWZsEwaovJlSOBtkaTQ6hwjJaiPfw2D8m+zBjtjNxm7l6k6hO3uNKnkm4euG3ZcxmbxKSw71GlNttqJCSSUdDJtkX8GaAHCxr2RsFxKbic6izDMWF4dCs6ivJcuK4S6ya4lxyC7yjma2kqQg0L3J34S5OK8dw/K/2R8IoMbxGjw7Ns0xybhtAeLxbqtmRSnS6wzJSmJHUjrZVupJGS0tJWkzM0KTuA7/E/Zv05wi1rbPGCs4SanFHMPjRe0pcaKGt7qqdUa0m4p417mazWZHue5GfiA9MP2acFhaT4No41bXp02n8ysnVj6n2TlPOQV82ieV0uCiM/wBbihJmXyNIClstJ8etdRnNTpEyyTauY67jJsocbKP2Vx0nVL4mg1dTkWxHy47f8X+cBN6X+zTguk0vGZmO2d7LViuJpwyGme+ytLsEnid5ukhpPJ3kW25cU7f8X+cBwqj2V8IqK+wxtGX5m/isyFPgR8bXaIbr4LUvl1ibJptDrm3NXAn3HSb3+AkgOpa9jTTp2GuBf5dmd8xJxZ3DJrdjOjmiXVK26TSkNMIS2bRluhbRNqM/Fw3AFXp5oBQ6f5g3nZ5pluR3bdA3jZybuYw7yhtu9RvdLTLaSWk/DkRFyLxVyV8QDw9qn/ue83/8nl/8VAC+gfxxv/f/ANRgO5AAGVaBfeP+IFx+UA7bULRvGNSsqwrL7yfaMTMFnSp9ciK62lp1yRFXGWTxKQo1JJDijLiaT5EW5mXgAn6X2ZMEoafTmjiW1+tjTHthVCnH2TW/2llbS+0GTRErZLijLgSPEi33+QD9o/Zm0/oanTijYsbyRF0xOYVUmQ8yo5RSWVtOFJ2aIllxcVtw4eO2+/yAcXGPZexLFKaZikDOM1dx1ynlUVdSO2LJQqmHITxUlhDbKTcNJeCFSTeNBeCTIjMjDuWvZ/w5qTpjLTZ3Jr0oYXHpiN5raQlUUoxnJ/g/jPgW/wAHD4v9XgA6y09l/Tq2LM2XLC+jxM0t4mQvRY8ptDVbbR+Jpmw/4Pdp1Sm0KWSjWhRp8U+KiMPyw9mnHrnH4lfd6g5xY3tbdJyCvyeRYMHZQJyWjaJcdBMFEaR0zUk2kxybUS1GpJmozAeyy9nCjta2jJzUbOiyPHZUqXCyldkzItEqklxfQZPsuRukoiIukTBNp4lwSky3Adpj2hOJY/lcHMvfGQ2k+Fja8XUdpP7WUmMt/rLceUtPUU6avDfkSSSexJIiLYJrEPZF0nw/S/ItJ4715Y0+TSUyZMidMSqWx00tpjIZcbQgkIYS02TRbGZcS3NW57hz3/ZsorSnjVeU6iZvkL0bIK7IkzLGfHNzrQlcmWktNMIjttf/AHxNtJWv5qWZ+IDYQGMYh/Lzqz/sof7moBr0D+Kt/wC//rMByQGVaBfeP+IFx+UA7vONJMcz/MsFzi4nWTM/T+ykWla3FcbSy869GXHUT5KQpSkkhwzIkqSe+3iZeACYu/Zg05vXc4KXKum4meToVxMhMyW0MQrSKaDbnxf4Pk2+am2lKM1KSZtpPj89w8LD2Z8ft6SLDvNQs5sr6uuG76uyaRYRzsa+Yho2kqjtpYKI0jpqWlTaY/BfNRqSozMwHtsvZwo7Wtoyc1Gzosjx2VKlwspXZMyLRKpJcX0GT7LkbpKIiLpEwTaeJcEpMtwHbUOhmL49mNHnCLvIrGzosefxttVlP7V2lh59LzjrylpNxbprT8yUSSI9iSRbbBNYr7Iuk+LYRmWn7a7uyqs2fNyYc6ak3ojSfFhiMttCDaaZPxbLxUR+JqUA99n7MWP5DiMjEsv1GzzIUybartzmz58brNrr5Db8dpttuOiO2g1NkSzS0S1kZ8lmrZRB6889mCqzbU+dqzW6s6h4hc2lJHx+cjHJsJlqRDZdccSkzeiuuoVydVuptxJ/LbYy3AdnQ+zTpnithhs3GW7Srbwetsqyujx5h8XUziT2h15xRG8t41J5E4ThHyUZnv4bBB6leyqqXhuX21bluXZzlzmH2mOY83fzoZnGbktkRspdQyyp01KbQXUkuOKLx+ItzMByKP2OcKusPKDqTb5TcTrDCmsQUzNsml+5Ya0NqfYiONtko93W0ma3VuqPgkuRpLYByrD2M8EumLr31n+eTpWRUcOitJr06J1ZCIb3VhvklMYm2nWVb8SbQhs+SubazPcBQWvs2U1jkTmVQdTtQKWysoMOBfv1FmxEVeojEZNLkKQxyac2UZG5FNhRke2+2xAFx7MOBXdVqRTybfIUs6pWMSzuFIktGth2Oyw0go6lNHxSaYyDPnzMzNXiW5bByYPs745UZrJy6izLL6qHPtE3c6ghWDbVdLnkhKTeXs12jZXFJqaS8lpZlupB7mAo9K9K8f0ioJ+OY5NsZMayu7K+dVOcQtaZE2QuQ6lJoQgiQS3DJJGRmRbbmZ+ICb9pr/7QKP8At7hv/wC8MAB+ezx/FdQv7fW/5QDlay6DVGs1hiN1KzfKsWtsJnvWVTYY+7EQ8h51hTK+RSY7yFFwWotuJfMBMn7ImBzKPIq/I8xzPILfKptbMtcgspsZVi8UB1LsZkuDCGG2kqSfwoaLfkrx3PcgotRfZ6xrUHJpGXx8ryjFbeyqio7aTj8phlVpXEpSkx3+sy7sSTW5xW3wcTzVsstwEte+xbpFcKsIcSbklJS3OLRsPsqWrnoaiS6+MlSYxKNTankqb5q/UcSle/8ACJWW5APXaexlgdvEyCulZ7nSImWRoLd8wzOiNIsZURDaGZrhJjFs/wAWkJUSOLSiIt2z2LYKad7OeOOZrZZrQ5xmuNe/pMWZe1tJZoixbaRHQlDbryiaN9CjQhCFdF1slpSRKJRAOjP2QcFasl2EHN82hIbytWZ18Viwj9Cus1rNTy2iUwZuJc5KSaXzd4ko+mbZ+IDu6/2c6FrNK3MsgzvM8mVQrluUsG6mx32a45CFIc4PJYTKd+BakkT7zpER+HyLYOFgXsrYVp7a4Xa12VZVP+wDdmxRx58iKppiPN482FdNhCloRx+AzVyLc+Sl+GweyX7LWnsmDklbGt8ihR7/ACRrLoyIsplBUtsjYzkQd2j4c1FzWhzqIUalfDsoyMPfZ+zdQXWP1VfZZ9m79/R2672typywYVaRJq2zbUppKmTiobNtSkGyUcmtjP4Nz3Aeqy9mjH7BmhnJ1FzxjJ8fKa3Hyk7NmTaONS1JVIZWcllxgmlGhBk2hpKW+CemSNgHdVehOIVWUwctVZXthLh4svETRZTu1pkQ1uk4tby3Em648pReKjXtsZ/CAlsY9kDSjF9KbXSRqTfWFda2TNsuwmzELsGZMc2uyKbdS2lJdnJhlLe6T2JsuXLx3DuFezrRTqythZJnua5BKrckh5QmfZT2FOuSopbNNdNtlDDTO3zQy03ue6jPkZmYa2AxfAv5ZdZP/LVR/wDseIA2CD/FW/8AYf8A1gOQAAADKtAvvH/EC4/KAaqAAAAAAADKtAvvH/EC4/KAaqAAAAAAAAAyrQL7x/xAuPygGqgAAAAAAAAAAAAJ3UX+T7J//I03/wCAsBJ6I/yS4B/Zyq/uzYDTgABlWgX3j/iBcflANVAAAAAAAAAAAAAAAAAAAAAAAAAZx7SX/c7apf2KvP7i8A73Fv8A7GVP/wCjM/8A6hAKoAAZVoF94/4gXH5QDVQAAAAAAAAAAAAAAAAABj+q/wDLXpF/429/uJANQqf++/8Am/8AzAdgAAMq0C+8f8QLj8oBqoAAAAAAAAAAAAAAAAAAAMf1U/lv0k/2X/8AdGwGoVP/AH3/AM3/AOYDsAGVaBfeP+IFx+UA1UAAAAAAAAAAAAAAAAAAAGUe1T/3Peb/APk8v/ioAX0D+ON/7/8AqMB3IAAyrQL7x/xAuPygGqgAAAAAAAAAAAAAAAxjEP5edWf9lD/c1ANegfxVv/f/ANZgOSAyrQL7x/xAuPygGqgAAAAAAAAAAAAAAAAAAAAMl9pr/wC0Cj/t7hv/AO8MAB+ezx/FdQv7fW/5QDWwAAAAAAAAAAAAAAAAAAAAGL4F/LLrJ/5aqP8A9jxAGwQf4q3/ALD/AOsByAAAAZVoF94/4gXH5QDVQAAAAAAAZVoF94/4gXH5QDVQAAAAAAAAGVaBfeP+IFx+UA1UAAAAAAAAAAAABO6i/wAn2T/+Rpv/AMBYCT0R/klwD+zlV/dmwGnAADKtAvvH/EC4/KAaqAAAAAAAAAAAAAAAAAAAAAAAAAzj2kv+521S/sVef3F4B3uLf/Yyp/8A0Zn/APUIBVAADKtAvvH/ABAuPygGqgAAAAAAAAAAAAAAAAADH9V/5a9Iv/G3v9xIBqFT/wB9/wDN/wDmA7AAAZVoF94/4gXH5QDVQAAAAAAAAAAAAAAAAAAAY/qp/LfpJ/sv/wC6NgNQqf8Avv8A5v8A8wHYAMq0C+8f8QLj8oBqoAAAAAAAAAAAAAAAAAAAMo9qn/ue83/8nl/8VAC+gfxxv/f/ANRgO5AAGVaBfeP+IFx+UA1UAAAAAAAAAAAAAAAGMYh/Lzqz/sof7moBr0D+Kt/7/wDrMByQGVaBfeP+IFx+UA1UAAAAAAAAAAAAAAAAAAAABkvtNf8A2gUf9vcN/wD3hgAPz2eP4rqF/b63/KAa2AAAAAAAAAAAAAAAAAAAAAxfAv5ZdZP/AC1Uf/seIA2CD/FW/wDYf/WA5AAAAMq0C+8f8QLj8oBqoAAAAAAAMq0C+8f8QLj8oBqoAAAAAAAADMrb2ctGb21mXVnhvXm2EhyVId94y083XFGpatkukRbmZnsREX9ADoH/AGZ9EkvOJThaiIlGRF7zmf0/+NAeH6NOif7GK/ecz/NAP0adE/2MV+85n+aAfo06J/sYr95zP80A/Rp0T/YxX7zmf5oDocb9lzA41zlT2TUMeXXS7dt7HmWbGWS4deUCIhbTpkpO6zlomubma/hdQXItiQgO+/Rp0T/YxX7zmf5oB+jTon+xiv3nM/zQD9GnRP8AYxX7zmf5oB+jTon+xiv3nM/zQD9GnRP9jFfvOZ/mgNHoKmvoYFbRVEYo8GuZZiRWSUZk202kkoTuZmZ7JIi8T3AVAAAzK29nLRm9tZl1Z4b15thIclSHfeMtPN1xRqWrZLpEW5mZ7ERF/QA6B/2Z9EkvOJThaiIlGRF7zmf0/wDjQHh+jTon+xiv3nM/zQD9GnRP9jFfvOZ/mgH6NOif7GK/ecz/ADQD9GnRP9jFfvOZ/mgH6NOif7GK/ecz/NAP0adE/wBjFfvOZ/mgH6NOif7GK/ecz/NAP0adE/2MV+85n+aA6HJPZcwOTc4q9jNDHiV0S3ceyFl6xlmuZXnAloQ00ZqVsspa4Tm5Gj4WllyPc0LDvv0adE/2MV+85n+aAfo06J/sYr95zP8ANAP0adE/2MV+85n+aAfo06J/sYr95zP80A/Rp0T/AGMV+85n+aAfo06J/sYr95zP80A/Rp0T/YxX7zmf5oB+jTon+xiv3nM/zQD9GnRP9jFfvOZ/mgPB/wBmHQyUw5Gk4MTzLyDbcbcsZakrSZbGkyN3YyMvDYBqdaw1FVGjMI4tskltCdzPZJFsReP+oB34AAzK29nLRm9tZl1Z4b15thIclSHfeMtPN1xRqWrZLpEW5mZ7ERF/QA6B/wBmfRJLziU4WoiJRkRe85n9P/jQHh+jTon+xiv3nM/zQD9GnRP9jFfvOZ/mgH6NOif7GK/ecz/NAP0adE/2MV+85n+aAfo06J/sYr95zP8ANAP0adE/2MV+85n+aA6HAPZcwOuwPG6/UKhj2eVRqiGzeToljLSxKsEsoKQ62RKQRIU6S1ERIR4GXwp+RB336NOif7GK/ecz/NAP0adE/wBjFfvOZ/mgH6NOif7GK/ecz/NAP0adE/2MV+85n+aAfo06J/sYr95zP80A/Rp0T/YxX7zmf5oDs8a0N0txC8i5JjuMHFsYfM2HjnSXOHNCkK+FbhpPdKlF4l/OA0yp/wC+/wDm/wDzAdgAAMytvZy0ZvbWZdWeG9ebYSHJUh33jLTzdcUalq2S6RFuZmexERf0AOgf9mfRJLziU4WoiJRkRe85n9P/AI0B4fo06J/sYr95zP8ANAP0adE/2MV+85n+aAfo06J/sYr95zP80A/Rp0T/AGMV+85n+aAfo06J/sYr95zP80A/Rp0T/YxX7zmf5oDoZvsuYEvO6ewhUMdvFmaiyZs4KrGX137Bb0I4TqD5GZIbabnpUXNO5vN/CvYjQHffo06J/sYr95zP80A/Rp0T/YxX7zmf5oB+jTon+xiv3nM/zQD9GnRP9jFfvOZ/mgH6NOif7GK/ecz/ADQD9GnRP9jFfvOZ/mgH6NOif7GK/ecz/NAdpjOhul2H3kbI8dxg4ljE59F450lzhzQaFfCtw0nulRl4l/OA0up/77/5v/zAdgAzK29nLRm9tZl1Z4b15thIclSHfeMtPN1xRqWrZLpEW5mZ7ERF/QA6B/2Z9EkvOJThaiIlGRF7zmf0/wDjQHh+jTon+xiv3nM/zQD9GnRP9jFfvOZ/mgH6NOif7GK/ecz/ADQD9GnRP9jFfvOZ/mgH6NOif7GK/ecz/NAP0adE/wBjFfvOZ/mgOhyT2XMDk3OKvYzQx4ldEt3HshZesZZrmV5wJaENNGalbLKWuE5uRo+FpZcj3NCw779GnRP9jFfvOZ/mgH6NOif7GK/ecz/NAP0adE/2MV+85n+aAfo06J/sYr95zP8ANAP0adE/2MV+85n+aAfo06J/sYr95zP80A/Rp0T/AGMV+85n+aA/F+zLog4hTbmEmpCiNKkqs5hkZH8yMuqA1eB/HG/9/wD1GA7kAAZlbezloze2sy6s8N682wkOSpDvvGWnm64o1LVsl0iLczM9iIi/oAdA/wCzPokl5xKcLUREoyIvecz+n/xoDw/Rp0T/AGMV+85n+aAfo06J/sYr95zP80A/Rp0T/YxX7zmf5oB+jTon+xiv3nM/zQHQ5/7LmB2OB5JX6e0MesyqTUTGaOdLsZamItgplZR3XCNSyNCXTQoyNC/Aj+FXyMO+/Rp0T/YxX7zmf5oB+jTon+xiv3nM/wA0A/Rp0T/YxX7zmf5oB+jTon+xiv3nM/zQD9GnRP8AYxX7zmf5oB+jTon+xiv3nM/zQFPhOmeEaddtPDqU6/3h0+07yXnufT5cP+EWrbbmr5bfPx/mAX8D+Kt/7/8ArMByQGZW3s5aM3trMurPDevNsJDkqQ77xlp5uuKNS1bJdIi3MzPYiIv6AHQP+zPokl5xKcLUREoyIvecz+n/AMaA8P0adE/2MV+85n+aAfo06J/sYr95zP8ANAP0adE/2MV+85n+aAfo06J/sYr95zP80A/Rp0T/AGMV+85n+aAfo06J/sYr95zP80A/Rp0T/YxX7zmf5oDocJ9lzA4NNIZzihj2Niq3tXmHo1jLShNe5PfXAaMiUj424io7aj2PdSFGal/rqDvv0adE/wBjFfvOZ/mgH6NOif7GK/ecz/NAP0adE/2MV+85n+aAfo06J/sYr95zP80A/Rp0T/YxX7zmf5oB+jTon+xiv3nM/wA0A/Rp0T/YxX7zmf5oD3QvZy0Yr7CFaRsLR2mvlsToynJ0pwm32XEuNL4qcNJmlaEqLcj8UkA5b3s76PX11LtrXETflz3nZUhfvGWnm6tRqUrZLpEW5mZ7EREA4L/sz6JJecSnC1ERKMiL3nM/p/8AGgPD9GnRP9jFfvOZ/mgH6NOif7GK/ecz/NAP0adE/wBjFfvOZ/mgH6NOif7GK/ecz/NAP0adE/2MV+85n+aAfo06J/sYr95zP80B0Of+y5gdjgeSV+ntDHrMqk1ExmjnS7GWpiLYKZWUd1wjUsjQl00KMjQvwI/hV8jDvv0adE/2MV+85n+aAfo06J/sYr95zP8ANAP0adE/2MV+85n+aAfo06J/sYr95zP80A/Rp0T/AGMV+85n+aAfo06J/sYr95zP80A/Rp0T/YxX7zmf5oB+jTon+xiv3nM/zQFThGm2FacszY+GUia5Fg6l6URPuum6tKeJKM3FKPckkRf7CIBfQf4q3/sP/rAcgAAAGVaBfeP+IFx+UA1UAAAAAAAGVaBfeP8AiBcflANVAAAAAAAAAAAAAAAAAQ+AQsOjZXqS/i9tLmWUzKGHsjZeSZIh2JUtYhDLRmhO6DhohOGZGv43VlyLY0IC4AAGaYZe5q9q/mmM5Jdx5VfDraufXxI8VLaIhPuzEqTz/wCEcUaWWzUaj25EfFKSPYBeqmSCtEV5VMo2FR1PHO5NdBKyURE0Zc+pzMjNRGSDTsR7qI9iMI+ZqzDbzebhVZiGRW7lU7FZtJsBEVbMBUhJKbNxtb6ZC08VJM1NMrSRGe5/CriHGga10djdx4KMbyBqonWT1NDyBxqP7vkzmjWlTKSJ45CfjacSla2UtqUnYlHyTuHWUvtGYpaRq63n41klJSW1bKs4NtYsxkx30RmzcfQSG31vJUlCVHupskKJPwKVuW4QmpOu+dIr8hscfqcgxFuJp1bZDBbtIkJxxchtbHZ5KCbW+Xgla/4JZkfiXNv5APDTbVm4rL81T9QtQspoWccn3NiWbYk1QzEvMJaWlFeg4EA5KeBvGv4HEp/g91o3IjDR2dc2Z8WpVT6Z5lY2FzDctI1YymA3J93p4EUtZuykNoQo3EklClk6Z7/wZbGZB+2+vVHAx6LmNXhuWXePuVabmZaQYbLceBEMz5Kd7Q60pa0khZqaZS64kk+KS5J5B0mrmuEunoMoi4LTXkiXQtxUSb2OzFXCgPv9JaG1pdc6q1G24gzNDK0pJwjUafHYK+DqtEs8neoq3Dcnl10ec7VuX8eKy7XomNkfUaMkunIIkqI0m4bPSJRcee4DrEa8UkV6anJ8NyvGWI9PMvoz9tEZT22FF49ZbbTTy3m1JJxs+m+hpeyy+HclEQdPlGvl7WYhGySp0jyon5VrUQ2I8tVcpMiPNkob6rTrUw2TPYzSSTdJSVKQakkg+QDYmluONpWttbSlERmhW26T/oPYzLf/AGGZAPaAAIfP4WHScr02fyi2lw7KHlD72OMspM0TLE6WzQtl0yQrZBQ1zXCMzR8bSC5HuSFhcAAAAAAAAAAAAAAAAAAAAAAAAAAAAh9EYeG12jGA12nNtKtMUi4xVM0M6Wk0vyq5MVsozzhGhBktTRIUZGhHiZ/Cn5EFwAAAAAAAAAAAAAAAAAAAAAAACHtoWGua0YtYTbeW3ljGMX7NZBSk+g/XLl1JzXlnwMiW263XpSXNO5POfCvYzQFwAAAAAAAAAAAAAAAAAAAAAAACHz+Fh0nK9Nn8otpcOyh5Q+9jjLKTNEyxOls0LZdMkK2QUNc1wjM0fG0guR7khYXAAAAAAAAAAAAAAAAAAAAAAAh9boeG2OjGfV2o1tKq8UlYxas306Ik1Pxa5UVwpLzZEhZmtLRrUREhfiRfCr5GFwAAAAAAAAAAAAAAAAAAAAAAACG0ih4dBxWfHwW2l2NarKMlefekpNK02Ll1NXYMkRoR8DcxUltB7HuhCTJS/wBdQXIAAAAAAAAAAAAAAAAAAAACH1uh4bY6MZ9XajW0qrxSVjFqzfToiTU/FrlRXCkvNkSFma0tGtRESF+JF8KvkYXAAAAAAAAAAAAAAAAADKtAvvH/ABAuPygGqgAAAAAAAyrQL7x/xAuPygGqgAAAAAAAAAAAAAAAAJLDppy8kztj7C+4exZAyx7w6PD7Qb1UBzt2/BPPj1Ox8t3P4jx5Fx4ICtAAGUYxpzq5Vas2uoV5qViE+rt4zEJ6siYfKiyEx46n1MEmSqycTzI5B81GyZK4lslADSVNWvvVDyZkQq4mFJVHOMo3ze5FssnepxJBJ3I0dMzMzI+RbbGGaagaLWWe5hCv5d/QNxYMqPJivO40ly6r+mpKlNwrFD6DYQs0/FyacPZbhb7GRJDqsU9mehxLMEXldHw5NexaSrZlScOje+es+tbhoXYqWrdtK3FGk0soc2Sgup4HyDnP+z3XzsMxPB7bIVyYWOVNhUSFIiEhUxuVEXHUot1n0jSSzUX6++2wDrbn2f8AL8sp7OBlupkCVJl4hNxCK9Dx447bLL5tGmQ4g5KzccLpfERKQlW5cUt7HyDluaLag5Uw03qfqTQWiq2tnQakqLF3qxtlyVFVGU8+T06Sp40oUrilKmy+I99/AyDvLHSzIYk+iyDBswgVV3U0RY8+/Y06p0eVFI0KJXSQ+ypDiVo3SfUNOy1EaVeBkEDkXsh1V5XfZ97IKqZVrx9FIS7rHm7GdCcSTu8mE8t1LcVbi3eayJlW5pLY07EZB3FjoBl0+vvqZvUmsjQsuREfvEtY6o3HJrDLTXVjqVKMmWlpjtcm1pdV+txcTvuQUVbpZmNDc2UXHdRI9bidpYy7Z2uZpSOwTIkmpTqUzVPGhLRurU5sUfqEZ7E4RAI3EPZhs8NuazJKbMMZrreBVzKaROrcObYfnsPJbMn5C1yHFPSicZQtTjhqQojWXSSauRB74/s2WzFfeORMpxaot7OZWWEcqPE1wapEqFK7Ql9+EUxRvuuK+FxaXmzUhKS8DLcBujRPk0knloW4SS5qSjikz/nMiMz2L/Vuf+0Bjmq3td6FaH6mU2luquVrx2xv64rGFNlRlHBNJuqaJC3k79NW6FHusiQRF4qIzIjDWaS9pMnqo17jlxBta2YgnY0yDIQ+w8g/kpDiDNKi/wBZGA6LMZpxMkwRj7C+/u25A8x7w6PP7P7VU9zt2/BXDl0+x8t2/wCPceR8uCwrQAAAAAAAAAAAAAAAAAAAAAAAAAAASWlE07LS3DbL7DFhXa6Cuf8Asz0el7k5Rmz7Dw4N8Ojv0uPTRtw24p+RBWgAAAAAAAAAAAAAAAAAAAAAAAkrGaaNUserTwXtfXoLl/7TdHf3b05NaXYefA+Paup1ePUTy9378V8d2wrQAAAAAAAAAAAAAAAAAAAAAAASWYzTiZJgjH2F9/dtyB5j3h0ef2f2qp7nbt+CuHLp9j5bt/x7jyPlwWFaAAAAAAAAAAAAAAAAAAAAAAJLVeadbpbmVl9hizXslBYv/Zno9X33xjOH2Hhwc59bbpcemvfntxV8jCtAAAAAAAAAAAAAAAAAAAAAAABJaazTsMclv/YU8SNGQXrHu/o9LrdO1lN9u24I37Xx7Zy2Pl2nlyc35qCtAAAAAAAAAAAAAAAAAAAAASWq8063S3MrL7DFmvZKCxf+zPR6vvvjGcPsPDg5z623S49Ne/Pbir5GFaAAAAAAAAAAAAAAAAAyrQL7x/xAuPygGqgAAAAAAAyrQL7x/wAQLj8oBqoAAAAAAAAAAAAAAAADIpWjuXZFmWXX+SatZfXV8+0YXQQKK5VHaiQE18RtaHEG1sTipaJjngaiNLiPEj3SkOR3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YD4C9vn2GNddZtecPqdK4+TZTXJx0kzb7JbRK40BXand2zeUSfkRpV00EpZ8tyIy32Cq0f/ANH9l3srJxG5sPaIy73lkuT19TY1eLz3q6sJt5DiVmo9+UlSeJGhSkoIti3Qoi8Q+0+4Lzq1V9R/TAcHRyPa0WpOo2HS8uyC9h03ufsjlxPVKdR1o7ji9jPYi3M/5iLwSnffbcBsYAAAAAAAAAAAAAAAAAAAAAAAAAAnNPYOZV2A41X6i2sS0yyLTwmb6dESSWJVillBSXmyJCCJCnSWoiJCPAy+FPyIKMAAAAAAAAAAAAAAAAAAAAAAAE5Nh5ivP6ewg2sVvFGKizZtIKkF137Fb0I4TyD4GZIbabsEqLmnc3m/hXsRoCjAAAAAAAAAAAAAAAAAAAAAAABOZNCzOXdYo/jFtFh1sO4ceyJl5JGuZXHXy0IZaM0K2WUxcJwzI0fA0suR7mhYUYAAAAAAAAAAAAAAAAAAAAAAnNQoOZWOA5LX6dWsSryyVTzWaGdLSSmItiplZRnnCNCyNCXTQoyNC/Aj+FXyMKMAAAAAAAAAAAAAAAAAAAAAAAE5hELMoFNJZzu1i2Nkq4tnmHoqSShNc5YSFwGTIkI+NuGqM2s9j3WhRmpf66gowAAAAAAAAAAAAAAAAAAAAE5qFBzKxwHJa/Tq1iVeWSqeazQzpaSUxFsVMrKM84RoWRoS6aFGRoX4Efwq+RhRgAAAAAAAAAAAAAAAAMq0C+8f8QLj8oBqoAAAAAAAMq0C+8f8QLj8oBqoAAAAAAAAAAAAAAAACHwCFh0bK9SX8XtpcyymZQw9kbLyTJEOxKlrEIZaM0J3QcNEJwzI1/G6suRbGhAXAAAAAAAAAAAAAAAAAAAAAAAxX2nrxWP1Wn9izTz7Z9Of1CWIEBCDkSXNndm0G4pDaTPbbk4tCC/4yiLxAaFgmdw87rZclqmsqedWzF19lV2RNdqhSUpSs23DZccaVuhaFEptxaTJRbGYDEbXFItlrBrJlk7N81pY1DX1El5jHrMo3WQ3BccUZpMtlL2SZFuZEAs6fRdi6qodzG1m1XQzPjNym0uZERKJK0koiMibMt9j8djMBzu4Lzq1V9R/TAO4Lzq1V9R/TAdfpDqTJgaHYZkWaz7G3m2U1qkOWaW1POuuTVRmVun8JHsRI5K/WPYz2Mz8Q2UAAdbZ3lXUSK+NYy+i7ayihQ0mlRm6901ucS2I9vgbWe57F4fP5AODiOX12a1sqzqo8lpqJZTatwn0pSo3YshbDhlxUZcTU2ZpPffbbciPwAc472rK7axtUtJWL0Rc5DHFW5sIWlCl77bFspaS2338f9RgOyAdbbXlVRNx3LWWUdEuWzCZM0qVzfeWSG0eBH81GRb/ACL+fYgHZAAAAAAAAAAAAh9EYeG12jGA12nNtKtMUi4xVM0M6Wk0vyq5MVsozzhGhBktTRIUZGhHiZ/Cn5EFwAAAAAAAAAAAAAAAAAAAAAAACHtoWGua0YtYTbeW3ljGMX7NZBSk+g/XLl1JzXlnwMiW263XpSXNO5POfCvYzQFwAAAAAAAAAAAAAAAAAAAAAAACHz+Fh0nK9Nn8otpcOyh5Q+9jjLKTNEyxOls0LZdMkK2QUNc1wjM0fG0guR7khYXAAAAAAAAAAAAAAAAAAAAAAAh9boeG2OjGfV2o1tKq8UlYxas306Ik1Pxa5UVwpLzZEhZmtLRrUREhfiRfCr5GFwAAAAAAAAAAAAAAAAAAAAAAACG0ih4dBxWfHwW2l2NarKMlefekpNK02Ll1NXYMkRoR8DcxUltB7HuhCTJS/wBdQXIAAAAAAAAAAAAAAAAAAAACH1uh4bY6MZ9XajW0qrxSVjFqzfToiTU/FrlRXCkvNkSFma0tGtRESF+JF8KvkYXAAAAAAAAAAAAAAAAADKtAvvH/ABAuPygGqgAAAAAAAyrQL7x/xAuPygGqgAAAAAAAAAAAAAAAAJLDppy8kztj7C+4exZAyx7w6PD7Qb1UBzt2/BPPj1Ox8t3P4jx5Fx4ICtAAAAAAAAAAAAAAAAAAAAAAGAe2Jk9liWH4Tb1NfKlTkZ5Upj9nrJVj0lmT2zio8VCnnUp23NDaTUr9UvE9wF5oo9jkjFX5FAeTvvyJzki0nZDjs6mmTpqySa3jYmMMr47cUp4o4JSlKSP4QGZUMGtl2vtA1+HUDsVmRURGo0BqrciLU8qBIJRJjqQle63DUZfD8Zq5FvyIzCDs9P0TsOnwdO9McjpZCtOJVbk6VUMmE/Y2S+y9BJrU2RzpCeEk+q2pzbkfxfERGFhn+nOK0d7kNdaaTSravdxyOxhKKvHnZqK+xNUlT5traQpMF9bzjLhyFm1ueyjc3QZkErqXj+czbCTMm4Gb+WY2zRuNWaMRsbefLbYTHclPwbNDqY8Q+XaEHHaQp1w0qM0LNwiASmguNVdjp5jD7emt1IzI87j2Tt4nHJJJVUpsyNJnONsm1NIbIkmxz5IUk1G2XE1EGp4tWwMfgXLr+jeTWmq0Vm7esbWNWS69FlyU+bKVWhcESW3EG0lpttx1bRmgyQ2aN0hIUOIqjPZDVnh9zWYxk2MxUmrFdN7CqSuXHl8nTlRZS3FyFk2pJLNxKFyG+aEE6fgQdlWYZWvt4hYWujsJ2joc+bkNyKzAJ9c2tp2vdb7QmnfS6/FSmQplK3EpJtSkk8fEiNRBR43pgrIsux+Dm+DPzqVFrnD0qPZV6lxFpfsUKj9ZC08FocTutBLIyVxJSd+JGQRLumWTM1kR2mwKbEy13Aryko7JVG6p2FIRPcNlk5BJScXeKZpaNbjRGlRJQrx2Ac6Np3PmYxkbGNUMmNXTDoY8itpsAn4ww68izZW4+TT8hbrryGuXN9tsk8eJm4o0bJDt8+0lxyryK5ZZ0jaexKvynGLhEGDjSpUf5OpmPsxmmldRXijqm2hStvFQD6mb48E8E8S2LYttti/2fzAPMAAAAAAAAAASWlE07LS3DbL7DFhXa6Cuf+zPR6XuTlGbPsPDg3w6O/S49NG3Dbin5EFaAAAAAAAAAAAAAAAAAAAAAAACSsZpo1Sx6tPBe19eguX/ALTdHf3b05NaXYefA+Paup1ePUTy9378V8d2wrQAAAAAAAAAAAAAAAAAAAAAAASWYzTiZJgjH2F9/dtyB5j3h0ef2f2qp7nbt+CuHLp9j5bt/wAe48j5cFhWgAAAAAAAAAAAAAAAAAAAAACS1XmnW6W5lZfYYs17JQWL/wBmej1fffGM4fYeHBzn1tulx6a9+e3FXyMK0AAAAAAAAAAAAAAAAAAAAAAAElprNOwxyW/9hTxI0ZBese7+j0ut07WU327bgjftfHtnLY+XaeXJzfmoK0AAAAAAAAAAAAAAAAAAAABJarzTrdLcysvsMWa9koLF/wCzPR6vvvjGcPsPDg5z623S49Ne/Pbir5GFaAAAAAAAAAAAAAAAAAyrQL7x/wAQLj8oBqoAAAAAAAMq0C+8f8QLj8oBqoAAAAAAAAAAAAAAAACcxqFmcS6yt/J7aLMrZlw29jrLKSJcOuKviIWy6ZITus5iJrhGZr+B1Bci2JCAowAAAAAAAAAAAAAAAAAAAAABlHtBM2vurDrSqx+1uPc+X19nJjVkRUh/oNJdUsyQn/cW5mRbmRGZbgOnuPaig01rXUrmiWq78+1S+qLHbomULdJokm5xJx9PIyJRHsnc9iM/kQCXwfVO9r9SM9zCz0H1UiQsk919ibXQoU6XZ46m3OZIeUReJlt4nuR/7gGMe07/AKUt/R5Eqnwr2f8ALnLJqQ5A96ZRG7HWNyUIJSkJ6SlqfUnkRLb5tKT47/LYBlfsDe3PrrrNrzmFtqpIybKa5OOmqFQ41VpXGgK7U1s4TKTT8iNSeos1LPlsZmW2wffvf75K6q+nPqAOX7OtJd45oxjNTkdTIrLBLLzzsORxJ1knZDjiUrJJmSVcVp3TvuR7kfiRgNKAAAAAAAAAAAAAAAAAAAAATmnsHMq7Acar9RbWJaZZFp4TN9OiJJLEqxSygpLzZEhBEhTpLUREhHgZfCn5EFGAAAAAAAAAAAAAAAAAAAAAAACcmw8xXn9PYQbWK3ijFRZs2kFSC679it6EcJ5B8DMkNtN2CVFzTubzfwr2I0BRgAAAAAAAAAAAAAAAAAAAAAAAnMmhZnLusUfxi2iw62HcOPZEy8kjXMrjr5aEMtGaFbLKYuE4ZkaPgaWXI9zQsKMAAAAAAAAAAAAAAAAAAAAAATmoUHMrHAclr9OrWJV5ZKp5rNDOlpJTEWxUysozzhGhZGhLpoUZGhfgR/Cr5GFGAAAAAAAAAAAAAAAAAAAAAAACcwiFmUCmks53axbGyVcWzzD0VJJQmucsJC4DJkSEfG3DVGbWex7rQozUv9dQUYAAAAAAAAAAAAAAAAAAAACc1Cg5lY4Dktfp1axKvLJVPNZoZ0tJKYi2KmVlGecI0LI0JdNCjI0L8CP4VfIwowAAAAAAAAAAAAAAAAGVaBfeP+IFx+UA1UAAAAAAAGVaBfeP+IFx+UA1UAAAAAAAAAAAAAAAABD4BCw6NlepL+L20uZZTMoYeyNl5JkiHYlS1iEMtGaE7oOGiE4Zka/jdWXItjQgLgAAAAAAAAAAAAAAAAAAAAAAAGVah/y7aR/7L/8AuaAGqgMa0eoqTJqrU+iyKng2tbNzy4akw50dD7DyD6W6VtrI0qL/AFGQD16U+yJoVofqZc6paVYovHbG/rjrpsKLJUcE0m6l01oZVv01boSWyDJBEXgkjMzMNpAAAAAAAAAAAAAAAAAAAAAAABD6Iw8NrtGMBrtObaVaYpFxiqZoZ0tJpflVyYrZRnnCNCDJamiQoyNCPEz+FPyILgAAAAAAAAAAAAAAAAAAAAAAAEPbQsNc1oxawm28tvLGMYv2ayClJ9B+uXLqTmvLPgZEtt1uvSkuadyec+FexmgLgAAAAAAAAAAAAAAAAAAAAAAAEPn8LDpOV6bP5RbS4dlDyh97HGWUmaJlidLZoWy6ZIVsgoa5rhGZo+NpBcj3JCwuAAAAAAAAAAAAAAAAAAAAAABD63Q8NsdGM+rtRraVV4pKxi1Zvp0RJqfi1yorhSXmyJCzNaWjWoiJC/Ei+FXyMLgAAAAAAAAAAAAAAAAAAAAAAAENpFDw6Dis+PgttLsa1WUZK8+9JSaVpsXLqauwZIjQj4G5ipLaD2PdCEmSl/rqC5AAAAAAAAAAAAAAAAAAAAAQ+t0PDbHRjPq7Ua2lVeKSsYtWb6dESan4tcqK4Ul5siQszWlo1qIiQvxIvhV8jC4AAAAAAAAAAAAAAAAAZVoF94/4gXH5QDVQAAAAAAAZVoF94/4gXH5QDVQAAAAAAAAAAAAAAAAElh005eSZ2x9hfcPYsgZY94dHh9oN6qA527fgnnx6nY+W7n8R48i48EBWgAAAAAAAAAAAAAAAAAAAAAAAyrUP+XbSP/Zf/wBzQA1UBlWgX3j/AIgXH5QDVQAAAAAAAAAAAAAAAAAAAAAAAAElpRNOy0tw2y+wxYV2ugrn/sz0el7k5Rmz7Dw4N8Ojv0uPTRtw24p+RBWgAAAAAAAAAAAAAAAAAAAAAAAkrGaaNUserTwXtfXoLl/7TdHf3b05NaXYefA+Paup1ePUTy9378V8d2wrQAAAAAAAAAAAAAAAAAAAAAAASWYzTiZJgjH2F9/dtyB5j3h0ef2f2qp7nbt+CuHLp9j5bt/x7jyPlwWFaAAAAAAAAAAAAAAAAAAAAAAJLVeadbpbmVl9hizXslBYv/Zno9X33xjOH2Hhwc59bbpcemvfntxV8jCtAAAAAAAAAAAAAAAAAAAAAAABJaazTsMclv8A2FPEjRkF6x7v6PS63TtZTfbtuCN+18e2ctj5dp5cnN+agrQAAAAAAAAAAAAAAAAAAAAElqvNOt0tzKy+wxZr2SgsX/sz0er774xnD7Dw4Oc+tt0uPTXvz24q+RhWgAAAAAAAAAAAAAAAAMq0C+8f8QLj8oBqoAAAAAAAMq0C+8f8QLj8oBqoAAAAAAAAAAAAAAAACcxqFmcS6yt/J7aLMrZlw29jrLKSJcOuKviIWy6ZITus5iJrhGZr+B1Bci2JCAowAAAAAAAAAAAAAAAAAAAAAAAZVqH/AC7aR/7L/wDuaAGqgMq0C+8f8QLj8oBqoAAAAAAAAAAAAAAAAAAAAAAAACc09g5lXYDjVfqLaxLTLItPCZvp0RJJYlWKWUFJebIkIIkKdJaiIkI8DL4U/IgowAAAAAAAAAAAAAAAAAAAAAAATk2HmK8/p7CDaxW8UYqLNm0gqQXXfsVvQjhPIPgZkhtpuwSouadzeb+FexGgKMAAAAAAAAAAAAAAAAAAAAAAAE5k0LM5d1ij+MW0WHWw7hx7ImXkka5lcdfLQhlozQrZZTFwnDMjR8DSy5HuaFhRgAAAAAAAAAAAAAAAAAAAAACc1Cg5lY4Dktfp1axKvLJVPNZoZ0tJKYi2KmVlGecI0LI0JdNCjI0L8CP4VfIwowAAAAAAAAAAAAAAAAAAAAAAATmEQsygU0lnO7WLY2Sri2eYeipJKE1zlhIXAZMiQj424aozaz2PdaFGal/rqCjAAAAAAAAAAAAAAAAAAAAATmoUHMrHAclr9OrWJV5ZKp5rNDOlpJTEWxUysozzhGhZGhLpoUZGhfgR/Cr5GFGAAAAAAAAAAAAAAAAAyrQL7x/xAuPygGqgAAAAAAAyrQL7x/xAuPygGqgAAAAAAAAAAAAAAAAIfAIWHRsr1Jfxe2lzLKZlDD2RsvJMkQ7EqWsQhlozQndBw0QnDMjX8bqy5FsaEBcAAAAAAAAAAAAAAAAAAAAAAAAy3WDDcSyjJMCO+zzJcZtl2kyuolUbyWlzJDkF991lbhsuG2RMQ3nORKb3Nviaj58FB4dwXnVqr6j+mAp9PNO6vTWqnVdXa2tj7xsHbOTJs30vPuPuJQlZmtKU778CPcyM9zMzMBXAAAAAAAAAAAAAAAAAAAAAAAAAIfRGHhtdoxgNdpzbSrTFIuMVTNDOlpNL8quTFbKM84RoQZLU0SFGRoR4mfwp+RBcAAAAAAAAAAAAAAAAAAAAAAAAh7aFhrmtGLWE23lt5YxjF+zWQUpPoP1y5dSc15Z8DIltut16UlzTuTznwr2M0BcAAAAAAAAAAAAAAAAAAAAAAAAh8/hYdJyvTZ/KLaXDsoeUPvY4yykzRMsTpbNC2XTJCtkFDXNcIzNHxtILke5IWFwAAAAAAAAAAAAAAAAAAAAAAIfW6Hhtjoxn1dqNbSqvFJWMWrN9OiJNT8WuVFcKS82RIWZrS0a1ERIX4kXwq+RhcAAAAAAAAAAAAAAAAAAAAAAAAhtIoeHQcVnx8FtpdjWqyjJXn3pKTStNi5dTV2DJEaEfA3MVJbQex7oQkyUv9dQXIAAAAAAAAAAAAAAAAAAAACH1uh4bY6MZ9XajW0qrxSVjFqzfToiTU/FrlRXCkvNkSFma0tGtRESF+JF8KvkYXAAAAAAAAAAAAAAAAADKtAvvH/EC4/KAaqAAAAAAADKtAvvH/EC4/KAaqAAAAAAAAAAAAAAAAAksOmnLyTO2PsL7h7FkDLHvDo8PtBvVQHO3b8E8+PU7Hy3c/iPHkXHggK0AAAAAAAAAAAAAAAAAAZ3m2sULBrCU3NwrKJ1VVIbduLuIxHKDWNrLclOdZ5t10iLY1dnbeNJfrbANCSolpJST3Iy3IwHR2mXVtTk9HikmPJVMv0S3Iy0JSbaCjpSpfMzURluSy22I/wCffYB3wDMtZ8d+1U/T+jOtvzJ/KHD98Uj/AEH6LjUWKu1qc6TnFC9uyH4t7nMSXM9+msOP3BedWqvqP6YDoq/TKusstuMMZ1i1dTNpIsOVIcVkKSaUiT1eBIMkbmZdFW+5F8y238dg73uC86tVfUf0wHF0cReUeoGo2BzsxvcgrqR6qeguXMlMiQyciKanUk5xIzSakkZJPwLx2+Z7hZZFqVjONxZ0l92RLVWWkComMxm91syJjjKGiPmaUmX/AGQ2ozIz2SZ/My2AVoDraW9q8hgIs6aUUmI4460lwkqSRrbcU2stlER+C0KLf+fbcty8QHZAI7MtTsXwiqvLOyckyl483EdnxojZKdQiS502jLmaUHuZKPbl4ER/6twsQAB0DeW1sjNZWCJjye3w6xi1ccNKeibLrrjaUkfLly5Mq3LjtsZeJ+JEHfgOgyHL63GbOgq57Elx7JLI6uIplKTSh0mHX93N1EZJ4srLciM9zLw23Mg78AAAAAAAABJaUTTstLcNsvsMWFdroK5/7M9Hpe5OUZs+w8ODfDo79Lj00bcNuKfkQVoAAAAAAAAAAAAAAAAAAAAAAAJKxmmjVLHq08F7X16C5f8AtN0d/dvTk1pdh58D49q6nV49RPL3fvxXx3bCtAAAAAAAAAAAAAAAAAAAAAAABJZjNOJkmCMfYX3923IHmPeHR5/Z/aqnudu34K4cun2Plu3/AB7jyPlwWFaAAAAAAAAAAAAAAAAAAAAAAJLVeadbpbmVl9hizXslBYv/AGZ6PV998Yzh9h4cHOfW26XHpr357cVfIwrQAAAAAAAAAAAAAAAAAAAAAAASWms07DHJb/2FPEjRkF6x7v6PS63TtZTfbtuCN+18e2ctj5dp5cnN+agrQAAAAAAAAAAAAAAAAAAAAElqvNOt0tzKy+wxZr2SgsX/ALM9Hq+++MZw+w8ODnPrbdLj01789uKvkYVoAAAAAAAAAAAAAAAADKtAvvH/ABAuPygGqgAAAAAAAyrQL7x/xAuPygGqgAAAAAAAAAAAAAAAAJzGoWZxLrK38ntosytmXDb2OsspIlw64q+IhbLpkhO6zmImuEZmv4HUFyLYkICjAAAAAAAAAAAAAAAAAAGD+0FO9/x5eIow7N3siioS/i7tfCmyaifLUkjR2zokqGTaHEkSkztkltyR47GA1y4RUuQq4snqU2CimRlNJTXLmJZlkouDpElCjbJKvHqnsSPmai+YDMNdqOms860/n5bg9lkuNQSt/ebMenes2WyWw2TfXjtIWbqTUXgjgr4iJW3wmZBCYBpXZWN7jcm0w6WxDq6LI3sdas4KyaqzctWnKxC21kRMutskhTbatltknYiSaT2DpKnGMmp04fZaZaYW8C+oqG0PUdDtbIq3chdOtcQ1G7abZdtkKnm06l9pxZpShwycLmSVh08bFMngs5BBxXB5MWmyHH683o9FgFlQtTDjT0rnsvNSHFuLkqiKWgnHemp4jNKOoe+wU0zEIDs3Kp+EaXXNdgcmTi7k6pTjMmEVhXsvSe2ttQFtocUkuSDWyTfJZEr4F9QuYc/K6jE7Wrx2ixHROTBw+VNnPsNZBg1pZQWX+DKUJTRtuNHFQ5ydNLkhDKG1NuGSSN3moMJwjGrKTkORnk+GWFnk9JVVVYmsu9PrC3YW+00tPBl1hXUrXT4IJuWbht8DSpKnCRyINxmYOqNa6hlA0ymQL24yzE7hT8akdWT8QnK83tpiW+D3TdakKcTzNSdlOKIiUSjDsouliG1RMrPAnVZF3oynFT3K5SpbdU7OdJzis08kRVtnuoiMm1Eo1HvyMzDPYGm71TTYzWL07hwcZqVX0W1rJ+mU+6jnarmoUxIKFGU0p3eMRpblpS42REpJLIzAW8HR5dyiVFzzFZuSlC03ix4TlzVGr/s1L8xbaSaU48kpLaVNEn+EW4nf9bdR7hL5FhEqBjGfWD2mty7k9xiGLyu2RsakyJcvodIpjZvNNKUt8lISamORuq4EfAyTuQajqBmdNqxR1hVOK5lPxypvor+U1lrhFxBXNrjbeLiiNLjNKmIS6bK1ttpcM0p8Un4EYRjmKRmCZsLbTe1kaQllcqU3jH2XkyDbinWobacOoS0bxM9sS8smuiZkpxLnAi+Ig8cgwzJJeStWmGYNfQ8Di43AO0x6bEkJmWkFE+YpUJlZuGbZklaXeyn8Sm+mwaWiUaQHpsMHyefqlZS5tebFnLySHNx+0Z09nSZ0WqSTBobbtu0tx4bSUpdQ5HcSg9jd/g1m4RKD3YdhyGcswVcnTe6azeszOxl5XeqopDTchpbFglp5c5SCblNGTjJIJK1k2RpSZI+QD6uAAAAAAAAATmnsHMq7Acar9RbWJaZZFp4TN9OiJJLEqxSygpLzZEhBEhTpLUREhHgZfCn5EFGAAAAAAAAAAAAAAAAAAAAAAACcmw8xXn9PYQbWK3ijFRZs2kFSC679it6EcJ5B8DMkNtN2CVFzTubzfwr2I0BRgAAAAAAAAAAAAAAAAAAAAAAAnMmhZnLusUfxi2iw62HcOPZEy8kjXMrjr5aEMtGaFbLKYuE4ZkaPgaWXI9zQsKMAAAAAAAAAAAAAAAAAAAAAATmoUHMrHAclr9OrWJV5ZKp5rNDOlpJTEWxUysozzhGhZGhLpoUZGhfgR/Cr5GFGAAAAAAAAAAAAAAAAAAAAAAACcwiFmUCmks53axbGyVcWzzD0VJJQmucsJC4DJkSEfG3DVGbWex7rQozUv9dQUYAAAAAAAAAAAAAAAAAAAACc1Cg5lY4Dktfp1axKvLJVPNZoZ0tJKYi2KmVlGecI0LI0JdNCjI0L8CP4VfIwowAAAAAAAAAAAAAAAAGVaBfeP+IFx+UA1UAAAAAAAGVaBfeP+IFx+UA1UAAAAAAAAAAAAAAAABD4BCw6NlepL+L20uZZTMoYeyNl5JkiHYlS1iEMtGaE7oOGiE4Zka/jdWXItjQgLgAAAAAAAAAAAAAAAAAAAAAASOomolXprVQbS0qrWx942DVZGjVjCXn3H3ErUgiQpSd9+BlsRme5kREAzbJdRqrKrrE7yw0e1fbfw64cu4KWcfSSHH118uCaXSNRmaOlOdURJNJ80oPfYjSoMg9pP/SSRdDIcuHVez3qFY2kZLBOv20NMCuiKeJXSJ15CnVE4fHkTRpQai22URGRgPlr2Wv9Ib7Q+uXta479upVnKxpMOyUWJ4lXF0ln2VZpUbZq6j3FRJVu44rjx3Ttue4f0m7/AHyV1V9OfUAcXRxd5eagajZ5Ow69x+uu3qpmC3cxkx5Dxx4ppdUTfIzJJKUREo/A/Hb5HsGvgAAAAAAAAAAAAAAAAAAAAACH0Rh4bXaMYDXac20q0xSLjFUzQzpaTS/KrkxWyjPOEaEGS1NEhRkaEeJn8KfkQXAAAAAAAAAAAAAAAAAAAAAAAAIe2hYa5rRi1hNt5beWMYxfs1kFKT6D9cuXUnNeWfAyJbbrdelJc07k858K9jNAXAAAAAAAAAAAAAAAAAAAAAAAAIfP4WHScr02fyi2lw7KHlD72OMspM0TLE6WzQtl0yQrZBQ1zXCMzR8bSC5HuSFhcAAAAAAAAAAAAAAAAAAAAAACH1uh4bY6MZ9XajW0qrxSVjFqzfToiTU/FrlRXCkvNkSFma0tGtRESF+JF8KvkYXAAAAAAAAAAAAAAAAAAAAAAAAIbSKHh0HFZ8fBbaXY1qsoyV596Sk0rTYuXU1dgyRGhHwNzFSW0Hse6EJMlL/XUFyAAAAAAAAAAAAAAAAAAAAAh9boeG2OjGfV2o1tKq8UlYxas306Ik1Pxa5UVwpLzZEhZmtLRrUREhfiRfCr5GFwAAAAAAAAAAAAAAAAAyrQL7x/xAuPygGqgAAAAAAAyrQL7x/xAuPygGqgAAAAAAAAAAAAAAAAJLDppy8kztj7C+4exZAyx7w6PD7Qb1UBzt2/BPPj1Ox8t3P4jx5Fx4ICtAAAAAAAAAAAAAAAAAAAAAAGVa+/dx+IFP8AmgNVAZFg0WNO1r1hhzI7UiO+iibdadQS0OIOEsjSpJ+BkZeBkYDqsd9jL2eMK1lrtdsEwRnGMmr2pLJoqFnHgyCeaNtXOMX8GkySpW3TJHie6uWxbBuQAAAAAAAAAAAAAAAAAAAAAAAACS0omnZaW4bZfYYsK7XQVz/2Z6PS9ycozZ9h4cG+HR36XHpo24bcU/IgrQAAAAAAAAAAAAAAAAAAAAAAASVjNNGqWPVp4L2vr0Fy/wDabo7+7enJrS7Dz4Hx7V1Orx6ieXu/fivju2FaAAAAAAAAAAAAAAAAAAAAAAACSzGacTJMEY+wvv7tuQPMe8Ojz+z+1VPc7dvwVw5dPsfLdv8Aj3HkfLgsK0AAAAAAAAAAAAAAAAAAAAAASWq8063S3MrL7DFmvZKCxf8Asz0er774xnD7Dw4Oc+tt0uPTXvz24q+RhWgAAAAAAAAAAAAAAAAAAAAAAAktNZp2GOS3/sKeJGjIL1j3f0el1unaym+3bcEb9r49s5bHy7Ty5Ob81BWgAAAAAAAAAAAAAAAAAAAAJLVeadbpbmVl9hizXslBYv8A2Z6PV998Yzh9h4cHOfW26XHpr357cVfIwrQAAAAAAAAAAAAAAAAGVaBfeP8AiBcflANVAAAAAAABlWgX3j/iBcflANVAAAAAAAAAAAAAAAAATmNQsziXWVv5PbRZlbMuG3sdZZSRLh1xV8RC2XTJCd1nMRNcIzNfwOoLkWxIQFGAAAAAAAAAAAAAAAAAAAAAAMq19+7j8QKf80BqoDKtPP5dtXP9lB/c1gNVAAAAAAAAAAAAAAAAAAAAAAAAAAE5p7BzKuwHGq/UW1iWmWRaeEzfToiSSxKsUsoKS82RIQRIU6S1ERIR4GXwp+RBRgAAAAAAAAAAAAAAAAAAAAAAAnJsPMV5/T2EG1it4oxUWbNpBUguu/YrehHCeQfAzJDbTdglRc07m838K9iNAUYAAAAAAAAAAAAAAAAAAAAAAAJzJoWZy7rFH8YtosOth3Dj2RMvJI1zK46+WhDLRmhWyymLhOGZGj4GllyPc0LCjAAAAAAAAAAAAAAAAAAAAAAE5qFBzKxwHJa/Tq1iVeWSqeazQzpaSUxFsVMrKM84RoWRoS6aFGRoX4Efwq+RhRgAAAAAAAAAAAAAAAAAAAAAAAnMIhZlAppLOd2sWxslXFs8w9FSSUJrnLCQuAyZEhHxtw1Rm1nse60KM1L/AF1BRgAAAAAAAAAAAAAAAAAAAAJzUKDmVjgOS1+nVrEq8slU81mhnS0kpiLYqZWUZ5wjQsjQl00KMjQvwI/hV8jCjAAAAAAAAAAAAAAAAAZVoF94/wCIFx+UA1UAAAAAAAGVaBfeP+IFx+UA1UAAAAAAAAAAAAAAAABD4BCw6NlepL+L20uZZTMoYeyNl5JkiHYlS1iEMtGaE7oOGiE4Zka/jdWXItjQgLgAAAAAAAAAAAAAAAAAAAAAAZVr793H4gU/5oDVQGVaefy7auf7KD+5rAaqAAAAAAAAAAAAAAAAAAAAAAAAAAIfRGHhtdoxgNdpzbSrTFIuMVTNDOlpNL8quTFbKM84RoQZLU0SFGRoR4mfwp+RBcAAAAAAAAAAAAAAAAAAAAAAAAh7aFhrmtGLWE23lt5YxjF+zWQUpPoP1y5dSc15Z8DIltut16UlzTuTznwr2M0BcAAAAAAAAAAAAAAAAAAAAAAAAh8/hYdJyvTZ/KLaXDsoeUPvY4yykzRMsTpbNC2XTJCtkFDXNcIzNHxtILke5IWFwAAAAAAAAAAAAAAAAAAAAAAIfW6Hhtjoxn1dqNbSqvFJWMWrN9OiJNT8WuVFcKS82RIWZrS0a1ERIX4kXwq+RhcAAAAAAAAAAAAAAAAAAAAAAAAhtIoeHQcVnx8FtpdjWqyjJXn3pKTStNi5dTV2DJEaEfA3MVJbQex7oQkyUv8AXUFyAAAAAAAAAAAAAAAAAAAAAh9boeG2OjGfV2o1tKq8UlYxas306Ik1Pxa5UVwpLzZEhZmtLRrUREhfiRfCr5GFwAAAAAAAAAAAAAAAAAyrQL7x/wAQLj8oBqoAAAAAAAMq0C+8f8QLj8oBqoAAAAAAAAAAAAAAAACSw6acvJM7Y+wvuHsWQMse8Ojw+0G9VAc7dvwTz49TsfLdz+I8eRceCArQAAAAAAAAAAAAAAAAAAAAABI6iad1epVVBq7S1ta73dYNWcaTWPpZfbfbStKDJakq225me5ER7kRkYDNsm0ntqO6xOsr9Q9XrSPkNw5WzpbORq4VLCa+XJKU7swouCnYzUcuRoLnJR8RnshQX+n2ldVp5PubSJkWQXM297OUuTcykyHT6CVpRsskJP5L28TPwSki22AXAAAAAAAAAAAAAAAAAAAAAAAAAAAktKJp2WluG2X2GLCu10Fc/9mej0vcnKM2fYeHBvh0d+lx6aNuG3FPyIK0AAAAAAAAAAAAAAAAAAAAAAAElYzTRqlj1aeC9r69Bcv8A2m6O/u3pya0uw8+B8e1dTq8eonl7v34r47thWgAAAAAAAAAAAAAAAAAAAAAAAksxmnEyTBGPsL7+7bkDzHvDo8/s/tVT3O3b8FcOXT7Hy3b/AI9x5Hy4LCtAAAAAAAAAAAAAAAAAAAAAAElqvNOt0tzKy+wxZr2SgsX/ALM9Hq+++MZw+w8ODnPrbdLj01789uKvkYVoAAAAAAAAAAAAAAAAAAAAAAAJLTWadhjkt/7CniRoyC9Y939Hpdbp2spvt23BG/a+PbOWx8u08uTm/NQVoAAAAAAAAAAAAAAAAAAAACS1XmnW6W5lZfYYs17JQWL/ANmej1fffGM4fYeHBzn1tulx6a9+e3FXyMK0AAAAAAAAAAAAAAAABlWgX3j/AIgXH5QDVQAAAAAAAZVoF94/4gXH5QDVQAAAAAAAAAAAAAAAAE5jULM4l1lb+T20WZWzLht7HWWUkS4dcVfEQtl0yQndZzETXCMzX8DqC5FsSEBRgAAAAAAAAAAAAAAAAAAAAAD0vvsxWXJMh1LTTSTW4tZ7JSki3MzM/kREAwDDc31i1A1radq8hhwcGZRBuzq34iCekU7se2joUlw21L6jspFfI2NSOLTZp33NaXA3dibJdsZUJyplssx0tqblrU0bUg1Ee6UElZuEadi35oSXxFxNXjsEXVawV1zkM+nr8UyFdbWTpFbKviRFVBZksEZuIWhL5yWyLiZc1spQfhsrZSTMI/NNeLB/TazynHMUyyjQUeHY1VtMrmHo8+IuUwhS2iaW6aFKbd3S2+ht0yVulG6T4hSr1zp4jdvHt8Oyast6uTCitVD7cVcueuYpSYvQNp9bWzikLL+EcQaOKjWSCLcB4q11qm2kQPsTlB5M5a+5/s1wh9vJ/s/aOXPtHZen0C6nMn+O3w78/hAcDSXV6Rkr6KXI2LIrS0usibhJejNNdljwJZN9F3gf6yUuISRkSyVxM+R+BmHAZ17gryd69cesEYx9nESWK84rRyXZ52jkJKUcTMzU4tKUJSa+PiRnx8TIOm1o1utmMK6VYvNNP7yLkdLAtGo9REtLZiFLe26kdhlE5l81pSsi4IdURpUXEjIB44FrU/jVfl9lk2SZjlVFRqqyhu3+PN1OQuOSnVMr5wOzw1dAlm3wcVHQatnSSbnEgFtZa8V1R216fgGWtx6OO1IyJ9KYK0URLQThJkcZJm4omzJxRRie2SZb+JkQDsbrV2FQZAxVTsLyj3S9PiVp5D2ZhNcmTJ4kygiW8mQ4k1ONo6jbK2yUrY1FxVxCWudbpttk2JQ8VpL2FS2GXO0r1zIjxThWKGGJfWbb/hFPIInWPBa22yV0z4qMjLcKOh1og38V+2i4FmSKc4L1jW2aa1Eli1Yb28Y6Y7jjqVKIyNCHm2lLI/hI9j2DgyfaApauBbOZFhWT0lrUuV7R083sPaZBznDaim243JXGIluJWj+EeRxNJ8uJbGYeNhrNfs5ThdC3pZksRORTpsWwTOKElyEhiObnMlJlGhxJ7pVyaN0jSlaS+MuIDoct18nXWjuX5rhGN5DTlFxydbUd7LYhvRJBtJPitJNvOmhW/FRNyG21GW/w7pURBpWQ5xCw/E4V/bR5k9+YqLFjRITaFSJkt80pbabJSkoJSlH81KSki3MzIiMwE2rXaqS0qAeFZN9p02RVRYxxh+8FPnHOQRkvtHZeHQI3OfX47Fx35/CA6h7WxyDnG9rBvIVa9jjT8egkVyUWLtmqc6x0kI+a1qJGxGlZtGkuoSuG6wHe2WttNWWsuG9imRO11VLj11vctNxjh1kp4mzS07u+Ty+PWa5LabcQnn4qLZWwT+V61zpdtTV2H0l4xXLzSJj0m8WxFVBlGl5TcmOgjcU+nZSVI6htITyQZJXvtuGjaewcyrsBxqv1FtYlplkWnhM306IkksSrFLKCkvNkSEESFOktRESEeBl8KfkQUYAAAAAAAAAAAAAAAAAAAAAAAJybDzFef09hBtYreKMVFmzaQVILrv2K3oRwnkHwMyQ203YJUXNO5vN/CvYjQFGAAAAAAAAAAAAAAAAAAAAAAACcyaFmcu6xR/GLaLDrYdw49kTLySNcyuOvloQy0ZoVsspi4ThmRo+BpZcj3NCwowAAAAAAAAAAAAAAAAAAAAABOahQcyscByWv06tYlXlkqnms0M6WklMRbFTKyjPOEaFkaEumhRkaF+BH8KvkYUYAAAAAAAAAAAAAAAAAAAAAAAJzCIWZQKaSzndrFsbJVxbPMPRUklCa5ywkLgMmRIR8bcNUZtZ7HutCjNS/11BRgAAAAAAAAAAAAAAAAAAAAJzUKDmVjgOS1+nVrEq8slU81mhnS0kpiLYqZWUZ5wjQsjQl00KMjQvwI/hV8jCjAAAAAAAAAAAAAAAAAZVoF94/4gXH5QDVQAAAAAAAZVoF94/4gXH5QDVQAAAAAAAAAAAAAAAAEPgELDo2V6kv4vbS5llMyhh7I2XkmSIdiVLWIQy0ZoTug4aIThmRr+N1Zci2NCAuAAAAAAAAAAAAAAAAAAAAAAB1WR47U5bQWOL38dyRW2sVyHLaQ84ypxlxJpWkltqStO5GfikyMv5jAYUj2cdJtP8AXDCNRZOb5qmxdmlWY3WWOSW1u0/Pbq7Y3EuKmPvlw7G9LWncm+CmNkr/AIVbbgb4yi1TYSnZE2IuCtLZRWURlIdaURH1DW4bhk4R/DsRIRx2Pc1b+AZbaaESsh1DbzK/vMfeYYdeUl6NjaY109HcbWjsb9gh/i7GIlmXT6BGZIRuo1EajD0x9E85dwxWndzqpDlY/AixINQxGx0mHW2Y7zK0KluKkLOQ6SWOBKaJhHxqM21GRbBy9RNAKvUSxvrG1sILqrQ6d+JGnVSJkZiRXuPLQp5payKQ2vrGlSPgPbfZRGZGQddX+z3OoolZZYxdYjRZLUW7lpHeq8NaiVXFyMqOtlUNp9LqyNtRq5qkmsl7GR8S6YDk1uh+UUD1XdUOocIr6vtbie7KnURvx32bJ0nXmSYbkNmg0qSjgvqHsSfiSrfcB1T3sr1NhjqscvcmTaM+6kQEnJq21pU83ZrsG3nGzUaHE81JQpvYiURK8U8tiBI9m+1axZcLGrzBMcvk3lbdMSanBERK1Jw3OaG3IjUpLzvLdW6lSvDf4ST4kYdpK0PynJ/eFxqFn9bYZBPRWRUyKmhXAhMRYc4pZITHclPuGtaiMlLU8ZEXHZJbGSg5uYaMWl/YZS3SZjFqqTPG2m8jhuVJyJLvFlLClRnyeQTClspSgzW26RGRGREe+4TWTey8q9yKbkcfIsdRLTcRLuqnzcWKZZwnY7jS0RlS1SCUqIXSMiabS0oiV+ufjyDt6rQe9gSqWufz2I9jGPZBJv4Naik4SFG+UnqMPSDfNK2yVKWaDS0hRElJKNfiZh4Q9D85YwiXpmWq7LGLsVaqqljxKHoyWWt09Ptj5yFHJJKU8NmkxuSVK5bmZGQddR+zhb4tKu5uM5BhFT9o6+LEsa2FgbbVU8uO44aTOMUnc21oeUlxKnDWoySaXUEXAw7DF/Z+ssS+z8mkyyngSKW6lWSokHH1MVbceRHJh2NDiFJPspcUktJm44knFLUaFEriQdVZ+zLaXqMqctMwx6LNyShm0bk6mxQq96WqQSSKTY8ZJpmuIJBceKWSI1ubbErYg79WlurV9TsVmdamYhMdqJMKxopNRh0mCcWbGWRpW+l6ykE+2pO6FIT0lbKPZZHtsHpk6HZQ7OPOGs+rE5+m196NWa6FaqxBFEOIUc4RSicNvpGZn/2SSzX8XLb4AEbr1h+N6eYjY+0JrFmE2xn4dStssXFZSNIl1stU7kiTFbJwiJCTeQ2bSlGamkqJxxfJW4dFpZT6Ce0HkUvUXSvULTLKSt5ca6u0vYexIu2ZCENNrNpUlzqQ2lmyXwusOGk1L4r3MjSGps6GXkWwYhQ86iNYxDyv7Wxq4qUzllIVIU+6yuT1+KmjW44admUqTundSyIyUFTojDw2u0YwGu05tpVpikXGKpmhnS0ml+VXJitlGecI0IMlqaJCjI0I8TP4U/IguAAAAAAAAAAAAAAAAAAAAAAAAQ9tCw1zWjFrCbby28sYxi/ZrIKUn0H65cupOa8s+BkS23W69KS5p3J5z4V7GaAuAAAAAAAAAAAAAAAAAAAAAAAAQ+fwsOk5Xps/lFtLh2UPKH3scZZSZomWJ0tmhbLpkhWyChrmuEZmj42kFyPckLC4AAAAAAAAAAAAAAAAAAAAAAEPrdDw2x0Yz6u1GtpVXikrGLVm+nREmp+LXKiuFJebIkLM1paNaiIkL8SL4VfIwuAAAAAAAAAAAAAAAAAAAAAAAAQ2kUPDoOKz4+C20uxrVZRkrz70lJpWmxcupq7BkiNCPgbmKktoPY90ISZKX+uoLkAAAAAAAAAAAAAAAAAAAABD63Q8NsdGM+rtRraVV4pKxi1Zvp0RJqfi1yorhSXmyJCzNaWjWoiJC/Ei+FXyMLgAAAAAAAAAAAAAAAABlWgX3j/iBcflANVAAAAAAABlWgX3j/iBcflANVAAAAAAAAAAAAAAAAAcGHU1ddIny6+sixZFpIKXOdZZShcp8mm2SddURbrWTTLTZKVufBtCfkkiIOcAAAAAAAAAAAAAAAAAAAAAAADMtZ8d+1U/T+jOtvzJ/KHD98Uj/QfouNRYq7WpzpOcUL27Ifi3ucxJcz36aw4/cF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgMW9sr2cMpuvZmzqqwnM9T8uu5MRhEOldtzlolq7Uzuk2iQRrIk8lfPw47/wAwD5D9m3/RCazrsq7NNWdR16criuIkNQ8ekk/btqLfw7Q2rpMK2PwWlTv85GkvmA+4tAtJrTLNIccvLnW7VV2Y6y6y66rJ1rU4bT7jRLUpaVKNSiQRme/zM9iItiIOZkWmR6L93cbDtQs192/aiooW6p+1/wCwWoPiRMoZbQhKUElpKCT+qSdy2+WwfRgAAAAAAAAAAAAAAAAAAAAAAAJKxmmjVLHq08F7X16C5f8AtN0d/dvTk1pdh58D49q6nV49RPL3fvxXx3bCtAAAAAAAAAAAAAAAAAAAAAAABJZjNOJkmCMfYX3923IHmPeHR5/Z/aqnudu34K4cun2Plu3/AB7jyPlwWFaAAAAAAAAAAAAAAAAAAAAAAJLVeadbpbmVl9hizXslBYv/AGZ6PV998Yzh9h4cHOfW26XHpr357cVfIwrQAAAAAAAAAAAAAAAAAAAAAAASWms07DHJb/2FPEjRkF6x7v6PS63TtZTfbtuCN+18e2ctj5dp5cnN+agrQAAAAAAAAAAAAAAAAAAAAElqvNOt0tzKy+wxZr2SgsX/ALM9Hq+++MZw+w8ODnPrbdLj01789uKvkYVoAAAAAAAAAAAAAAAADKtAvvH/ABAuPygGqgAAAAAAAyrQL7x/xAuPygGqgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACcyaFmcu6xR/GLaLDrYdw49kTLySNcyuOvloQy0ZoVsspi4ThmRo+BpZcj3NCwowAAAAAAAAAAAAAAAAAAAAAAAAABlPsufyF4z/z3++PAP3X37uPxAp/zQGqgAAAAAAAAAAAAAAAAAAAAAAAnJsPMV5/T2EG1it4oxUWbNpBUguu/YrehHCeQfAzJDbTdglRc07m838K9iNAUYAAAAAAAAAAAAAAAAAAAAAAAJzJoWZy7rFH8YtosOth3Dj2RMvJI1zK46+WhDLRmhWyymLhOGZGj4GllyPc0LCjAAAAAAAAAAAAAAAAAAAAAAE5qFBzKxwHJa/Tq1iVeWSqeazQzpaSUxFsVMrKM84RoWRoS6aFGRoX4Efwq+RhRgAAAAAAAAAAAAAAAAAAAAAAAnMIhZlAppLOd2sWxslXFs8w9FSSUJrnLCQuAyZEhHxtw1Rm1nse60KM1L/XUFGAAAAAAAAAAAAAAAAAAAAAnNQoOZWOA5LX6dWsSryyVTzWaGdLSSmItiplZRnnCNCyNCXTQoyNC/Aj+FXyMKMAAAAAAAAAAAAAAAABlWgX3j/iBcflANVAAAAAAABlWgX3j/iBcflANVAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAEPn8LDpOV6bP5RbS4dlDyh97HGWUmaJlidLZoWy6ZIVsgoa5rhGZo+NpBcj3JCwuAAAAAAAAAAAAAAAAAAAAAAAAAABlPsufyF4z/AM9/vjwD919+7j8QKf8ANAaqAAPnjHM3sJerlo1letGoNUy1k8iurqI8XiNUDzSOKGmFWKq1RmtxRmZJ7alxRmRJL+YBvMqdJjzIcVqolyW5KlJdkNKaJuKRJMyU4S1pWZKMuJcErPc/EiLxASOXarRMYydvD4WIZBkVocD3pIYqSi8o8TqGjqGl99pTvilXwMk4vw/V3UkjDh2ut1FWW8yGWNZDJqqmZHr7W8ZYYKFXSXibNLbqVupkHsTzXJTbS0o5/EZcVcQ9MXXajk3DdcnEsmbr1X72MruXGoyYbdg24psmzLr9ZSVqT8K0NKQW5Eo0mRkQTc3XC1yLI8NLFKS9rKG7lWRN2MyPEONbMMwZC0LZ4uOOtp6jaFl1ENKUnxIlJ3ASGL606lWHsvu2U2+ac1JU01CYsDhspJTsmOmUzK6BJJsyRFcNZp48TNlRbAL6o16iQcZrpdvQX1z7upK6dk1zBYjFFrFvx0Omp5KnUOK+FXUUlhpzikyMyLciAUVdq/XW+Wz8aqMRyGdEq5xVs65joirhxpJtpc4LbJ/tRJ2Wj+E6HT+LfltuZBx8U1rgZdUv5JAwTLkU3YXbCBPTEYlIsmmz2NLKIrzrqXD3I0tuobWe5lx3SokhxVa81MOHbFeYNlVRcVS69tNHJbhuTJipzhtRSZUzIcYPqOJWn43UcTSZr4l4gPNGvVG4iwiKxDJmr2BasUfuJxmMUyROdjFJS00rr9BREyZqNw3SbLir4j2AVuF5nBzWvlSo9bPrJddLXAsK6elspEOQkkqNtfTWts/hWhRKQtSTJRGRmAowAAAQ9tCw1zWjFrCbby28sYxi/ZrIKUn0H65cupOa8s+BkS23W69KS5p3J5z4V7GaAuAAAAAAAAAAAAAAAAAAAAAAAAQ+fwsOk5Xps/lFtLh2UPKH3scZZSZomWJ0tmhbLpkhWyChrmuEZmj42kFyPckLC4AAAAAAAAAAAAAAAAAAAAAAEPrdDw2x0Yz6u1GtpVXikrGLVm+nREmp+LXKiuFJebIkLM1paNaiIkL8SL4VfIwuAAAAAAAAAAAAAAAAAAAAAAAAQ2kUPDoOKz4+C20uxrVZRkrz70lJpWmxcupq7BkiNCPgbmKktoPY90ISZKX+uoLkAAAAAAAAAAAAAAAAAAAABD63Q8NsdGM+rtRraVV4pKxi1Zvp0RJqfi1yorhSXmyJCzNaWjWoiJC/Ei+FXyMLgAAAAAAAAAAAAAAAABlWgX3j/iBcflANVAAAAAAABlWgX3j/AIgXH5QDVQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABJZjNOJkmCMfYX3923IHmPeHR5/Z/aqnudu34K4cun2Plu3/HuPI+XBYVoAAAAAAAAAAAAAAAAAAAAAAAAAA+c9GtRrXTvTanw+60e1JemwO0dVyLjylNK6khxwuJqUk/kst9yLx3/wBoD25ZqBY6rSsAj0ulGotY3Fyyqt3n7jHXYiGo6TUSjWRmamzLqEZktKeJErfYy2AfQ4AAzLINMs3ym8TFvtRYkrDkWke3TWHR8LEnWH0PtM9tS+TfQJxCfh7N1DItjdP5gL+W1armw1QZsVmKhajmNOxlOOPJNJ8SbWTiSbMlbGZmle5eGxfMBnesGj9nqqbcQ7jGk15M9PpXOMJsn4T25/8AZUF9L7K4z+xlss+oRGhBkktj5BOyPZfoftlKyOMWKSWbOdGsbB+4xCNZW6nmm2m1dGc6vi2lZMpM+bLhkanDSpJmXEKhOjZFUsVK8h5E1mbmXGvsn6xKlLf7Ntz8P1+PPf8Am34/zAOkx3Qe+pF4xWSc+iSMfwtUtFJCapTZfJh6M6wlEh831E6ptLpcVIbbIySfJJmZKSH7U+zlDq/cqjyl1w6nEE4ytBQyS3IkIYUw1ONPM+K0tOvo4bnuTn63wkA6aT7KNIqzZmx1YdKVIg18OzkXWGx7OapUSO2wTkR51zjH5Ntp3S42+kleJF89wo7rRCxvNQ4GaSshoENVctEmHKYxhDV6y0nx7IVih4knGP8AVNs45mpHwmoz+IB0T/s23E+ZfWc7NaODYXFW/WrnUOMe7H5puLbUT1kaJKkzFETfH4Us+Dju3ElESQ6G49neVgsK1y+jn1EJ9ZVMtcLEcFJptmbAlKcZktRG5XUdb4OrJ5olLecT/wAGtJklAD10Ojthq/HyjIMyTHfffymJdUz2QYg4zDfUzWtxV9SqlLS+TBmbpElxwnNyJZL24mYbVplgEfTrHV0zTVIh2RJXLfKko2amGS1ERbNx2zUZERJIt1rcWf8AOoy2IgsQGG5F7Zvs8YVrLY6E53nTOMZNXtRniXboOPBkE80TieEk/wCDSZJUnfqGjxPZPLY9g2qLLjTorUyHIafjvoJxp1pZLQ4gy3JSVF4GRl4kZAJqxmmjVLHq08F7X16C5f8AtN0d/dvTk1pdh58D49q6nV49RPL3fvxXx3bCtAAAAAAAAAAAAAAAAAAAAAAABJZjNOJkmCMfYX3923IHmPeHR5/Z/aqnudu34K4cun2Plu3/AB7jyPlwWFaAAAAAAAAAAAAAAAAAAAAAAJLVeadbpbmVl9hizXslBYv/AGZ6PV998Yzh9h4cHOfW26XHpr357cVfIwrQAAAAAAAAAAAAAAAAAAAAAAASWms07DHJb/2FPEjRkF6x7v6PS63TtZTfbtuCN+18e2ctj5dp5cnN+agrQAAAAAAAAAAAAAAAAAAAAElqvNOt0tzKy+wxZr2SgsX/ALM9Hq+++MZw+w8ODnPrbdLj01789uKvkYVoAAAAAAAAAAAAAAAADKtAvvH/ABAuPygGqgAAAAAAAyrQL7x/xAuPygGqgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACcyaFmcu6xR/GLaLDrYdw49kTLySNcyuOvloQy0ZoVsspi4ThmRo+BpZcj3NCwowAAAAAAAAAAAAAAAAAAAAAAAAAAAYbhHs+5dGwygj6ga6aiSsoaq4qLqRXZEoors8mklIWySmUmTZu8zTukj4mW5F8gHe9wXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mA/mz7Uv+jy9ofXL2tci+wsWzlY0mHWpPLMtsS6Sz7KglJJwk9R7iolJ2bbVx47K23LcN80N9iS99mXMMHwZftGagz2ssTbHOh1Nk5XV0dbLCHUqaj8lkpzluk3F/MjMySkz8A+qu4Lzq1V9R/TAePs+vWvurMau1yC1uPc+X2FZGk2ctUh/oNJaSgjWr/eexERbmZkRbgNXAAAAAAAAAAAAAAAAAAAAAAE5k0LM5d1ij+MW0WHWw7hx7ImXkka5lcdfLQhlozQrZZTFwnDMjR8DSy5HuaFhRgAAAAAAAAAAAAAAAAAAAAACc1Cg5lY4Dktfp1axKvLJVPNZoZ0tJKYi2KmVlGecI0LI0JdNCjI0L8CP4VfIwowAAAAAAAAAAAAAAAAAAAAAAATmEQsygU0lnO7WLY2Sri2eYeipJKE1zlhIXAZMiQj424aozaz2PdaFGal/rqCjAAAAAAAAAAAAAAAAAAAAATmoUHMrHAclr9OrWJV5ZKp5rNDOlpJTEWxUysozzhGhZGhLpoUZGhfgR/Cr5GFGAAAAAAAAAAAAAAAAAyrQL7x/xAuPygGqgAAAAAAAyrQL7x/xAuPygGqgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACHz+Fh0nK9Nn8otpcOyh5Q+9jjLKTNEyxOls0LZdMkK2QUNc1wjM0fG0guR7khYXAAAAAAAAAAAAAAAAAAAAAAAAAAAAIfRGHhtdoxgNdpzbSrTFIuMVTNDOlpNL8quTFbKM84RoQZLU0SFGRoR4mfwp+RBcAAAAAAAAAAAAAAAAAAAAAADAtcc4LGNc9JWIGMXOS2SWb6UqtqEsHJKP2ZCTd/h3WkbF4nx5c1cTJCVH4ANygzEWEGPPabfaTJaQ6lt9pTTqSURGRLQoiUhRb+KTIjI/AwHy9jGL1y73MLSx1C1CqlX2p8+mjxaG2SxGbeWhKkrUgy8C2QfIyMz328AGr9wXnVqr6j+mAdwXnVqr6j+mAmtSdJJOI6d5TlcPWnVTtFLSzrFrfISV8bLC3E+Bt7H4pLwAWmi2p0fUfFK91ZS1WTFLTzbB59pttLrkyE3IJSSQe3yX4lsREfgW5eIDn57qrjmnV9iGP38SxW5mdqdTDfjtIUzGe6ZqSp81LI0IUZJQRpJXxLSRkRGZkH7M1Vx6HqzXaOHCsXbmxpn7spDbSDiMMtOIRwcWayUTizUZpIkmWyFGZl4bhbAAAAAAAAAAAAAACHz+Fh0nK9Nn8otpcOyh5Q+9jjLKTNEyxOls0LZdMkK2QUNc1wjM0fG0guR7khYXAAAAAAAAAAAAAAAAAAAAAAAh9boeG2OjGfV2o1tKq8UlYxas306Ik1Pxa5UVwpLzZEhZmtLRrUREhfiRfCr5GFwAAAAAAAAAAAAAAAAAAAAAAACG0ih4dBxWfHwW2l2NarKMlefekpNK02Ll1NXYMkRoR8DcxUltB7HuhCTJS/11BcgAAAAAAAAAAAAAAAAAAAAIfW6Hhtjoxn1dqNbSqvFJWMWrN9OiJNT8WuVFcKS82RIWZrS0a1ERIX4kXwq+RhcAAAAAAAAAAAAAAAAAMq0C+8f8QLj8oBqoAAAAAAAMq0C+8f8QLj8oBqoAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAksxmnEyTBGPsL7+7bkDzHvDo8/s/tVT3O3b8FcOXT7Hy3b/j3HkfLgsK0AAAAAAAAAAAAAAAAAAAAAAAAAAAElpRNOy0tw2y+wxYV2ugrn/sz0el7k5Rmz7Dw4N8Ojv0uPTRtw24p+RBWgAAAAAAAAAAAAAAAAAAAAAD5I9pWhah+01p3ld2nMl1VhQ2VcwWKHPOaiYyhxaPCB/DER9dHif8FslXU+EgG/0TU5ektU1q7VFaTjqYxXsT3ec7rP8E9QjYaQvqHz8TJKTLcjMvABi0DGFZbGsqtVO/YQka2SX5zTTS1JTGS2fI3ST8m/Ekq5fCZK2PwPYBxr/DvcEGzwtvS9gsSPOZL0Nh/E51vVQo/YGVINNTCNHXaW8p7ZR/wTbm6z+IiAcfSrTN3JrXGMb1M08lTK7H6rKa6TDtKF5qsTzs4zkRtDT3UZU30OJtJJbiUkgySozbPYJbKsUlM6Wyj1R0/vLgy0oVAx3rUEmcqrtENyuvyUSFdidNJxdnXOmWyNufwmQBjuF53M0zoHWaN06M4uHPXDE3HJNsxMhN0BNrScJlbbktCJBtc2kGZkafFJ8TSArIWk97f1mO0kaLOZrZNlfP16kYw/TRqYlwSKOpqI6665Ga7QjmhLptnyMyJCS47hy0Y3qrk9jGz9jHLGlyzJMeyUkE8ypJVb/QiMQmlr2ImzUbKnEkZl4rUZfIwHUYXp465TXzCa26qIj2MOV91HxvTuXRPSZK3G/wCFcVMkue8ZCOLx82kOEtK3S5qNaEqD9jYjBmYTPqpOBqg0UW7iTIsiq0vsI9dOdJh1PGbjbpm+42gzQSnGSSS3DaX8BNmaQ8bXH7PIcYxbGbTRurjRG4doVb23CLW1rzdVJ4tE3Wm+gqpTiCS4hUpZdFKjbSaS5AJ+6grkaaXFpqjp5llldM6YRoVFOk0ExT1fOjxpKJySfWjaE6bnBRvLW2TqOHFSyIB3T2C5dPzWVItYxFazLOqk43bR8CmWM2LARHi7ExbJlNxobaVIfJxpzjuRuGaXOoRKD7CAAAAAAABJZjNOJkmCMfYX3923IHmPeHR5/Z/aqnudu34K4cun2Plu3/HuPI+XBYVoAAAAAAAAAAAAAAAAAAAAAAktV5p1uluZWX2GLNeyUFi/9mej1fffGM4fYeHBzn1tulx6a9+e3FXyMK0AAAAAAAAAAAAAAAAAAAAAAAElprNOwxyW/wDYU8SNGQXrHu/o9LrdO1lN9u24I37Xx7Zy2Pl2nlyc35qCtAAAAAAAAAAAAAAAAAAAAASWq8063S3MrL7DFmvZKCxf+zPR6vvvjGcPsPDg5z623S49Ne/Pbir5GFaAAAAAAAAAAAAAAAAAyrQL7x/xAuPygGqgAAAAAAAyrQL7x/xAuPygGqgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACcyaFmcu6xR/GLaLDrYdw49kTLySNcyuOvloQy0ZoVsspi4ThmRo+BpZcj3NCwowAAAAAAAAAAAAAAAAAAAAAAAAAAATmnsHMq7Acar9RbWJaZZFp4TN9OiJJLEqxSygpLzZEhBEhTpLUREhHgZfCn5EFGAAAAAAAAAAAAAAAAAAAAAAMc1jkWtFqTpzmMTEcgvYdN747W3TwFSnUdaO22jci2ItzP8AnMvBKtt9tgHAl+1NEi354yxoZq1MsihpnnHZomErJg1qRz4uSEqMuSdjMiMiMy8fEBO6SaoXmLNZa7eaG6pMKvMpn3MZsqBK1FHeJvgS+LpkSvhPciM/9pgPmn2lv9Lde4AZ0WluhN5AlySdKPc5kz0I7pNuG2tcdhlaifRuR7OE8RbkW6T3Aez/AEcPti6v6oM6k5Dq4jOM9lKnV3ZE01Sh2LXINt7khLTZoQzy2T4EXxcNzMz3Mw+s881km5Bg2RUFdopqiqXZVUuGwS8e4pNxxlSE7n1PAtzLxAahp7Vz6PAsapbNnoTa+nhxZLXJKuDrbKErTukzI9jIy3IzL+gBRgAAAAACMy/SXBM7sU2eSVkx182SivpjWsuI1Mjko1EzJaYdQiS2RqV8DyVp+JRbbKPcLBCENoJtCSSlJERERbERf0APMAAAAAAAE5k0LM5d1ij+MW0WHWw7hx7ImXkka5lcdfLQhlozQrZZTFwnDMjR8DSy5HuaFhRgAAAAAAAAAAAAAAAAAAAAACc1Cg5lY4Dktfp1axKvLJVPNZoZ0tJKYi2KmVlGecI0LI0JdNCjI0L8CP4VfIwowAAAAAAAAAAAAAAAAAAAAAAATmEQsygU0lnO7WLY2Sri2eYeipJKE1zlhIXAZMiQj424aozaz2PdaFGal/rqCjAAAAAAAAAAAAAAAAAAAAATmoUHMrHAclr9OrWJV5ZKp5rNDOlpJTEWxUysozzhGhZGhLpoUZGhfgR/Cr5GFGAAAAAAAAAAAAAAAAAyrQL7x/xAuPygGqgAAAAAAAyrQL7x/wAQLj8oBqoAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAh8/hYdJyvTZ/KLaXDsoeUPvY4yykzRMsTpbNC2XTJCtkFDXNcIzNHxtILke5IWFwAAAAAAAAAAAAAAAAAAAAAAAAAAACH0Rh4bXaMYDXac20q0xSLjFUzQzpaTS/KrkxWyjPOEaEGS1NEhRkaEeJn8KfkQXAAAAAAAAAAAAAAAAAAAAAAAAMq/8Kb/0f/8A1EBqoDDtDcJw7UH2b8fxnOsWqsgqZBzDchWURuQyoymP7K4rIyJRfzGXiX8xgO00K9l7SD2bpmUv6RU0yni5ZIjyZcBcxb8dhbKVknok5utJH1FGZGpXzIi2IiIg10AAAAAAAAAAAAAAAAAAAEPn8LDpOV6bP5RbS4dlDyh97HGWUmaJlidLZoWy6ZIVsgoa5rhGZo+NpBcj3JCwuAAAAAAAAAAAAAAAAAAAAAABD63Q8NsdGM+rtRraVV4pKxi1Zvp0RJqfi1yorhSXmyJCzNaWjWoiJC/Ei+FXyMLgAAAAAAAAAAAAAAAAAAAAAAAENpFDw6Dis+PgttLsa1WUZK8+9JSaVpsXLqauwZIjQj4G5ipLaD2PdCEmSl/rqC5AAAAAAAAAAAAAAAAAAAAAQ+t0PDbHRjPq7Ua2lVeKSsYtWb6dESan4tcqK4Ul5siQszWlo1qIiQvxIvhV8jC4AAAAAAAAAAAAAAAAAZVoF94/4gXH5QDVQAAAAAAAZVoF94/4gXH5QDVQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABJZjNOJkmCMfYX3923IHmPeHR5/Z/aqnudu34K4cun2Plu3/HuPI+XBYVoAAAAAAAAAAAAAAAAAAAAAAAAAAAJLSiadlpbhtl9hiwrtdBXP/Zno9L3JyjNn2Hhwb4dHfpcemjbhtxT8iCtAAAAAAAAAAAAAAAAAAAAAAABlX/hTf+j/AP8AqIDVQGU+y5/IXjP/AD3++PANWAAAAAAAAAAAAAAAAAAAAASWYzTiZJgjH2F9/dtyB5j3h0ef2f2qp7nbt+CuHLp9j5bt/wAe48j5cFhWgAAAAAAAAAAAAAAAAAAAAACS1XmnW6W5lZfYYs17JQWL/wBmej1fffGM4fYeHBzn1tulx6a9+e3FXyMK0AAAAAAAAAAAAAAAAAAAAAAAElprNOwxyW/9hTxI0ZBese7+j0ut07WU327bgjftfHtnLY+XaeXJzfmoK0AAAAAAAAAAAAAAAAAAAABJarzTrdLcysvsMWa9koLF/wCzPR6vvvjGcPsPDg5z623S49Ne/Pbir5GFaAAAAAAAAAAAAAAAAAyrQL7x/wAQLj8oBqoAAAAAAAMq0C+8f8QLj8oBqoAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAnMmhZnLusUfxi2iw62HcOPZEy8kjXMrjr5aEMtGaFbLKYuE4ZkaPgaWXI9zQsKMAAAAAAAAAAAAAAAAAAAAAAAAAAAE5p7BzKuwHGq/UW1iWmWRaeEzfToiSSxKsUsoKS82RIQRIU6S1ERIR4GXwp+RBRgAAAAAAAAAAAAAAAAAAAAAAAyr/AMKb/wBH/wD9RAaqAyn2XP5C8Z/57/fHgGrAAAAAAAAAAAAAAAAAAAAAJzJoWZy7rFH8YtosOth3Dj2RMvJI1zK46+WhDLRmhWyymLhOGZGj4GllyPc0LCjAAAAAAAAAAAAAAAAAAAAAAE5qFBzKxwHJa/Tq1iVeWSqeazQzpaSUxFsVMrKM84RoWRoS6aFGRoX4Efwq+RhRgAAAAAAAAAAAAAAAAAAAAAAAnMIhZlAppLOd2sWxslXFs8w9FSSUJrnLCQuAyZEhHxtw1Rm1nse60KM1L/XUFGAAAAAAAAAAAAAAAAAAAAAnNQoOZWOA5LX6dWsSryyVTzWaGdLSSmItiplZRnnCNCyNCXTQoyNC/Aj+FXyMKMAAAAAAAAAAAAAAAABlWgX3j/iBcflANVAAAAAAABlWgX3j/iBcflANVAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAEPn8LDpOV6bP5RbS4dlDyh97HGWUmaJlidLZoWy6ZIVsgoa5rhGZo+NpBcj3JCwuAAAAAAAAAAAAAAAAAAAAAAAAAABKd7Glv9ZeK/vmN/jAZ/prq77NunmiOFxqbWvG28Qp8crIdVPubmPGkOwm4zaI6nkr6ZpdU2lJqQbaFcjMuCT+Eg+Zde/8ATC6I4IiTT6KUszP7hJGhE5wlwqttfiXLksuq9xMv1UoSlRbcXNj3AfW+nWu+A5Zp/jGU3meYnBsrmmhWEyKi3YSlh91hC3GyJS+RElSjLY/Hw8QFH3saW/1l4r++Y3+MA72NLf6y8V/fMb/GAd7Glv8AWXiv75jf4wDvY0t/rLxX98xv8YB3saW/1l4r++Y3+MA72NLf6y8V/fMb/GAd7Glv9ZeK/vmN/jAO9jS3+svFf3zG/wAYB3saW/1l4r++Y3+MA72NLf6y8V/fMb/GAd7Glv8AWXiv75jf4wDvY0t/rLxX98xv8YB3saW/1l4r++Y3+MA72NLf6y8V/fMb/GAd7Glv9ZeK/vmN/jAO9jS3+svFf3zG/wAYDOMjq9KdUtXqxyg1tu4eTuY3MSiFi1sySHoEeVH6rrriGlmg0Oy2Ukk3EksnDMkL6ZqQHe9wXnVqr6j+mAssCwuq09xODh1HIlvwq/qk25KWlTp9R1Th8jSlJfNZ7bEXht/tAUYAAAAAAAAAAAAAAAAAAAACHz+Fh0nK9Nn8otpcOyh5Q+9jjLKTNEyxOls0LZdMkK2QUNc1wjM0fG0guR7khYXAAAAAAAAAAAAAAAAAAAAAAAh9boeG2OjGfV2o1tKq8UlYxas306Ik1Pxa5UVwpLzZEhZmtLRrUREhfiRfCr5GFwAAAAAAAAAAAAAAAAAAAAAAACG0ih4dBxWfHwW2l2NarKMlefekpNK02Ll1NXYMkRoR8DcxUltB7HuhCTJS/wBdQXIAAAAAAAAAAAAAAAAAAAACH1uh4bY6MZ9XajW0qrxSVjFqzfToiTU/FrlRXCkvNkSFma0tGtRESF+JF8KvkYXAAAAAAAAAAAAAAAAADKtAvvH/ABAuPygGqgAAAAAAAyrQL7x/xAuPygGqgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACSzGacTJMEY+wvv7tuQPMe8Ojz+z+1VPc7dvwVw5dPsfLdv+PceR8uCwrQAAAAAAAAAAAAAAAAAAAAAAAAABhHs5afYDd6M49aXeEUFhNkHL6smVWMuur2lvJLktSTM9iIiLc/kREA6X2n9K8HXi2KV1FpNhNi9ZZhWRF106K3DiT0K6h9CQ62w6pLZqSkzPpr2NJHxMyIB8kZT/otsS1ps8vVhlDU6S3FC+1HbbqMkk5FSTJSmzccZMpMOK/HUnm3yUg3EJ5kSUbpNID7w0t0e0tp8BosSk4rg9zaYxWxKW2kRa6M8XbWI7aXSUZp5JUZ/FsvZWyiMy8QHtrIvsyXNo5R0sbTCws2o6pbkOKivdfQwXzdNtO6iQW5bq22AceRI9lKHVe/5b+k7FWUgonbXFVqWOvwJfS6h/Dz4KSrjvvxMj+RgOfd1Ps4Yz7tPJazTWqK6USK3tzMBjtqj22JnmRdQz5J8E7/ADL+kB5SaP2dYWTx8KmU+nDGQS0E5HqXY8FE11JkZkpDBlzUWyVHuRfzH/QA8IVX7N1lfzcVr63TWTdVyVLm1rLMBcqMlO3I3GiLmgi3LfkRbbkAj83yXQKnwC0zbTjAdPdQX6ybCgrr6Y69alPSJDbKUG4hC0oX/CbkSiLfbbct9yDuoLGjNtfYjXU2lmKyoGX1Uy1YmnVx0GyhgmD4m2bW5mfX2PcyNJoMjI/5g7Ctj+zJcxrGZUR9MJ8anWTdi7GRXuohrNXEkvKTuTZmojLZW3iWwDhZnJ9mbT+VChZTSafw5E2wjViWXYsBtbbr5Gps1pXsaUmlKlb/ANBeBGAz2symskNrymx9mfTZnDftO5jDUuNZIetXHSsDhJd7CquQ1sbhcjSmSpRI3MuRlsA1N6s9myM/ZMSIGmjTtM0t+ybW1ASqE0hfBS3iMt20kr4TNWxEfh8wHh2T2ZTxY84KNph9myX0zuOFf2El8uPHr/8AB78vDbf5+AD9mwfZoq6CBlVlC0yiU1qpKYNk+3XtxZJqIzSTTp/AszIj24me+wDosdtfZ0m4jSZXluMacYuV/JkRoLFimA32hxt9bRJaU4hHUUrgR7JLf4iLx+ZhRTKf2cq7JIuGWNVpvFyCcklRal5iA3MfIyMyNDJlzUR8VfIj+R/0AONIj+zHEt7Cgfj6YotaplyTOr1Irykxmm0kpa3Gv1kJSkyMzMiIiMjMBwdNb3S/IsjxHJ9KdNKRVVk2MWs77VVkKOjsHSkVxFWuuMoNJLfN5SzR1S8a9Xwr4maA2MAAAAAAAAAAAAAAAAAAAAAAAElmM04mSYIx9hff3bcgeY94dHn9n9qqe527fgrhy6fY+W7f8e48j5cFhWgAAAAAAAAAAAAAAAAAAAAACS1XmnW6W5lZfYYs17JQWL/2Z6PV998Yzh9h4cHOfW26XHpr357cVfIwrQAAAAAAAAAAAAAAAAAAAAAAASWms07DHJb/ANhTxI0ZBese7+j0ut07WU327bgjftfHtnLY+XaeXJzfmoK0AAAAAAAAAAAAAAAAAAAABJarzTrdLcysvsMWa9koLF/7M9Hq+++MZw+w8ODnPrbdLj01789uKvkYVoAAAAAAAAAAAAAAAADKtAvvH/EC4/KAaqAAAAAAADKtAvvH/EC4/KAaqAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAJzJoWZy7rFH8YtosOth3Dj2RMvJI1zK46+WhDLRmhWyymLhOGZGj4GllyPc0LCjAAAAAAAAAAAAAAAAAAAAAAAAAAGU+y5/IXjP8Az3++PAPD2g1OIZ08cZb6jic9qTQjfbkrZ3YvEBR6P47cYzp1TwckilHvJDa59u31EuGU+QtT0j4kmaVfwjii3IzLYi2PYBUQpkmWuUl+plQijvqZbU+toykIIiMnUdNajJB7mREvirdJ7pItjMPmHTDF8syzAdOq2r0/drYWPv21q5c9qi9nlk8zMZS00lLvX6ji5CVL6jaUFwV8Svh3CvxvBcx04Y02u4Wn0m7Tj2EqxubTV0mEh+HKX2ZZuJN91tlSTNlaFmlwz/VMiURmZB1WLadag6ZojHL0zPOWLfFmqF+sizYSI1Y4UqU8pl45S0coppkobNTSXVbM/wDBGWxAPVJ0gz/7W3VPKj5U9S32URMgQusmUbNTGbb7OpKXVvR1WBLaOPxSTSTSpKWyJTZGrgHLj6fZwm+mUcHTy5LGHTt1Taq7tKx6qcOS2+alVsprlYsOPOO7K6qUoShbhEktk7hxGcA1Xt6ubXLxvIU17MzHTgt5M9TKs2m4tk08823IgrMnIrbKTUkn1G8a+W3LcgHPzHRnPZeo1hWYsgo2J2uOZF2SxTIbT7ps7BLCVs9PfqGhbiFPpUlKiSpTpHt8BGHpxDT7PYrMmxvMNyu2l1eKu45Fr8gn0DcKX1VNEbLaK9pJqjF0+XN9SVpTuSWjUoyIFVpXqFguN1NCVBJyqRjmXwL1dnHmMJlXEc2ltuGrtLydnGOSUbLWRG023xNSt0kEXQaK20Nd9Er/AGYjpc/nZJZToWpn/wCA2+iy9YLdbe7SzKVYq2YUSekbOytumrZJmYDSJWluWMY7cyYuOokTk6jnlpwEvMJXbQ25CFpSSlK4Es0JJSEuKSRKQklGj5kHWXGK6rSrmfnNZgtpVou8kYnLgwzpZF5XtM13ZiktKluLgoccWXFZpWtfR8CPczJIdfgOB6rYHa0V9cYBc5AdHPyaO8w1PqutIRZS25LM1G7jDOxJSptwtmlEpSuDak+Jh7sJ0/1E02jwJk7Sd3KW5+OyKB+lhz68ma9R2El/+EOQ42lUZ1t9slG2S1kTSSNo/kQcbPcB1itrC1bq8MsI0SLe01xGgUi6NirsI0NcRZtqdeSmauWRNLQk1Gw1xbbLkki2MKCuw3UFjO0xqPDr2oon7SwsJ7NrLqp1OlT7bxnJhLSo7BmStx1O6TImUkp5JblxUoKTRyi1QqKTT2tuInuGnx/FZlLeU8tyO7IkWbbkFEKU0tg3E9EmmZ57dVCtpDXNvkWzYa4AAAAAAAAAAAAAAAAAAAAAAACcyaFmcu6xR/GLaLDrYdw49kTLySNcyuOvloQy0ZoVsspi4ThmRo+BpZcj3NCwowAAAAAAAAAAAAAAAAAAAAABOahQcyscByWv06tYlXlkqnms0M6WklMRbFTKyjPOEaFkaEumhRkaF+BH8KvkYUYAAAAAAAAAAAAAAAAAAAAAAAJzCIWZQKaSzndrFsbJVxbPMPRUklCa5ywkLgMmRIR8bcNUZtZ7HutCjNS/11BRgAAAAAAAAAAAAAAAAAAAAJzUKDmVjgOS1+nVrEq8slU81mhnS0kpiLYqZWUZ5wjQsjQl00KMjQvwI/hV8jCjAAAAAAAAAAAAAAAAAZVoF94/4gXH5QDVQAAAAAAAZVoF94/4gXH5QDVQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABD5/Cw6Tlemz+UW0uHZQ8ofexxllJmiZYnS2aFsumSFbIKGua4RmaPjaQXI9yQsLgAAAAAAAAAAAAAAAAAAAAAAAAAAZT7Ln8heM/8APf748A/dffu4/ECn/NAaqAAOHW1dbTwm6yogR4MNgjJqPGaS022RnuZJSkiIvEzPw/pAcwAAAAAAAAAAAAAAAAAAAEPbQsNc1oxawm28tvLGMYv2ayClJ9B+uXLqTmvLPgZEtt1uvSkuadyec+FexmgLgAAAAAAAAAAAAAAAAAAAAAAAEPn8LDpOV6bP5RbS4dlDyh97HGWUmaJlidLZoWy6ZIVsgoa5rhGZo+NpBcj3JCwuAAAAAAAAAAAAAAAAAAAAAABD63Q8NsdGM+rtRraVV4pKxi1Zvp0RJqfi1yorhSXmyJCzNaWjWoiJC/Ei+FXyMLgAAAAAAAAAAAAAAAAAAAAAAAENpFDw6Dis+PgttLsa1WUZK8+9JSaVpsXLqauwZIjQj4G5ipLaD2PdCEmSl/rqC5AAAAAAAAAAAAAAAAAAAAAQ+t0PDbHRjPq7Ua2lVeKSsYtWb6dESan4tcqK4Ul5siQszWlo1qIiQvxIvhV8jC4AAAAAAAAAAAAAAAAAZVoF94/4gXH5QDVQAAAAAAAZVoF94/4gXH5QDVQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABJZjNOJkmCMfYX3923IHmPeHR5/Z/aqnudu34K4cun2Plu3/AB7jyPlwWFaAAAAAAAAAAAAAAAAAAAAAAAAAAMIwLC/aP09xOFh1JI02fhV/VJpyUuep0+o6pw+RpSkvms9tiLw2/wBoDr6Cr9obWKg06yvPqbEMcjImVGVyqpKZzNjCcJBOLiupdRsTiCcWhSTJJ807GZeID6HAAAAAAAAAAAAAAAAAAAAAAABJWM00apY9Wngva+vQXL/2m6O/u3pya0uw8+B8e1dTq8eonl7v34r47thWgAAAAAAAAAAAAAAAAAAAAAAAksxmnEyTBGPsL7+7bkDzHvDo8/s/tVT3O3b8FcOXT7Hy3b/j3HkfLgsK0AAAAAAAAAAAAAAAAAAAAAASWq8063S3MrL7DFmvZKCxf+zPR6vvvjGcPsPDg5z623S49Ne/Pbir5GFaAAAAAAAAAAAAAAAAAAAAAAACS01mnYY5Lf8AsKeJGjIL1j3f0el1unaym+3bcEb9r49s5bHy7Ty5Ob81BWgAAAAAAAAAAAAAAAAAAAAJLVeadbpbmVl9hizXslBYv/Zno9X33xjOH2Hhwc59bbpcemvfntxV8jCtAAAAAAAAAAAAAAAAAZVoF94/4gXH5QDVQAAAAAAAZVoF94/4gXH5QDVQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABOZNCzOXdYo/jFtFh1sO4ceyJl5JGuZXHXy0IZaM0K2WUxcJwzI0fA0suR7mhYUYAAAAAAAAAAAAAAAAAAAAAAAAAAAJzT2DmVdgONV+otrEtMsi08Jm+nREkliVYpZQUl5siQgiQp0lqIiQjwMvhT8iCjAAAAAAAAAAAAAAAAAAAAAAABOTYeYrz+nsINrFbxRios2bSCpBdd+xW9COE8g+BmSG2m7BKi5p3N5v4V7EaAowAAAAAAAAAAAAAAAAAAAAAAATmTQszl3WKP4xbRYdbDuHHsiZeSRrmVx18tCGWjNCtllMXCcMyNHwNLLke5oWFGAAAAAAAAAAAAAAAAAAAAAAJzUKDmVjgOS1+nVrEq8slU81mhnS0kpiLYqZWUZ5wjQsjQl00KMjQvwI/hV8jCjAAAAAAAAAAAAAAAAAAAAAAABOYRCzKBTSWc7tYtjZKuLZ5h6KkkoTXOWEhcBkyJCPjbhqjNrPY91oUZqX+uoKMAAAAAAAAAAAAAAAAAAAABOahQcyscByWv06tYlXlkqnms0M6WklMRbFTKyjPOEaFkaEumhRkaF+BH8KvkYUYAAAAAAAAAAAAAAAADKtAvvH/EC4/KAaqAAAAAAADKtAvvH/EC4/KAaqAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAIfP4WHScr02fyi2lw7KHlD72OMspM0TLE6WzQtl0yQrZBQ1zXCMzR8bSC5HuSFhcAAAAAAAAAAAAAADpMuyqrwnHJ+UXXW7JAQSlJZRzccUpRJQhBeG6lKUlJbmRbmW5kXiA6vC9Q4+W2VpQzMbuMeuahLDsqttezKeSy8Suk6S4zzzSkqNtwvBe5GgyMi8NwrwHQXeXVtBe4/j8uPJXIyOU/EiqbSk0IW0wt5RuGaiMi4tqItiPxMvkXiA8syyyuwbE7fMbdiS9CpYbs2Q3GSlTqm20moyQSjSRnsXhuZF/rAc6unvT1SOpVyojbThIacfNoykoNCVdRHBajJO6jTsskq3Srw22Mw54Dq8itJ1LTSbStxuxvpMdHJutrnIyJMg9/1UKkutMkf8/xuJLw+YCF0x1mtNTLGVGLRbOcbhQ5EqG/ZXEimVHRJjudNxnjEsH3jVyJREom+B8T+L5bh2WpOqJ6dy8fqoeB5Jltnkkp6LCgUi4KHSNplTq1rVNkx2ySSUn/AMczM/5gGGezVheDZ1plj8DBdStb6GuqqGpVArbi7jtSU1b0VKoL20Yls8FspLYkK+E0qSaUGRpINa7gvOrVX1H9MBxJGh0hmbGio1V1dfafJw3JLeStk3H47GRLJREs+W5kXBKvke/Hw3CYz/HbHRu0wTJIer2dy27HMa2klRra1KXGfYlc21IU2bfzMzTsfzSexkZbbgPocAAAGPSfaKjyp1NW4bpRm+XTLiDMsUsVa6lg47EaV2ZanTmzmE7m5+qSDUe3ie3yAXWC5zAzypkTotbY1cqBLcgWFbYoQmVCkoIjU0501rbM+KkqJSFrQolEZKMjAU4AA4cOW/JclIdrJMRMd42m1vKbMpCeKT6iOC1GSdzNOyySrdJ/DtsZhzAHQZzl9dgOH3WbXEeU9AooL0+Q3GSlTq220mpRIJSkkati8NzIv9YDuIr6JcVmW2Rkl5tLiSV8yIy3Lf8A9YD3gOgcy2tj5rFwRUeT2+ZWP2rbhJT0SZadbbUkz5cuXJ5OxcdtiPxLwIw78B0GWZbXYbEgTbJmS6iws4dU0TCUqMnpLqWkKVyUXwkpRGZlue2+xH8gHTW0LDXNaMWsJtvLbyxjGL9msgpSfQfrly6k5ryz4GRLbdbr0pLmncnnPhXsZoC4AAAAAAAAAAAAAAAAAAAAAAABD5/Cw6Tlemz+UW0uHZQ8ofexxllJmiZYnS2aFsumSFbIKGua4RmaPjaQXI9yQsLgAAAAAAAAAAAAAAAAAAAAAAQ+t0PDbHRjPq7Ua2lVeKSsYtWb6dESan4tcqK4Ul5siQszWlo1qIiQvxIvhV8jC4AAAAAAAAAAAAAAAAAAAAAAABDaRQ8Og4rPj4LbS7GtVlGSvPvSUmlabFy6mrsGSI0I+BuYqS2g9j3QhJkpf66guQAAAAAAAAAAAAAAAAAAAAEPrdDw2x0Yz6u1GtpVXikrGLVm+nREmp+LXKiuFJebIkLM1paNaiIkL8SL4VfIwuAAAAAAAAAAAAAAAAAGVaBfeP8AiBcflANVAAAAAAABlWgX3j/iBcflANVAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAElmM04mSYIx9hff3bcgeY94dHn9n9qqe527fgrhy6fY+W7f8e48j5cFhWgAAAAAAAAAAAAACD1vrry20vvKygrznvyUMtvRkspdcdim8jtJNIURkp3o9TgW363Hbx2AS+iVUzDyvJ52L1mUMYrKiV5MSMoZsE2L85HWJ4udkXbFtJbNgi6hmglGrh/xgGlU7dMhVt7gquwPLnOHMUdcuL2iVwTu9upCevuXAuqnkR8duR8TIg+X8QwqxRIwZuk0+t6vUGtVbpyi+foH4zb1g5AlIbfdlrQSJaVPKI0OIW4lJGSd0ciSYT9TpxlH2Bv2IGPy2LpODT4N1FrtPZ1S5Zz1toIikyn5LhWUjqJcNLrKHN+Th80ksiUGh2dHluI3mTasVGG3FhY0WX9RqFGhOKfsq2TUw47yWUkW7iSeQ0szTuW8dX9BgJfLdJ8jo51XAyith3FYvGm+kt/BJ2Sm3fPSH3pzzSYr7fY3lrdbNL69i2SREtPE9w1LR/Tt+s1ByLJ8uqZU24iwKWHEubOGSXlmmvQiQttXJaUrUouLnTWrcyIjUrYgE3YYrcLxRpvIcUsJ+Ntak3FhkFT7rdlLm1i3ZXSX2VKTXJa6q47nFKF8kkSiJREA6BnTbLbzJsSThj2XYFjKcvt5tM9WU7DL1TWqqktkk406K63EadkE8aW3GUmXMtiSZ7AJ/BsP1Jk+z47Dm4dc1eSS/s1NsmLGjNxCseJLTiaxMOG3DbPsrRux3YjLbKlG2v8Agy6yUgO7h4QuLQRnp+KTLPT9zL48q0x6vwCZVxOzJhOoNbVO44/IWz2k461o6SSNSDWSDLdZh6MZxw7efYpo8ayqLSNXGYRenXRlNTITLsGIhlDZL2JhZpLZptZp47JTxTx4kGcanYrUxsaoIC9LI8rGoef4pJdeh6d2FK5OSh1/rsrqXkqVLeSyRcnmGv4UnOnxPiSQGo12C5Q5qk7JsoPZr08tKfBtY2n01yYmnJ4lMslcqkojMx+zl0lxzSSkkay6SlGRqDjT9KLauwKgsGcPYbalZfcS8tascUk3TkyKb87sZvwWVtvymULdbUhBckp5pWSdiMwHOrMLRWowp3P8StckwRp28caq04TKNmBJecZOEZVRKkvsMpQmSTRuERtE4RGTXgRBxtPtJ9RJ99hbNdkec6cprsSuI70qDXQXVk4u35tx31T4slvfh8ZEnZSiLclGXiAXGE5mWGY7ByGkkzl1+Tz3M6Xb4rKv2LqWcc0R7DsMZbRyY6j6RpS0SkMmaCNBdIzSHtiYfCqIuLO6p4Vc5ZgqItumFWoweY+mvmPS2lReNWhUp+M2TPVSytwiNlHwq6W/EBwc0wPJbLLLE5dDYRCnQaxrD3peFTcgtqhtMdtKkNWbc0mYD6HyUpanlkSj2Upxwi2IKuzpMvxCzyLVSrxK4s7SgzSQ6iJGhLN+0rZVdFYd6KCLdxPWQ05unct2FePgYCUy3SfI6OdVwMorYdxWLxpvpLfwSdkpt3z0h96c80mK+32N5a3WzS+vYtkkRLTxPcOxybSm4ewvV+zt8Ytb3Jzw+HXVc+RWm5MlO+6Sbf6CUG5u6tfwuE0pe5kSd1bEAr8tziDqhp43Q4xjOcOtVkupkX0Gywq5qzmViJDfamWymRmSkmaCVyZbNalpJSeJ8tjCTm4nHJM2fTacWjGkRZPBkzMYTjElHaIqILyJDjdR0ieUycpUZSmiZ3WppSyQoviUHhk2GZFa21DK0fwe8pMTj49aFMp5sGTCfnQzsIzioMZS1kdebyEuG20tKDJBcCQ0RmaQ4OX4VfXebW8tukXCRZe7VYVILTmdOsKmImMyRNx5hSGWatSHkumpt4m/mZnzI+JBz5mHm5ki0W2nVxNzxGo8Wyev28ffNC6Yp6FMn28kdJTKGOmk2ScM0Gg1GguJqIPoCxmmjVLHq08F7X16C5f+03R3929OTWl2HnwPj2rqdXj1E8vd+/FfHdsK0AAAAAAAAAAAAAAAAAAAAAAAElmM04mSYIx9hff3bcgeY94dHn9n9qqe527fgrhy6fY+W7f8e48j5cFhWgAAAAAAAAAAAAAAAAAAAAACS1XmnW6W5lZfYYs17JQWL/2Z6PV998Yzh9h4cHOfW26XHpr357cVfIwrQAAAAAAAAAAAAAAAAAAAAAAASWms07DHJb/2FPEjRkF6x7v6PS63TtZTfbtuCN+18e2ctj5dp5cnN+agrQAAAAAAAAAAAAAAAAAAAAElqvNOt0tzKy+wxZr2SgsX/sz0er774xnD7Dw4Oc+tt0uPTXvz24q+RhWgAAAAAAAAAAAAAAAAMq0C+8f8QLj8oBqoAAAAAAAMq0C+8f8AEC4/KAaqAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAJzJoWZy7rFH8YtosOth3Dj2RMvJI1zK46+WhDLRmhWyymLhOGZGj4GllyPc0LCjAAAAAAAAAAAAAAAAAAAAAAAAAAGO1ftJVd7AatKXSjUmwhvGrpSYtGl1peyjSfFaXTI9jIyPY/mRkAhi9qm/wBKtH8dsNWdItUb7Ko9bXV9s7V422krK3NpCXlR0GtrdK3ScWSUoSZJ3MkERGRB/N72o/8ASo6y6utvYfhGKN6c1sWWTrb5PLdumXm1KLkmRsjs6vmX8Gglp3UnmZAP6S6Bavxsf0Zw2M1pBqlNkSqWHOmzypVyVT5jzKHH5Snlump1TjilLNaj3PkA7fM8xu9UrrBaSi0oz2vOvy2Bay5VtUpixmYzCXFOKU4pz57GWxfMzMiLxMB9AAAAAAAAAAAAAAAAAAAAAnJsPMV5/T2EG1it4oxUWbNpBUguu/YrehHCeQfAzJDbTdglRc07m838K9iNAUYAAAAAAAAAAAAAAAAAAAAAAAJzJoWZy7rFH8YtosOth3Dj2RMvJI1zK46+WhDLRmhWyymLhOGZGj4GllyPc0LCjAAAAAAAAAAAAAAAAAAAAAAE5qFBzKxwHJa/Tq1iVeWSqeazQzpaSUxFsVMrKM84RoWRoS6aFGRoX4Efwq+RhRgAAAAAAAAAAAAAAAAAAAAAAAnMIhZlAppLOd2sWxslXFs8w9FSSUJrnLCQuAyZEhHxtw1Rm1nse60KM1L/AF1BRgAAAAAAAAAAAAAAAAAAAAJzUKDmVjgOS1+nVrEq8slU81mhnS0kpiLYqZWUZ5wjQsjQl00KMjQvwI/hV8jCjAAAAAAAAAAAAAAAAAZVoF94/wCIFx+UA1UAAAAAAAGVaBfeP+IFx+UA1UAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAQ+fwsOk5Xps/lFtLh2UPKH3scZZSZomWJ0tmhbLpkhWyChrmuEZmj42kFyPckLC4AAAAAAAAAAAAAAAAAAAAAAAAAdJlOYYng1O7kOa5NVUFWyezk2zmNxWEnsZkRrcMk77Efhv/ADAID2V3mpGguLPx3UOtOpmLQtCiUlSTmPGRkZfMjL+cB7dffu4/ECn/ADQHD1y9kzQD2iIbjWqOnVfNsFt9Nu4jJ7NYs7Fsk0yG9lKJPzJC+SP6UmW5ANIxHGq/C8UpcPqFPqg0VfGrIpvr5uGyw2ltBrVsW6uKS3PYtzAdyAAAAAAAAAAAAAAAAAAAAAh7aFhrmtGLWE23lt5YxjF+zWQUpPoP1y5dSc15Z8DIltut16UlzTuTznwr2M0BcAAAAAAAAAAAAAAAAAAAAAAAAh8/hYdJyvTZ/KLaXDsoeUPvY4yykzRMsTpbNC2XTJCtkFDXNcIzNHxtILke5IWFwAAAAAAAAAAAAAAAAAAAAAAIfW6Hhtjoxn1dqNbSqvFJWMWrN9OiJNT8WuVFcKS82RIWZrS0a1ERIX4kXwq+RhcAAAAAAAAAAAAAAAAAAAAAAAAhtIoeHQcVnx8FtpdjWqyjJXn3pKTStNi5dTV2DJEaEfA3MVJbQex7oQkyUv8AXUFyAAAAAAAAAAAAAAAAAAAAAh9boeG2OjGfV2o1tKq8UlYxas306Ik1Pxa5UVwpLzZEhZmtLRrUREhfiRfCr5GFwAAAAAAAAAAAAAAAAAyrQL7x/wAQLj8oBqoAAAAAAAMq0C+8f8QLj8oBqoAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAksxmnEyTBGPsL7+7bkDzHvDo8/s/tVT3O3b8FcOXT7Hy3b/j3HkfLgsK0AAAAAAAAAAAAAAAAAAAAB0+TZTjeE0UvKMwv6+lp4KUqkzp8hDDDRKWSU8lrMiI1KUlJF8zUoiLczIgE45qXJtImOWmn+D3WU1t+6s1z2VMwmYMdDiUrddKWtt1XLdRtpQ2rnwM90pMlGHv+z+otje3hXmdQWscmRHodbCqKpcWdFU4SdpC5a33CU6guZJJDaE+JKMjPZJB7cW0vxDEaxqsiRJln0rD3sUq7nv2krt3TJvtBPSluLSvgXEuJkSSMySREewCY9lz+QvGf+e/3x4B+6+/dx+IFP8AmgNVAAAAAAAAAAGWaj2mVWeoFFp7RZ1Kw6PMqJ9y7Zw40R195yM7HSlgu1tOtE3s8pTmyeexFxUjxMB2uk2fTs80ho8/djJsZ02uN5xqv4JTKfbNSFdHqLJBEtSD48lkn4i3Vt4gO1zfPa7AMOdzS/rLJUZhUZDkWM0h6SlbzqGkpJJK4qMlOFvxUfyPjy8NwmFa8VEc51ZZ4Tk9fkUSVFiMY8+mGqdOXJJxTCmVNyFRzSomXj5LeQSekvnx2AeEnX2njlXQk4TlLt7YW71EdE2zEOZHmNxjkmh1RyCYJJskSyWl1SDJSfiLx2CW0+1wyeS7eKyLEsxtLaxyWzgUlAymoSbcSE4bbimlk+hJJQXA3FSHtzWsib3IyIBXV2vOO3FxQ0NHjGR2E+8ZlvraaYYR7vTEkojSikqceSlJtOL2MkGvkST4c/DcOsrPaOrrxurXT6X5zLVfV71lSoS1AQdi2yaSeSg1yyJpSOaT3eNtKiP4FL3IjDNsw12x281IhIuPaoe0gxWxw6uvapt5+ghOTX335CXSUq0ivmpSEttEaWzIkmZ777kYDQcG1kt04hTt2tXcZje20yzRUlVxo0Z+1rYr5oRYKJ9xhhCFNqZUaiUlKzcI207KIiD02Gu1FMznGHMX0kvsskv43bTpljXRY/b8fYRMiMriuJeUjfrOtOmpppxTilVxcWnSLkgKaw1xoYFnJjpxrI5FTWS48G0vG4zKIddIeS2pDbyHHUyNyJ5rmaGVpRz+I08VcQ88V1Wx+wvnMadkXS3nZNybUuxZjoZI4Ekmn2UKa28E8yNHJPI2yM1KNRGA6svaJx2TWIuqzC8ssoLNWzeWb8ePGIquC9zU08+lb6VL5NtqdJDJOuEgiM0kZpIw7LTPMbjKc21Hr5lmmXW0txCYqUpaQkmo7tbGfMiUkiNW63Vq3UZn8W3yIiAfvfbRe+uxnjt6VKdv7gLIzRG93e8Or0eht1u0f8N/B8+j0+Xhy/nATGa67TpWm+U5Dh2M5NWtx6SdPosjkQozsCYplJ7OI4uOKbLfxT2hpsll+qSgFTJ1ox9iA+/CqLm4U3PapofYmWtrWxUlRrjxVOOISo2+CubijQ0k0qLnuhZJDjr10qOzxosfDcmkZHItHac8bbREKe1IbZJ9zmtUgoxISypDnMnzSZLSRGaj4gOXojl11m+Hzby9W+chOQXMNtD7CGnGWWJzzTTSkoLbdKEJTv477b7nvuYaEAAAAAAAAAAACSzGacTJMEY+wvv7tuQPMe8Ojz+z+1VPc7dvwVw5dPsfLdv+PceR8uCwrQAAAAAAAAAAAAAAAAAAAAABJarzTrdLcysvsMWa9koLF/7M9Hq+++MZw+w8ODnPrbdLj01789uKvkYVoAAAAAAAAAAAAAAAAAAAAAAAJLTWadhjkt/7CniRoyC9Y939Hpdbp2spvt23BG/a+PbOWx8u08uTm/NQVoAAAAAAAAAAAAAAAAAAAACS1XmnW6W5lZfYYs17JQWL/wBmej1fffGM4fYeHBzn1tulx6a9+e3FXyMK0AAAAAAAAAAAAAAAABlWgX3j/iBcflANVAAAAAAABlWgX3j/AIgXH5QDVQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABOZNCzOXdYo/jFtFh1sO4ceyJl5JGuZXHXy0IZaM0K2WUxcJwzI0fA0suR7mhYUYAAAAD5/wzDrvVK6zq7vdV89rzr8tn1USLU2yYsZmMwltLaUtpb+exnufzMzMz8TAVfcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgMx120y1Ew2mxmRpZr3ncHIbLKa+ujOXNkU6Co18zJD7BoI1tGpCSURGR8d9jAaZojrd3k+8sNzKj+y2pGK8Gsjx11zlw5eCJcVZ/8PDd23bdL/8ANVsojIBpFjcVtSy9JsZrTCI8d2W4RnuomWyI3FkkviMk7lvsR/Mv6SARBapW+U4p9o9JMEscidXY9gabukv4+ypBN8zlcpTPVVH3NKCW2y4ajPdJKSRqIO6Kmzqdk9PkEvMEVlXFgkU7HoURp9uTMUlZLNcx1BOKaSakGgm0MqNTRGozSo2yD9xTTLCcKl2tjj9IaJ14+3IspkqS9MkyltqUpvm8+tazShTizQnlxRyPiRAKsAAAGU+y5/IXjP8Az3++PAP3X37uPxAp/wA0BqoAAAAAAAAAAzrVzTCfqU1VNRZ+Lk3XSDfOLkeLM3cU17p4vtoU40tqQ3sfBwlmkuR8kL8NgoscxVOHYVDxHGpiW1VsPs0WTMa6xG6Sf+FdQhSOe6jNSiSpO+57Gn+YJXXamv73Sl+nrEvyLVydUHzgRDWpKkz46luoa+PZKSSpexmoiJJ7mZEZgJzJfZyfzxdhd59kNBdZC/LgSYa3ca5VTKIaX0tNOwXZDhvkfanzWfWSZmpJp4cSAdxiehkfGZGNTmZlBEdo7eXbPR6TG2auG+p6IuMTbbLazNBJJZK5OLdWZkZbkWxJDwTonaVM2NkeLZfCiZDCtrmfHlTqhUqN2eyeJ16O4yl9tauJob2Wl1Pijcy2M0gOdhGjSMOvq/IPtEubIj19nGm8opI7VKnTES3ny2WfTSS0qJLeytiUXxHtuYeWI6N/Zb7C/wD4x9q+xdLNqP4nw7X2jo/wn659Pj0f1fi35fMtvEOFp1oTEwaYb0+9RcRnMTg4q9Gcg9NLqI7shanD3WrwWUk08NvDj+se+xB0tz7NEe3qcbizLXH7ydiBTYVS5lGNJt4xVz6kGhh5lT6FOOtJaaSl8nEGZJPdJ8jAeotC9Q8Xy/H7bSrNsYxuC3jUukyA3MZbWUh0pbT8NyJFYUy2waOrYfEpa0lzb5NPGZrQHumezHSv5vMytssSlIt7CPaWT9thsSdbKkNttoV0JilEhlK+ikzI2F8TUs0GndPEObk3s+LvsTfx+JmjldNdyOwu0WDULdTcect0pMUk9Qv1mH3Gyc38FcF8T48TDjZx7NGP5NlkjKKyLhrR2ECJXTE3eHsXDrLcclJbVDcccSlhfBXEyWh5BmlB8PAyUHMi6U6q4tnGQ5Dp7qRh9fTZJLhy5Nba4bImPtGxFZjGlp9mxjoSk0MkZF0T4mZ/MvAB7UaHWCZRUv2ti/YwsiPKDqU1RlMOYck5XT7X1+PQ7QfPj0Oe3w89gHA7hstXgVtpKvUyGjDHqeXTVcJnHyTJYaeSaUdpfU+opBNJPZJNojmfhyNQD8u/ZloZ1EvFayTTtUEKzYu6OmsaNE6BXTUoWh5JsG4lLkZwnFK6JcDQtS1JWW5EkPZA0Bn0EKmsMSu8Vo8kpLOTYNPQMQbjVSkSGOi6ycJl9DhkaCSZLVIUslJLdRp+AgutNMIf0/xx+klXarZ+TZz7R2UccmN1ypLj6kkgjMiJJuGkvH5EQCuAAAAAAABwrJNmuvkpqHIzc421FGXJSpTSXNvhNaUmRqIj8TIjLf5bl8wHyBkntsY7prBx/S3UnUd2pzLJbWay3ksqsZRDjQG7mREU4fFPSQ6lphXHmg0EZJUs1eJKDe9E8yk5CrIaBedN5rEpJMU6/IknFNVhFkRkOpUtURCI6lJUbieTSEJMiT4b7mYd3kU7M3jxzIa1aqCpqLeZKyeJPS06/Lqm4M1tCWel1S5HKOE+Wy0K6baiUZGamlB0tZrzTTsVdzObheT1lUuNHk1r77cR5NqmQ4lthEdUeQ4knFrcbIkOm2r4yMyIiUaQ6bJNZMmi3uFMxcEy+resLqZXT6GTCiKlTCTXPPtE2+l1cY08koV1EyCSRkaVqSe6QHcRteKq0hVpUGEZTbXdh27qUUZENMyGUN8mJJvLdkoj7IdMkfA8rmZ7o5ERmQcLI9fKmRQPu4NQ5BdSnMdcvVuwo8cvdTJk4ltyQiQ6hRq6jThdNtLi92lfD8tw92Oa2VjWnc7I8gYspUrFqKtsLlxphojkLkQ23zNlJKSkz+LxI+BEfgXgA4lVrFYwcgyajmVVvk9n9p5EClqatuKh8obMOK64o1vOMtEhCnzM1OObmbiUluexAIbVLXKml5bg7Un2gpGkGMXNXcuzJUlymiPKsYkiOz2RxdnHfaStBqfI0t/M0bko0luYVWnusciJi86XOvLDUKG7kCabErWCxETJyRCoyHjUhSOhEXwUUhJvJ6TRpZM/AyPcKXv0qXShV9fheTTcilTpMB3Hmkw0zoi46ELeU6pchMckJQ6yrkl5RKJ1HHkZ7AOne9orCUR42VqVkrUFyrnSUQVxIzaHDZsGoZ8+Zktt03lpSnktLZJUo3OJkRkHJvdbMir5mIw4+kmUR3shyH3NKjzewEtlrsrr5OocRLNlwjJBHuha9iS4kyJfFJh2ULXGhnWbDRY1kMennzJNdXXzrUfsM+UwThrabJLxvp36LxJW40hCjR8Kj5J5Bwqf2gK/IaOit6LTnMJkjJ2FTaatJEBqVMhpbbW5KLqSktttpN5CD6q0KNR/ClRGkzDxste6iZWEnD8byO5nu1MizfRDjxyXUNtrWyapKXnUfETzTqem2TizNpeyTIiMwxpetKpmUUtdqZ7Wx6WRpGB0N0wjtGPQvec2T1+0r3sobpq26bfwtcSTy+XiQDVcI1nu/snSxrbHrvLsisjsnoaKqLGjOzauNJNpuwWUh1llsnEKYVsSi5G5uhHH5B3buvFBJagO4pieT5QcqtRcSmqmKyb1fEWtSCW82862tS+bbqek0Tjpm2oiQfhuHS6naxWjuMZbR6c19vBtWo7tHBy2RFY91VF7IZJMTtDbilSFJQ89H6ikxnUJ5GSt+KyIO8xHWDGLF9+skzLrdtm3sEzbVqM2lbMKc5HkISbJkWzRknbdJGbamzUalctg4f6Q1AqD74ZwzKnqyJBi2N1NSxFJFI1IbS6gpKVPk4pRNKS4pLCHTSkyM9tyIBS4rqXBzDK7/F6jHbom8bknCmWjyGEwzkdJl0m2z6puLM0PpPcm+JcVEZkexGFmAAAAAAAAAAAAAAAAAAACcwiFmUCmks53axbGyVcWzzD0VJJQmucsJC4DJkSEfG3DVGbWex7rQozUv9dQUYAAAAAAAAAAAAAAAAAAAACc1Cg5lY4Dktfp1axKvLJVPNZoZ0tJKYi2KmVlGecI0LI0JdNCjI0L8CP4VfIwowAAAAAAAAAAAAAAAAGVaBfeP+IFx+UA1UAAAAAAAGVaBfeP+IFx+UA1UAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAQ+fwsOk5Xps/lFtLh2UPKH3scZZSZomWJ0tmhbLpkhWyChrmuEZmj42kFyPckLC4AAAAAZVoF94/4gXH5QD5U1P8AadraC91uscE9pTQFcQ8QavoqISpSJWQSDRZNN17T8W+bIrFtiHEaVMjtpfUl6IXBKWY6SDXs51Zqse9lCVkeAZtQZxX4mxKi2RaURkwYjkKFDeknAjyG5chFIhEZtknJBurcS0SkxUokvxOAdXkWN6o6f5tgkUtL8ffkW1wxj1GVJrNk9DSkuDWSJTSlUTMRyHDjKYrFkcVJyEEpaULU8RrdML/NqDK9VtaMx0uf1TyDHsTgYRRSXqmtq6WUxYLsZdyxKKSVjAlGtCmoTKOBGlG3Pcj5GAoKDAMj0jxWyn2PtI5VcwKageYYk54iqkQa7pNkaZ0t6PGiSX+mlszcU7KLmk3DUolGTiQ/mbimpGQu4XjMJ3JdNslg0uMUHVxrUPUSVR1shacVfSzBdxuzWiK/GQ+mmkIkoSaH5KifbkoakOJrA+/8OrLDWT2bNLr3TO9w0nqN6stKlyG+qXUyGoRqZS0amYkI2VdMvjaREZJl1K2ekkkeAS+vml3tE5FAZ1Zg2OnWP5jgEaRZVN5WduKR0UINb0J4lkaXozpJ2W0sjLfZSeKi3AblppimGyY0PVyLhNDAyvLamJJtbOJBQiQ91Gm1m31T3X0+XjwNRlv8R7qMzMNAAAAAAAABjtX7NtXRQGqul1X1Jr4bJq6UaLeJaaRuo1HxQloiLczMz2L5mZgILEfZcxDP8cxTPK32mNcckqJbMHIaaTYZPul5K20usSOm5GSptSkLI/1UKLkZbJ+QDSu4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TASVloNhxapY83P1x1UTliqC5VWM+/TPnXFJre2q6nQMk8XTry4808uW/FXEzQFb3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YDr732fbyRTy2Ma171FgWi2lFEkzrdcuO05/MpxlBtKcSX86ScQZ//AHxAP5u+1B/o6vamzHVfGcRxu9c1BiqrJEt/IpUNdXArTfnPOuNLU9KfNaubhucUHzMlkfA9lKAa7o9/o+ss9lQ8TvLH2hcsK1ybJ66nsq3Fpz1dW9J1DiVcz35yVJ2I0KUlBEfzQZfMPrR/2XTuW66TlGtWo022pZz8+rmx7hTJQ3lsvx0OIQvqfGUeS42ozPZXNZ8UkZJIMr0g9meVB1A1Xxe1z7tcuIdJIjzYtX2dDk3mU1qXJZN5ZPuoWy2nfkjdBrLw5FxD6Cgac5tOuscyTPM/g29hQWUieluvo+wRODkNccmmm1PuuI8XDcNS3XTM9yLiWxEEddey3WWUiNcFJxWytYsy5dQeS4qi2h9CwmdqNBMG+2pLjatkpcJwiMuW6PiIkh2jmglnWqJOE5jW0iJ+Ot41cJPHWjS8w2p1SHYrbDjLUVwjkPfNDiPFPwbkZmHDsvZ4yB6ltsYqdRYsKqyGjr6m1S7Rm9IU7Ejkwl5lztCUtJWlKeTakOH4HxWkz3IGV+zNV5JcSclclY5YWarmVaRm8gxtNpAbakRozLjS45vINav+xULS4laDIzMtjLcjCixPRWNiuR4tfRbKA2jHKeyq1w4FMzBjvLmPsPKcbbZMkMpSbJkSOKjMlEalmojNQdNfezhWXOMpxV2xqpdfU5Au9xyHcUqZ8OuJaFpXEeZU6kpDG7zxoSRtmglISR/ARmHQ3WnMjSNnGckx51iDZwZU9uS/jWnbkqtSzKQ1zbOtgOlJSW8dng7zeMlJPqGaTLYPbpto3ljmI1F1KyBVPc+67eH0LGnbkc0TbMpaVSWTUlOym0EhxkuJ/wAIoiWg0kA7Ci9nazxyPXyKbJ8dq59fkzWQtxazGVxaZpJRlxnWWYJSzNpTjbq1G4Txl1TJZoMt0GHjivswUOJ5GVhWN4c3XR5MyXFcbw2MVyS3zcPi7YqWo1oQp1WxoabcMkoI3DIlcg9177NdVbYxglW4/jVlaYLU+547+R40m0gyWVNtJcUqIbyDSvdhCkqS78PxEfIjAe+PoLa0DsWRhGW09I5Io/cFwlONNEy/H6zrqVxWWHWURXEqfe2MydTsouSVmRqMO1wnRCBis2S5Y2zdzCl4jVYk/EdhEhLjUMn0qcUfNW5OE+ZGjbw4/NW/gGJe0lS6baD4FiGoGruRU9pHopC8SiWGQYX9oGWoUpfUZ68ftCHDcZRHSk30L3V8W7SjUREFnppV4tnVZAzT2Y9b8RUydLGoreTDoo8pJtNrcdaW1GYdYRBkEp90yS42tGyi5NHtuYdXrV7MWnLuOah5Pm1vjldiNrAnW2UWruHR5t/EYKIZS3Yk74ia+Bs1kRRXVpUaunsZpJIXF37OzdxidfjKcvciuxbuznyJbcLxkwLCQ85KgmnqfCS0PEjnuextpXx/4oD1Zh7NNBkeb2GXQo2HJK87N7x98YfHtZjZstpaScN91ZJY3bQkjS408ncuRJLxIw0LDMLRiEzJZTc8pKchuFWpNkx0yjkcdlnp+Bny26O++yf1ttvDcwqgAAAAAAAAAAAAAAAAAAAQ2kUPDoOKz4+C20uxrVZRkrz70lJpWmxcupq7BkiNCPgbmKktoPY90ISZKX+uoLkAAAAAAAAAAAAAAAAAAAABD63Q8NsdGM+rtRraVV4pKxi1Zvp0RJqfi1yorhSXmyJCzNaWjWoiJC/Ei+FXyMLgAAAAAAAAAAAAAAAABlWgX3j/AIgXH5QDVQAAAAAAAZVoF94/4gXH5QDVQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABJZjNOJkmCMfYX3923IHmPeHR5/Z/aqnudu34K4cun2Plu3/HuPI+XBYVoAAAADKtAvvH/ABAuPygHjk+hh5ZhOrVBZ5RyutVq+fUu2xweSauC5DXEhxmWlOGs2WELW8bZuklcmRMdSTJSDbQFVqxgvehpbmWmnvT3Z9raCxou29Drdl7VGcZ6vT5J58epy48k77bblvuA6rMdPMqy3UvBcq+1tVExvCrB6590+5XHJ0uc5Xz4O/be0k22yTc/l0+zKVya/wCE2VskO1pcFOt1GyfUiXadol5BX1dMzHbY6TcaDBOU62SjNSjceU/PmKU4XBPTNhBNkptbjod/bN2r1XNYop0SHZrYcTDkS4ypLDL5pPprcaS42pxBK2M0E4g1ERkSk77kHxux/o+Mwq5VA5Qa/wDZWcf90PMt9hvY3ORCxxijUvaBexW2+o2ybvNtCXy59JTzjPJCw+qdJ8F7r9LcN0096e8/slQV1F23odHtXZYzbPV6fJXDl0+XHkrbfbc9twH7qz/JZmX9n7H+7OAGk38lmG/2frv7s2ArAAAAAAAAAElpRNOy0tw2y+wxYV2ugrn/ALM9Hpe5OUZs+w8ODfDo79Lj00bcNuKfkQVoAAAAAAAAAAAAAAAAAAAAAAAJKxmmjVLHq08F7X16C5f+03R3929OTWl2HnwPj2rqdXj1E8vd+/FfHdsK0AAAAAAAAAAAAAAAAAAAAAAZVr793H4gU/5oDVQHzxD05+22v+qUos6y+g7M3RN9OjtOyod3iLPksuJ8j/m3/oAWXcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgPl//AEhnsr6i5roVAx3S+31Gzy7k5LBSmrnWyZLCG+m8RvLJSUpbJJmkuopRJTy8TIvEBhejH+in1V0xx6z1a1A1tssLuqeplzo0DC5i25iFJYNZIcmkZJR8SSJSW0rJRbbLLYB9141oLDz/AE0qlZTqnqNOi5JRMHYwZF71oz7ciOXVaWh1tXNtRLUk0rNW5GZHvuYD3ni9pp5rPp7VRNR81uYV7727XGuLdUho+hE5I2QRJL5r38SPxSky22AbPdvWzFZIeooTMuwJO0dl97pNKWZkRGtZEZkkt9z2Iz2I9iM9gGQTdXdRm/Z1iaqwaigfu0wXJk83CdbhtJbUolG21zNxRq4kkkm4W3I1Go+PFQbHKntQa96wktvqaYZU8tLDC3nDIi3MkNtkpa1f0JSRqM/AiMwHWZLmuNYfjTmXZNYKrqhomTckPMOkbfVWlCOSCTzT8S0ke5Fx38dtj2DoWdbdOHa2xtFWtjHKrktRJEOVSTo8/rO/8ChuG4ymQ6bmx8Om2rnsfHfYwHg/rnpnFrIFo9dTv/wnOdrI0QqacqcqY22bi46ohM9dDpISauCmyUZbGRHuW4S+De0FXW8TIr3K3ZkSGxksmhpq+Pi1oU57oKWk9kGhTktZk0txRMtJJpJGSy8DUAq2tbtM3n6WJGvZUmVkBPHBisVUx19XReQy+TjSWjWybTjiScJ0km38Rr4klRkHWR/aM0lmNtOQbm4lFJiKmxCj43ZuqnNIMicOKlMczkqQZlzS0S1I+ayTsYCfm6hanagajnjmjmaYVWUCMWrsibnW+My7VyX2t6QhJI6U+J00klgj2UlR7qPfbbYB3OI63Vp4VHus4WkrJdtYUiG6Gvlz/eLsR9xpb0WOwh182zJs1mREsmy3I1GRcjDvLDWrTitrKu3XdS5bFw06/ERXVMyc90mjJLrjjMdpbjKG1GSVqcSkkKPioyPwAJutOmldOYgu5Ep8n247xy4sCTJhR0PkRsKfltNqYjkslJNPVWjclEZeBkA5WOaiY7b38zFvtJFmWjUiebbLVe/GJDUZ1DbqDU4akuLbU4glKSZErkRkkiAdXI170qjx4ks8hlOxpcVNh2iPUTX2o0VS1JRIkrbZNMVpRoWaXHjQlRJUojNJGYDQW1ocQTjaiUlREaVEe5GX9JAPYAktNZp2GOS3/sKeJGjIL1j3f0el1unaym+3bcEb9r49s5bHy7Ty5Ob81BWgAAAAAAAAAAAAAAAAAAAAJLVeadbpbmVl9hizXslBYv8A2Z6PV998Yzh9h4cHOfW26XHpr357cVfIwrQAAAAAAAAAAAAAAAAGVaBfeP8AiBcflANVAAAAAAABlWgX3j/iBcflANVAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAE5k0LM5d1ij+MW0WHWw7hx7ImXkka5lcdfLQhlozQrZZTFwnDMjR8DSy5HuaFhRgAAAAMq0C+8f8QLj8oBqoAAAAAAAACT1Z/kszL+z9j/dnADSb+SzDf7P1392bAVgAAAAAAAACc09g5lXYDjVfqLaxLTLItPCZvp0RJJYlWKWUFJebIkIIkKdJaiIkI8DL4U/IgowAAAAAAAAAAAAAAAAAAAAAAATk2HmK8/p7CDaxW8UYqLNm0gqQXXfsVvQjhPIPgZkhtpuwSouadzeb+FexGgKMAAAAAAAAAAAAAAAAAAAAAAZVr793H4gU/5oDVQGVaefy7auf7KD+5rAaqAAAAAAAAAAAAAAAAAAACT1Z/kszL+z9j/dnADSb+SzDf7P1392bAS2of8ALtpH/sv/AO5oAX2SyMmi00h/EKmss7dCS7NFsrByDGcVuW5LfbYfUgttz3JpXiRFsW+5BgRafe0h+j4/pCWF6bHZPR3a7tP21n9AmFmpfW/+xHLmSlcenttsW/Pf4QG7U0rMlYq3JvqGmiZETCzVAiWzsmETpb8ElKVGbcNJ/DuroblueyVbeIQ/tFFZPaPSUJRFZnuWVKSUqNTrKHjsY3gZ7JNaCV/qSZl/MQCSy3Q7Pc+t5+eX5VNdetTqt6vrKvI57DDrMNEpHFywZZZkMrc7a74ttqJHBJfGRqAdxiei13U3OM5DKjVcORAyCZc2bKbyxtnFE5XriN7S5u7j7hbo3UaWkkktiTundQfrWkucUUiuyaiepZtxT5Te3DEGXNejxZMSxU58C3kMrU26hK0K3JpZbpUn5K5EHvwLSDJ8fzeHm95Mqlvvxr52wZircUlmVYS4zyW2TUgjW2hDBpNauClHsfAt9kh7cG0jyPGO7ft0ytX9j6Cwq53RccPqPP8AQ4Ka3QW6S6StzVxPxLwPx2CLx/2QcUt5sM9aMMwrLocHDIOORymQUzXIslt6St91hTzW7RKS81stJkrdHiRbEYDsp+hGbKocJQmTAn2WCNzqmKzFyOwx9udXO9NLTi5EFvqNPJQy0amyQttSuXiW5GkOZUaT6g6fy6bJdP6zF5dqmsmVlnX2V5YojIORMKUchuU43JffWlZrJROJR1TPfk1+qA66+9n/ACu1ye9nyjr7eLmK4ki4WvKrqsjR3URmmHkorYyzalNqJrkROPNqLlxUpREQDmZJodnUikmqxO7q6+/cye0nxpS3XSQ3WWCTafQZk2auqlCidSnbibjLZGoiPkQdPqZoijG3shyusVERirmORolkiZltzUsVkSCw4hThxa/4ZrfRPc21G0fwGRL2X8Ia/pdqFpzqPiEG60vzCnyKmaYbZQ/WyieS3skiJCy5GttZEXihfxl8j8dwFkAmsLi5lW0UtGd2sezsfe9tIYdiIIkprlz5DkBnYkI3W3DVGaV4GZrQo+SzPmoJ3FdUrnJarMZ56f2DE/FbBcFmqRLaclTT7Ky+2R/qttLUT6UmnmpKTIzNR/zB78Ez3KbvJbbC87xCBQ3lZDjWSUVluuyiuxH1uobPqrjsKS6SmV8kdMyLw4rWW+wWUG1qrN6YxXWkSW7XP9lmIYeS4qO9xSvpuERmaF8VoVxPY9lJP5GQCIk6tMxtW4+mZ0a1w3WktuXBSPgasVtLfbhm3x+ZsNLc5ci2+AuJ8twHfStSdO4K7Nqbn+OR10rSn7JLtqwg4TaV8FLeI1fwaSX8JmrYiPw+YD1SNU9MYtGrJ5eo+LsUyHG2lWC7iOmMS3EkpCTdNfDdSVEZFvuZGRl8wHfV9tVWtYzdVdnFmV8hon2Zcd5LjLjZluS0rSZpNO3juR7AOlqNTNN8hiPz6DUDG7KNFbdffeh2zDzbTbZkTi1KQsySlJqTyM/AuRb/ADAdNc686QU1BXZS7qLjsqotLVmnjTottGcYVJcURcep1OPwkfJWx7kkjPYBJK1n1cnSMut8c0mxafi+IWk2vfkvZk+xZSUxUEpxbUX3cpncyP4UqkkR/wA6kgNLq8+xC2oPtK1kEGPCbhsT5SpElts4bLzROtm/uf8ABboUR/Ft4APCRqPp9EvmMXlZ7jrNzKkHFYrnLRhMp18kko20tGrmpZJUk+JFvsZH/OAQtRtPrC+bxaBnmOybp5Lqm61q0YXKWlpakOGTRK5mSFJUlXh4GkyPYyMB19nq1grELJ00GV0N5cYrXyZ06oh2rK5LXRQozQ6hBqW1uaeO6k+B/wA38wDppuvWGd3t3mePW1LeWNBVosrClhW7Tj8VSkEZNPGjkpo99y3UgvkfgArX8+wiLkkbDZuY0cfIJiCcYqHbFlM11JkZkaGTVzUWyVeJF/Mf9AD81Cg5lY4Dktfp1axKvLJVPNZoZ0tJKYi2KmVlGecI0LI0JdNCjI0L8CP4VfIwowAAAAAAAAAAAAAAAAGVaBfeP+IFx+UA1UAAAAAAAGVaBfeP+IFx+UA1UAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAQ+fwsOk5Xps/lFtLh2UPKH3scZZSZomWJ0tmhbLpkhWyChrmuEZmj42kFyPckLC4AAAAAYXo3nuDY9N1Hrr/M6Ktlpz22WbEyxZZcJJ9LY+K1EexgNJ72NLf6y8V/fMb/ABgHexpb/WXiv75jf4wDvY0t/rLxX98xv8YB3saW/wBZeK/vmN/jAO9jS3+svFf3zG/xgHexpb/WXiv75jf4wDvY0t/rLxX98xv8YCV1S1S0yf0yy9iPqLjDrrlDYIQhFvHNSlHHWREREvxMzAVOk38lmG/2frv7s2ArAAAAAAAAAEPojDw2u0YwGu05tpVpikXGKpmhnS0ml+VXJitlGecI0IMlqaJCjI0I8TP4U/IguAAAAAAAAAAAAAAAAAAAAAAAAQ9tCw1zWjFrCbby28sYxi/ZrIKUn0H65cupOa8s+BkS23W69KS5p3J5z4V7GaAuAAAAAAAAAAAAAAAAAAAAAABlHtBM2vurDrSqx+1uPc+X19nJjVkRUh/oNJdUsyQn/cW5mRbmRGZbgOvtPacpqedUV1npNqdHk3sxVfXtrx3xkyEx3pCm0/wnzJmO+v8A2NqAezRyRa3upOo2Yy8RyCih3PufsjdxAVFdX0Y7ja9iPcj2Mv5jPwUnfbfYBsYAAAAAAAAAAAAAAAAAAAJPVn+SzMv7P2P92cANJv5LMN/s/Xf3ZsBnWuNVklxq3pNCxXLPs9PNy8Umd2FuXskoaeSem4ZJ8f6f5gHed3muv/KJ/wCiMP8AxAHd5rr/AMon/ojD/wAQB3ea6/8AKJ/6Iw/8QB3ea6/8on/ojD/xAHd5rr/yif8AojD/AMQB3ea6/wDKJ/6Iw/8AEAd3muv/ACif+iMP/EAd3muv/KJ/6Iw/8QB3ea6/8on/AKIw/wDEAd3muv8Ayif+iMP/ABAHd5rr/wAon/ojD/xAHd5rr/yif+iMP/EAd3muv/KJ/wCiMP8AxAHd5rr/AMon/ojD/wAQCJ1s0/1tb0az1yVryuey3jFqpyK3icVKn0lEc3bI0qNRGovDcvHx8AH8q/ZL9iL23MruIOeaZLt9LoTiUrbyGxmPVpuNGaVfwbSC6zyFJ8S+Dpr22NWx7gP6iaIUPtK3+O3FdlPtLrn2WN302ifmniEBJSjYNOzhJTx4lssi2PkZ8TMz8diDwyfCNXfZ70tvL/H9bX7KPFs5twuEvHISFvSbKzXIf2cMzJJdaW4pJGWxFsncvmA73Cch1gqlaoXLXs7ZdBsLeY5eUbFlaUXCW6mFGYRGUqPYucHDWyo91bI4/NZGewCo0MLI227JeW6ZZpR3k3pybO6yOTUuHZv+JcGUQJ0npNtl4IbMkJSk/A1KNZmGlQp8mU5Nbfp5kRMV7pNLeUyZSk8Eq6jfBajJO6jTs4SFbpP4dtjMPnd/QXVKdQWOoB6h5TFyuXd/a9vEElTHXduaWXZ4qpBxDkbdBttlSkyiT4q2MknsA591hOQUGJ2N1LxuKmU7qY3k3u16dEactGDko6TSXHHEtdY/gNtC1p3W2kjMtwE/XU13kl7neQ1unuTRLOJnCJkf7PWlY3a1j6qaM0pzjJWcF9RktaXEKWtJcty5mXgGgSaXOZug9zg+SVrNTa2WP2/Us1KhxokVS1L6RSuivih5aHCW6tlBskpLpkovhIwy/LMdyrVbKF1lJiMjE7FrCKx1DPbK956S1HtmHibSptT8cm1pacS0bpmSj5ckJSR7hTnp9qG43IzheM5lZWp5HRT3oN1Pok2EqPCWvmpCIRNREKJLvga31KWlBEfHilJhw7H2are6rNQ79S8jj5BZ5TNt62sVmFg3T2sTds0x5MBqT2TpvpSttfNrl8W6vkA7LOsR1JnxNQqag0xmOt55T15w3EWEBpitdajE05GfI3iVzLiXE2krbPciNSS8QHZWeleUuVObph4412281Bp71lRPMkqRDjuVxrdNXLw4pjvbJUZK+HwI9y3BRaXZbVUWJR2ceYYl1+odvkM0uszsmM+5YG2+o0q+Lkl9jci3WRK2Mi2PYIQtOdbpyamTYYdeKVGpL2lk1qXKCLXw5E2Nsl2CiMaXOydRG38M6bvxJM2zPkog9mY4bls3FK6ussKYw5yrwd3DIbc63r22rayknF6UaIpD57pLszhl1SbUZqLZJ+Owd5O0i1AczS+q5acvm0uQ5VFyFEiDLo2K1lDfZ1J663mF2BOtHH2STRKSoibIloI1GkNc1uh4bY6MZ9XajW0qrxSVjFqzfToiTU/FrlRXCkvNkSFma0tGtRESF+JF8KvkYXAAAAAAAAAAAAAAAAADKtAvvH/EC4/KAaqAAAAAAADKtAvvH/EC4/KAaqAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAJLMZpxMkwRj7C+/u25A8x7w6PP7P7VU9zt2/BXDl0+x8t2/49x5Hy4LCtAAAAATlpp9gd3Ods7vCMfsJz5p6kmVWMuur2SSS5LUkzPYiIi3P5ERAOP3T6W/1aYr+5o3+AA7p9Lf6tMV/c0b/AAAHdPpb/Vpiv7mjf4ADun0t/q0xX9zRv8AB3T6W/wBWmK/uaN/gAO6fS3+rTFf3NG/wAHdPpb/Vpiv7mjf4ADun0t/q0xX9zRv8ACrAAAAAAAAAAElpRNOy0tw2y+wxYV2ugrn/ALM9Hpe5OUZs+w8ODfDo79Lj00bcNuKfkQVoAAAAAAAAAAAAAAAAAAAAAAAJKxmmjVLHq08F7X16C5f+03R3929OTWl2HnwPj2rqdXj1E8vd+/FfHdsK0AAAAAAAAAAAAAAAAAAAAAAAElmM04mSYIx9hff3bcgeY94dHn9n9qqe527fgrhy6fY+W7f8e48j5cFhWgAAAAAAAAAAAAAAAAAAAAJfU2JLn6b5ZBgxnpMiVRz2WWWUGtbi1R1klKUl4mZmZERF4mZgMwxTWt7DsCpqm00Z1Vedo6ePGknExhx7mpllKV9NKVc17mk9iJPI/AiLc9gA8otNQ9Z9PbWJpxmtNCove3a5NxUKjtF14nFGyyNRfNG3iZeKkkW+4DdwAAAAAAAAAAAAAAAAAAAAGVaBfeP+IFx+UA9HtW9p7gMs7B0u09OJ0etvw59rZ48tvHbfbfbx2Ae//tpvKr/3iAf9tN5Vf+8QD/tpvKr/AN4gH/bTeVX/ALxAcG4pfaNyOrk0mQVOj9nXTGzakw5jE95h5B/NK0LI0qL/AFGQD04/jPtBYjVNUWKUGjVNWsbm1Dr4s6Mw2ZnufFtsiSW5+J7EA7JaPajcSaFp0pUlRbGRlYmRl/QA6LFsD1swdElvCsL0Ox9E53rSk1VdLiE+5/8AfrJtKeSv9Z7mA7//ALabyq/94gH/AG03lV/7xAP+2m8qv/eIB/203lV/7xAP+2m8qv8A3iAf9tN5Vf8AvEB8Z/6Vvvr/AEXo3eD9h/dn2pg8Pc/a+0dboyNv+F+Hjx5b/wA//tAYr7Hd1/pQcbxZ3I8Sbfc0/qoDs3hqISzgqYaa5pKMSjKWaeKT49Ayb335GXgA/ovXr9onUHB47lnWaSWFRk1UhUiFPjTltSI0hkuTTzSjWlSVIWaVIM1EZGZbmQDsoObaz0epGKYdqAxhS4eT9v4uUyJZuo7NHNw9zdURFuZo/mPw5fLwMBsQAAAAAAAAAAAAAAAMq0C+8f8AEC4/KAaqAAAAAAADKtAvvH/EC4/KAaqAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAJzJoWZy7rFH8YtosOth3Dj2RMvJI1zK46+WhDLRmhWyymLhOGZGj4GllyPc0LCjAAAAAAAAAAAAAAAAAAAAAAAAAAABOaewcyrsBxqv1FtYlplkWnhM306IkksSrFLKCkvNkSEESFOktRESEeBl8KfkQUYAAAAAAAAAAAAAAAAAAAAAAAJybDzFef09hBtYreKMVFmzaQVILrv2K3oRwnkHwMyQ203YJUXNO5vN/CvYjQFGAAAAAAAAAAAAAAAAAAAAAAACcyaFmcu6xR/GLaLDrYdw49kTLySNcyuOvloQy0ZoVsspi4ThmRo+BpZcj3NCwowAAAAAAAAAAAAAAAAAAAAABOahQcyscByWv06tYlXlkqnms0M6WklMRbFTKyjPOEaFkaEumhRkaF+BH8KvkYUYAAAAAAAAAAAAAAAAAAAAAAyrQL7x/wAQLj8oB+e1H/IXk3/Mv74yA1YAAAAAAAAAAAAAAAAAAddaUVJeojN3lNBsEQ5Lc2MmXHQ8TMhH6jqCUR8Vp3PZReJb+BgOk1Z/kszL+z9j/dnADSb+SzDf7P1392bAS2of8u2kf+y//uaAGqgAAAAAAAAAAAAAAAyrQL7x/wAQLj8oBqoAAAAAAAMq0C+8f8QLj8oBqoAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAh8/hYdJyvTZ/KLaXDsoeUPvY4yykzRMsTpbNC2XTJCtkFDXNcIzNHxtILke5IWFwAAAAAAAAAAAAAAAAAAAAAAAAAAACH0Rh4bXaMYDXac20q0xSLjFUzQzpaTS/KrkxWyjPOEaEGS1NEhRkaEeJn8KfkQXAAAAAAAAAAAAAAAAAAAAAAAAIe2hYa5rRi1hNt5beWMYxfs1kFKT6D9cuXUnNeWfAyJbbrdelJc07k858K9jNAXAAAAAAAAAAAAAAAAAAAAAAAAIfP4WHScr02fyi2lw7KHlD72OMspM0TLE6WzQtl0yQrZBQ1zXCMzR8bSC5HuSFhcAAAAAAAAAAAAAAAAAAAAAACH1uh4bY6MZ9XajW0qrxSVjFqzfToiTU/FrlRXCkvNkSFma0tGtRESF+JF8KvkYXAAAAAAAAAAAAAAAAAAAAAAA+c9ONZdN9O7TUGlzHJfd85/N7aU212OQ7yaNSEkrdtCi+aFFtvv4f7AHX64e0Lo9qDpJfUGG5m1ZzpTjLLTTUSQklLZmN9VPNTZJI09NZGRmRkaTL5+AD6cAAAAAAAAAAAAAAAAAAABJ6s/yWZl/Z+x/uzgBpN/JZhv8AZ+u/uzYCW1D/AJdtI/8AZf8A9zQA1UAAAAAAAAAAAAAAAGVaBfeP+IFx+UA1UAAAAAAAGVaBfeP+IFx+UA1UAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAASWYzTiZJgjH2F9/dtyB5j3h0ef2f2qp7nbt+CuHLp9j5bt/wAe48j5cFhWgAAAAAAAAAAAAAAAAAAAAAAAAAAAktKJp2WluG2X2GLCu10Fc/8AZno9L3JyjNn2Hhwb4dHfpcemjbhtxT8iCtAAAAAAAAAAAAAAAAAAAAAAABJWM00apY9Wngva+vQXL/2m6O/u3pya0uw8+B8e1dTq8eonl7v34r47thWgAAAAAAAAAAAAAAAAAAAAAAAksxmnEyTBGPsL7+7bkDzHvDo8/s/tVT3O3b8FcOXT7Hy3b/j3HkfLgsK0AAAAAAAAAAAAAAAAAAAAAASWq8063S3MrL7DFmvZKCxf+zPR6vvvjGcPsPDg5z623S49Ne/Pbir5GFaAAAAAAAAAAAAAAAAAAAAAAACS01mnYY5Lf+wp4kaMgvWPd/R6XW6drKb7dtwRv2vj2zlsfLtPLk5vzUFaAAAAAAAAAAAAAAAAAAACT1Z/kszL+z9j/dnADSb+SzDf7P1392bAS2of8u2kf+y//uaAGqgAAAAAAAAAAAAAAAyrQL7x/wAQLj8oBqoAAAAAAAMq0C+8f8QLj8oBqoAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAnMmhZnLusUfxi2iw62HcOPZEy8kjXMrjr5aEMtGaFbLKYuE4ZkaPgaWXI9zQsKMAAAAAAAAAAAAAAAAAAAAAAAAAAAE5p7BzKuwHGq/UW1iWmWRaeEzfToiSSxKsUsoKS82RIQRIU6S1ERIR4GXwp+RBRgAAAAAAAAAAAAAAAAAAAAAAAnJsPMV5/T2EG1it4oxUWbNpBUguu/YrehHCeQfAzJDbTdglRc07m838K9iNAUYAAAAAAAAAAAAAAAAAAAAAAAJzJoWZy7rFH8YtosOth3Dj2RMvJI1zK46+WhDLRmhWyymLhOGZGj4GllyPc0LCjAAAAAAAAAAAAAAAAAAAAAAE5qFBzKxwHJa/Tq1iVeWSqeazQzpaSUxFsVMrKM84RoWRoS6aFGRoX4Efwq+RhRgAAAAAAAAAAAAAAAAAAAAAAAnMIhZlAppLOd2sWxslXFs8w9FSSUJrnLCQuAyZEhHxtw1Rm1nse60KM1L/AF1BRgAAAAAAAAAAAAAAAAAAAk9Wf5LMy/s/Y/3ZwA0m/ksw3+z9d/dmwGM6z2FXnftGaXaaR7bK6aXHavJjlnVLOIW6Yje7SHjI+Z/GjlxSafHbluRkQeiHUwJOVw8cl5pr5WQraa9W1V5NvI7cKfLaS6pbTaCUclHwsOmS3WEIUSd0qVuW4aJ3BedWqvqP6YB3BedWqvqP6YDi6OIvKPUDUbA52Y3uQV1I9VPQXLmSmRIZORFNTqSc4kZpNSSMkn4F47fM9w/JHtCvIsciTA0V1Bs6bFrCRXWV9DOoOIhbCSU8tLK56ZbiUkf/ABY5qP8A4qTAarW2MK4rottWyEvxJrCJMd1PycbWklJUX+oyMjAcsBxpT70aM/IZiOyltNqWhho0Et0yLckJNakpIz+RclEW5+JkXiA8o7i3mW3XIzjCnEJUppw0mpszLxSfEzTuXyPYzL+gzAe8AAAGVaBfeP8AiBcflANVAAAAAAABlWgX3j/iBcflANVAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAEPn8LDpOV6bP5RbS4dlDyh97HGWUmaJlidLZoWy6ZIVsgoa5rhGZo+NpBcj3JCwuAAAAAAAAAAAAAAAAAAAAAAAAAAAAQ+iMPDa7RjAa7Tm2lWmKRcYqmaGdLSaX5VcmK2UZ5wjQgyWpokKMjQjxM/hT8iC4AAAAAAAAAAAAAAAAAAAAAAABD20LDXNaMWsJtvLbyxjGL9msgpSfQfrly6k5ryz4GRLbdbr0pLmncnnPhXsZoC4AAAAAAAAAAAAAAAAAAAAAAABD5/Cw6Tlemz+UW0uHZQ8ofexxllJmiZYnS2aFsumSFbIKGua4RmaPjaQXI9yQsLgAAAAAAAAAAAAAAAAAAAAAAQ+t0PDbHRjPq7Ua2lVeKSsYtWb6dESan4tcqK4Ul5siQszWlo1qIiQvxIvhV8jC4AAAAAAAAAAAAAAAAAAAAAAABDaRQ8Og4rPj4LbS7GtVlGSvPvSUmlabFy6mrsGSI0I+BuYqS2g9j3QhJkpf66guQAAAAAAAAAAAAAAAAAAASerP8lmZf2fsf7s4AaTfyWYb/Z+u/uzYDBdTs7qf00dJIpUuXKTWQr+skSU4lanFN99lg2+nIKP0nUbEZqcbWpCCIzWpICqnzHcq1hxm0psMzmBktVZrYt0W0OauljViWpCDejuukcAnXN29lxFdcyWSXPhJSSDY3m6H7URXHac13JQXiYn+7Vq6cbm31Gu1cOCOSumfSNZGrhyJJkgzIMJ1jxaNOy7NH8r0+tclm2NFGYwWXEpH53u+YSHiWTb7aVJgO9dTThvLU0RlwPmfTPiGZU+m9hO1H1mu7TFF2WYVWS4S/BsEwjelMfwMRMl6MvjyQlSUu81I2I0krl4EYCyl6V6sv02qF5SZ1nMGPJy+xlHiLESuZjXNcfS6yWH3IZzEreaJwkOIkEXPbbYtwHD1Swa0t8smOpx9xNLLx6tjYdz0/sLmbVKShZLRGdbkspq5CVm0rm90/kjdezZkkKmgx2sg6h2Sc909yG6z168Q/S5LHopHFqvJhlLZnZIMmWGUGTnUjG+RrPns25z+IM+w/TzNGqmQ3YVbrWWR8euGcicgYBNhP2r7kR1HCVaOyVNWPJ00KQbKHD5EnYm07kAufsPlDNzV6fMY5ZfZzPGKW1upBRl9GE5BZQUxl9Rlsg30MQ2yQrY1bu+B7GA6XAMEy6PqfBk38Uo+TxsnsJs60i4HMRJk16nXzbbeu1yijPR1MqaSTKUqUnZsiaI2zNIfV4AAyrQL7x/xAuPygGqgAAAAAAAyrQL7x/xAuPygGqgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACSzGacTJMEY+wvv7tuQPMe8Ojz+z+1VPc7dvwVw5dPsfLdv+PceR8uCwrQAAAAAAAAAAAAAAAAAAAAAAAAAAASWlE07LS3DbL7DFhXa6Cuf+zPR6XuTlGbPsPDg3w6O/S49NG3Dbin5EFaAAAAAAAAAAAAAAAAAAAAAAACSsZpo1Sx6tPBe19eguX/tN0d/dvTk1pdh58D49q6nV49RPL3fvxXx3bCtAAAAAAAAAAAAAAAAAAAAAAABJZjNOJkmCMfYX3923IHmPeHR5/Z/aqnudu34K4cun2Plu3/HuPI+XBYVoAAAAAAAAAAAAAAAAAAAAAAktV5p1uluZWX2GLNeyUFi/wDZno9X33xjOH2Hhwc59bbpcemvfntxV8jCtAAAAAAAAAAAAAAAAAAAAAAABJaazTsMclv/AGFPEjRkF6x7v6PS63TtZTfbtuCN+18e2ctj5dp5cnN+agrQAAAAAAAAAAAAAAAAAAASerP8lmZf2fsf7s4AnqTM6rT3QHHMxvI0t+FX0FUbjcVCVOn1G2Wy4kpSS+ay33MvDf8A2AM7y7U65udUcByuFonqecDHfevblKoNlF2iOltvinqfF8RHv/QAndfvb+rdDaV2UXs+6oWtimGucTL9aiHFYZSskdSS9ycWw2alERL6SkmZGW+5GA+FMQ/0mntH62e0pp5VypDtFiSsmhk/jeLxd35zXV2NpxazN141JPiaOSUKPx4ke2wf1J7/AHyV1V9OfUAcXRxd5eagajZ5Ow69x+uu3qpmC3cxkx5Dxx4ppdUTfIzJJKUREo/A/Hb5HsGvgAAAAAAAAADKtAvvH/EC4/KAaqAAAAAAADKtAvvH/EC4/KAaqAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAJzJoWZy7rFH8YtosOth3Dj2RMvJI1zK46+WhDLRmhWyymLhOGZGj4GllyPc0LCjAAAAAfP+GYdd6pXWdXd7qvntedfls+qiRam2TFjMxmEtpbSltLfz2M9z+ZmZmfiYCr7gvOrVX1H9MA7gvOrVX1H9MA7gvOrVX1H9MA7gvOrVX1H9MA7gvOrVX1H9MA7gvOrVX1H9MBmOu2mWomG02MyNLNe87g5DZZTX10Zy5sinQVGvmZIfYNBGto1ISSiIyPjvsYDTNEdbu8n3lhuZUf2W1IxXg1keOuucuHLwRLirP/AIeG7tu26X/5qtlEZANXAAAAAAAAAAE5p7BzKuwHGq/UW1iWmWRaeEzfToiSSxKsUsoKS82RIQRIU6S1ERIR4GXwp+RBRgAAAAAAAAAAAAAAAAAAAAAAAnJsPMV5/T2EG1it4oxUWbNpBUguu/YrehHCeQfAzJDbTdglRc07m838K9iNAUYAAAAAAAAAAAAAAAAAAAAAAAJzJoWZy7rFH8YtosOth3Dj2RMvJI1zK46+WhDLRmhWyymLhOGZGj4GllyPc0LCjAAAAAAAAAAAAAAAAAAAAAAE5qFBzKxwHJa/Tq1iVeWSqeazQzpaSUxFsVMrKM84RoWRoS6aFGRoX4Efwq+RhRgAAAAAAAAAAAAAAAAAAAAAAAnMIhZlAppLOd2sWxslXFs8w9FSSUJrnLCQuAyZEhHxtw1Rm1nse60KM1L/AF1BRgAAAAAAAAAAAAAAAAAAAk9Wf5LMy/s/Y/3ZwBlepf8A3HET+z9F/wDEigPoABlJkR+1LsZbken/AP8AUQE9kPsU+zjeanY9rFE08i0GWY5aM2zE6kPsSZLrauZE+0guk4Rq8VK4ks9v1tjMjDdwAAAAAAAAAAAAGVaBfeP+IFx+UA1UAAAAAAAGVaBfeP8AiBcflANVAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAEPn8LDpOV6bP5RbS4dlDyh97HGWUmaJlidLZoWy6ZIVsgoa5rhGZo+NpBcj3JCwuAAAAAGVaBfeP8AiBcflAPhXL8ztciy/CMoj0uoGWV93n91e0tu1lGpdc3YVcyFcvwGo0ZmoNqJ04jzKSVXm8tTbCi26Dj7iA3SHqteYr7IGRWa7DNLe0w64vvtFWxvfZW7cMkybZqtO1uGY0uGhuufhJXYutdTopUiL/2U7GUkO1yLG9UdP82wSKWl+PvyLa4Yx6jKk1myehpSXBrJEppSqJmI5DhxlMViyOKk5CCUtKFqeI1umF/m1Bleq2tGY6XP6p5Bj2JwMIopL1TW1dLKYsF2Mu5YlFJKxgSjWhTUJlHAjSjbnuR8jAUFBgGR6R4rZT7H2kcquYFNQPMMSc8RVSINd0myNM6W9HjRJL/TS2ZuKdlFzSbhqUSjJxIfzNxTUjIXcLxmE7kum2SwaXGKDq41qHqJKo62QtOKvpZgu43ZrRFfjIfTTSESUJND8lRPtyUNSHE1gff+HVlhrJ7Nml17pne4aT1G9WWlS5DfVLqZDUI1MpaNTMSEbKumXxtIiMky6lbPSSSPAJfXzS72icigM6swbHTrH8xwCNIsqm8rO3FI6KEGt6E8SyNL0Z0k7LaWRlvspPFRbgPpTB7uTkuGUORzm2kSbSrizXkMkZISt1pK1EkjMzIt1HtuZnt/OYDvgAAAAAAAAEPojDw2u0YwGu05tpVpikXGKpmhnS0ml+VXJitlGecI0IMlqaJCjI0I8TP4U/IguAAAAAAAAAAAAAAAAAAAAAAAAQ9tCw1zWjFrCbby28sYxi/ZrIKUn0H65cupOa8s+BkS23W69KS5p3J5z4V7GaAuAAAAAAAAAAAAAAAAAAAAAAAAQ+fwsOk5Xps/lFtLh2UPKH3scZZSZomWJ0tmhbLpkhWyChrmuEZmj42kFyPckLC4AAAAAAAAAAAAAAAAAAAAAAEPrdDw2x0Yz6u1GtpVXikrGLVm+nREmp+LXKiuFJebIkLM1paNaiIkL8SL4VfIwuAAAAAAAAAAAAAAAAAAAAAAAAQ2kUPDoOKz4+C20uxrVZRkrz70lJpWmxcupq7BkiNCPgbmKktoPY90ISZKX+uoLkAAAAAAAAAAAAAAAAAAAEnqz/JZmX9n7H+7OAMr1L/7jiJ/Z+i/+JFAfQADKv8Awpv/AEf/AP1EBqoAAAAAAAAAAAAAAyrQL7x/xAuPygGqgAAAAAAAyrQL7x/xAuPygGqgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACSzGacTJMEY+wvv7tuQPMe8Ojz+z+1VPc7dvwVw5dPsfLdv+PceR8uCwrQAAAAGVaBfeP8AiBcflAIhj2IdOK6Bp1T0+a6ktQcAkNqJL2e3qlymG6uTBQhomprbUFe8hCzXGbbLg2tlKUtOqSQVUr2bcZi6UaqaX4xkuQMN6px7BMufd2Mi6fhvyapmu5pdkuG+8hLcdtZIceUe/JKVJRwSgKDMdPMqy3UvBcq+1tVExvCrB6590+5XHJ0uc5Xz4O/be0k22yTc/l0+zKVya/4TZWyQ7WlwU63UbJ9SJdp2iXkFfV0zMdtjpNxoME5TrZKM1KNx5T8+YpThcE9M2EE2Sm1uOh39s3avVc1iinRIdmthxMORLjKksMvmk+mtxpLjanEErYzQTiDURGRKTvuQfG7H+j4zCrlUDlBr/wBlZx/3Q8y32G9jc5ELHGKNS9oF7Fbb6jbJu820JfLn0lPOM8kLD6p0nwXuv0tw3TT3p7z+yVBXUXbeh0e1dljNs9Xp8lcOXT5ceStt9tz23AfurP8AJZmX9n7H+7OAJzRzUDDp+O45gEW8acyCtxmulSoJIWSmmjisHz5GXEy/hW/kZ7Goi+YDuqzWbSK6xObnlXqhisnGq2QUObbot4/Y4z58CJp17lwQs+q18KjI/wCER4fEW4d19scR4VLv2qqOF+hLlSrtzW1ghRJNKmD5fwpGTiDI0b780/0kA7JudDdkuRG5bKn2tjW0SyNaNyIy3T8y8DL/ANYDkgAAAAJLSiadlpbhtl9hiwrtdBXP/Zno9L3JyjNn2Hhwb4dHfpcemjbhtxT8iCtAAAAAAAAAAGZ4ffZmrV3NMbye+jSa2DV1dhAjMxkNNwyedmJUXP8AXWZpYbNRqVtuR8UpI9gHS6daj5bmmsNshU5tOGS8eanUEQo6CW4lMtxlUxTm3Iye47oTvxJskK23UYDV2Zsh2zkwV1MtphhptxExamjZfUo1ckIJKzcJSOJcuSEl8aeJq+LYM7n6/wBDAvLatXh2UuV2P3bFBbXaY8YoMOS8llTalcnyeW2faWiNbbSySZny4kRmA5kDWujsbuPBRjeQNVE6yepoeQONR/d8mc0a0qZSRPHIT8bTiUrWyltSk7Eo+SdwhbLXfIsystNl4RimZVFFlNy4UixU1UpVIisxXHVJJL761IbNaDJaibJZpbX0z3NBqCrje0NjTkZu2l4tkkKmnw5c2mtH2ovZ7hEdlb6yYSh9TqFG02taSfQ1yIvAB6l+0NBNquOHpjm8qTbVEi/hxG2YCXXK9lTJKfM3JSUI3J9BpQtSVmRK+HfYjCV1P9opDmleVW+KN32JzEYsWSUl7YR4ZNPwlLbSqU0hxbvDp9VBmmS0jbkR8TLcB6NONTa+VnVRW4P7TvfHUSWZbl/ydpZSaVptk1tvm9VRmEtcnCJHB7kaue6duJgO8Y16xmVnFHbXGmNpUUsnEru5rsytWoxEqGyuA4uPHS0px5CX2zJ80O9FRlDR/BrMt2wpe+2LHp3rO706zOneU5EaroUuJGU7arkr4MIjrZfWyS1K+aHXG1IL4lkhPxAOvstcalpcTtlHmNZaw7KTCl4+iPBckrdbrnZiWnTJxbakLaQS0KYd8V8EqURc0kHaW+uWE1LEKShNhMZn0rN4wuKyhZKafeaZitbGoj6rzjpJbTtsfBe6kkW4Ccy7WS2cZqYFXV2+KXbOWUddaVtq3Ecf7FMeMtyUy48yaHEpWRKQs1JNCi+EyAaBm2eRcNOsiJo7S8tLqQuNX1lYTPaJCkNqccUSn3GmkJShJmaluJL5EW5mRGEyWu9PMZhMY/hWUXV1JOaUijhtxEzYPZHUtSTeN6Qhn4HFoT8DizXyI2+ZeIDr6PWNS84yGonN2sxMmTUtUVOiATM1CX4aHnjcbcJCkJRyNbhumXD9X5mlJh3EHWygsLmPCLHL9inm2T9RDyF1uOVfImtGtK2kkTxvp+JpxJLWyltRp2JR8k8g6Cu1psswzvAY2PUV7WY3kS7F1EycxF6FvHajKW040aHHHW08uKiJxLSlEZeBlvsGzgAAAAAAAAAAAksxmnEyTBGPsL7+7bkDzHvDo8/s/tVT3O3b8FcOXT7Hy3b/AI9x5Hy4LCtAAAAAAAAAAAAAAAAAAAAAAElqvNOt0tzKy+wxZr2SgsX/ALM9Hq+++MZw+w8ODnPrbdLj01789uKvkYVoAAAAAAAAAAAAAAAAAAAAAAAJLTWadhjkt/7CniRoyC9Y939Hpdbp2spvt23BG/a+PbOWx8u08uTm/NQVoAAAAAAAAAAAAAAAAAAAJPVn+SzMv7P2P92cAZXqX/3HET+z9F/8SKA+gAGVf+FN/wCj/wD+ogNVAAAAAAAAAAAAAAGVaBfeP+IFx+UA1UAAAAAAAGVaBfeP+IFx+UA1UAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAATmTQszl3WKP4xbRYdbDuHHsiZeSRrmVx18tCGWjNCtllMXCcMyNHwNLLke5oWFGAAAAAyrQL7x/xAuPygGqgAAAAAAAAJPVn+SzMv7P2P92cANJv5LMN/s/Xf3ZsBTSIsaW0bMqO082fzQ4glJ/8AUYDrp2K4xZPQJFjjtZKdqlpXAcfhtrVEUkyNJtGZbtmRpSZGnbbiX9ADjxcFwmHk8vNoeHUjGR2LHZZlu1XsomyGfg/g3HyT1Fo/g2/hNRl8Cf6CAdJF0R0kr8QmafVOntJV43PlFNk1ddGKHHdfLhs4aGeJcv4Nvx//ACE/0EA5U3S3DJy8acch2LJ4iTSalEW4mR0NJb4cEuIadSl9P8EgjS6SyMiMj3Iz3DkR8BqYeU2eZRLfJEWFtFOK80vIJz0FojJBc2YTrqorDhdNPxttJPc1me/NW4dZF0sKBiErEK/UTOGe0yilFaOXByrBnYkF00PSEubN/B+qZH+sr+kBwsYwPUleLafll+rd1FyLHquA3kxUzcFyDfzW22u0m6cmGp1La3EObGybC+Lh/qmSeId5GxvMGMotLp3UeZJrZsdbUKncrYpMQHT48XUuJQTzm3FW6VrMjNZ/LZJEHXMY5q5ExGTWJ1RqJWRnLJ2NbTMY5R24+ySNpyK1JbNZ7ks+ZOo/WLwPj4hy5cPVonsbKBkWJLYjoZTkan6WSlc1RGjqqhkmUZRiMicNKXDf25JI1K4mag9kNWqn2ktu3tYqdB2dw6k2XJPbOvunpk/unhwMuZqNHiXwkRH4mA4Me21rZxGVMn4Nhr+TolkiNAi5PJTCejbI3cXJXB5tubm5/Bkyoj4p+MuR8Q5MvJdSI03HozOmTEpmc1HO6kM3rZIq3FmROpbJbaVSUt/EfIiQaiItk7nsQe6vyrLZOTWtLP0xt4dbCaddh3Hb4LjE9SVJJLaGye6yFqJRmXUQlJcD3UXhuHXNanXacSeye00azuBKamlEKlU1XyZ7ieJK66SjS3Wel4mW5ukrdJ/D4p3DKGoGobWqKrXOrB20odSamJVzKut03smXIcQ1Pk0zIntWTqYryO1KJ5xTfHZO5dPxMg5+Aae6WYHrNa2+O2OqTT+O42uC7FvXcpm1pMNukpS2Jk5xyJIIkrSSWWjWZbKUktyVsF1G1ywNNPMzyyzBqJixSmqtg5lHNhSGZpINxwnDeIlLSpBoNOzSSTxVupRnskMdYsMcusrvccyPWrHMbodQswhW9TVWkdDU3II/RgGx7tfcebJSHnW0oPZp5RmakpNJmWwd/hWlWkWPakWDVFmOljz2PTp17MhtUMBeQwjccU452mabqloabW/+t0G3CLpkbnz5BX4rgWLIxvTpiBqFV2sHBXHo5yWOmpqwW5DdZNvdLpk2oku89t1HsX+vcBwmPZ7sXaKuxKxzyNIxzHa6ZBxxhinNqRH68VyKhyS911Jk9Nl5ZJJDbJGZ7nuYCtb0s4WtVZ+/d/dmJv4xw7L/AMJ1DZPr78/Dbo/qbHvy/W8PEJbIPZzO9xStxj7Y9D3fhn2R6/u7lz/hIy+0ceqW38W24bn+v+t4eIWdzpuxZ5lGy6JZdk61Y9S3cXoc0WkNRGbSVHyLgtpalGleyvhccTt8RGkMjx32SkUVszQKLBCwdeJ2uPyPdeIx667deeOK3Gdel/whvGlhMwlqR0CNa0Gptwj/AIMK290PyzO8UcxrUzPKK/TDkQpNQ0WKIbgsuxVmpC5Ud191UpSyPg4RONINP6iW1fEA92K6Bxsdl0FimXjcN6nvHrh6Pj+Ls1MN4lwnYqWktIcUtOxO8zW446ozIyLikyJIdez7MFC5h2T4fc3SbSPdS4zlccmvQtuuhxH+vDhm2pRpfabcNe5Hx5pWadk/MBxpHs32rWLLhY1eYJjl8m8rbpiTU4IiJWpOG5zQ25EalJed5bq3UqV4b/CSfEjCje041UuSqL/JNQ8Udy3HZzz9VPgYlIjQSjvMdJ1h+K5YOuOGe5qJaH29jSjwMiPkHXRtDMqoJkTKcO1BroeWqOw962FjQqlxJiZshD7xIjIktKZNCm0k0fVWSS35k4Z7gPZ+j8lOYStSmssM8zUqu7NeOVrZPE3HYS0+w8ltSEusyCJRqbSSEpUaVJ+JCVEHBxf2Y8exfLU20OLhp1jdlLtGzTh0b3ybr63HDQuxUtW7aVuq4mllDmyUF1PA+QdlhWiWQYrY4emdncSfTYIiTGpYTNKcd44zrBspTIeN9ZOOITx2WhDZGRK3QZmRpDXgAAAAAAAAGLare13oVofqZTaW6q5WvHbG/risYU2VGUcE0m6pokLeTv01boUe6yJBEXiojMiMNZpL2kyeqjXuOXEG1rZiCdjTIMhD7DyD+SkOIM0qL/WRgOsyaFmcu6xR/GLaLDrYdw49kTLySNcyuOvloQy0ZoVsspi4ThmRo+BpZcj3NCwowAAAAAAAAAAAAAAAAAAAAABOahQcyscByWv06tYlXlkqnms0M6WklMRbFTKyjPOEaFkaEumhRkaF+BH8KvkYUYAAAAAAAAAAAAAAAAAAAAAAAJzCIWZQKaSzndrFsbJVxbPMPRUklCa5ywkLgMmRIR8bcNUZtZ7HutCjNS/11BRgAAAAAAAAAAAAAAAAAAAnNQqufeYFktLWM9ebYU8yLGa5JTzdcZWlCd1GRFuZkW5mRf0gMWsYGr95pVE0pu9CJS4TECFBdlRctgtOr7N0zJaSUhZJ3NotyPfwMy3/AJwHS0eleuF5mdIzcXWp+M46jtPvR1WdRZynN2j6PHg2Sk7OERHsSt+XjsRANkwvRurwvKnsxLLsqvbJ2Aqs6l3YJlcGDcS5sk+BKLZSfAt9viV4eO4DQgAAAAAAAAAAAAABlWgX3j/iBcflANVAAAAAAABlWgX3j/iBcflANVAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAEPn8LDpOV6bP5RbS4dlDyh97HGWUmaJlidLZoWy6ZIVsgoa5rhGZo+NpBcj3JCwuAAAAAGF6N57g2PTdR66/zOirZac9tlmxMsWWXCSfS2PitRHsYDSe9jS3+svFf3zG/wAYB3saW/1l4r++Y3+MA72NLf6y8V/fMb/GAd7Glv8AWXiv75jf4wDvY0t/rLxX98xv8YB3saW/1l4r++Y3+MA72NLf6y8V/fMb/GAldUtUtMn9MsvYj6i4w665Q2CEIRbxzUpRx1kRERL8TMwFTpN/JZhv9n67+7NgKwAAAAAAAAB8+6T6DYc5pbhrmnOuWqkjE1UFcqie9+mz1K44zfZldM2EGjdrgfE0J2+XFPyIK3uC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wHXPezLRPKkuPan6jOKmNLYkmu7QZvNL/XQvdr4kq/nI/A/5wHSzPYu0vn0D2KWGQ5XKpJEkpjta9LjriuSCSSeqpo2OBr2Ii5GW+xEW4DwkexTpbMl4/PlZFl7knFGmGKJ057HKraZMjaRGPo/wKUGlPFKNiLiW3yAe+L7HWDQ7O1uoeoepDE+8aeZsJDeRGlclDpkbhKMkf8YyI9/mA9kb2Scfh0y6CBrVrTEiLf7QfQzqWh0l7EXg6XxknYi+Elcf9XiA9z3suMrkV8lHtCa3te7kNNpbRmSzQ8SD3/hSU2fUM/kpSvE/5zAdfZaDYcWqWPNz9cdVE5YqguVVjPv0z51xSa3tqup0DJPF068uPNPLlvxVxM0BW9wXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mA4ljoFZKr5CajWzUtE021FGXJyFSmkubfCa0pQRqIj8TIjIz+W5fMB/Lr2xPZu141YyfEr2tcy7NIcVudTT8isyXJZirReSorTaUtIUsz4E0o220rX8RqIjLfYNe9m32OX9E6zGMpq/aUzhdhdZdX0F3RUnbqCPEU80rkiRHkpRIW4SemtButNeBpPgottw+zp/s/wBPaxIWVZTqJq3DdxeXJs4RTMjbceiuIZkRjkI6JLT8cd54iLc1cHtjSlW5EElohrJGq7PUKbaSc+sKCFHopNIu8lsz35ZS0GTaGlsvLaS48460SUKNtWxp5EXBXEL3JNZMmi3uFMxcEy+resLqZXT6GTCiKlTCTXPPtE2+l1cY08koV1EyCSRkaVqSe6QHcRteKq0hVpUGEZTbXdh27qUUZENMyGUN8mJJvLdkoj7IdMkfA8rmZ7o5ERmQcLI9fKmRQPu4NQ5BdSnMdcvVuwo8cvdTJk4ltyQiQ6hRq6jThdNtLi92lfD8tw92Oa2VjWnc7I8gYspUrFqKtsLlxphojkLkQ23zNlJKSkz+LxI+BEfgXgA4lVrFYwcgyajmVVvk9n9p5EClqatuKh8obMOK64o1vOMtEhCnzM1OObmbiUluexAIbVLXKml5bg7Un2gpGkGMXNXcuzJUlymiPKsYkiOz2RxdnHfaStBqfI0t/M0bko0luYVWnusciJi86XOvLDUKG7kCabErWCxETJyRCoyHjUhSOhEXwUUhJvJ6TRpZM/AyPcKXv0qXShV9fheTTcilTpMB3Hmkw0zoi46ELeU6pchMckJQ6yrkl5RKJ1HHkZ7AOne9orCUR42VqVkrUFyrnSUQVxIzaHDZsGoZ8+Zktt03lpSnktLZJUo3OJkRkHJvdbMir5mIw4+kmUR3shyH3NKjzewEtlrsrr5OocRLNlwjJBHuha9iS4kyJfFJh2ULXGhnWbDRY1kMennzJNdXXzrUfsM+UwThrabJLxvp36LxJW40hCjR8Kj5J5Bwqf2gK/IaOit6LTnMJkjJ2FTaatJEBqVMhpbbW5KLqSktttpN5CD6q0KNR/ClRGkzDxste6iZWEnD8byO5nu1MizfRDjxyXUNtrWyapKXnUfETzTqem2TizNpeyTIiMwxpetKpmUUtdqZ7Wx6WRpGB0N0wjtGPQvec2T1+0r3sobpq26bfwtcSTy+XiQDVcI1nu/snSxrbHrvLsisjsnoaKqLGjOzauNJNpuwWUh1llsnEKYVsSi5G5uhHH5B3buvFBJagO4pieT5QcqtRcSmqmKyb1fEWtSCW82862tS+bbqek0Tjpm2oiQfhuEprjqzpvY6b6g4znlXlsbAjq5mP5TlVfFZNmqTJi8HSJtSlSVOIQ+jxRGdSlaiJW/FREFbiOsGMWL79ZJmXW7bNvYJm2rUZtK2YU5yPIQk2TItmjJO26SM21Nmo1K5bBw/0hqBUH3wzhmVPVkSDFsbqaliKSKRqQ2l1BSUqfJxSiaUlxSWEOmlJkZ7bkQClxXUuDmGV3+L1GO3RN43JOFMtHkMJhnI6TLpNtn1TcWZofSe5N8S4qIzI9iMLMAAAAAAAAAAAAAAAAAAAENpFDw6Dis+PgttLsa1WUZK8+9JSaVpsXLqauwZIjQj4G5ipLaD2PdCEmSl/rqC5AAAAAAAAAAAAAAAAAAAAAQ+t0PDbHRjPq7Ua2lVeKSsYtWb6dESan4tcqK4Ul5siQszWlo1qIiQvxIvhV8jC4AAAAAAAAAAAAAAAAAZVoF94/4gXH5QDVQAAAAAAAZVoF94/wCIFx+UA1UAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAASWYzTiZJgjH2F9/dtyB5j3h0ef2f2qp7nbt+CuHLp9j5bt/x7jyPlwWFaAAAAAnLTT7A7uc7Z3eEY/YTnzT1JMqsZddXskklyWpJmexERFufyIiAcfun0t/q0xX9zRv8AB3T6W/1aYr+5o3+AA7p9Lf6tMV/c0b/AAd0+lv9WmK/uaN/gAO6fS3+rTFf3NG/wAAB3T6W/1aYr+5o3+AA7p9Lf6tMV/c0b/AAd0+lv8AVpiv7mjf4AFWAAAAAAAAAAJLSiadlpbhtl9hiwrtdBXP/Zno9L3JyjNn2Hhwb4dHfpcemjbhtxT8iCtAAAAAAAAAAAAAAAAAAAAAAABJWM00apY9Wngva+vQXL/2m6O/u3pya0uw8+B8e1dTq8eonl7v34r47thWgAAAAAAAAAAAAAAAAAAA668ZvJFRLYxqygwLRbSiiSZ0JcuO05/MpxlDrSnEl/OknEGf/wB8QDLsC0s1jw/DrHGpWruOuzHLJ+0r5tdiDkZDTr8lyQ81Iaenv9dlSnTSRIUyskl+vv8AEAmM4we3xZ3E77KcjjXeQ5HqNRSLGVDrzgxC6TK2m0MR1OvKQkkp3+J1xRmo/i22Ig3aPFvHKl+LY3DHbneuluXCh9EmUqUrpGTbi3SNaEmkjMzNKlJM+KSPiQfLOCez+9a6vawR7bIqiLYGWPvNPUVAVfFXMb4ymZb8Y3nCedSthO58k8iW7tx5lxDcYGnObTrrHMkzzP4NvYUFlInpbr6PsETg5DXHJpptT7riPFw3DUt10zPci4lsRBHXXst1llIjXBScVsrWLMuXUHkuKotofQsJnajQTBvtqS42rZKXCcIjLluj4iJIdo5oJZ1qiThOY1tIifjreNXCTx1o0vMNqdUh2K2w4y1FcI5D3zQ4jxT8G5GZhw7L2eMgepbbGKnUWLCqsho6+ptUu0ZvSFOxI5MJeZc7QlLSVpSnk2pDh+B8VpM9yBlfszVeSXEnJXJWOWFmq5lWkZvIMbTaQG2pEaMy40uObyDWr/sVC0uJWgyMzLYy3IwosT0VjYrkeLX0WygNoxynsqtcOBTMwY7y5j7DynG22TJDKUmyZEjiozJRGpZqIzUHTX3s4VlzjKcVdsaqXX1OQLvcch3FKmfDriWhaVxHmVOpKQxu88aEkbZoJSEkfwEZh0N1pzI0jZxnJMedYg2cGVPbkv41p25KrUsykNc2zrYDpSUlvHZ4O83jJST6hmky2D26baN5Y5iNRdSsgVT3Puu3h9Cxp25HNE2zKWlUlk1JTsptBIcZLif8IoiWg0kA7Ci9nazxyPXyKbJ8dq59fkzWQtxazGVxaZpJRlxnWWYJSzNpTjbq1G4Txl1TJZoMt0GHjivswUOJ5GVhWN4c3XR5MyXFcbw2MVyS3zcPi7YqWo1oQp1WxoabcMkoI3DIlcg9177NdVbYxglW4/jVlaYLU+547+R40m0gyWVNtJcUqIbyDSvdhCkqS78PxEfIjAe+PoLa0DsWRhGW09I5Io/cFwlONNEy/H6zrqVxWWHWURXEqfe2MydTsouSVmRqMO1wnRCBis2S5Y2zdzCl4jVYk/EdhEhLjUMn0qcUfNW5OE+ZGjbw4/NW/gGJe0lS6baD4FiGoGruRU9pHopC8SiWGQYX9oGWoUpfUZ68ftCHDcZRHSk30L3V8W7SjUREFnppV4tnVZAzT2Y9b8RUydLGoreTDoo8pJtNrcdaW1GYdYRBkEp90yS42tGyi5NHtuYcXVf2eqqirs51FrsJxPNiXHl5HKop2FRrC5uZbUbkqCxMUS0oTJUylHE4rqiN1fD5pJIV137OzdxidfjKcvciuxbuznyJbcLxkwLCQ85KgmnqfCS0PEjnuextpXx/4oDqNVPZti3t5eZnjLWDx5Nwywqc7fYaxcSo5sNE2S4briyS0fTQkuDjbqORErYvEjCVx7JL/D9NcNkzM7nxLTVuU1cTrWrokSZ8Yk1DSnURIqGnkOvKVHSZbMKSSVr2b+HcBu+ml6dzg0Syayp7LFpN9vty64q+S6ptxSem/HUSCafTtwWk0tFzSr4G/wBUg9+W55BwjT2x1HyamtIsWprlWU2ChDT0xpKU8lN8W1qbW4Xy2QtRGfyM/AwEwvXukgrsoeQYflFNZw0Q3IlbLZiqkWiJbxsxzjdF9aPidLgZOqbUg/FZIT4gPGZr3T10VLc/BsrZvTt2qNVAbURc1Ml5hb7PxJkHHNtxCD2Wl40kfgo08VcQmcU1qyRGRZtJyvGstV08maxzHKFCKozecKIh5SGlIe36hkbji1PvE2SCRxNJkogFOnXumdm1lHHwvKHr+wnyqxynQ1E7TCkR22nXUvLOQTJF0nkOEtDikmk/AzMyIw4cT2i6uxkwo9bpxmspNtLsK6qdQxCSifNhrdS8wjnJI0Hsw6tLjpNtGlP6/IySYZvn2u9Fb5xiZWntGS9GMat8cnzjKW/QxH3rBiahhbDi7JiS3yb/AIQjS0rYzLfdRbGAucD1hmx8SS/ZyrPO3J1/IqMXm10aK1JyKMhnqlJI+TMXYiS8RukbbSya3SXxJIwoV641T8KAimw3JrW+nPzI68ejIiNz4y4ikpk9U3pCI5Eg1tluTyiV1EGjmR7gPG212pa0nnWMLy6ezXVrNtdrZgtMqpYzpKUk5LUh1t01kltxRtsocWRI34/EnkHtodW8dnZrJxRuVdSVzLR6FGkyWYxQ23m4MeV0Glt7LNK2nTcSbhKUZoeLciJBAOIjX+knsodxzC8pv1dmdsX24DUQlR4CH3WUS1dWQglNumy4ptLZrdUkt+BfIBotLbV1/UQb2oklJg2EduVGeSRkTjS0kpKvHx8SMj8QHPASWms07DHJb/2FPEjRkF6x7v6PS63TtZTfbtuCN+18e2ctj5dp5cnN+agrQAAAAAAAAAAAAAAAAAAAAElqvNOt0tzKy+wxZr2SgsX/ALM9Hq+++MZw+w8ODnPrbdLj01789uKvkYVoAAAAAAAAAAAAAAAADKtAvvH/ABAuPygGqgAAAAAAAyrQL7x/xAuPygGqgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACcyaFmcu6xR/GLaLDrYdw49kTLySNcyuOvloQy0ZoVsspi4ThmRo+BpZcj3NCwowAAAAAAAAAAAAAAAAAAAAAAAAAAATmnsHMq7Acar9RbWJaZZFp4TN9OiJJLEqxSygpLzZEhBEhTpLUREhHgZfCn5EFGAAAAAAAAAAAAAAAAAAAAAAACcmw8xXn9PYQbWK3ijFRZs2kFSC679it6EcJ5B8DMkNtN2CVFzTubzfwr2I0BRgAAAAAAAAAAAAAAAAAAAAADKtffu4/ECn/NAaqA+eIenP221/1SlFnWX0HZm6Jvp0dp2VDu8RZ8llxPkf8ANv8A0ALLuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wHy/8A6Qz2V9Rc10KgY7pfb6jZ5dyclgpTVzrZMlhDfTeI3lkpKUtkkzSXUUokp5eJkXiAwvRj/RT6q6Y49Z6taga22WF3VPUy50aBhcxbcxCksGskOTSMko+JJEpLaVkottllsA+9sE0bk3+D49fTdZtUUSLKriTHUt5Go0ktxlK1EXJBq23M/mZn/SZ/MB6zxe0081n09qomo+a3MK997drjXFuqQ0fQickbIIkl817+JH4pSZbbANWzery23x2VXYTf1FNaSNkJl2tS7Yx0IPwWRsNSI6jMy32PqkRH4mSvkAznHNF9QaTAcVoJeo+PyMkwY0oobePi7zMUmCjHHNuVEVOWp7khSjUaH2vEkmRFse4XOn+FysJx2TXvXSLK1sZsq0nTji9Fp2XIWa1qQySz4NkZkSUczPikt1Ge6jDoNcKXJrnQTKqFs12N7Ko3I5qqoSkqekGkiNTLBqdUXjuaUGpe3gRmr5mE9k3s8TNREWU3UjKKi7nPM17FYRY+SIcZEOQchHaI7r7hSTW4eznxNkaS2SlB/EA52NaBsUKad5MvGIEityFF661juKs1MN0kRnWEspaQ4tZH/DGs1uOOnvuREkjIiDmWOiz8iXOvKrKGYt19qzyuskP1xvsxXjhpiKZdaJ1JvIU2S9zSts91FsZcfEPPGNGZVNlsDOLbKW59wmbY2FmpmAcdmU9KYYYSTSDdWbKG24zZEk1OGfiZq3Ae6h0c9yfZL/8AGPrfZW5trf8AifHtPbe0/wAH+ufDh2n9b4uXD5Fv4Bx8G0TThmTRMicyJM9MattK5UdULpk4UyxOYajPmr9Xfp7bHy/W3L9UB0137Nlbb4xV409Y01ixi1y/Y44xe0CbKHDiutrR2N5hTqeu2gnFk2pKm1JJLRePA+QciHoTa45FoLPA8ixihyWlKc0b7WIoRVOsTFtqebTBjvsqRsbLXBRvqWXH4zc3MB12UezLHyTIVZPOs8UtrWyr4kG4sciwyJZyXFMEoifiGa0NxlmlZkaVNut/Cg+B7K5B219oIqypcjg02XrqLC0vo99Vz2a9JqqnG4jMU0JQS0pWSmmnE+HAiJ3bb4fEOszf2asZs50K9o6zEzcraFihUzfYk3eEUSNzNo4yVOt9J4uay3PmlXw7oPYBT6X6saY3rcPAqO3XWXtXEQyrHreAdRZtobTx5FCWhvdv4fBbKDa2/UPjsA0sBNYZCzaDRSmM2tYdlaqt7d6O7HSSW0wHJ8hdeyrZCPibhqjNLPiZmpCj5LP41BG6eys8ytjUXGckzuQxYVt8ddCs6qDFaXAbXBivbMIebdbVxW8vibyXTMj+Lf5EHq0YiZt9pMtk22rOS5jj0GSiorjuolU2s5bJq7W6hUCHG3QS1EyRL5fEy4f85ANOgTZMt6Y0/UTISYr/AEWnH1NGmUjilXVb6a1GSN1GnZZIVulXw7bGYZLO1JylvWAnWLNJYLCsmcQmMGwjY7V5k3kyOrx57JWqPH4kok8nVbluRGA7eTr3SMe+1R8NyiUimuPs8hbLMUisLM3ktpixyW+kzUfIlc1khsk78lpMjIg47vtD0LbjlYnC8oVfIuk4+VKpMJqSuYcQpakpcckpjmSWVb8utsoyMkc/DcLZWaVsTCpGdXsKwpIUKG9NmsWDHCRFQ0SjcJaUmojMuJ7Gg1JV4Gk1EZGYRZe0Lj0M3Y+R4ZldDMTWN2rMSdHjLckNOyG47CGzYfcQbjjrqUpQaiNP/H4eG4cDJ9frCqYhQq/TDI0Xyr+sqp1NMVAKQxHlqPg+S0y+gslpQ4lJpdVstJksi2MBlFjrM19odQW7X2xFY5k9Ff2EGlwcjx5w3kNNpOOz2RcRVg/1FHtsh7mrf4TIBujOslfXYvf3OTUllFnYhWwpl5FabQZpdfjJeNtnksuRp5Gk+Rp8S/n+YDjy9e6KJdzqv7IZO5Bqb6PjtjbpYjFDiy3yYNnfk+TriFHJaTyabXxMz5Eki3AftJrxRXlzXQWcRyePAs7ibQR7eQzGTEOwjLeStniT5v8AxdncUlZNG2ZbEaiV8JBw8k1rdRW3lc1iGU43PXR21hRWFrEjoZmriNGalIaJ1brZluhZJkNN8k/zH4kAmbv2gLc9LZ7jdNf49kqsR9/1VhYxonSsUoS0TrzKEOOceKnWzNt5ttWzhfB4HsF1O1spYFzJgrxrIJFVXWLFRY5Ay3HOvhzXemSWlkbxPq2U80lS0MqQk1+Ki4q4hS6hQcyscByWv06tYlXlkqnms0M6WklMRbFTKyjPOEaFkaEumhRkaF+BH8KvkYUYAAAAAAAAAAAAAAAADKtAvvH/ABAuPygGqgAAAAAAAyrQL7x/xAuPygGqgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACHz+Fh0nK9Nn8otpcOyh5Q+9jjLKTNEyxOls0LZdMkK2QUNc1wjM0fG0guR7khYXAAAAAAAAAAAAAAAAAAAAAAAAAAAAIfRGHhtdoxgNdpzbSrTFIuMVTNDOlpNL8quTFbKM84RoQZLU0SFGRoR4mfwp+RBcAAAAAAAAAAAAAAAAAAAAAAAAh7aFhrmtGLWE23lt5YxjF+zWQUpPoP1y5dSc15Z8DIltut16UlzTuTznwr2M0BcAAAAAAAAAAAAAAAAAAAAAADKPaCZtfdWHWlVj9rce58vr7OTGrIipD/AEGkuqWZIT/uLczItzIjMtwHX2ntOU1POqK6z0m1Ojyb2Yqvr21474yZCY70hTaf4T5kzHfX/sbUA9mjki1vdSdRsxl4jkFFDufc/ZG7iAqK6vox3G17Ee5HsZfzGfgpO+2+wDYwAAAAAAAAAAAAAAAAAAASerP8lmZf2fsf7s4AaTfyWYb/AGfrv7s2AzrXHGTy7VzSam+0N1S8nLxztVRL7NILjDT8JL2PwP8AnLYB3ncF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgPTI0KOPHdkFrJqw700KXwbyLda9i32SXT8TP+YBhmUezdqLr9GYr7GTlmNY4y+l9mXnVu3bWe5GRk4xXsbIjK8CNLi5PUSZpM2tyNICp0C0TsnqDIcfs9eNWp54zk1hSx5UjKFreeZZNBoNZqSfiXMy2TskiJPhvuZhyNTdGO5jSa+v8D1T1CjvxZbk9tiRfLXHVKmz+pIdWlJJUZrdkuuHsoviUf8AN4ANJ0r071WwvI8luMz1ExS/i5LM95PR6vE5NY4zK6DLJGlxyxkkbfBhPwGjlyMz5kXgArtPMQXgWH12LLsTsXIhOLfmG10jkPOOKcccNPJWxqWtR7bn8/mA7eEzbNuzTsJ0R9tx7lDS1FU2bLPBJcHDNxXVVyJZ8iJBbGRcdyNSgxtz2R9KZWNyV2GK4vIz+Up2arOlY5G96osluG6mWl093UmhZlxT1fBKSRvsA5OZ6bzsbwOxmRrmxl2jGWFl0WVXUapqoz6niUZKhIdJyQ0STWS0NLJxSVHwLlsRhO4hp3m2oFTms+0nVao+S5EiWbGWYMtcKyjIgRmFc6yQ61IZQTjS+nzd5fDurmRkAuG9JpdXpNM0xlXJ2VM7S2MKQzEg9OUp19SlITENb5tstNpWtttlZK2Imy6hEk+QZbV6dX2utxYR88fnTK6PjUWtKVZYTIpmUzmpzcltJxJq1qlbGyXVUlZsqIySgy3UYC5g+zzJrqJbNRZYVSXTdzX27D1HhTdfXpVEWakocjNyOs7yJS9zVJ8DMjSSS3JQd8rRKK/iWXYxMv3FO5NfSchizmI5Nu1ktakLZW3upXJTTjaVErw3+RltvuHVZZofl2TFkkZrUaBDiZnWxYt4n3Abjq5LDXT60dfaSSyhZEXJtSHD2I+K0me4Dt5+jPbqvIa77R8Pf2Vwcn59j36PZlxFdDbn8XLsm3Pw26n6p8fEPGFox2Oqx+sLJjP3FlU/JuqmJxU72lctXRL4z4Gntf6/jv0/1S38AgK/2SpVcuJLj5bjTFlHgWNRIs4+IE3Msokxg21rmP8AajcflEokL6pqJBmS92viI0hK+0M1jmh+m8fJdfNUXHKQqxOBVc+txhfOCUskKXKloTIUb58Yad+klvbx2Qo1FsHfYBgGi2sGQy9UdJsx01yamtbZm6nvljESxt2JPFBm2mW45vGSvppPg6wpxPJfFSTNJpDYtboeG2OjGfV2o1tKq8UlYxas306Ik1Pxa5UVwpLzZEhZmtLRrUREhfiRfCr5GFwAAAAAAAAAAAAAAAAAyrQL7x/xAuPygGqgAAAAAAAyrQL7x/xAuPygGqgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACSzGacTJMEY+wvv7tuQPMe8Ojz+z+1VPc7dvwVw5dPsfLdv+PceR8uCwrQAAAAAAAAAAAAAAAAAAAAAAAAAAASWlE07LS3DbL7DFhXa6Cuf+zPR6XuTlGbPsPDg3w6O/S49NG3Dbin5EFaAAAAAAAAAAAAAAAAAAAAAAACSsZpo1Sx6tPBe19eguX/tN0d/dvTk1pdh58D49q6nV49RPL3fvxXx3bCtAAAAAAAAAAAAAAAAAAAAAAABJZjNOJkmCMfYX3923IHmPeHR5/Z/aqnudu34K4cun2Plu3/HuPI+XBYVoAAAAAAAAAAAAAAAAAAAACT1Z/kszL+z9j/dnADSb+SzDf7P1392bAS2of8u2kf8Asv8A+5oAaqAAAAAAAAAAAAAAAAAAAAAyrQL7x/xAuPygHo9q2P2vQDLIvXdZ6zcRvqMq4rRvLZLkk/5jL5kf9ID39wXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mA+X/9IZ7K+oua6FQMd0vt9Rs8u5OSwUpq51smSwhvpvEbyyUlKWySZpLqKUSU8vEyLxAYXox/op9VdMces9WtQNbbLC7qnqZc6NAwuYtuYhSWDWSHJpGSUfEkiUltKyUW2yy2AfdeNaCw8/00qlZTqnqNOi5JRMHYwZF71oz7ciOXVaWh1tXNtRLUk0rNW5GZHvuYD3ni9pp5rPp7VRNR81uYV7727XGuLdUho+hE5I2QRJL5r38SPxSky22AbuAAAAAAAAAAAAAAADKtAvvH/EC4/KAaqAAAAAAADKtAvvH/ABAuPygGqgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACcyaFmcu6xR/GLaLDrYdw49kTLySNcyuOvloQy0ZoVsspi4ThmRo+BpZcj3NCwowAAAAAAAAAAAAAAAAAAAAAAAAAAATmnsHMq7Acar9RbWJaZZFp4TN9OiJJLEqxSygpLzZEhBEhTpLUREhHgZfCn5EFGAAAAAAAAAAAAAAAAAAAAAAACcmw8xXn9PYQbWK3ijFRZs2kFSC679it6EcJ5B8DMkNtN2CVFzTubzfwr2I0BRgAAAAAAAAAAAAAAAAAAAAAAAnMmhZnLusUfxi2iw62HcOPZEy8kjXMrjr5aEMtGaFbLKYuE4ZkaPgaWXI9zQsKMAAAAAAAAAAAAAAAAAAAABL6mxJc/TfLIMGM9JkSqOeyyyyg1rcWqOskpSkvEzMzIiIvEzMBm2Eawycawygx2bo1qe5Jq6uLCeWzjxmhS2mkoUaTNZGZbpPbciPb+YgHpPKLTUPWfT21iacZrTQqL3t2uTcVCo7RdeJxRssjUXzRt4mXipJFvuA3cAAAAAAAAAAAAAAAAAAAABlWgX3j/iBcflAPz2o/5C8m/wCZf3xkBqwAAAAAAAAAAAAAAAAAAAJPVn+SzMv7P2P92cANJv5LMN/s/Xf3ZsBjOs9hV537Rml2mke2yumlx2ryY5Z1SziFumI3u0h4yPmfxo5cUmnx25bkZEHFqoNHbX8KpRqHrnGrrayk1FZeP38fsM2ZH6vUaQSVKkI/4B4iU4yhKumexnunkGj9wXnVqr6j+mAdwXnVqr6j+mA4ujiLyj1A1GwOdmN7kFdSPVT0Fy5kpkSGTkRTU6knOJGaTUkjJJ+BeO3zPcPyR7QryLHIkwNFdQbOmxawkV1lfQzqDiIWwklPLSyuemW4lJH/AMWOaj/4qTAarW2MK4rottWyEvxJrCJMd1PycbWklJUX+oyMjAcsBxpT70aM/IZiOyltNqWhho0Et0yLckJNakpIz+RclEW5+JkXiA8o7i3mW3XIzjCnEJUppw0mpszLxSfEzTuXyPYzL+gzAe8AAAGVaBfeP+IFx+UA1UAAAAAAAGVaBfeP+IFx+UA1UAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAQ+fwsOk5Xps/lFtLh2UPKH3scZZSZomWJ0tmhbLpkhWyChrmuEZmj42kFyPckLC4AAAAAAAAAAAAAAAAAAAAAAAAAAABD6Iw8NrtGMBrtObaVaYpFxiqZoZ0tJpflVyYrZRnnCNCDJamiQoyNCPEz+FPyILgAAAAAAAAAAAAAAAAAAAAAAAEPbQsNc1oxawm28tvLGMYv2ayClJ9B+uXLqTmvLPgZEtt1uvSkuadyec+FexmgLgAAAAAAAAAAAAAAAAAAAAAAAEPn8LDpOV6bP5RbS4dlDyh97HGWUmaJlidLZoWy6ZIVsgoa5rhGZo+NpBcj3JCwuAAAAAAAAAAAAAAAAAAAAAABD63Q8NsdGM+rtRraVV4pKxi1Zvp0RJqfi1yorhSXmyJCzNaWjWoiJC/Ei+FXyMLgAAAAAAAAAAAAAAAAAAAAAAfP2E5vbabWucVdppXn9j7xzCzs40mso1OsOMOKSlBktSk778DPciMtjIyMB0upOsyNadKrzH8I0u1EeclTF15SHsfWllEiFP6cltRpUoyNDsZ5s9iP4k/0eID6bAAAAAAAAAAAAAAAAAAABJ6s/wAlmZf2fsf7s4AaTfyWYb/Z+u/uzYDBdTs7qf00dJIpUuXKTWQr+skSU4lanFN99lg2+nIKP0nUbEZqcbWpCCIzWpIDkYVjFnFzXHHixvK2syj5TYzL85UOamgYgunJ5PReZdgQtaVMcVxtn1GtfUM+T24fQrzdD9qIrjtOa7koLxMT/dq1dONzb6jXauHBHJXTPpGsjVw5EkyQZkGE6x4tGnZdmj+V6fWuSzbGijMYLLiUj873fMJDxLJt9tKkwHeuppw3lqaIy4HzPpnxDMqfTewnaj6zXdpii7LMKrJcJfg2CYRvSmP4GImS9GXx5ISpKXeakbEaSVy8CMBZS9K9WX6bVC8pM6zmDHk5fYyjxFiJXMxrmuPpdZLD7kM5iVvNE4SHESCLnttsW4Dh6pYNaW+WTHU4+4mll49WxsO56f2FzNqlJQslojOtyWU1chKzaVze6fyRuvZsySFTQY7WQdQ7JOe6e5DdZ69eIfpclj0Uji1XkwylszskGTLDKDJzqRjfI1nz2bc5/EGfYfp5mjVTIbsKt1rLI+PXDOROQMAmwn7V9yI6jhKtHZKmrHk6aFINlDh8iTsTadyAXP2Hyhm5q9PmMcsvs5njFLa3Ugoy+jCcgsoKYy+oy2Qb6GIbZIVsat3fA9jAdLgGCZdH1Pgyb+KUfJ42T2E2daRcDmIkya9Tr5ttvXa5RRno6mVNJJlKVKTs2RNEbZmkPq8AAZVoF94/4gXH5QDVQAAAAAAAZVoF94/4gXH5QDVQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABJZjNOJkmCMfYX3923IHmPeHR5/Z/aqnudu34K4cun2Plu3/HuPI+XBYVoAAAAAAAAAAAAAAAAAAAAAAAAAAAJLSiadlpbhtl9hiwrtdBXP/Zno9L3JyjNn2Hhwb4dHfpcemjbhtxT8iCtAAAAAAAAAAAAAAAAAAAAAAABJWM00apY9Wngva+vQXL/2m6O/u3pya0uw8+B8e1dTq8eonl7v34r47thWgAAAAAAAAAAAAAAAAAAAAAAA4Mypq7GRAl2FZFlSKuQcuC68yla4r5tOMm60oy3Qs2nnWzUnY+Di0/JRkYc4AAAAAAAAAAAAAAAAAAAAAASWq8063S3MrL7DFmvZKCxf+zPR6vvvjGcPsPDg5z623S49Ne/Pbir5GFaAAAAAAAAAAAAAAAAAAAAAAACS01mnYY5Lf+wp4kaMgvWPd/R6XW6drKb7dtwRv2vj2zlsfLtPLk5vzUFaAAAAAAAAAAAAAAAAAAACX1NiS5+m+WQYMZ6TIlUc9llllBrW4tUdZJSlJeJmZmREReJmYDI6z2gY+mOmdMzk2kOpqSo62BAlPIokoa6hJbZ3JbrqEkk1mXio0+Hz2+QDp7/VrI8k1QwPK2/Z91ZhwMb96dsVJo2eR9ojk23wS2+rf4k+O+225fMB1Gv3t/VuhtK7KL2fdULWxTDXOJl+tRDisMpWSOpJe5OLYbNSiIl9JSTMjLfcjAfCmIf6TT2j9bPaU08q5Uh2ixJWTQyfxvF4u785rq7G04tZm68aknxNHJKFH48SPbYP6k9/vkrqr6c+oA4uji7y81A1Gzydh17j9ddvVTMFu5jJjyHjjxTS6om+RmSSUoiJR+B+O3yPYNfAAAAAAAAAAGVaBfeP+IFx+UA1UAAAAAAAGVaBfeP+IFx+UA1UAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAATmTQszl3WKP4xbRYdbDuHHsiZeSRrmVx18tCGWjNCtllMXCcMyNHwNLLke5oWFGAAAAAAAAAAAAAAAAAAAAAAAAAAACc09g5lXYDjVfqLaxLTLItPCZvp0RJJYlWKWUFJebIkIIkKdJaiIkI8DL4U/IgowAAAAAAAAAAAAAAAAAAAAAAATk2HmK8/p7CDaxW8UYqLNm0gqQXXfsVvQjhPIPgZkhtpuwSouadzeb+FexGgKMAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAHTZZXX1xitxU4tkf2fuptfIj11v2NEv3fKW2pLUnoLMkO9NZpX01HxVx2PwMB3IAAAAAAAAAAAAAAAAAAAAAAAJzCIWZQKaSzndrFsbJVxbPMPRUklCa5ywkLgMmRIR8bcNUZtZ7HutCjNS/11BRgAAAAAAAAAAAAAAAAAAAAMp9qP8AkLyb/mX98ZAasAykyI/al2MtyPT/AP8AqICeyH2KfZxvNTse1iiaeRaDLMctGbZidSH2JMl1tXMifaQXScI1eKlcSWe362xmRhu4AAAAAAAAAAAADKtAvvH/ABAuPygGqgAAAAAAAyrQL7x/xAuPygGqgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACHz+Fh0nK9Nn8otpcOyh5Q+9jjLKTNEyxOls0LZdMkK2QUNc1wjM0fG0guR7khYXAAAAAAAAAAAAAAAAAAAAAAAAAAAAIfRGHhtdoxgNdpzbSrTFIuMVTNDOlpNL8quTFbKM84RoQZLU0SFGRoR4mfwp+RBcAAAAAAAAAAAAAAAAAAAAAAAAh7aFhrmtGLWE23lt5YxjF+zWQUpPoP1y5dSc15Z8DIltut16UlzTuTznwr2M0BcAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAJ3BsKqsBppFFTyJTzEq4trpapK0qWT1hPkTnkkaUpLgl2S4lBbbkgkkZqMjUYUQAAAAAAAAAAAAAAAAAAAADKfaj/kLyb/AJl/fGQGrAMq/wDCm/8AR/8A/UQGqgAAAAAAAAAAAAADKtAvvH/EC4/KAaqAAAAAAADKtAvvH/EC4/KAaqAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAJLMZpxMkwRj7C+/u25A8x7w6PP7P7VU9zt2/BXDl0+x8t2/49x5Hy4LCllS40GM7MmSWmI7CFOOuurJCG0EW5qUo/AiIvEzMBiuO+2b7PGa6y12hOCZ2zk+TWDUl411CDkQY5MtG4rnJL+DUZpSrbpmvxLZXHctw3IAAAAAAAAAAAAAAAAAAAAAAAABJaUTTstLcNsvsMWFdroK5/wCzPR6XuTlGbPsPDg3w6O/S49NG3Dbin5EFaAAAAAAAAAAAAAAAAAAAAAAACSsZpo1Sx6tPBe19eguX/tN0d/dvTk1pdh58D49q6nV49RPL3fvxXx3bCtAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABlPtR/yF5N/zL++MgNWAZV/4U3/AKP/AP6iA1UAAAAAAAAAAAAAAZVoF94/4gXH5QDVQAAAAAAAZVoF94/4gXH5QDVQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAGUe0E9a+6sOq6rILWn98ZfX1kmTWS1R3+g6l1KyJaf8AcexkZbkRmR7APLuC86tVfUf0wHyrrl7El77TWYZxgyPaM1BgNYmmpODDtrJyxrpC3mFuqU7H5IJLnLZJOI+RERmlRl4hgfstf6PL2h9Dfa1x37dRbOLjSodkkssxKxLpIPsqySk3DT1GeSjSnZxtPLlsnfY9g/pN3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YB3BedWqvqP6YDosI9n3Lo2GUEfUDXTUSVlDVXFRdSK7IlFFdnk0kpC2SUykybN3mad0kfEy3IvkA73uC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wDuC86tVfUf0wHRS/Z+y5WZ1T8LXPUVOLoq7BFkwvIldqXPU7EOGts+jt00tJnEvdRHyWzsR+JpDve4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAO4Lzq1V9R/TAdXkWk9JiVNKyLI9etUINfDTyddXkZntuexJSkmjUpRmZElKSNSjMiIjMyIB0VxjmD0mNVGWSde9YJEC/bbcrEQJ0ubLmJW31CNuKxGW+rZHxK2b+EtzVtsAoKDSGoyimh5Bj+vGp06unsk9HkNZJulaD/wBre5H/ADGR+JGRkexkA8ch0iqcVpJ+R32uuqUSurI65Up9WQqUTbSC3UeyWjM/AvkRGZ/IiMwHjQ6TQMijHKr9YtZm2yS2v/s6wkwlGS0EtOyX46FH4KLciLdKt0q2URkQcp3RRlmfHrV6zaum7KQ44hSLpxTREjjvzdJk0IP4i2JSiNXjxI9j2DjX2k1NjTEaVea6aqRm5k2PXsK9/rXzkPuE20jZLR7clqItz8C38TIgHadwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAdwXnVqr6j+mAk9VfZgucu05yPHsa1o1A99yq91VT72vurBKegucY5COgvkyTyWzWRJUfEj2LfYBWdwXnVqr6j+mA7PC9G6vC8qezEsuyq9snYCqzqXdgmVwYNxLmyT4EotlJ8C32+JXh47gNCAAAAAAAAAAAAAAGVaBfeP+IFx+UA1UAAAAAAAGVaBfeP+IFx+UA1UAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABlWvv3cfiBT/AJoDVQGVaefy7auf7KD+5rAaqAAAAAAAAAAAAAAAAAAAAAAAAAAIfRGHhtdoxgNdpzbSrTFIuMVTNDOlpNL8quTFbKM84RoQZLU0SFGRoR4mfwp+RBcAAAAAAAAAAAAAAAAAAAAAAAAh7aFhrmtGLWE23lt5YxjF+zWQUpPoP1y5dSc15Z8DIltut16UlzTuTznwr2M0BcAAAAAAAAAAAAAAAAjdVsrwrC8FtLzPMlpKKtQytpEy3mMxWEvqQom0k46ZJJZn4JLfc/5gGBUupmFRtLtH9VcR1HwKZ7gqCoX12t/2aq6jsSP1mXZ7DT6I0lCmm+Lbid1makeBmRkGt+z/AF9nW6RtOZLIJtVjOtrTmlC4qURpM195tREoyW2XTcSotzJREZb7GA67XuvoLv2ZMkYqraXNqToSdhTYtzIcXIaSlJtr7UlzqPEotjNSlq5kfxGrcwERNO6k6rx8BbzLKY1KWYR65TTN5LJ1cQsaW6bJvG4bpEpwiWaiUS+fx8iV8QDi1d/k0V1jHm8/voEWE1ncFmc+/IsnWG4sllEZ5xKlKclLZSo+JrNS/n4+JmAnYt3FyWk+y1bljto2xlGKKXcU2cSckgrUuwJKunIlEbsSSaUbrYJSkpSptST5GajDeNHEya2+1BxT3vbTa6iyBpivKzsn57zLbkCK8pHXkLW6pPUdWZEpR7cti2IiIg1AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABD63Q8NsdGM+rtRraVV4pKxi1Zvp0RJqfi1yorhSXmyJCzNaWjWoiJC/Ei+FXyMLgAAAAAAAAAAAAAAAABlWgX3j/iBcflANVAAAAAAABlWgX3j/AIgXH5QDVQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAGVa+/dx+IFP+aA1UBlWnn8u2rn+yg/uawGqgADH9Z5OoeNpey2i1KegKaXHi0OMxq+I43dTVH4sSVvNLeVz8duzrZ6aUqWo1ER7B0l9m+fITkGp0LMpEaqxfJW6IsbRDiqiTI5Px2H3HXVNHI6/Jx00G26hsiJHJCvEwG2y5smNLhR2KiXLalOqbdkMqaJERJIUoluEtaVGkzIkl00rPdRbkSd1EGR6waiZVjOpOOY7AbyWHQe4rm9tJtS3WLNwoqGiSgylqNRJR1TUZIQRmo2vE0k4QDmue0JQV7Ms04rltlCoyr029qliGlqIiUw06285u8g1+Dpc0sIWpJkfwEk0mYe9etVVVzF08CoyjJreZeWddFr2irmXjOIaesTSnXWGjaTzTx5LN1RGZmR7GZB0lhrta4/m+SvTcUyixoazGKm8cgx4UZl+rbcXM7S8/13GjMySyjdtKnF/AfBB+JmEZl2rlGvWPKaDKvbBPS+DXxKh+kqe049GOYiRHNxxzjYw3X3d1bFshZEXyIiMBf4prZdoxHH0XmF5BkOTSqg7axjVERiOtiGTi0Ny3G5TzXDqkjkTKTU5vyIkfCYDu165UUqVHbxXFslyiIuBEs5k2nitONwI0ojNhTjbjqHnFKSlSumw26siLxSW5bh7W9aqN29KCnHL73Oq2OhLIunH93HYE4bXQ263aP+FI2+fR6fPw5gOBiut+JScelTZsm+Nqsx88jcmWkeOhyTDJ15CzSTBkg1oUyZKSSUkXNv58gHtc14pG5C1OYfk5VkJ+JEtbXoxuzVUmQhpSWXy6/VUpPXaJamm3EINXir4VGQd7gGpMPUU7ORUY3dQ4FbNkV5TpzbCGpL7Ehxh5LRJdU4ZJU1vyUhKTJadjMyUSQsgAAAAElpRNOy0tw2y+wxYV2ugrn/ALM9Hpe5OUZs+w8ODfDo79Lj00bcNuKfkQVoAAAAAAAAAAAAAAAAAAAAAAAJKxmmjVLHq08F7X16C5f+03R3929OTWl2HnwPj2rqdXj1E8vd+/FfHdsK0AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAElqvNOt0tzKy+wxZr2SgsX/sz0er774xnD7Dw4Oc+tt0uPTXvz24q+RhWgAAAAAAAAAAAAAAAAMq0C+8f8QLj8oBqoAAAAAAAMq0C+8f8QLj8oBqoAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAADKPaCZtfdWHWlVj9rce58vr7OTGrIipD/QaS6pZkhP+4tzMi3MiMy3AeXf75K6q+nPqAODo5Itb3UnUbMZeI5BRQ7n3P2Ru4gKiur6MdxtexHuR7GX8xn4KTvtvsA2MAAZHlGmWrM7VFeoeLajYbFYbhNwYMK8w+VZOwG/m+bDzVnHSk3T25KNo1bISW5kWwD22ujVzY282KnMorWI3Nsxe21R7pUqS9MbUys0tSuuSWmFrYQpbamXFHyUROJI/ANIlotFzITkGbFZjIdUc1t2Mp1bzfBRJS2snEk0ol8TNRpWRkRp2IzJRBJZ5pgeb3Ldud32Lp47cUHT7N1N+3EyXV35l+p0f1dvi5fNO3iHRK0I5Ynl2LnlP/wBtKK5HX7F/FuyxY7H6vU+Pl2fl807ctvHbcw9GQaGWVpTWlLFvcXnwre8n3EqBk2Jpt4C+0KJSUmz2hpfUbMjIlk4RGSlbo32Mg/avQH3bjuR0BZlIk+/8Qi4r2iRFNa2eimUXX8XPjI+1eCNy2Jsi5HvuQdxWaM1LUnMk3k4rSvzKvgV0mIqP0+m3GimwZ8uR7mrfkXgXE/6fmAkcg9mleQu01xbXGJ5Bf19OiilWGUYe3bJkxm3FradQ2p9HSkFzPk5yUhZmZm2XgRB3rejuTY5dHO02zyvx2LYQIFfbtuY40+64mIhSG3YnTcaYjOGhXEyUy63slPFtOx7h1dP7NNFRZ0vJoEbDuxOXL18ansPjvXPaXHTdUgrFazIm+oozLZknElsSXC2IwHskezmxJocLojy1aCxhTkewWmD4W1e4+h9yIpPU/gyU40zurdfwpUW3x7kHqt/ZpoJ+oFlmMWNhxsXdmzbWB2WHxrCzS8hDaFJjTHV8WkKJpO5LZcNJqWaVJMy4ho2nmGHgmPOUPvIp3UsrCx6vR6W3apbsjhx5K/V6vHffx477FvsQU4Drb+3TQ0sy5VWz7AobKnjiwI5vyXtv+K22XipR/wAxAJ7BNXdO9SVSI2I5MzIsIREc2qktORLKFvt4SIb6UPsn4/8AHQkBZgJzT2DmVdgONV+otrEtMsi08Jm+nREkliVYpZQUl5siQgiQp0lqIiQjwMvhT8iCjAAAAAAAAAAAAAAAAAAAAAAABOTYeYrz+nsINrFbxRios2bSCpBdd+xW9COE8g+BmSG2m7BKi5p3N5v4V7EaAowAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAATmoUHMrHAclr9OrWJV5ZKp5rNDOlpJTEWxUysozzhGhZGhLpoUZGhfgR/Cr5GFGAAAAAAAAAAAAAAAAAyrQL7x/wAQLj8oBqoAAAAAAAMq0C+8f8QLj8oBqoAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAzHWbFMOzGfgFTlGYZBjtknKHHscepHzZfkWJVFiS2VOE2vgjsapqzMzRubaS5eJIWHo7gvOrVX1H9MA7gvOrVX1H9MA7gvOrVX1H9MA7gvOrVX1H9MA7gvOrVX1H9MA7gvOrVX1H9MA7gvOrVX1H9MA7gvOrVX1H9MA7gvOrVX1H9MA7gvOrVX1H9MA7gvOrVX1H9MA7gvOrVX1H9MA7gvOrVX1H9MA7gvOrVX1H9MA7gvOrVX1H9MA7gvOrVX1H9MA7gvOrVX1H9MA7gvOrVX1H9MB11/oyqipplynVXWSyOGyp3skC+J2S/t/xW0GgiUo/5i3IBi157JWouuVjW3GXZHk2EV9avqwn7a7buMmZ3I/Fl1gksV7nj+sh2T/OWxbgLfQPSa0yvSHHLy51u1VdmOsusuuqyda1OG0+40S1KWlSjUokEZnv8zPYiLYiD322lLehcTTWqwjUTOVVLGSU2OM1k28W7FbgJI0pZJBJTuRJaSnYzMuO5GRgPpEAAAAAAAAAAAAAAAAAAAAAAAEPbQsNc1oxawm28tvLGMYv2ayClJ9B+uXLqTmvLPgZEtt1uvSkuadyec+FexmgLgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAEPrdDw2x0Yz6u1GtpVXikrGLVm+nREmp+LXKiuFJebIkLM1paNaiIkL8SL4VfIwuAAAAAAAAAAAAAAAAAGVaBfeP+IFx+UA1UAAAAAAAGVaBfeP+IFx+UA1UAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAASWYzTiZJgjH2F9/dtyB5j3h0ef2f2qp7nbt+CuHLp9j5bt/x7jyPlwWFaAAAAAAAAAAAAAAAAAAAAAAAAAAMp9lz+QvGf+e/3x4B+6+/dx+IFP+aA1UAAAAAAAAAAAAAAAAAAAAAAAElYzTRqlj1aeC9r69Bcv/abo7+7enJrS7Dz4Hx7V1Orx6ieXu/fivju2FaAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACS1XmnW6W5lZfYYs17JQWL/wBmej1fffGM4fYeHBzn1tulx6a9+e3FXyMK0AAAAAAAAAAAAAAAABlWgX3j/iBcflANVAAAAAAABlWgX3j/AIgXH5QDVQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABOZNCzOXdYo/jFtFh1sO4ceyJl5JGuZXHXy0IZaM0K2WUxcJwzI0fA0suR7mhYUYAAAAAAAAAAAAAAAAAAAAAAAAAAyn2XP5C8Z/57/fHgH7r793H4gU/5oDVQAAAAAAAAAAAAAAAAAAAAAAATk2HmK8/p7CDaxW8UYqLNm0gqQXXfsVvQjhPIPgZkhtpuwSouadzeb+FexGgKMAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAHTZZXX1xitxU4tkf2fuptfIj11v2NEv3fKW2pLUnoLMkO9NZpX01HxVx2PwMB3IAAAAAAAAAAAAAAAADKtAvvH/ABAuPygGqgAAAAAAAyrQL7x/xAuPygGqgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACHz+Fh0nK9Nn8otpcOyh5Q+9jjLKTNEyxOls0LZdMkK2QUNc1wjM0fG0guR7khYXAAAAAAAAAAAAAAAAAAAAAAAAAAA+c9GtRrXTvTanw+60e1JemwO0dVyLjylNK6khxwuJqUk/kst9yLx3/2gPVbarN66RNNbXCNO85TUv5JTZGzZzaNbUVyAojUl4lkpWxGl1KtzIi47mZkA+kQAAAAAAAAAAAAAAAAAAAAAAAQ9tCw1zWjFrCbby28sYxi/ZrIKUn0H65cupOa8s+BkS23W69KS5p3J5z4V7GaAuAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABlWgX3j/AIgXH5QDVQAAAAAAAZVoF94/4gXH5QDVQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABJZjNOJkmCMfYX3923IHmPeHR5/Z/aqnudu34K4cun2Plu3/HuPI+XBYVoAAAAAAAAAAAAAAAAAAAAAAAAAAAJLSiadlpbhtl9hiwrtdBXP/Zno9L3JyjNn2Hhwb4dHfpcemjbhtxT8iCtAAAAAAAAAAAAAAAAAAAAAAABJWM00apY9Wngva+vQXL/ANpujv7t6cmtLsPPgfHtXU6vHqJ5e79+K+O7YVoAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAyrQL7x/xAuPygGqgAAAAAAAyrQL7x/xAuPygGqgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACcyaFmcu6xR/GLaLDrYdw49kTLySNcyuOvloQy0ZoVsspi4ThmRo+BpZcj3NCwowAAAAAAAAAAAAAAAAAAAAE/mOVIxGoKzKitrqQ68iNFr6thLkmS8s/hSnmpDaC8DM1uLQhJFupREAkpGu+OQMBus4tMbyOI/j7z0ayojjNO2LElprrKZMmnVsGfSMl8yeNriZGayIBoddMbsYMawZJSW5TKHkEoiJRJUkjLfb+fxAcLKMjrcPxq1yq4U4mDTw3p0k20cl9JpBrVxL+c9iPYgHpxfIJ2R1zdhMxW2ouqy08hqwXFUs+aORp/7HedSRp32Pc9t/kai8QGQ4foTmkLF8Wrc71i1NsciVVsJv7CryNCYCLBDKOupBOIQ501u8+GzZmRF8RIAe/L9Mq7C66LZWusWrrrUuyhVaCj5AlSidlPoYbM+SCLiSnCNR777b7EZ+ADrdTsKZ0zpoNovUDW7IZNpZx6iFW0+SRSkPyHjPgRKlKZaSXwnuanC/wB4CPZv7vGM0w6Id3rDjt7IyOvZk0eY20CczNrJButOLScR19hREpJF/wAIS0GaT2LdJgPrYAAcMpb/ALzOD7skkyTBO9sNTfRNXIy6W3Pqcti5b8OOx/rb+ADmAACcwPNKzUDG2MppmJbMSQ/JYSiUhKXCUy8tlRmSVKLY1NqMvH5GXy+QCjAdDk2W1uKO0zdixJcO8tGaiP0UpPi84lakqXuotk7NnuZbn8vAB3wDgXFoxS1M24ktuLZgR3JLiWyI1GlCTUZERmRb7F4bmQD149dRMjoa3IoLTrca1hszWUukRLJDqCWklERmRHsot9jMt/5zAdmAAAAAnJsPMV5/T2EG1it4oxUWbNpBUguu/YrehHCeQfAzJDbTdglRc07m838K9iNAUYAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAyrQL7x/xAuPygGqgAAAAAAAyrQL7x/xAuPygGqgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACHz+Fh0nK9Nn8otpcOyh5Q+9jjLKTNEyxOls0LZdMkK2QUNc1wjM0fG0guR7khYXAAAAAAAAAAAAAAAAAAAAACF1hzu208wp26o8ftLawfkNQ2EwaabadmU4rY5DseG24+tpst1GSE7nsSd078iDLrvI8WrfZ3zD3HV6jXNhkEexiyHpGn923ZWNo/GUXUVE7Gl1DZ/AhK+mTKEpSglFxAa3p9kNXlGn9fZMVlyzFTDKO9GtqKZXyd20ElaTiymm3jI9jIvg2UX6u5GAnNXMapsp9nLJsbocQTJgSsXfbq6b3OptSdo5mw0mGtBKbWkyRxbNBKSZEWxGQDMZWli3c1i0iMCfaxpOT0K1RmK5TcMordK+hwjSlJI6JOGTay/V3VwV+tsYSeiuCQYGEYhWafw8wscBjV96jH7JmMtma/VSKqF2SS2pxDSUOv7rWkzS2RumvwI9yIOVV4I/Jo5mL49p84dOq6xhx2wgYTYYw/IJqybN1MiK7sT7iGi5rltJQk0maT2JIDu8t0ouo9hYYnp/TW+IU5alUM+BIx6qZQiCx2FByJUdt1hyMSSd5c1KaUjkauXxAID2i8B1DqnfdVxd5Pmd3MyKjTRZHYQWTJys5Om/Xm1BbjMk+TnJRkkm1vIW3sr4DNIWUDB7FrF7iTUY885h71vSOW+O1WnszH4sqEy6s5nRrX3nH3lKQprqpS0lLqWiSknT5EA9+ZYtW2TOLy8M05fqdNmHrM5FNeafWNtGOWtMfs76KNtxt9hvZL6U8mkEhZrV0y6hOGHvgaV5jY1yoBx7R2bCwtqTSzJtcuGTM+PauyobHA3Xjb4EllJNqdUsm9iVse5EHGyzGc2zOlrtTLXF3o0PJMlVNvKW4xaVcLYq2YbkeC1IrGXG3nkk6RPG38XFb5KNJ8TMg7PTTSo5WcYZKyXHZNhR11dfTK9uZjjtbDr1qnQnIraIjzryo5J4OOMtuqS4gi8EI4bEHYaMZ0uBp87pQzj+c1OXqfvGYrk3B7piA0+uTJcZcOcuKUUkGRpUS+rxPciIzMyIBHIwm5extiLpFp7eY1krGF2ETK5K6h+sXZWKm2SQg5DiUFOkG6mQpL6FuEnkpXULmXIO5u8Xrbqjq4/s66d3GDSk5LWrXNnYlMgQWXkNSkqke73eipRt8i5v9NKHOTX8I4SdiDq8lw+7kM4gm6wZLNDWQbOHd19vhVhljTl6bzJnM6MdxDj/AFUk6puWaVkRKURmhStiD2z8Fkxo7EHUrDMhzAl4G1X4u85jj0p6FZc5JuJV01PlBeNC4iSecdLwb8Xd0mA+jNNIkuDp1ikGdGdjSItJBaeZdQaFtrSwglJUk/EjIyMjI/EjIBUAAAAAIe2hYa5rRi1hNt5beWMYxfs1kFKT6D9cuXUnNeWfAyJbbrdelJc07k858K9jNAXAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAyrQL7x/wAQLj8oBqoAAAAAAAMq0C+8f8QLj8oBqoAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAksxmnEyTBGPsL7+7bkDzHvDo8/s/tVT3O3b8FcOXT7Hy3b/j3HkfLgsK0AAAAAAAAAAAAAAAAAAAAAAAAAAY7V+0lV3sBq0pdKNSbCG8aulJi0aXWl7KNJ8VpdMj2MjI9j+ZGQCLle1VA0r0porG59mPUyiXHgwYDWNU1Cy43DkKbSlMGKZLbS4hsyNCemgvgRyJBEWxB/O32hf9Lrr9m7k3GNMKBjTKu3Ww46Zdqt/AzSojdcSSWT8P+I2S0HvsszIjAf0k0i9oOxlaUYVJuNLdU7ae9j1a5KnlQ9UpbxxmzW9z6nx81bq5fz77gPZmeY3eqV1gtJRaUZ7XnX5bAtZcq2qUxYzMZhLinFKcU589jLYvmZmRF4mA+gAAAAAAAAAAAAAAAAAAAAcFdTVP20e9erYq7KHHeiR5imUm+yw8ptTrSHNuSULUwypSSPZRtNme/Etg5wAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAyrQL7x/xAuPygGqgAAAAAAAyrQL7x/xAuPygGqgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACcyaFmcu6xR/GLaLDrYdw49kTLySNcyuOvloQy0ZoVsspi4ThmRo+BpZcj3NCwowAAAAAAAAAAAAAAAAAAAAAAAAABlPsufyF4z/AM9/vjwD919+7j8QKf8ANAcPXL2TNAPaIhuNao6dV82wW3027iMns1izsWyTTIb2Uok/MkL5I/pSZbkA0jEcar8LxSlw+oU+qDRV8asim+vm4bLDaW0GtWxbq4pLc9i3MB3IAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAADKtAvvH/ABAuPygGqgAAAAAAAyrQL7x/xAuPygGqgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACHz+Fh0nK9Nn8otpcOyh5Q+9jjLKTNEyxOls0LZdMkK2QUNc1wjM0fG0guR7khYXAAAAAAAAAAAAAAAAAAAAAAAAAAAyn2XP5C8Z/57/fHgH7r793H4gU/5oDVQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAZVoF94/4gXH5QDVQAAAAAAAZVoF94/4gXH5QDVQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABJZjNOJkmCMfYX3923IHmPeHR5/Z/aqnudu34K4cun2Plu3/AB7jyPlwWFaAAAAAAAAAAAAAAAAAAAAAAAAAAMp9lz+QvGf+e/3x4B+6+/dx+IFP+aA1UAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAGVaBfeP+IFx+UA1UAAAAAAAGVaBfeP8AiBcflANVAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAGO1fs21dFAaq6XVfUmvhsmrpRot4lppG6jUfFCWiItzMzPYvmZmA67AfZ3uY1DhczVPVPL8jyihbrbCyJdv16522YQk3XGkOMpMmzd6nHwSrgrb4dwG5gAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAyrQL7x/wAQLj8oBqoAAAAAAAMq0C+8f8QLj8oBqoAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAIfRGHhtdoxgNdpzbSrTFIuMVTNDOlpNL8quTFbKM84RoQZLU0SFGRoR4mfwp+RBcAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAyrQL7x/xAuPygGqgAAAAAAAyqb7P1I5eXFzSZ/nuPleTl2UuJU3qmYxyVpSTjiUKSriajSRmW+25nsRF4AHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgHcF51aq+o/pgH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

