# Android MediaPlayer




## Android 播放器相关知识

译自 MediaPlayer官方文档  https://developer.android.com/reference/android/media/MediaPlayer

MediaPlayer 类可用于控制音视频文件和流的播放，其不是线程安全的，Player 实例的创建和访问都应该在同一个线程上，且线程必须有一个Looper以注册回调

音视频文件和流的播放控制作为状态机进行管理，下图显示了受支持的播放控制操作驱动的 MediaPlayer 对象的生命周期和状态。蓝椭圆表示MediaPlayer 对象可能驻留的状态，线段代表驱动对象状态转换的播放控制操作，其中单箭头的线段为同步调用，双箭头的为异步调用

![mediaplayer_state_diagram](https://developer.android.com/images/mediaplayer_state_diagram.gif)

从这个状态图中可以看出MediaPlayer 对象具有以下状态：

当 MediaPlayer 对象刚使用 new 创建或调用 reset() 后，处于 Idle 状态；在调用 release() 之后，其处于 End 状态，这两种状态之间是 MediaPlayer 对象的生命周期

- 新构造的对象和 reset() 后的对象有着席位但终于的区别：
  
  当 MediaPlayer 对象处于 Idle 状态时，
  
  `getCurrentPosition(),` `getDuration()`, 
  
  `getVideoHeight(), getVideoWidth(),` 
  
  `setAudioAttributes(android.media.AudioAttributes)`, 
  
  `setLooping(boolean)`, 
  
  `setVolume(float, float)`,
  
   `pause()`, `start()`, `stop()`, `seekTo(long, int)`, `prepare()` or `prepareAsync()`
  
  均不可调用，否则播放器会回调错误：  `OnErrorListener.onError()`

- 建议一旦不再使用 MediaPlayer 对象就立即调用 release() 以便立即释放与 MediaPlayer 对象关联的内部播放器引擎使用的资源。资源可能包括单例资源。一旦MediaPlayer 对象处于 End 状态，就不能再使用它，也无法将其恢复到任何其他状态

- 使用 new 创建的 MediaPlayer 对象会处于 Idle 状态，但使用 create 方法创建的不会处于 Idle 而是 Prepared 状态（已完成初始化）

通常，播放器的某些播放控制操作可能和由于各种原因而失败，例如不支持的音视频格式、音视频交错不良、分辨率太高、流媒体超时等。因此，在这些情况下错误报告和恢复是一个重要的事情。在这些所有错误的情况下，如果用户先通过 setOnErrorListener( android.media.MediaPlayer.OnErrorListener ) 注册了 OnErrorListener，则内部播放器引擎会调用用户提供的 OnErrorListener.onError() 方法。

- 一旦发生错误，即使没有注册OnErrorListener，MediaPlayer 对象也会进入 Error 状态
- 为了重用处于 Error 状态的MediaPlayer 对象并从错误中恢复，可以调用 reset() 将对象恢复成 Idle 状态
- 强烈建议用户自行实现 OnErrorListener
- 抛出 IIIegalStateException 以防止编程错误，如在无效状态调用 `prepare(), prepareAsync(), setDataSource `

调用 `setDataSource(java.io.FileDescriptor) `或 

`setDataSource(java.lang.String) `或

 `setDataSource(android.content.Context, android.net.Uri) `或

`` setDataSource(java.io.FileDescriptor, long, long)``，或 

`setDataSource(android.media.MediaDataSource) `将 Idle 状态的 MediaPlayer 对象转移到 Initialized 状态

- 在其他任何非Idle状态下调用 setDataSource() 会抛出 IIlegalStateException 异常
- 始终注意可能从重载的 setDataSource 方法抛出的 IIIlegalArgumentException 和 IOException 是一种很好的编程习惯

MediaPlayer 对象应先进入 Prepared 状态，才能开始播放

- 有两种方式（同步和异步）来达到 Prepared 状态：调用 prepare() （同步），一旦方法调用返回即可将对象切换成 Prepared 状态，或调用 prepareAsync() （异步）先进入 Preparing 状态，完成后再进入 prepared()。当成功进入 Prepared 状态时，如果用户实现了 OnPreparedListener 的 onPrepared() 方法，则其会被回调。
- Preparing 状态是瞬态状态，当 MediaPlayer 对象处于此状态时调用任何方法的结果都是不可预知的
- 在 Prepared 状态下，可以通过调用相应的 set 方法来调整音频音量、screenOnWhilePlaying、looping 等属性。

要开始播放，必须调用 start() 。start() 成功返回后，MediaPlayer 处于 Started 状态，可以调用 isPlaying 来测试

- 在 Started 状态下，如果预先通过`setOnBufferingUpdateListener(android.media.MediaPlayer.OnBufferingUpdateListener)` 注册了 OnBufferingUpdateListener，则内部播放器引擎会调用用户提供的 `OnBufferingUpdateListener.onBufferingUpdate()` 回调方法。此回调允许应用程序在播放流媒体时跟踪缓冲状态
- 在 started 状态中调用 Start() 方法不会对 MediaPlayer 对象产生任何影响

播放可以暂停或停止，当前的播放位置也可以调整。播放可以通过 pause() 来暂停，当对 pause() 的调用返回时，MediaPlayer 对象进入暂停状态。请注意，从起始状态到暂停状态的转换或与之相反的过程都是在播放引擎中异步发生的，在调用 isPlaying() 时更新状态可能需要一些时间，而在流媒体内容的情况下可能需要几秒钟。

- 调用 start() 以恢复暂停了的 MediaPlayer 对象的播放，并且恢复的回放位置与暂停的位置相同。当 start() 的调用返回时，暂停的MediaPlayer对象返回到 Started 状态。
- 调用 pause() 对已暂停的MediaPlayer 对象不产生影响

调用 stop() 来停止播放，并会将一个处于 Started, Paused, Prepared, PlaybackCompleted 状态的 MediaPlayer 对象进入 Stoped 状态

- 一旦进入了 stoped 状态，就无法启动播放，直到调用 prepare() 或 prepareAsync() 将MediaPlayer 对象再次设置为 Prepared 状态
- 调用 stop() 对已处于 stopped 状态的 MediaPlayer 对象没有影响

播放位置可以通过 seekTo(long, int)来调整

- 尽管异步的 seekTo(long, int)调用能够立即返回，但实际发生的查找操作可能需要一定的时间来完成，尤其是流媒体。当实际的查找操作完成时，如果事先通过 setOnSeekCompleteListener(android.media) 注册过 OnSeekCompleteListener，则内部播放器引擎将调用在用户注册的监听类
- 请注意，seekTo(long, int) 也可以在其他状态中调用，比如 Prepared、Paused和PlaybackCompleted 状态。在这些状态中调用 seekTo(long, int) 时，如果文件流具有视频且请求的位置有效，则将显示一个视频帧
- 此外，还能通过调用 getCurrentPosition() 检索当前的实际播放位置，这对于需要跟踪播放进度的应用程序（如音乐播放器）很有帮助

当播放到流的末尾时，播放完成

- 如果用 setLooping(boolean) 将循环模式设置为 true，则MediaPlayer对象将会保持在 started 状态
- 如果循环模式为false，则播放器引擎会回调用户注册了的 OnCompletion.onCompletion()，当该回调调用完成，说明MediaPlayer对象现在处于 PlaybackCompleted 状态
- 当MediaPlayer 对象处在 PlaybackCompleted 状态时，使用 start() 即可从开头重新播放媒体文件

