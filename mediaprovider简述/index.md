# MediaProvider简述



## android MediaProvider

- The document base on Android 9.0

Android 系统提供了对多媒体的统一处理机制，通过一套良好的框架实现了多媒体信息的扫描、存储、读取。用户可以给予这套框架非常方便地对多媒体信息进行处理，这套框架主要包含三部分：

1. MediaScannerReceiver：多媒体扫描广播接收者，继承 BroadcastReceiver，主要响应APP发送的广播命令，并开启 MediaScannerService 执行扫描工作
2. MediaScannerService：多媒体扫描服务，继承Service，主要是处理APP发送的请求，用要到Framework中的MediaScanner来共同完成具体扫描工作，并获取媒体文件的metadata，最后将数据写入或删除MediaProvider提供的数据库中。
3. MediaProvider：多媒体内容提供者，继承ContentProvider，主要是负责操作数据库，并提供给别的程序insert、query、delete、update等操作。

本文就从上面三个部分作为入口，分析他们是如何工作的，如何对设备上的多媒体进行扫描，如何将多媒体信息进行存储，用户如何读取、修改多媒体信息？

### 1. 如何调用MediaScannerService

#### 1.1 MediaScannerActivity

我们可以从Android自带的 Dev Tools 中的MediaScannerActivity 入手，看看它是如何扫描多媒体的

    /development/apps/Development/src/com/android/development/MediaScannerActivity.java

```java
package com.android.development;

public class MediaScannerActivity extends Activity
{
    private TextView mTitle;

    /** Called when the activity is first created or resumed. */
    @Override
    public void onCreate(Bundle icicle) {
        super.onCreate(icicle);

        setContentView(R.layout.media_scanner_activity);

        IntentFilter intentFilter = new IntentFilter(Intent.ACTION_MEDIA_SCANNER_STARTED);
        intentFilter.addAction(Intent.ACTION_MEDIA_SCANNER_FINISHED);
        intentFilter.addDataScheme("file");
        registerReceiver(mReceiver, intentFilter);

        mTitle = (TextView) findViewById(R.id.title);
    }

    /** Called when the activity going into the background or being destroyed. */
    @Override
    public void onDestroy() {
        unregisterReceiver(mReceiver);
        super.onDestroy();
    }

    private final BroadcastReceiver mReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            if (intent.getAction().equals(Intent.ACTION_MEDIA_SCANNER_STARTED)) {
                mTitle.setText("Media Scanner started scanning " + intent.getData().getPath());
            }
            else if (intent.getAction().equals(Intent.ACTION_MEDIA_SCANNER_FINISHED)) {
                mTitle.setText("Media Scanner finished scanning " + intent.getData().getPath());
            }
        }
    };

    public void startScan(View v) {
        sendBroadcast(new Intent(Intent.ACTION_MEDIA_MOUNTED, Uri.parse("file://"
                + Environment.getExternalStorageDirectory())));

        mTitle.setText("Sent ACTION_MEDIA_MOUNTED to trigger the Media Scanner.");
    }
}
````

主要做了两件事：

- 注册扫描开始和结束的广播，用来展示扫描状态；
- 在点击事件中，发送 ACTION_MEDIA_MOUNTED 广播。

那么系统肯定存在一个接收者，在收到 ACTION_MEDIA_MOUNTED 后进行扫描，这就是 MediaScannerReceiver。

#### 1.2 MediaScannerReceiver

首先关注 AndroidManifest，对接收的广播一目了然

    /packages/providers/MediaProvider/AndroidManifest.xml

```xml
<receiver android:name="MediaScannerReceiver">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED" />
        <action android:name="android.intent.action.LOCALE_CHANGED" />
    </intent-filter>
    <intent-filter>
        <action android:name="android.intent.action.MEDIA_MOUNTED" />
        <data android:scheme="file" />
    </intent-filter>
    <intent-filter>
        <action android:name="android.intent.action.MEDIA_UNMOUNTED" />
        <data android:scheme="file" />
    </intent-filter>
    <intent-filter>
        <action android:name="android.intent.action.MEDIA_SCANNER_SCAN_FILE" />
        <data android:scheme="file" />
    </intent-filter>
</receiver>
```

    /packages/providers/MediaProvider/src/com/android/providers/media/MediaScannerReceiver.java

``` java
package com.android.providers.media;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.net.Uri;
import android.os.Bundle;
import android.os.Environment;
import android.provider.MediaStore;
import android.util.Log;

import java.io.File;
import java.io.IOException;

public class MediaScannerReceiver extends BroadcastReceiver {
    private final static String TAG = "MediaScannerReceiver";

    @Override
    public void onReceive(Context context, Intent intent) {
        final String action = intent.getAction();
        final Uri uri = intent.getData();
        if (Intent.ACTION_BOOT_COMPLETED.equals(action)) {
            // 开机广播，只处理内部存储
            scan(context, MediaProvider.INTERNAL_VOLUME);
        } else if (Intent.ACTION_LOCALE_CHANGED.equals(action)) {
            // 处理系统语言变换
            scanTranslatable(context);
        } else {
            if (uri.getScheme().equals("file")) {
                // 处理外部存储
                String path = uri.getPath();
                String externalStoragePath = Environment.getExternalStorageDirectory().getPath();
                String legacyPath = Environment.getLegacyExternalStorageDirectory().getPath();

                try {
                    path = new File(path).getCanonicalPath();
                } catch (IOException e) {
                    Log.e(TAG, "couldn't canonicalize " + path);
                    return;
                }
                if (path.startsWith(legacyPath)) {
                    path = externalStoragePath + path.substring(legacyPath.length());
                }

                Log.d(TAG, "action: " + action + " path: " + path);
                if (Intent.ACTION_MEDIA_MOUNTED.equals(action)) {
                    // 每当挂载外部存储时
                    scan(context, MediaProvider.EXTERNAL_VOLUME);
                } else if (Intent.ACTION_MEDIA_SCANNER_SCAN_FILE.equals(action) &&
                        path != null && path.startsWith(externalStoragePath + "/")) {
                    // 扫描单个文件，并且路径是在外部存储路径下
                    scanFile(context, path);
                }
            }
        }
    }

    // 扫描内部或者外部存储，根据volume进行区分
    private void scan(Context context, String volume) {
        Bundle args = new Bundle();
        args.putString("volume", volume);
        context.startService(new Intent(context, MediaScannerService.class).putExtras(args));
    }

    // 扫描单个文件，不可以是文件夹
    private void scanFile(Context context, String path) {
        Bundle args = new Bundle();
        args.putString("filepath", path);
        context.startService(new Intent(context, MediaScannerService.class).putExtras(args));
    }

    // 扫描可转换语言的多媒体
    private void scanTranslatable(Context context) {
        final Bundle args = new Bundle();
        args.putBoolean(MediaStore.RETRANSLATE_CALL, true);
        context.startService(new Intent(context, MediaScannerService.class).putExtras(args));
    }
}
```

再对比下Android 6.0中 MediaScannerReceiver源码：

``` java
package com.android.providers.media;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.net.Uri;
import android.os.Bundle;
import android.os.Environment;
import android.util.Log;

import java.io.File;
import java.io.IOException;

public class MediaScannerReceiver extends BroadcastReceiver {
    private final static String TAG = "MediaScannerReceiver";

    @Override
    public void onReceive(Context context, Intent intent) {
        final String action = intent.getAction();
        final Uri uri = intent.getData();
        if (Intent.ACTION_BOOT_COMPLETED.equals(action)) {
            // Scan both internal and external storage
            scan(context, MediaProvider.INTERNAL_VOLUME);
            scan(context, MediaProvider.EXTERNAL_VOLUME);

        } else {
            if (uri.getScheme().equals("file")) {
                // handle intents related to external storage
                String path = uri.getPath();
                String externalStoragePath = Environment.getExternalStorageDirectory().getPath();
                String legacyPath = Environment.getLegacyExternalStorageDirectory().getPath();

                try {
                    path = new File(path).getCanonicalPath();
                } catch (IOException e) {
                    Log.e(TAG, "couldn't canonicalize " + path);
                    return;
                }
                if (path.startsWith(legacyPath)) {
                    path = externalStoragePath + path.substring(legacyPath.length());
                }

                Log.d(TAG, "action: " + action + " path: " + path);
                if (Intent.ACTION_MEDIA_MOUNTED.equals(action)) {
                    // scan whenever any volume is mounted
                    scan(context, MediaProvider.EXTERNAL_VOLUME);
                } else if (Intent.ACTION_MEDIA_SCANNER_SCAN_FILE.equals(action) &&
                        path != null && path.startsWith(externalStoragePath + "/")) {
                    scanFile(context, path);
                }
            }
        }
    }

    private void scan(Context context, String volume) {
        Bundle args = new Bundle();
        args.putString("volume", volume);
        context.startService(new Intent(context, MediaScannerService.class).putExtras(args));
    }

    private void scanFile(Context context, String path) {
        Bundle args = new Bundle();
        args.putString("filepath", path);
        context.startService(new Intent(context, MediaScannerService.class).putExtras(args));
    }
}
```

扫描时机为以下几点：

1. `Intent.ACTION_BOOT_COMPLETED.equals(action)`

6.0 中接到设备重启的广播，对 Internal 和 External 扫描，而 9.0 中只对 Internal 扫描。

2. `Intent.ACTION_LOCALE_CHANGED.equals(action)`

9.0 相比 6.0 增加了系统语言发生改变时的广播，用于进行扫描可以转换语言的多媒体。

3. `uri.getScheme().equals("file")`

6.0 和 9.0 处理的一致，都是先过滤 scheme 为 "file" 的 Intent，再通过下面两个 action 对 External 进行扫描：

3.1 `Intent.ACTION_MEDIA_MOUNTED.equals(action)`

插入外部存储时扫描 scan()。

3.2 `Intent.ACTION_MEDIA_SCANNER_SCAN_FILE.equals(action) && path != null && path.startsWith(externalStoragePath + "/")`

扫描外部存储中的单个文件 scanFile()。

**注意：不支持扫描外部存储中的文件夹，需要遍历文件夹中的文件，使用扫描单个文件的方式。**

### 2. MediaScannerService如何工作？

MediaScannerService继承Service，并实现Runnable工作线程。通过ServiceHandler这个Handler把主线程需要大量计算的工作放到工作线程中。

    /packages/providers/MediaProvider/src/com/android/providers/media/MediaScannerService.java

#### 2.1 在onCreate() 中启动线程

``` java
@Override
public void onCreate() {
    // 获取电源锁，防止扫描过程中休眠
    PowerManager pm = (PowerManager)getSystemService(Context.POWER_SERVICE);
    mWakeLock = pm.newWakeLock(PowerManager.PARTIAL_WAKE_LOCK, TAG);
    // 获取外部存储扫描路径
    StorageManager storageManager = (StorageManager)getSystemService(Context.STORAGE_SERVICE);
    mExternalStoragePaths = storageManager.getVolumePaths();

    // Start up the thread running the service.  Note that we create a
    // separate thread because the service normally runs in the process's
    // main thread, which we don't want to block.
    Thread thr = new Thread(null, this, "MediaScannerService"); // 启动最重要的工作线程，该线程也是个消息泵线程
    thr.start();
}
```

可以看到，onCreate()里会启动最重要的工作线程，该线程也是个消息泵线程。每当用户需要扫描媒体文件时，基本上都是在向这个信息泵里发送Message，并在处理Message时完成真正的scan动作。请注意，创建Thread时传入的第二个线程就是MediaScannerService自身，也就是说线程的主要行为其实就是MediaScannerService的run() 方法，该方法的代码如下：

``` java
@Override
public void run() {
    // reduce priority below other background threads to avoid interfering
    // with other services at boot time.
    // 设置进程优先级，媒体扫描比较费时，防止 CPU 一直被 MediaScannerService 占用
    // 这会导致用户感觉系统变得很慢
    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND +
            Process.THREAD_PRIORITY_LESS_FAVORABLE);
    Looper.prepare();

    mServiceLooper = Looper.myLooper(); // 消息looper
    mServiceHandler = new ServiceHandler(); // 发送消息的handler

    Looper.loop();
}
```

#### 2.2 向工作线程发送Message

比较常见的向消息泵(message pump)发送Message的做法是调用startService()，并在MediaScannerService的onStartCommand()方法里sendMessage()。比如，和MediaScannerService配套提供的MediaScannerReceiver，当它收到类似ACTION_BOOT_COMPLETED这样的系统广播时，就会调用自己的scan()或scanFile()方法，里面的startService() method would call onStartCommand() in service and further send the message, the method is selected as follows:

``` java
@Override
public int onStartCommand(Intent intent, int flags, int startId) {
    ...
    Message msg = mServiceHandler.obtainMessage();
    msg.arg1 = startId;
    msg.obj = intent.getExtras();
    mServiceHandler.sendMessage(msg); // 发送消息

    // Try again later if we are killed before we can finish scanning.
    return Service.START_REDELIVER_INTENT;
}
```

Another more common way to send a Message is to directly or indirectly call `bindService()`, that would gain a `IMediaScannerService` interface while bind success, then the outside world sends commands to MediaScannerService through this interface, requesting it to scan specific files or directories.

IMediaScannerService interface only offer two interface method:

- `void requestScanFile(String path, String mineType, IMediaScannerListener listener);`
- `void scanFile(String path, String mineType);`

The entity that handles these two requests is the mBinder object inside the service. The reference code is as follows:

``` java
private final IMediaScannerService.Stub mBinder = new IMediaScannerService.Stub() {
    public void requestScanFile(String path, String mimeType, IMediaScannerListener listener) {
        if (false) {
            Log.d(TAG, "IMediaScannerService.scanFile: " + path + " mimeType: " + mimeType);
        }
        Bundle args = new Bundle();
        args.putString("filepath", path);
        args.putString("mimetype", mimeType);
        if (listener != null) {
            args.putIBinder("listener", listener.asBinder());
        }
        startService(new Intent(MediaScannerService.this,
                MediaScannerService.class).putExtras(args));
    }

    public void scanFile(String path, String mimeType) {
        requestScanFile(path, mimeType, null);
    }
};
```

After all, it is still calling `startService()`.

When specifically handling messages in the message pump thread, the handleMessage() method of ServiceHandler is executed:

``` java
private final class ServiceHandler extends Handler {
    @Override
    public void handleMessage(Message msg) {
        Bundle arguments = (Bundle) msg.obj;
        String filePath = arguments.getString("filepath");

        try {
            if (filePath != null) {
                // scan one file
                ...
                try {
                    uri = scanFile(filePath, arguments.getString("mimetype"));
                } catch (Exception e) {
                    Log.e(TAG, "Exception scanning file", e);
                }
                ...
            } else if (arguments.getBoolean(MediaStore.RETRANSLATE_CALL)) {
                // switch language
                ContentProviderClient mediaProvider = getBaseContext().getContentResolver()
                    .acquireContentProviderClient(MediaStore.AUTHORITY);
                mediaProvider.call(MediaStore.RETRANSLATE_CALL, null, null);
            } else {
                // scan internal or external
                String volume = arguments.getString("volume");
                String[] directories = null;

                if (MediaProvider.INTERNAL_VOLUME.equals(volume)) {
                    // while internal, the path is
                    directories = new String[] {
                            Environment.getRootDirectory() + "/media",
                            Environment.getOemDirectory() + "/media",
                            Environment.getProductDirectory() + "/media",
                }
                else if (MediaProvider.EXTERNAL_VOLUME.equals(volume)) {
                    // while external, the path is 
                    if (getSystemService(UserManager.class).isDemoUser()) {
                        directories = ArrayUtils.appendElement(String.class,
                                mExternalStoragePaths,
                                Environment.getDataPreloadsMediaDirectory().getAbsolutePath());
                    } else {
                        directories = mExternalStoragePaths;
                    }
                }
                // 调用 scan 函数开展文件夹扫描工作
                if (directories != null) {
                    scan(directories, volume);
                }
            }
        } catch (Exception e) {
            Log.e(TAG, "Exception in handleMessage", e);
        }

        stopSelf(msg.arg1); // 扫描结束，MediaScannerService完成本次使命，可以stop自身了
    }
}
```

the `scanFile()` method in MediaScannerService is use for scan a single file:

``` java
private Uri scanFile(String path, String mimeType) {
    String volumeName = MediaProvider.EXTERNAL_VOLUME;

    try (MediaScanner scanner = new MediaScanner(this, volumeName)) {
        // make sure the file path is in canonical form
        String canonicalPath = new File(path).getCanonicalPath();
        return scanner.scanSingleFile(canonicalPath, mimeType);
    } catch (Exception e) {
        Log.e(TAG, "bad path " + path + " in scanFile()", e);
        return null;
    }
}
```

The `scan()`  method in MediaScannerService is use for scan internal or external path:

``` java
private void scan(String[] directories, String volumeName) {
    Uri uri = Uri.parse("file://" + directories[0]);
    // don't sleep while scanning
    mWakeLock.acquire();

    try {
        ContentValues values = new ContentValues();
        values.put(MediaStore.MEDIA_SCANNER_VOLUME, volumeName);
        // 通过 insert 这个特殊的 uri，让 MeidaProvider 做一些准备工作
        Uri scanUri = getContentResolver().insert(MediaStore.getMediaScannerUri(), values);
        // 发送开始扫描的广播
        sendBroadcast(new Intent(Intent.ACTION_MEDIA_SCANNER_STARTED, uri));

        try {
            if (volumeName.equals(MediaProvider.EXTERNAL_VOLUME)) {
                // 打开数据库文件
                openDatabase(volumeName);
            }
            // 创建媒体扫描器，并调用 scanDirectories 扫描目标文件夹
            try (MediaScanner scanner = new MediaScanner(this, volumeName)) {
                scanner.scanDirectories(directories);
            }
        } catch (Exception e) {
            Log.e(TAG, "exception in MediaScanner.scan()", e);
        }
        // 通过 delete 这个 uri，让 MeidaProvider 做一些清理工作
        getContentResolver().delete(scanUri, null, null);

    } finally {
        // 发送结束扫描的广播
        sendBroadcast(new Intent(Intent.ACTION_MEDIA_SCANNER_FINISHED, uri));
        mWakeLock.release();
    }
}
```

In the above code, the more complicated is the interaction between MediaScannerService and MediaProvider. MediaScannerService ususally uses some special Uri to handle database operations, and the MediaProvider would make some special handle for these Uri, such as open database file etc, we'll focus on this later in the MediaProvider, but let's go back to the logic of scanning.

scanFile() or scan() is where the actual scanning is performed. MediaScanner is manily used in the scanning action, which is the key to get through the java layer and the C++ layer. The scanning action will finally call a native function of MediaScanner, so the program process starts to go to the C++ layer.

Now, we can draw a schematic diagram:
![](http://upload-images.jianshu.io/upload_images/7345261-8e663017c1e395ed.png)

### 3. How MediaScanner works?

As the name suggests, the MediaScanner is just a "Media Scanner". it must get through the java layer and the C++ layer. Please pay attention to its two native functions: `native_init()` and `native_setup()` , and two important member variables: `mClient` and `mNativeContext`, which will be explained in detail later.

    /frameworks/base/media/java/android/media/MediaScanner.java

``` java
public class MediaScanner implements AutoCloseable {
    static {
        System.loadLibrary("media_jni");
        native_init();    // 将java层和c++层联系起来
    }
    ...
    private long mNativeContext;
    ...
    public MediaScanner(Context c, String volumeName) {
        native_setup();
        ...
    }
    ...
    // 一开始就具有明确的mClient对象
    private final MyMediaScannerClient mClient = new MyMediaScannerClient();
    ...
}
```

When the MediaScanner class is loaded, the dynamic link library "media_jni" is loaded at the same time, and `native_init()` is called to link the java layer and the C++ layer.

    /frameworks/base/media/jni/android_media_MediaScanner.cpp

``` java
// This function gets a field ID, which in turn causes class initialization.
// It is called from a static block in MediaScanner, which won't run until the
// first time an instance of this class is used.
static void
android_media_MediaScanner_native_init(JNIEnv *env)
{
    ALOGV("native_init");
    // Java 层 MediaScanner 类
    jclass clazz = env->FindClass(kClassMediaScanner);
    if (clazz == NULL) {
        return;
    }
    // Java 层 mNativeContext 对象（long 类型）保存在 JNI 层 fields.context 对象中
    fields.context = env->GetFieldID(clazz, "mNativeContext", "J");
    if (fields.context == NULL) {
        return;
    }
}
```

After analyzing the code, we found that there will be a class corresponding to MediaScanner in C++ layer, named `StrageFrightMediaScanner`. when the java layer creates the MediaScanner object, the constructor of MediaScanner calls `native_setup()`, which corresponding to the C++ layer is `android_media_MediaScanner_native_setup()`, the code is as follows:

    /frameworks/base/media/jni/android_media_MediaScanner.cpp

``` C++
static void
android_media_MediaScanner_native_setup(JNIEnv *env, jobject thiz)
{
    ALOGV("native_setup");
    // 创建 native 层的 MediaScanner 对象，StagefrightMediaScanner（frameworks/av/ 中定义）
    MediaScanner *mp = new StagefrightMediaScanner;

    if (mp == NULL) {
        jniThrowException(env, kRunTimeException, "Out of memory");
        return;
    }
    // 将 mp 指针保存在 Java 层 MediaScanner 类 mNativeContext 对象中
    env->SetLongField(thiz, fields.context, (jlong)mp);
}
```

The last sentence `env->SetLongField()` is actually assigning a value to the `mNativeContext`  field of the java layer MediaScanner.

Later, we will see that whenever the C++ layer performs a scanning action, it will create a `MyMediaScannerClient` object, which corresponds to the class of the same name in the java layer.

Let's draw a picture to illustrate:

![](http://upload-images.jianshu.io/upload_images/7345261-5d9d21e338a1c410.png)

#### 3.1 scanSingleFile() action

``` java
// this function is used to scan a single file
public Uri scanSingleFile(String path, String mimeType) {
    try {
        prescan(path, true); // ① 扫描前预准备

        File file = new File(path);
        if (!file.exists() || !file.canRead()) {
            return null;
        }

        // lastModified is in milliseconds on Files.
        long lastModifiedSeconds = file.lastModified() / 1000;

        // always scan the file, so we can return the content://media Uri for existing files
        // ② 扫描前预准备
        return mClient.doScanFile(path, mimeType, lastModifiedSeconds, file.length(),
                false, true, MediaScanner.isNoMediaPath(path));
    } catch (RemoteException e) {
        Log.e(TAG, "RemoteException in MediaScanner.scanFile()", e);
        return null;
    } finally {
        releaseResources();
    }
}
```

Let's look at the code at 1 first. The `prescan` function is more critial. First, let's think about a problem.

During the media scanning process, there is a headache, to give an example: suppose there are 100 media files in the SD card before a certain scan, there will be 100 records of these files in the database. Now delete 50 of those files, so when will the media database be updated?

MediaScanner takes this into account, and the main function of the prescan function is to traverse the datacase information obtained from the previous scan before scanning, and detect whether it is lost. If it is lost, delete it from the database.

Looking at the code at 2, with the help of `mClient.doScanFile()`, the `mClient` type here is `MyMediaScannerClient`.

``` java
public Uri doScanFile(String path, String mimeType, long lastModified,
        long fileSize, boolean isDirectory, boolean scanAlways, boolean noMedia) {
    try {
        FileEntry entry = beginFile(path, mimeType, lastModified,
                fileSize, isDirectory, noMedia);
        ...

        // rescan for metadata if file was modified since last scan
        if (entry != null && (entry.mLastModifiedChanged || scanAlways)) {
            if (noMedia) {
                result = endFile(entry, false, false, false, false, false);
            } else {
                ...
                // we only extract metadata for audio and video files
                if (isaudio || isvideo) {
                    mScanSuccess = processFile(path, mimeType, this);
                }

                if (isimage) {
                    mScanSuccess = processImageFile(path);
                }
                ...
                result = endFile(entry, ringtones, notifications, alarms, music, podcasts);
            }
        }
    } catch (RemoteException e) {
        Log.e(TAG, "RemoteException in MediaScanner.scanFile()", e);
    }
    ...
    return result;
}
```

Due to `MyMediaScannerClient` is a internal class of MediaScanner, so it can directer call `processFile()` in MediaScanner.

Now let's draw a call diagram of `MediaScannerService.scanFile()` :
![](http://upload-images.jianshu.io/upload_images/7345261-a6dd0c8e8e595a74.png)

#### 3.2 scanDirectories() action

``` java
public void scanDirectories(String[] directories) {
    try {
        prescan(null, true);  // 扫描前预准备
        ...
        for (int i = 0; i < directories.length; i++) {
            // native 函数，调用它来对目标文件夹进行扫描
            processDirectory(directories[i], mClient); 
        }
        ...
        postscan(directories);  // 扫描后处理
    } catch (SQLException e) {
        ...
    } finally {
        ...
    }
}
```

Let's draw a call diagram of `MediaScannerService.scan()`:

![](http://upload-images.jianshu.io/upload_images/7345261-04e11669643e4550.png)

### 4. Call to C++ layer

Watch this : [https://links.jianshu.com/go?to=https%3A%2F%2Fmy.oschina.net%2Fyouranhongcha%2Fblog%2F787223%23h3_17]

### 5. How MediaProvider works?

    /packages/providers/MediaProvider/src/com/android/providers/media/MediaProvider.java

#### 5.1 When the MediaProvider create database?

Generally, the creation of the database should be done in the `onCreate()` method of the contentProvider.

``` java
@Override
public boolean onCreate() {
    ...
    // DatabaseHelper缓存
    mDatabases = new HashMap<String, DatabaseHelper>();
    // 绑定内部存储数据库
    attachVolume(INTERNAL_VOLUME);
    ...
    String state = Environment.getExternalStorageState();
    if (Environment.MEDIA_MOUNTED.equals(state) ||
            Environment.MEDIA_MOUNTED_READ_ONLY.equals(state)) {
         // 如果已挂载外部存储，绑定外部存储数据库
        attachVolume(EXTERNAL_VOLUME);
    }
    ...
    return true;
}
```

Next analyze the attachVolume method

Create database:

- If the storageVolume had been linked, do nothing
- Else, query the ID if storage volume and create the corresponding database.

``` java
/**
 * Attach the database for a volume (internal or external).
 * Does nothing if the volume is already attached, otherwise
 * checks the volume ID and sets up the corresponding database.
 *
 * @param volume to attach, either {@link #INTERNAL_VOLUME} or {@link #EXTERNAL_VOLUME}.
 * @return the content URI of the attached volume.
 */
private Uri attachVolume(String volume) {
    ...
    // Update paths to reflect currently mounted volumes
    // 更新路径以反映当前装载的卷
    updateStoragePaths();

    DatabaseHelper helper = null;
    synchronized (mDatabases) {
        helper = mDatabases.get(volume);
        // 判断是否已经attached过了
        if (helper != null) {
            if (EXTERNAL_VOLUME.equals(volume)) {
                // 确保默认的文件夹已经被创建在挂载的主要存储设备上，
                // 对每个存储卷只做一次这种操作，所以当用户手动删除时不会打扰
                ensureDefaultFolders(helper, helper.getWritableDatabase());
            }
            return Uri.parse("content://media/" + volume);
        }

        Context context = getContext();
        if (INTERNAL_VOLUME.equals(volume)) {
            // 如果是内部存储则直接实例化DatabaseHelper
            helper = new DatabaseHelper(context, INTERNAL_DATABASE_NAME, true,
                    false, mObjectRemovedCallback);
        } else if (EXTERNAL_VOLUME.equals(volume)) {
            // 如果是外部存储的操作，只获取主要的外部卷 ID
            // Only extract FAT volume ID for primary public
            final VolumeInfo vol = mStorageManager.getPrimaryPhysicalVolume();
            if (vol != null) {// 判断是否存在主要的外部卷
                // 获取主要的外部卷
                final StorageVolume actualVolume = mStorageManager.getPrimaryVolume();
                // 获取主要的外部卷 ID
                final int volumeId = actualVolume.getFatVolumeId();

                // Must check for failure!
                // If the volume is not (yet) mounted, this will create a new
                // external-ffffffff.db database instead of the one we expect.  Then, if
                // android.process.media is later killed and respawned, the real external
                // database will be attached, containing stale records, or worse, be empty.
                // 数据库都是以类似 external-ffffffff.db 的形式命名的，
                // 后面的 8 个 16 进制字符是该 SD 卡 FAT 分区的 Volume ID。
                // 该 ID 是分区时决定的，只有重新分区或者手动改变才会更改，
                // 可以防止插入不同 SD 卡时数据库冲突。
                if (volumeId == -1) {
                    String state = Environment.getExternalStorageState();
                    if (Environment.MEDIA_MOUNTED.equals(state) ||
                            Environment.MEDIA_MOUNTED_READ_ONLY.equals(state)) {
                        // This may happen if external storage was _just_ mounted.  It may also
                        // happen if the volume ID is _actually_ 0xffffffff, in which case it
                        // must be changed since FileUtils::getFatVolumeId doesn't allow for
                        // that.  It may also indicate that FileUtils::getFatVolumeId is broken
                        // (missing ioctl), which is also impossible to disambiguate.
                        // 已经挂载但是sd卡是只读状态
                        Log.e(TAG, "Can't obtain external volume ID even though it's mounted.");
                    } else {
                        // 还没有挂载
                        Log.i(TAG, "External volume is not (yet) mounted, cannot attach.");
                    }

                    throw new IllegalArgumentException("Can't obtain external volume ID for " +
                            volume + " volume.");
                }

                // generate database name based on volume ID
                // 根据volume ID设置数据库的名称
                String dbName = "external-" + Integer.toHexString(volumeId) + ".db";
                // 创建外部存储数据库
                helper = new DatabaseHelper(context, dbName, false,
                        false, mObjectRemovedCallback);
                mVolumeId = volumeId;
            } else {
                // external database name should be EXTERNAL_DATABASE_NAME
                // however earlier releases used the external-XXXXXXXX.db naming
                // for devices without removable storage, and in that case we need to convert
                // to this new convention
                // 外部数据库名称应为EXTERNAL_DATABASE_NAME
                // 但是较早的版本对没有可移动存储的设备使用external-XXXXXXXX.db命名
                // 在这种情况下，我们需要转换为新的约定
                ...
                // 根据之前转换的数据库名，创建数据库
                helper = new DatabaseHelper(context, dbFile.getName(), false,
                        false, mObjectRemovedCallback);
            }
        } else {
            throw new IllegalArgumentException("There is no volume named " + volume);
        }
        // 缓存起来，标识已经创建过了数据库
        mDatabases.put(volume, helper);
        ...
    }

    if (EXTERNAL_VOLUME.equals(volume)) {
        // 给外部存储创建默认的文件夹
        ensureDefaultFolders(helper, helper.getWritableDatabase());
    }
    return Uri.parse("content://media/" + volume);
}
```

First pay attention to the related method `getPrimaryPhysicalVolume()`

    /frameworks/base/core/java/android/os/storage/StorageManager.java

``` java
// 获取主要的外部的 VolumeInfo
public @Nullable VolumeInfo getPrimaryPhysicalVolume() {
    final List<VolumeInfo> vols = getVolumes();
    for (VolumeInfo vol : vols) {
        if (vol.isPrimaryPhysical()) {
            return vol;
        }
    }
    return null;
}
```

    /frameworks/base/core/java/android/os/storage/VolumeInfo.java

``` java
// 判断该 VolumeInfo 是否是主要的，并且是外部的
public boolean isPrimaryPhysical() {
    return isPrimary() && (getType() == TYPE_PUBLIC);
}

// 判断该 VolumeInfo 是否是主要的
public boolean isPrimary() {
    return (mountFlags & MOUNT_FLAG_PRIMARY) != 0;
}
```

The following is the source of the analysis and creation of the database `DatabaseHelper`:

``` java
/**
 * Creates database the first time we try to open it.
 */
@Override
public void onCreate(final SQLiteDatabase db) {
    // 在此方法中对700版本以下的都会新建数据库
    updateDatabase(mContext, db, mInternal, 0, getDatabaseVersion(mContext));
}

/**
 * Updates the database format when a new content provider is used
 * with an older database format.
 */
@Override
public void onUpgrade(final SQLiteDatabase db, final int oldV, final int newV) {
    // 对数据库进行更新
    mUpgradeAttempted = true;
    updateDatabase(mContext, db, mInternal, oldV, newV);
}
```

It is emphasized here that the fromVersion obtained by the `getDatabaseVersion()` method is not the databse version, but the versionCode in
    /packages/providers/MediaProvider/AndroidManifest.xml

Now we had been found the method of create database `updateDatabase`, then roughly analyze this method:

``` java
/**
 * This method takes care of updating all the tables in the database to the
 * current version, creating them if necessary.
 * This method can only update databases at schema 700 or higher, which was
 * used by the KitKat release. Older database will be cleared and recreated.
 * @param db Database
 * @param internal True if this is the internal media database
 */
private static void updateDatabase(Context context, SQLiteDatabase db, boolean internal,
        int fromVersion, int toVersion) {
    ...
    // 对不同版本的数据库进行判断
    if (fromVersion < 700) {
        // 小于700，重新创建数据库
        createLatestSchema(db, internal);
    } else if (fromVersion < 800) {
        // 对700-800之间的数据库处理
        updateFromKKSchema(db);
    } else if (fromVersion < 900) {
        // 对800-900之间的数据库处理
        updateFromOCSchema(db);
    }
    // 检查audio_meta的_data值是否是不同的，如果不同就删除audio_meta，
    // 在扫描的时候重新创建
    sanityCheck(db, fromVersion);
}
```

Then there is another doubt. We know that the excution time of `onCreate()` of ContentProvider is earlier than that of `Application onCreate()`, so how to mount external storage after `onCreate()`?

Just search the place of `attachVolume()` been call, we can found in `insertInternal()`:

``` java
private Uri insertInternal(Uri uri, int match, ContentValues initialValues,
                           ArrayList<Long> notifyRowIds) {
    ...
    switch (match) {
        ...
        case VOLUMES:
        {
            String name = initialValues.getAsString("name");
            // 根据name绑定存储数据库
            Uri attachedVolume = attachVolume(name);
            ...
            return attachedVolume;
        }
    ...
}
```

Find the corresponding URI according to VOLUMES

    URI_MATCHER.addURI("media", null, VOLUMES);

The place where the `insertInternal()` method is called is in the
`insert()` method.

Then it means that there must be a URI that calls the `insert()` method and passes in "content://media/", which can be found in the `openDatabase()`:

    /packages/providers/MediaProvider/src/com/android/providers/media/MediaScannerService.java

``` java
private void openDatabase(String volumeName) {
    try {
        ContentValues values = new ContentValues();
        values.put("name", volumeName);
        getContentResolver().insert(Uri.parse("content://media/"), values);
    } catch (IllegalArgumentException ex) {
        Log.w(TAG, "failed to open media database");
    }         
}
```

The place where the openDatabase() method is called is when the external storage is scanned, and at this time, the DatabaseHelper is instantiated. The code of scan() has been analyzed in the previous section, for the convenience of viewing, the method is listed here:

``` java
private void scan(String[] directories, String volumeName) {
    Uri uri = Uri.parse("file://" + directories[0]);
    // don't sleep while scanning
    mWakeLock.acquire();

    try {
        ContentValues values = new ContentValues();
        values.put(MediaStore.MEDIA_SCANNER_VOLUME, volumeName);
        Uri scanUri = getContentResolver().insert(MediaStore.getMediaScannerUri(), values);

        sendBroadcast(new Intent(Intent.ACTION_MEDIA_SCANNER_STARTED, uri));

        try {
            if (volumeName.equals(MediaProvider.EXTERNAL_VOLUME)) {
                openDatabase(volumeName);
            }

            try (MediaScanner scanner = new MediaScanner(this, volumeName)) {
                scanner.scanDirectories(directories);
            }
        } catch (Exception e) {
            Log.e(TAG, "exception in MediaScanner.scan()", e);
        }

        getContentResolver().delete(scanUri, null, null);

    } finally {
        sendBroadcast(new Intent(Intent.ACTION_MEDIA_SCANNER_FINISHED, uri));
        mWakeLock.release();
    }
}
```

#### 5.2 MediaProvider update

``` java
@Override
public int (Uri uri, ContentValues initialValues, String userWhere,
        String[] whereArgs) {
    // 将uri进行转换成合适的格式，去除标准化
    uri = safeUncanonicalize(uri);
    int count;
    // 对uri进行匹配
    int match = URI_MATCHER.match(uri);
    // 返回查询的对应uri的数据库帮助类
    DatabaseHelper helper = getDatabaseForUri(uri);
    // 记录更新的次数
    helper.mNumUpdates++;
    // 通过可写的方式获得数据库实例
    SQLiteDatabase db = helper.getWritableDatabase();
    String genre = null;
    if (initialValues != null) {
        // 获取流派的信息，然后删除掉
        genre = initialValues.getAsString(Audio.AudioColumns.GENRE);
        initialValues.remove(Audio.AudioColumns.GENRE);
    }
    ...
    // 根据匹配的uri进行相应的操作
    switch (match) {
        case AUDIO_MEDIA:
        case AUDIO_MEDIA_ID:
        // 更新音乐人和专辑字段。首先从缓存中判断是否有值，如果有直接用缓存中的
        // 数据，如果没有再从数据库中查询是否有对应的信息，如果有则更新，
        // 如果没有插入这条数据.接下来的操作是增加更新次数，并更新流派
        ...
        case IMAGES_MEDIA:
        case IMAGES_MEDIA_ID:
        case VIDEO_MEDIA:
        case VIDEO_MEDIA_ID:
        // 更新视频，并且发出生成略缩图请求
        ...
        case AUDIO_PLAYLISTS_ID_MEMBERS_ID:
        // 更新播放列表数据
        ...
    }
    ...
}
```

From now on, update operation has been done.

#### 5.3 MediaProvider insert

About insert, there was two founction, one is a large number of inserts by bulkInsert() method passes in an array of ContentValues; the other is insert(), which passes in a single ContentValues. The following are analyzed separately:

``` java
@Override
public int bulkInsert(Uri uri, ContentValues values[]) {
    // 首先对传入的Uri进行匹配
    int match = URI_MATCHER.match(uri);
    if (match == VOLUMES) {
        // 如果是匹配的是存储卷，则直接调用父类的方法，进行循环插入
        return super.bulkInsert(uri, values);
    }
    // 对DatabaseHelper和SQLiteDatabase的初始化
    DatabaseHelper helper = getDatabaseForUri(uri);
    if (helper == null) {
        throw new UnsupportedOperationException(
                "Unknown URI: " + uri);
    }
    SQLiteDatabase db = helper.getWritableDatabase();
    if (db == null) {
        throw new IllegalStateException("Couldn't open database for " + uri);
    }

    if (match == AUDIO_PLAYLISTS_ID || match == AUDIO_PLAYLISTS_ID_MEMBERS) {
        // 插入播放列表的数据，在playlistBulkInsert中是开启的事务进行插入
        return playlistBulkInsert(db, uri, values);
    } else if (match == MTP_OBJECT_REFERENCES) {
        // 将MTP对象的ID转换成音频的ID，最终也是调用到playlistBulkInsert
        int handle = Integer.parseInt(uri.getPathSegments().get(2));
        return setObjectReferences(helper, db, handle, values);
    }

    ArrayList<Long> notifyRowIds = new ArrayList<Long>();
    int numInserted = 0;
    // insert may need to call getParent(), which in turn may need to update the database,
    // so synchronize on mDirectoryCache to avoid deadlocks
    synchronized (mDirectoryCache) {
         // 如果不满足上述的条件，则开启事务进行插入其他的数据
        db.beginTransaction();
        try {
            int len = values.length;
            for (int i = 0; i < len; i++) {
                if (values[i] != null) {
                    // 循环调用insertInternal去插入相关的数据
                    insertInternal(uri, match, values[i], notifyRowIds);
                }
            }
            numInserted = len;
            db.setTransactionSuccessful();
        } finally {
            // 结束事务
            db.endTransaction();
        }
    }

    // 通知更新
    getContext().getContentResolver().notifyChange(uri, null);
    return numInserted;
}

@Override
public Uri insert(Uri uri, ContentValues initialValues) {
    int match = URI_MATCHER.match(uri);

    ArrayList<Long> notifyRowIds = new ArrayList<Long>();
    // 只是调用insertInternal进行插入
    Uri newUri = insertInternal(uri, match, initialValues, notifyRowIds);

    // do not signal notification for MTP objects.
    // we will signal instead after file transfer is successful.
    if (newUri != null && match != MTP_OBJECTS) {
        // Report a general change to the media provider.
        // We only report this to observers that are not looking at
        // this specific URI and its descendants, because they will
        // still see the following more-specific URI and thus get
        // redundant info (and not be able to know if there was just
        // the specific URI change or also some general change in the
        // parent URI).
        getContext().getContentResolver().notifyChange(uri, null, match != MEDIA_SCANNER
                ? ContentResolver.NOTIFY_SKIP_NOTIFY_FOR_DESCENDANTS : 0);
        // Also report the specific URIs that changed.
        if (match != MEDIA_SCANNER) {
            getContext().getContentResolver().notifyChange(newUri, null, 0);
        }
    }
    return newUri;
}
```

#### 5.4 MediaProvider delete

``` java
@Override
public int delete(Uri uri, String userWhere, String[] whereArgs) {
    uri = safeUncanonicalize(uri);
    int count;
    int match = URI_MATCHER.match(uri);

    // handle MEDIA_SCANNER before calling getDatabaseForUri()
    if (match == MEDIA_SCANNER) {
        if (mMediaScannerVolume == null) {
            return 0;
        }
        DatabaseHelper database = getDatabaseForUri(
                Uri.parse("content://media/" + mMediaScannerVolume + "/audio"));
        if (database == null) {
            Log.w(TAG, "no database for scanned volume " + mMediaScannerVolume);
        } else {
            database.mScanStopTime = SystemClock.currentTimeMicro();
            String msg = dump(database, false);
            logToDb(database.getWritableDatabase(), msg);
        }
        if (INTERNAL_VOLUME.equals(mMediaScannerVolume)) {
            // persist current build fingerprint as fingerprint for system (internal) sound scan
            final SharedPreferences scanSettings =
                    getContext().getSharedPreferences(MediaScanner.SCANNED_BUILD_PREFS_NAME,
                            Context.MODE_PRIVATE);
            final SharedPreferences.Editor editor = scanSettings.edit();
            editor.putString(MediaScanner.LAST_INTERNAL_SCAN_FINGERPRINT, Build.FINGERPRINT);
            editor.apply();
        }
        mMediaScannerVolume = null;
        pruneThumbnails();
        return 1;
    }

    if (match == VOLUMES_ID) {
        detachVolume(uri);
        count = 1;
    } else if (match == MTP_CONNECTED) {
        synchronized (mMtpServiceConnection) {
            if (mMtpService != null) {
                // MTP has disconnected, so release our connection to MtpService
                getContext().unbindService(mMtpServiceConnection);
                count = 1;
                // mMtpServiceConnection.onServiceDisconnected might not get called,
                // so set mMtpService = null here
                mMtpService = null;
            } else {
                count = 0;
            }
        }
    } else {
        final String volumeName = getVolumeName(uri);
        final boolean isExternal = "external".equals(volumeName);

        DatabaseHelper database = getDatabaseForUri(uri);
        if (database == null) {
            throw new UnsupportedOperationException(
                    "Unknown URI: " + uri + " match: " + match);
        }
        database.mNumDeletes++;
        SQLiteDatabase db = database.getWritableDatabase();

        TableAndWhere tableAndWhere = getTableAndWhere(uri, match, userWhere);
        if (tableAndWhere.table.equals("files")) {
            String deleteparam = uri.getQueryParameter(MediaStore.PARAM_DELETE_DATA);
            if (deleteparam == null || ! deleteparam.equals("false")) {
                database.mNumQueries++;
                Cursor c = db.query(tableAndWhere.table,
                        sMediaTypeDataId,
                        tableAndWhere.where, whereArgs,
                        null /* groupBy */, null /* having */, null /* orderBy */);
                String [] idvalue = new String[] { "" };
                String [] playlistvalues = new String[] { "", "" };
                MiniThumbFile imageMicroThumbs = null;
                MiniThumbFile videoMicroThumbs = null;
                try {
                    while (c.moveToNext()) {
                        final int mediaType = c.getInt(0);
                        final String data = c.getString(1);
                        final long id = c.getLong(2);

                        if (mediaType == FileColumns.MEDIA_TYPE_IMAGE) {
                            deleteIfAllowed(uri, data);
                            MediaDocumentsProvider.onMediaStoreDelete(getContext(),
                                    volumeName, FileColumns.MEDIA_TYPE_IMAGE, id);

                            idvalue[0] = String.valueOf(id);
                            database.mNumQueries++;
                            Cursor cc = db.query("thumbnails", sDataOnlyColumn,
                                        "image_id=?", idvalue,
                                        null /* groupBy */, null /* having */,
                                        null /* orderBy */);
                            try {
                                while (cc.moveToNext()) {
                                    deleteIfAllowed(uri, cc.getString(0));
                                }
                                database.mNumDeletes++;
                                db.delete("thumbnails", "image_id=?", idvalue);
                            } finally {
                                IoUtils.closeQuietly(cc);
                            }
                            if (isExternal) {
                                if (imageMicroThumbs == null) {
                                    imageMicroThumbs = MiniThumbFile.instance(
                                            Images.Media.EXTERNAL_CONTENT_URI);
                                }
                                imageMicroThumbs.eraseMiniThumb(id);
                            }
                        } else if (mediaType == FileColumns.MEDIA_TYPE_VIDEO) {
                            deleteIfAllowed(uri, data);
                            MediaDocumentsProvider.onMediaStoreDelete(getContext(),
                                    volumeName, FileColumns.MEDIA_TYPE_VIDEO, id);

                            idvalue[0] = String.valueOf(id);
                            database.mNumQueries++;
                            Cursor cc = db.query("videothumbnails", sDataOnlyColumn,
                                        "video_id=?", idvalue, null, null, null);
                            try {
                                while (cc.moveToNext()) {
                                    deleteIfAllowed(uri, cc.getString(0));
                                }
                                database.mNumDeletes++;
                                db.delete("videothumbnails", "video_id=?", idvalue);
                            } finally {
                                IoUtils.closeQuietly(cc);
                            }
                            if (isExternal) {
                                if (videoMicroThumbs == null) {
                                    videoMicroThumbs = MiniThumbFile.instance(
                                            Video.Media.EXTERNAL_CONTENT_URI);
                                }
                                videoMicroThumbs.eraseMiniThumb(id);
                            }
                        } else if (mediaType == FileColumns.MEDIA_TYPE_AUDIO) {
                            if (!database.mInternal) {
                                MediaDocumentsProvider.onMediaStoreDelete(getContext(),
                                        volumeName, FileColumns.MEDIA_TYPE_AUDIO, id);

                                idvalue[0] = String.valueOf(id);
                                database.mNumDeletes += 2; // also count the one below
                                db.delete("audio_genres_map", "audio_id=?", idvalue);
                                // for each playlist that the item appears in, move
                                // all the items behind it forward by one
                                Cursor cc = db.query("audio_playlists_map",
                                            sPlaylistIdPlayOrder,
                                            "audio_id=?", idvalue, null, null, null);
                                try {
                                    while (cc.moveToNext()) {
                                        playlistvalues[0] = "" + cc.getLong(0);
                                        playlistvalues[1] = "" + cc.getInt(1);
                                        database.mNumUpdates++;
                                        db.execSQL("UPDATE audio_playlists_map" +
                                                " SET play_order=play_order-1" +
                                                " WHERE playlist_id=? AND play_order>?",
                                                playlistvalues);
                                    }
                                    db.delete("audio_playlists_map", "audio_id=?", idvalue);
                                } finally {
                                    IoUtils.closeQuietly(cc);
                                }
                            }
                        } else if (mediaType == FileColumns.MEDIA_TYPE_PLAYLIST) {
                            // TODO, maybe: remove the audio_playlists_cleanup trigger and
                            // implement functionality here (clean up the playlist map)
                        }
                    }
                } finally {
                    IoUtils.closeQuietly(c);
                    if (imageMicroThumbs != null) {
                        imageMicroThumbs.deactivate();
                    }
                    if (videoMicroThumbs != null) {
                        videoMicroThumbs.deactivate();
                    }
                }
                // Do not allow deletion if the file/object is referenced as parent
                // by some other entries. It could cause database corruption.
                if (!TextUtils.isEmpty(tableAndWhere.where)) {
                    tableAndWhere.where =
                            "(" + tableAndWhere.where + ")" +
                                    " AND (_id NOT IN (SELECT parent FROM files" +
                                    " WHERE NOT (" + tableAndWhere.where + ")))";
                } else {
                    tableAndWhere.where = ID_NOT_PARENT_CLAUSE;
                }
            }
        }

        switch (match) {
            case MTP_OBJECTS:
            case MTP_OBJECTS_ID:
                database.mNumDeletes++;
                count = db.delete("files", tableAndWhere.where, whereArgs);
                break;
            case AUDIO_GENRES_ID_MEMBERS:
                database.mNumDeletes++;
                count = db.delete("audio_genres_map",
                        tableAndWhere.where, whereArgs);
                break;

            case IMAGES_THUMBNAILS_ID:
            case IMAGES_THUMBNAILS:
            case VIDEO_THUMBNAILS_ID:
            case VIDEO_THUMBNAILS:
                // Delete the referenced files first.
                Cursor c = db.query(tableAndWhere.table,
                        sDataOnlyColumn,
                        tableAndWhere.where, whereArgs, null, null, null);
                if (c != null) {
                    try {
                        while (c.moveToNext()) {
                            deleteIfAllowed(uri, c.getString(0));
                        }
                    } finally {
                        IoUtils.closeQuietly(c);
                    }
                }
                database.mNumDeletes++;
                count = db.delete(tableAndWhere.table,
                        tableAndWhere.where, whereArgs);
                break;

            default:
                database.mNumDeletes++;
                count = db.delete(tableAndWhere.table,
                        tableAndWhere.where, whereArgs);
                break;
        }

        // Since there are multiple Uris that can refer to the same files
        // and deletes can affect other objects in storage (like subdirectories
        // or playlists) we will notify a change on the entire volume to make
        // sure no listeners miss the notification.
        Uri notifyUri = Uri.parse("content://" + MediaStore.AUTHORITY + "/" + volumeName);
        getContext().getContentResolver().notifyChange(notifyUri, null);
    }

    return count;
}
```

#### 5.5 MediaProvider query

``` java
public Cursor query(Uri uri, String[] projectionIn, String selection,
        String[] selectionArgs, String sort) {
    uri = safeUncanonicalize(uri);
    int table = URI_MATCHER.match(uri);
    List<String> prependArgs = new ArrayList<String>();
    // handle MEDIA_SCANNER before calling getDatabaseForUri()
    if (table == MEDIA_SCANNER) {
        if (mMediaScannerVolume == null) {
            return null;
        } else {
            // create a cursor to return volume currently being scanned by the media scanner
            MatrixCursor c = new MatrixCursor(
                new String[] {MediaStore.MEDIA_SCANNER_VOLUME});
            c.addRow(new String[] {mMediaScannerVolume});
            //直接返回的是有关存储卷的cursor
            return c;
        }
    }
    // Used temporarily (until we have unique media IDs) to get an identifier
    // for the current sd card, so that the music app doesn't have to use the
    // non-public getFatVolumeId method
    if (table == FS_ID) {
        MatrixCursor c = new MatrixCursor(new String[] {"fsid"});
        c.addRow(new Integer[] {mVolumeId});
        return c;
    }
    if (table == VERSION) {
        MatrixCursor c = new MatrixCursor(new String[] {"version"});
        c.addRow(new Integer[] {getDatabaseVersion(getContext())});
        return c;
    }
    //初始化DatabaseHelper和SQLiteDatabase
    String groupBy = null;
    DatabaseHelper helper = getDatabaseForUri(uri);
    if (helper == null) {
        return null;
    }
    helper.mNumQueries++;
    SQLiteDatabase db = null;
    try {
        db = helper.getReadableDatabase();
    } catch (Exception e) {
        e.printStackTrace();
        return null;
    }
    if (db == null) return null;
    // SQLiteQueryBuilder类是组成查询语句的帮助类
    SQLiteQueryBuilder qb = new SQLiteQueryBuilder();
    //获取uri里面的查询字符
    String limit = uri.getQueryParameter("limit");
    String filter = uri.getQueryParameter("filter");
    String [] keywords = null;
    if (filter != null) {
        filter = Uri.decode(filter).trim();
        if (!TextUtils.isEmpty(filter)) {
            //对字符进行筛选
            String [] searchWords = filter.split(" ");
            keywords = new String[searchWords.length];
            for (int i = 0; i < searchWords.length; i++) {
                String key = MediaStore.Audio.keyFor(searchWords[i]);
                key = key.replace("\\", "\\\\");
                key = key.replace("%", "\\%");
                key = key.replace("_", "\\_");
                keywords[i] = key;
            }
        }
    }
    if (uri.getQueryParameter("distinct") != null) {
        qb.setDistinct(true);
    }
    boolean hasThumbnailId = false;
    //对匹配的其他类型进行设置查询语句的操作
    switch (table) {
        case IMAGES_MEDIA:
                //设置查询的表是images
                qb.setTables("images");
                if (uri.getQueryParameter("distinct") != null)
                    //设置为唯一的
                    qb.setDistinct(true);
                break;
         //其他类型相类似
         ... ...
    }
    //根据拼装的搜索条件，进行查询
    Cursor c = qb.query(db, projectionIn, selection,
             combine(prependArgs, selectionArgs), groupBy, null, sort, limit);

    if (c != null) {
        String nonotify = uri.getQueryParameter("nonotify");
        if (nonotify == null || !nonotify.equals("1")) {
            //通知更新数据库
            c.setNotificationUri(getContext().getContentResolver(), uri);
        }
    }
    return c;
}
```

### 6. How MediaProvider update database?

From the perspective of the java layer, whether it is scanning a specific file or scanning a directory, it will eventually go to the `doScanFile()` of `MyMediaScannerClient` of the java layer. We have listed the code of this function in the previous article. In order to illustrate the problem, here are the important codes:

    /frameworks/base/media/java/android/media/MediaScanner.java

``` java
public Uri doScanFile(String path, String mimeType, long lastModified,
        long fileSize, boolean isDirectory, boolean scanAlways, boolean noMedia) {
    try {
        // ① beginFile
        FileEntry entry = beginFile(path, mimeType, lastModified,
                fileSize, isDirectory, noMedia);
        ...

        // rescan for metadata if file was modified since last scan
        if (entry != null && (entry.mLastModifiedChanged || scanAlways)) {
            if (noMedia) {
                result = endFile(entry, false, false, false, false, false);
            } else {
                // 正常文件处理走到这里
                ...
                // we only extract metadata for audio and video files
                if (isaudio || isvideo) {
                    // ② processFile 这边主要是解析媒体文件的元数据，以便后续存入到数据库中
                    mScanSuccess = processFile(path, mimeType, this);
                }

                if (isimage) {
                    mScanSuccess = processImageFile(path);
                }
                ...
                // ③ endFile
                result = endFile(entry, ringtones, notifications, alarms, music, podcasts);
            }
        }
    } catch (RemoteException e) {
        Log.e(TAG, "RemoteException in MediaScanner.scanFile()", e);
    }
    ...
    return result;
}
```

Look again at the `beginFile()` and `endFile()` in relation to MediaProvider.

`beginFile()` is to prepare a FileEntry for subsequent dealing with MediaProvider. FileEntry is defined as follows:

``` java
private static class FileEntry {
    long mRowId;
    String mPath;
    long mLastModified;
    int mFormat;
    boolean mLastModifiedChanged;

    FileEntry(long rowId, String path, long lastModified, int format) {
        mRowId = rowId;
        mPath = path;
        mLastModified = lastModified;
        mFormat = format;
        mLastModifiedChanged = false;
    }
    ...
}
```

Several member variables of FileEntry actually reflect the values of several columns when looking up the table.
The code snippet of `beginFile()` is as follows:

``` java
public FileEntry beginFile(String path, String mimeType, long lastModified,
        long fileSize, boolean isDirectory, boolean noMedia) {
    ...
    FileEntry entry = makeEntryFor(path); // 从MediaProvider中查出该文件或目录对应的入口
    ...
    if (entry == null || wasModified) {
        // 不管原来表中是否存在这个路径文件数据，这里面都会执行到
        if (wasModified) {
            // 更新最后编辑时间
            entry.mLastModified = lastModified;
        } else {
            // 如果前面没查到FileEntry，就在这里new一个新的FileEntry
            entry = new FileEntry(0, path, lastModified,
                    (isDirectory ? MtpConstants.FORMAT_ASSOCIATION : 0));
        }
        entry.mLastModifiedChanged = true;
    }
    ...
    return entry;
}
```

The call to `makeEntryFor()` will internally query the MediaProvider:

``` java
FileEntry makeEntryFor(String path) {
    String where;
    String[] selectionArgs;

    Cursor c = null;
    try {
        where = Files.FileColumns.DATA + "=?";
        selectionArgs = new String[] { path };
        c = mMediaProvider.query(mFilesUriNoNotify, FILES_PRESCAN_PROJECTION,
                where, selectionArgs, null, null);
        if (c.moveToFirst()) {
            long rowId = c.getLong(FILES_PRESCAN_ID_COLUMN_INDEX);
            int format = c.getInt(FILES_PRESCAN_FORMAT_COLUMN_INDEX);
            long lastModified = c.getLong(FILES_PRESCAN_DATE_MODIFIED_COLUMN_INDEX);
            return new FileEntry(rowId, path, lastModified, format);
        }
    } catch (RemoteException e) {
    } finally {
        if (c != null) {
            c.close();
        }
    }
    return null;
}
```

The definition of `FILES_PRESCAN_PROJECTION` used in the query statement is as follows:

``` java
private static final String[] FILES_PRESCAN_PROJECTION = new String[] {
        Files.FileColumns._ID, // 0
        Files.FileColumns.DATA, // 1
        Files.FileColumns.FORMAT, // 2
        Files.FileColumns.DATE_MODIFIED, // 3
};
```

Did you see that, specifically to check the last modification date of the file to be checked recorded in the MediaProvider. Returns a `FileEntry` if it can be found, and return null if an exception occurs during the query.
The lastModified parameter of `beginFile()` can be understood as the last modification date of the file to be checked obtained from the file system, which should be the most accurate.
The information recorded in the MediaProvider may be "older". By comparing the two "last modification dates" inside `beginFile()`, you can know whether the file has really changed. If it does change, it is necessary to adjust the `mLastModified` in FileEntry to the latest data.

Basically, `beginFile()` returns a `FileEntry`. If this stage fails to find the record corresponding to the file in the MediaProvider, the mRowId of the FileEntry object will be 0, and if it is found, it will be a non-zero value.
The opposite of beginFile() is `endFile()`, which is where the actual insertion or update of data to the MediaProvider database occurs. WHen FileEntry's mRowId is 0, it will consider calling:

``` java
result = mMediaProvider.insert(tableUri, values);
```

And when mRowId is a non-zero value, it will consider calling:

``` java
mMediaProvider.update(result, values, null, null);
```

This is the core code that changes the relevant information in the MediaProvider. The code snippet for `endFile()` is as follows:

``` java
private Uri endFile(FileEntry entry, boolean ringtones, boolean notifications,
        boolean alarms, boolean music, boolean podcasts)
        throws RemoteException {
    ...
    ContentValues values = toValues();
    String title = values.getAsString(MediaStore.MediaColumns.TITLE);
    if (title == null || TextUtils.isEmpty(title.trim())) {
        title = MediaFile.getFileTitle(values.getAsString(MediaStore.MediaColumns.DATA));
        values.put(MediaStore.MediaColumns.TITLE, title);
    }
    ...
    long rowId = entry.mRowId;
    if (MediaFile.isAudioFileType(mFileType) && (rowId == 0 || mMtpObjectHandle != 0)) {
        // Only set these for new entries. For existing entries, they
        // may have been modified later, and we want to keep the current
        // values so that custom ringtones still show up in the ringtone
        // picker.
        values.put(Audio.Media.IS_RINGTONE, ringtones);
        values.put(Audio.Media.IS_NOTIFICATION, notifications);
        values.put(Audio.Media.IS_ALARM, alarms);
        values.put(Audio.Media.IS_MUSIC, music);
        values.put(Audio.Media.IS_PODCAST, podcasts);
    } else if ((mFileType == MediaFile.FILE_TYPE_JPEG
            || mFileType == MediaFile.FILE_TYPE_HEIF
            || MediaFile.isRawImageFileType(mFileType)) && !mNoMedia) {
        ...
    }
    ...
    if (rowId == 0) {
        // 扫描的是新文件，insert记录。如果是目录的话，必须比它所含有的所有文件更早插入记录，
        // 所以在批量插入时，就需要有更高的优先权。如果是文件的话，而且我们现在就需要其对应
        // 的rowId，那么应该立即进行插入，此时不过多考虑批量插入。
        // New file, insert it.
        // Directories need to be inserted before the files they contain, so they
        // get priority when bulk inserting.
        // If the rowId of the inserted file is needed, it gets inserted immediately,
        // bypassing the bulk inserter.
        if (inserter == null || needToSetSettings) {
            if (inserter != null) {
                inserter.flushAll();
            }
            result = mMediaProvider.insert(tableUri, values);
        } else if (entry.mFormat == MtpConstants.FORMAT_ASSOCIATION) {
            inserter.insertwithPriority(tableUri, values);
        } else {
            inserter.insert(tableUri, values);
        }

        if (result != null) {
            rowId = ContentUris.parseId(result);
            entry.mRowId = rowId;
        }
    } else {
        ...
        mMediaProvider.update(result, values, null, null);
    }
    ...
    return result;
}
```

In addition to directly calling `mMediaProvider.insert()` to write data to the MediaProvider, there si another way in the function to use the inserter object, whose type is `MediaInserter`.
The `MediaInserter` is also write data to MediaProvider, eventually it will generally come to its `flush()` function, the code of which is as follows:

``` java
private void flush(Uri tableUri, List<ContentValues> list) throws RemoteException {
    if (!list.isEmpty()) {
        ContentValues[] valuesArray = new ContentValues[list.size()];
        valuesArray = list.toArray(valuesArray);
        mProvider.bulkInsert(tableUri, valuesArray);
        list.clear();
    }
}
```

