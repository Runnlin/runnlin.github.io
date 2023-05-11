# 使用LiveDataBus代替EventBus



> 对于Android系统来说，消息传递是最基本的组件，每一个App内的不同页面，不同组件都在进行消息传递。消息传递既可以用于Android四大组件之间的通信，也可用于异步线程和主线程之间的通信。对于Android开发者来说，经常使用的消息传递方式有很多种，从最早使用的Handler、BroadcastReceiver、接口回调，到前些年流行的通信总线类框架EventBus、RxBus。Android消息传递框架总在不断地演进中

## 从EventBus说起
EventBus是一个Android事件发布/订阅框架，通过解耦发布者和订阅者简化Android事件传递。EventBus可以代替Android传统的intent、Handler、Broadcast或接口回调，在Fragment、Activity、Service线程之间传递数据、执行方法。

EventBus最大的特点就是：简洁、解耦。在没有EventBus之前我们通常用广播来实现监听，或者自定义接口函数回调，有的场景我们也可以直接用Intent携带简单数据，或者在线程之间通过Handler处理消息传递。但无论是广播还是Handler机制远远不能满足我们高效的开发。EventBus简化了应用程序内各组件间、组件与后台线程间的通信。EventBus一经推出，便受到广大开发者的推崇。

现在看来，EventBus给Android开发者世界带来了一种新的框架和思想，就是消息的发布和订阅。这种思想在其后很多框架中都得到了应用。

![eventbus](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2018/7/27/164da80a8abe03d8~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)

图片摘自EventBus GitHub主页

## 发布/订阅模式

订阅发布模式定义了一种“一对多”的依赖关系，让多个订阅者对象同时监听某一个主题对象。这个主题对象在自身状态变化时，会通知所有订阅者对象，使它们能够自动更新自己的状态。

![发布/订阅模式](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2018/7/27/164da80d861343e9~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)

# RxBus的出现

RxBus不是一个库，而是一个文件，实现只有短短30行代码。RxBus本身不需要过多分析，它的强大完全来自于它基于的RxJava技术。响应式编程（Reactive Programming）技术这几年特别火，RxJava是它在Java上的实作。RxJava天生就是发布/订阅模式，而且很容易处理线程切换。所以，RxBus凭借区区30行代码，就敢挑战EventBus江湖老大的地位。

## RxBus原理

在RxJava中有个Subject类，它继承Observable类，同时实现了Observer接口，因此Subject可以同时担当订阅者和被订阅者的角色，我们使用Subject的子类PublishSubject来创建一个Subject对象（PublishSubject只有被订阅后才会把接收到的事件立刻发送给订阅者），在需要接收事件的地方，订阅该Subject对象，之后如果Subject对象接收到事件，则会发射给该订阅者，此时Subject对象充当被订阅者的角色。

完成了订阅，在需要发送事件的地方将事件发送给之前被订阅的Subject对象，则此时Subject对象作为订阅者接收事件，然后会立刻将事件转发给订阅该Subject对象的订阅者，以便订阅者处理相应事件，到这里就完成了事件的发送与处理。

最后就是取消订阅的操作了，RxJava中，订阅操作会返回一个Subscription对象，以便在合适的时机取消订阅，防止内存泄漏，如果一个类产生多个Subscription对象，我们可以用一个CompositeSubscription存储起来，以进行批量的取消订阅。

RxBus有很多实现，如：

AndroidKnife/RxBus（https://github.com/AndroidKnife/RxBus） Blankj/RxBus（https://github.com/Blankj/RxBus）

其实正如前面所说的，RxBus的原理是如此简单，我们自己都可以写出一个RxBus的实现：

### 基于RxJava1的RxBus实现：

```java
public final class RxBus {

    private final Subject<Object, Object> bus;

    private RxBus() {
        bus = new SerializedSubject<>(PublishSubject.create());
    }

    private static class SingletonHolder {
        private static final RxBus defaultRxBus = new RxBus();
    }

    public static RxBus getInstance() {
        return SingletonHolder.defaultRxBus;
    }

    /*
     * 发送
     */
    public void post(Object o) {
        bus.onNext(o);
    }

    /*
     * 是否有Observable订阅
     */
    public boolean hasObservable() {
        return bus.hasObservers();
    }

    /*
     * 转换为特定类型的Obserbale
     */
    public <T> Observable<T> toObservable(Class<T> type) {
        return bus.ofType(type);
    }
}
```

### 基于RxJava2的RxBus实现：

```java
public final class RxBus2 {

    private final Subject<Object> bus;

    private RxBus2() {
        // toSerialized method made bus thread safe
        bus = PublishSubject.create().toSerialized();
    }

    public static RxBus2 getInstance() {
        return Holder.BUS;
    }

    private static class Holder {
        private static final RxBus2 BUS = new RxBus2();
    }

    public void post(Object obj) {
        bus.onNext(obj);
    }

    public <T> Observable<T> toObservable(Class<T> tClass) {
        return bus.ofType(tClass);
    }

    public Observable<Object> toObservable() {
        return bus;
    }

    public boolean hasObservers() {
        return bus.hasObservers();
    }
}
```

# 引入LiveDataBus的想法

## 从LiveData谈起

LiveData是Android Architecture Components提出的框架。LiveData是一个可以被观察的数据持有类，它可以感知并遵循Activity、Fragment或Service等组件的生命周期。正是由于LiveData对组件生命周期可感知特点，因此可以做到仅在组件处于生命周期的激活状态时才更新UI数据。

LiveData需要一个观察者对象，一般是Observer类的具体实现。当观察者的生命周期处于STARTED或RESUMED状态时，LiveData会通知观察者数据变化；在观察者处于其他状态时，即使LiveData的数据变化了，也不会通知。

### LiveData的优点

-   **UI和实时数据保持一致** 因为LiveData采用的是观察者模式，这样一来就可以在数据发生改变时获得通知，更新UI。
-   **避免内存泄漏** 观察者被绑定到组件的生命周期上，当被绑定的组件销毁（destroy）时，观察者会立刻自动清理自身的数据。
-   **不会再产生由于Activity处于stop状态而引起的崩溃** 例如：当Activity处于后台状态时，是不会收到LiveData的任何事件的。
-   **不需要再解决生命周期带来的问题** LiveData可以感知被绑定的组件的生命周期，只有在活跃状态才会通知数据变化。
-   **实时数据刷新** 当组件处于活跃状态或者从不活跃状态到活跃状态时总是能收到最新的数据。
-   **解决Configuration Change问题** 在屏幕发生旋转或者被回收再次启动，立刻就能收到最新的数据。

### 谈一谈Android Architecture Components

Android Architecture Components的核心是Lifecycle、LiveData、ViewModel 以及 Room，通过它可以非常优雅的让数据与界面进行交互，并做一些持久化的操作，高度解耦，自动管理生命周期，而且不用担心内存泄漏的问题。

-   **Room** 一个强大的SQLite对象映射库。
-   **ViewModel** 一类对象，它用于为UI组件提供数据，在设备配置发生变更时依旧可以存活。
-   **LiveData** 一个可感知生命周期、可被观察的数据容器，它可以存储数据，还会在数据发生改变时进行提醒。
-   **Lifecycle** 包含LifeCycleOwer和LifecycleObserver，分别是生命周期所有者和生命周期感知者。

#### Android Architecture Components的特点

-   **数据驱动型编程** 变化的永远是数据，界面无需更改。
-   **感知生命周期，防止内存泄漏。**
-   **高度解耦** 数据，界面高度分离。
-   **数据持久化** 数据、ViewModel不与UI的生命周期挂钩，不会因为界面的重建而销毁。

## 重点：为什么使用LiveData构建数据通信总线LiveDataBus

### 使用LiveData的理由

-   **LiveData具有的这种可观察性和生命周期感知的能力，使其非常适合作为Android通信总线的基础构件。**
-   **使用者不用显示调用反注册方法。** 由于LiveData具有生命周期感知能力，所以LiveDataBus只需要调用注册回调方法，而不需要显示的调用反注册方法。这样带来的好处不仅可以编写更少的代码，而且可以完全杜绝其他通信总线类框架（如EventBus、RxBus）忘记调用反注册所带来的内存泄漏的风险。

### 为什么要用LiveDataBus替代EventBus和RxBus

-   **LiveDataBus的实现及其简单** 相对EventBus复杂的实现，LiveDataBus只需要一个类就可以实现。
-   **LiveDataBus可以减小APK包的大小** 由于LiveDataBus只依赖Android官方Android Architecture Components组件的LiveData，没有其他依赖，本身实现只有一个类。作为比较，EventBus JAR包大小为57kb，RxBus依赖RxJava和RxAndroid，其中RxJava2包大小2.2MB，RxJava1包大小1.1MB，RxAndroid包大小9kb。使用LiveDataBus可以大大减小APK包的大小。
-   **LiveDataBus依赖方支持更好** LiveDataBus只依赖Android官方Android Architecture Components组件的LiveData，相比RxBus依赖的RxJava和RxAndroid，依赖方支持更好。
-   **LiveDataBus具有生命周期感知** LiveDataBus具有生命周期感知，在Android系统中使用调用者不需要调用反注册，相比EventBus和RxBus使用更为方便，并且没有内存泄漏风险。

## LiveDataBus的设计和架构

### LiveDataBus的组成

-   **消息** 消息可以是任何的Object，可以定义不同类型的消息，如Boolean、String。也可以定义自定义类型的消息。
-   **消息通道** LiveData扮演了消息通道的角色，不同的消息通道用不同的名字区分，名字是String类型的，可以通过名字获取到一个LiveData消息通道。
-   **消息总线** 消息总线通过单例实现，不同的消息通道存放在一个HashMap中。
-   **订阅** 订阅者通过getChannel获取消息通道，然后调用observe订阅这个通道的消息。
-   **发布** 发布者通过getChannel获取消息通道，然后调用setValue或者postValue发布消息。

### LiveDataBus原理图

![LiveDataBus原理图](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2018/7/27/164da818e05d88e1~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)

## LiveDataBus的实现

#### 第一个实现：

```java
public final class LiveDataBus {

    private final Map<String, MutableLiveData<Object>> bus;

    private LiveDataBus() {
        bus = new HashMap<>();
    }

    private static class SingletonHolder {
        private static final LiveDataBus DATA_BUS = new LiveDataBus();
    }

    public static LiveDataBus get() {
        return SingletonHolder.DATA_BUS;
    }

    public <T> MutableLiveData<T> getChannel(String target, Class<T> type) {
        if (!bus.containsKey(target)) {
            bus.put(target, new MutableLiveData<>());
        }
        return (MutableLiveData<T>) bus.get(target);
    }

    public MutableLiveData<Object> getChannel(String target) {
        return getChannel(target, Object.class);
    }
}
```

短短二十行代码，就实现了一个通信总线的全部功能，并且还具有生命周期感知功能，并且使用起来也及其简单：

注册订阅：

```java
LiveDataBus.get().getChannel("key_test", Boolean.class)
        .observe(this, new Observer<Boolean>() {
            @Override
            public void onChanged(@Nullable Boolean aBoolean) {
            }
        });
```

发送消息：

```java
LiveDataBus.get().getChannel("key_test").setValue(true);
```

我们发送了一个名为"key_test"，值为true的事件。 这个时候订阅者就会收到消息，并作相应的处理，非常简单。

#### 问题出现

对于LiveDataBus的第一版实现，我们发现，在使用这个LiveDataBus的过程中，订阅者会收到订阅之前发布的消息。对于一个消息总线来说，这是不可接受的。无论EventBus或者RxBus，订阅方都不会收到订阅之前发出的消息。对于一个消息总线，LiveDataBus必须要解决这个问题。

#### 问题分析

怎么解决这个问题呢？先分析下原因：

当LifeCircleOwner的状态发生变化的时候，会调用LiveData.ObserverWrapper的activeStateChanged函数，如果这个时候ObserverWrapper的状态是active，就会调用LiveData的dispatchingValue。

![code](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2018/7/27/164da81e658d4ae8~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)

在LiveData的dispatchingValue中，又会调用LiveData的considerNotify方法。

![code](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2018/7/27/164da821ba1ff9ed~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)

在LiveData的considerNotify方法中，红框中的逻辑是关键，如果ObserverWrapper的mLastVersion小于LiveData的mVersion，就会去回调mObserver的onChanged方法。而每个新的订阅者，其version都是-1，LiveData一旦设置过其version是大于-1的（每次LiveData设置值都会使其version加1），这样就会导致LiveDataBus每注册一个新的订阅者，这个订阅者立刻会收到一个回调，即使这个设置的动作发生在订阅之前。

![code](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2018/7/27/164da825816a6d95~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)

#### 问题原因总结

对于这个问题，总结一下发生的核心原因。对于LiveData，其初始的version是-1，当我们调用了其setValue或者postValue，其vesion会+1；对于每一个观察者的封装ObserverWrapper，其初始version也为-1，也就是说，每一个新注册的观察者，其version为-1；当LiveData设置这个ObserverWrapper的时候，如果LiveData的version大于ObserverWrapper的version，LiveData就会强制把当前value推送给Observer。

#### 如何解决这个问题

明白了问题产生的原因之后，我们来看看怎么才能解决这个问题。很显然，根据之前的分析，只需要在注册一个新的订阅者的时候把Wrapper的version设置成跟LiveData的version一致即可。

那么怎么实现呢，看看LiveData的observe方法，他会在步骤1创建一个LifecycleBoundObserver，LifecycleBoundObserver是ObserverWrapper的派生类。然后会在步骤2把这个LifecycleBoundObserver放入一个私有Map容器mObservers中。无论ObserverWrapper还是LifecycleBoundObserver都是私有的或者包可见的，所以无法通过继承的方式更改LifecycleBoundObserver的version。

那么能不能从Map容器mObservers中取到LifecycleBoundObserver，然后再更改version呢？答案是肯定的，通过查看SafeIterableMap的源码我们发现有一个protected的get方法。因此，在调用observe的时候，我们可以通过反射拿到LifecycleBoundObserver，再把LifecycleBoundObserver的version设置成和LiveData一致即可。

![code](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2018/7/27/164da827d03c17d9~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)

对于非生命周期感知的observeForever方法来说，实现的思路是一致的，但是具体的实现略有不同。observeForever的时候，生成的wrapper不是LifecycleBoundObserver，而是AlwaysActiveObserver（步骤1），而且我们也没有机会在observeForever调用完成之后再去更改AlwaysActiveObserver的version，因为在observeForever方法体内，步骤3的语句，回调就发生了。

![code](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2018/7/27/164da82ad2eb99ed~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)

那么对于observeForever，如何解决这个问题呢？既然是在调用内回调的，那么我们可以写一个ObserverWrapper，把真正的回调给包装起来。把ObserverWrapper传给observeForever，那么在回调的时候我们去检查调用栈，如果回调是observeForever方法引起的，那么就不回调真正的订阅者。

### LiveDataBus最终实现

```java
public final class LiveDataBus {

    private final Map<String, BusMutableLiveData<Object>> bus;

    private LiveDataBus() {
        bus = new HashMap<>();
    }

    private static class SingletonHolder {
        private static final LiveDataBus DEFAULT_BUS = new LiveDataBus();
    }

    public static LiveDataBus get() {
        return SingletonHolder.DEFAULT_BUS;
    }

    public <T> MutableLiveData<T> with(String key, Class<T> type) {
        if (!bus.containsKey(key)) {
            bus.put(key, new BusMutableLiveData<>());
        }
        return (MutableLiveData<T>) bus.get(key);
    }

    public MutableLiveData<Object> with(String key) {
        return with(key, Object.class);
    }

    private static class ObserverWrapper<T> implements Observer<T> {

        private Observer<T> observer;

        public ObserverWrapper(Observer<T> observer) {
            this.observer = observer;
        }

        @Override
        public void onChanged(@Nullable T t) {
            if (observer != null) {
                if (isCallOnObserve()) {
                    return;
                }
                observer.onChanged(t);
            }
        }

        private boolean isCallOnObserve() {
            StackTraceElement[] stackTrace = Thread.currentThread().getStackTrace();
            if (stackTrace != null && stackTrace.length > 0) {
                for (StackTraceElement element : stackTrace) {
                    if ("android.arch.lifecycle.LiveData".equals(element.getClassName()) &&
                            "observeForever".equals(element.getMethodName())) {
                        return true;
                    }
                }
            }
            return false;
        }
    }

    private static class BusMutableLiveData<T> extends MutableLiveData<T> {

        private Map<Observer, Observer> observerMap = new HashMap<>();

        @Override
        public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<T> observer) {
            super.observe(owner, observer);
            try {
                hook(observer);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

        @Override
        public void observeForever(@NonNull Observer<T> observer) {
            if (!observerMap.containsKey(observer)) {
                observerMap.put(observer, new ObserverWrapper(observer));
            }
            super.observeForever(observerMap.get(observer));
        }

        @Override
        public void removeObserver(@NonNull Observer<T> observer) {
            Observer realObserver = null;
            if (observerMap.containsKey(observer)) {
                realObserver = observerMap.remove(observer);
            } else {
                realObserver = observer;
            }
            super.removeObserver(realObserver);
        }

        private void hook(@NonNull Observer<T> observer) throws Exception {
            //get wrapper's version
            Class<LiveData> classLiveData = LiveData.class;
            Field fieldObservers = classLiveData.getDeclaredField("mObservers");
            fieldObservers.setAccessible(true);
            Object objectObservers = fieldObservers.get(this);
            Class<?> classObservers = objectObservers.getClass();
            Method methodGet = classObservers.getDeclaredMethod("get", Object.class);
            methodGet.setAccessible(true);
            Object objectWrapperEntry = methodGet.invoke(objectObservers, observer);
            Object objectWrapper = null;
            if (objectWrapperEntry instanceof Map.Entry) {
                objectWrapper = ((Map.Entry) objectWrapperEntry).getValue();
            }
            if (objectWrapper == null) {
                throw new NullPointerException("Wrapper can not be bull!");
            }
            Class<?> classObserverWrapper = objectWrapper.getClass().getSuperclass();
            Field fieldLastVersion = classObserverWrapper.getDeclaredField("mLastVersion");
            fieldLastVersion.setAccessible(true);
            //get livedata's version
            Field fieldVersion = classLiveData.getDeclaredField("mVersion");
            fieldVersion.setAccessible(true);
            Object objectVersion = fieldVersion.get(this);
            //set wrapper's version
            fieldLastVersion.set(objectWrapper, objectVersion);
        }
    }
}
```

#### 注册订阅

```java
LiveDataBus.get()
        .with("key_test", String.class)
        .observe(this, new Observer<String>() {
            @Override
            public void onChanged(@Nullable String s) {
            }
        });
```

#### 发送消息：

```java
LiveDataBus.get().with("key_test").setValue(s);
```

### 源码说明

LiveDataBus的源码来源： https://github.com/JeremyLiao/LiveDataBus

# 总结

本文提供了一个新的消息总线框架——LiveDataBus。订阅者可以订阅某个消息通道的消息，发布者可以把消息发布到消息通道上。利用LiveDataBus，不仅可以实现消息总线功能，而且对于订阅者，他们不需要关心何时取消订阅，极大减少了因为忘记取消订阅造成的内存泄漏风险。

# Update

在项目使用的过程中，发现直接全编译打包出来的apk总会出现在执行hook中“getDeclaredMethod("get", Object.class)”时出现 NoSuchMethodException 问题，有可能是因为项目的特殊性导致混淆导致，同时使用反射的方式也不是Google所推荐的，下面尝试采用重写BusMutableLiveData的方式来重构：

```java
private class BusMutableLiveData<T> extends MutableLiveData<T>{

        int mCurrentVersion;
        /**
         * 是否需要更新数据,当主动调用setValue或者postValue的时候才触发
         */
        @Override
        public void setValue(T value) {
            mCurrentVersion++;
            super.setValue(value);
        }

        @Override
        public void postValue(T value) {
            mCurrentVersion++;
            super.postValue(value);
        }

        @Override
        public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
            super.observe(owner, new ObserverWrapper<T>(observer,mCurrentVersion,this));
        }
    }

    private class ObserverWrapper<T> implements Observer<T>{

        private Observer<? super T> mObserver;
        private int mVersion;
        private BusMutableLiveData<T> mLiveData;
        public ObserverWrapper(Observer<? super T> observer,int version,BusMutableLiveData<T> liveData) {
            mObserver = observer;
            mVersion = version;
            mLiveData = liveData;
        }

        @Override
        public void onChanged(T t) {
            if(mLiveData.mCurrentVersion>mVersion&&mObserver!=null){
                mObserver.onChanged(t);
            }
        }
    }
```

在这个方法中，我们不通过反射去获取系统的version信息，而是仿照系统自己弄一个mCurrentVersion，在新的观察者来的时候，吧当前mCurrentVersion传给他，setValue或者postValue的时候给通道的mCurrentVersion加一。

然后自定义一个观察者的包装类，ObserverWrapper，在其onChanged方法中判断通道的version大于自己的version的时候才更新数据。用法不变效果跟前面那个一样。

### 源码说明

更新的LiveDataBus的源码来源： https://blog.csdn.net/mingyunxiaohai/article/details/89605994
