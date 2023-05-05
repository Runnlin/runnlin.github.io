# MediaPlayer 错误收集及指北



# 项目中大量使用到了原生的 MediaPlayer 来播放音视频，这里记录一下遇到的问题及解决方式
  
  1. 切曲时出现 Failed to connect to surface ...
  
  LOG：
  ```log
    05-11 01:42:42.837 E 517      1356     BufferQueueProducer:   [SurfaceView - io.github.runnlin.exoplayerdemo/io.github.runnlin.exoplayerdemo.MainActivity#0](id:20500000107,api:3,p:561,c:517) connect: BufferQueue has been abandoned
    05-11 01:42:42.837 E 561      561      SurfaceUtils:  Failed to connect to surface 0xefb94788, err -19
    05-11 01:42:42.837 E 561      561      MediaPlayerService:  setVideoSurfaceTexture failed: -19
    05-11 01:42:42.837 D 561      561      NuPlayerDriver:  reset(0xf5a81a50) at state 2 
    05-11 01:42:42.838 D 561      5419     NuPlayerDriver:  notifyResetComplete(0xf5a81a50)
    05-11 01:42:42.840 W 4398     4398     System.err:  java.lang.IllegalStateException
    05-11 01:42:42.853 W 4398     4398     System.err:  at android.media.MediaPlayer.prepareAsync(Native Method)
    05-11 01:42:42.853 W 4398     4398     System.err:  at io.github.runnlin.exoplayerdemo.MainActivity.prepareMediaPlayer(MainActivity.kt:247)
    05-11 01:42:42.853 W 4398     4398     System.err:  at io.github.runnlin.exoplayerdemo.MainActivity.playMedia(MainActivity.kt:534)
    05-11 01:42:42.853 W 4398     4398     System.err:  at io.github.runnlin.exoplayerdemo.MainActivity.onPlayListener(MainActivity.kt:492)
  ```
  
  现象：切曲时在prepare阶段会闪退，报错为surface connect 失败
  
  分析原因：使用的surfaceView在播放结束时被回收了，需要重新setDisplay来再次让 mediaPlayer & surfaceHolder 绑定在一起
  
  解决方案：在prepare前进行 `setDisplay(null)`  然后在 surfaceCreated 回调中重新 `setDisplay(_playerView.holder)`
  
  2. 在播放结束时有概率出现 disconnect 超时，引发ANR
  
  分析原因：mediaPlayer放在了主线程，以及某些播放器使用了本地代理服务器来加速视频缓冲等事项，但同步进行接收代理服务器的回调时对方没有响应，导致ANR
  
  解决方案：mediaPlayer不要放在主线程，或是解决本地代理服务器回调超时问题
