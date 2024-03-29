# Android 大型项目media模块学习笔记(持续更新)


# **Android 大型项目media模块学习笔记(持续更新)**

## 整体架构

Media模块及大多数项目可以主要分为以下几个模块：

- Service
- View
- Proxy
- Data

### Service

1. 与底层jni连接
2. 确保其他模块正常运行
3. 负责基本操作，如U盘内容的解析
4. 提供解析数据
5. 存储解析数据到数据库

### View

1. 展示信息
2. 与Proxy通讯，获取信息更新，传递用户操作

### Proxy

1. 与View通讯，获取用户操作，进行相关操作或传递给其他模块
2. 维护 Handler

### Data

1. 对象或数据存储

## USB功能解析（USB Music 功能代码熟悉）

### **设备识别**

1. 开机（使用MEDIA_MOUNTEED广播为信号）启动MediaDeviceService服务，这是所有USB功能的基础服务，当服务启动完成后，在回调中创建设备插入处理线程，这样就不会影响UI线程；

2. 在设备状态改变的处理进程中处理刚插入的设备（handleDeviceAdd）：通过UsbInfoUtil的getUsbStorageStatus方法获取到设备信息及设备状态，进而处理未知设备或空设备情况，随后根据设备情况，开始进行USB索引和检索；

3. 设备识别完成后，除了索引外，还通过MediaDeviceBinder发送了设备类型和状态变化消息（此Binder包括设备所有变化，如索引/检索结果等），在该Binder中，自定义了一个名为IMediaDeviceListener的IInterface接口，其中也包括了各类USB设备变化方法，在实现中通过不同的code区分了binder中transact的不同数据对象，随后在其他模块中通过实现该IInterface接口即可异步获取到有关设备变化的数据；

### **连接/断开处理**

- HMI提示

1. 在实现了IMediaDeviceListener的LibDsvUsbMusic模块中，转发设备信息给了MusicPlayerService，随后通过Handler将设备信息转发，同时判断设备是插入还是拔出：

    - 若插入则通过状态判断是否已解析完成，完成了就可以建立音乐数据获取的线程；
    - 若拔出则通过Handler发送善后Msg（ACTION_CLEAR_METER、ACTION_ABANDON_FOCUS）及清理数据；

2. 在 Handler 接收的的这个设备信息状态改变 msg 触发了名为 DeviceStatusCallBack 的回调方法，在 Activity 和 Fragment 中都有创建这个 Callback 接口的实例，在其中重写该接口的 onDeviceStatusChange 方法中，有实例的 Activity 只进行判断是否移除设备然后finish()，而 UsbMusicFragment 中进行了 refreshView(status)：

    在 refreshView 中通过 Handler 发送 REFRESH_CURRENT_VIEW 的 msg，这是为了异步进行UI刷新不至于堵塞UI线程，随后在 Handler 处理中进行各类VIew更新，如插入和移除等

3. VIew更新时会实时移除或创建一系列View，当名为MusicLoadingVIew的view（view_usb_music_loading.xml）创建成功后，通过 MusicLoadingProxy 中的 startLoading 方法以动画形式显示该view，从而实现弹出自定义 Toast 的效果

- 回调连接/断开状态

    如开机识别章节中（3）中， MediaDeviceBinder 通过实现自定义的 aidl 来进行跨进程传输数据，在相应的模块接受到后通过 Handler 或其他方式进行处理，即为设备连接/断开状态回调

- 启动/停止USB文件扫描

    在 MediaDeviceService中，当确定设备插入时会进行设备判断分别处理，如果是刚插入未索引，会进行 MediaThreadMgr.getInstance().startIndex 来索引设备中的文件，如果索引完成发现没有媒体文件则弹出 Toast 说明没有可播放文件，如果索引完成发现有可播放的媒体文件则会进行解析

``` java
    // 已索引，无媒体文件
    if (devStatus.contains(MediaConstants.DeviceStatus.STATUS_NO_MUSIC) && 
devStatus.contains(MediaConstants.DeviceStatus.STATUS_NO_VIDEO) )   {
        ToastUtil
        .getInstance()
        .show(mContext, R.string.media_service_no_playable_files);
    }
    // 刚插入未索引
    if (devStatus.contains(MediaConstants.DeviceStatus.STATUS_MOUNTED)) {
        MediaThreadMgr
        .getInstance()
        .startIndex(mContext, addDevice);

    // 已索引，有媒体文件
    }else if (devStatus.contains(MediaConstants.DeviceStatus.STATUS_INDEXED)) {
        MediaThreadMgr
        .getInstance()
        .startParse(mContext, devStatus);
    }

```

### **列表数据库管理**

- 索引U盘文件

    在MediaThreadMgr 类中提供的索引方法：

``` java
    public void startIndex(Context context, String path) {
        LogUtil.d(TAG, "startIndex: path = " + path);
        DatabaseThread.getInstance().init(context);
        IndexingThread.getInstance().start(path);
    }

```

1. init：数据库线程的初始化，确保数据库正常运行且能随时响应数据库操作
2. start：
    1. 首先初始化索引线程，确保U盘可以被及时索引且响应对各类索引操作：
        - INDEX_NEXT_FOLDER

            1. 通过数据库线程插入正在索引的文件信息
            2. 给设备设置正在索引的状态（STATUS_INDEXING)，如果发现了有音乐文件（KEY_MUSIC)则再次给设备添加状态（STATUS_GOT_MUSIC），最后通过Handler发送索引状态改变通知

        - INDEX_EXCEPTION

            通过数据库线程插入正在索引的文件信息
        - INDEX_COMPLETED

            1. 回调告知索引结果，同时附上各类媒体文件的数量
            2. 通过Handler发送索引状态改变通知，通知内包括是否有媒体文件
            3. 结束索引线程

    2. 开始索引

        1. 判断 Handler 和 索引线程 是否为空，是则退出
        2. 重置统计过的各类文件数量
        3. 通过 Handler 通知设备状态变为索引中
        4. 数据库线程删除所有原本数据
        5. 从根目录开始，使用List分别遍历缓存文件和文件夹，依文件情况分别执行不同的索引线程中的Handler操作
        6. 在索引单个文件或文件夹时，创建一个FileInfo的模型，将文件中的设备类型（USB）、uuid、puid、文件名等信息放入，并判断文件类型（音乐/视频/图片），再用 mIndexingData List 存储，然后循环索引下一文件，直到缓存的文件队列为空，即 INDEX_COMPLETED
        7. 在数据库线程中插入改上述 List

- 解析U盘文件

1. init：同样是初始化数据库线程，同上
2. start：

    1. 初始化解析线程，索引操作：

        - ACTION_PARSE

            1. 判断是否解析完成，通过获取数据库中本文件数据中的音乐或视频数据大小，来分别进行音乐或视频的处理

                - 音乐文件处理
                    1. 通过原生的MediaMetadataRetriever 类从输入媒体文件中检索帧和元数据
                    2. 各种数据结构存储，将提取的数据存回文件中，放回数据库

                - 视频文件处理

                    同样使用原生的MediaMetadataRetriever 类，但只提取其视频长度

        - ACTION_GET_MUSIC_DATA

        通过 MediaDBManager 在数据库中获得对应的音乐文件信息

        - ACTION_GET_VIDEO_DATA

        通过 MediaDBManager 在数据库中获得对应的音乐文件信息

    2. 开始解析
       1. 更新设备状态为 PARSING
       2. 清楚数据库中原本解析文件
       3. 遍历进行对应的解析操作：先获取音/视频文件，再进行 ACTION_PARSE

- 数据排序

    打开USB音乐列表时，可以发现音乐播放列表或歌曲等已通过拼音进行排序，可以从这个页面着手分析数据排序的方法

    1. 打开 LibDsvUsbMusic 模块 proxy 中的 view 文件夹，其中包括了所有的音乐列表页面文件， 但其中大部分逻辑实现在 ui 文件夹内的 Proxy 文件中
    2. 分析 MusicLiberaryProxy 文件，可以发现里面定义了所有的 LiraryView ，当点开不同的音乐列表时，会通过不同的FragmentName启动不同的Fragment（handleFragmentLogic），同时传入不同内容的 bundle（里面装了 KEY_TYPE，是音乐文件夹类型，如 MUSIC_LIB_FAVORITE、 MUSIC_LIB_ARTIST 等）
    3. 在handleFragmentLogic中，显示获取到了 FragmentManager，然后通过此 manager 来实现滑动展示或隐藏子 Fragment，也是通过manager检索所有的子Fragment找到我们要打开的那个。
    4. 此处做了对 Album\Artist\Genre 三类文件夹的特殊处理，因为他们使用了不同的 Fragment ，其他的情况通过 Fragment 名字，再使用 FragmentTransaction.beginTransaction() 将目标Fragment展示出来
    5. 进入 MusicLibFragmentFolder 进行分析，首先引入眼帘的是mOnClickListener ，它规定了 mRecyclerView 的一些行为，如如果此时在滚动则点一下就能停下来。通过对 onLibItemClick 的重写，里面定义了点击文件夹或歌曲时的不同处理：
       - TYPE_FOLDER

        1. 将该文件夹加入栈（mListStack），同时使用线程来获得新的列表内容（#2）（这个线程使用 MediaThreadPoolMgr 中的线程池来管理，通过一个 Future 对象创建 task 即可将该多线程任务 submit 到线程池中进行），然后判断一下当前播放的歌曲是不是在这个点进去的文件夹里（isHightlight），如果是则再次使用新线程刷新列表同时传入显示那个正在播放高亮的曲目再进行列表刷新，否则直接进行列表刷新（Handler）。

        2. 在handler处理的 onRefreshListData 中，通过 MusicData 类提供的文件夹结构或文件结构来获取需要的 List\<FileInfo\>

        3. 最后进行adapter刷新列表,其中data来源是 albumList

       - TYPE_TRACK

            创建 ActInfo 对象，将获取到的音乐信息存入，通过 MusicPlayerService 播放

    6. 最后可以发现关于音乐的数据排序并不在view中完成，而是在从数据库取出信息时就已经排序好的状态，所以我们去 MusicDBProxy 看看：在MusicPlayerService中播放歌曲时，需要先取得音乐的数据， startGetMusicData 中可以看到取得ID3数据的方法中有调用 getID3DataFromID3Tables ，在这里面根据需要的音乐列表类型获得对应的列表，在取出列表后，通过 sortID3Struct 方法进行排序：直接用Collections的sort方法，通过预定的MediaSortRule对象设定要通过名字的拼音来排序，这样即可完成对音乐数据的排序

### **播放/显示/操作**

- **声音输出**
  
  前文提到过音乐是通过 MusicPlayerService 进行播放的，下面来看看具体播放流程
  
  在处理设备变化的Handler中，已经创建好了获取音乐数据的线程 getMusicDataTask ，在获取音乐数据的同时创建了播放器相关操作的线程（createPlayerTask），在 startUpMusicPlayer 方法中完成播放器的初始化和设置音乐数据路径给播放器，供给后续播放操作时使用

  在 startMusicPlayerService 方法中，启动音乐播放服务时用bundle传入了获取音乐信息时创建好的 ActInfo 对象，在 onStartCommand 中，判空该bundle，在 handleServiceCommand 中进行要做的动作识别，有很多，我们此处只分析 MUSIC_ACTION_PLAY 的情况。

  1. 获取音频焦点，通过handler来发送媒体信息（sendMediaInfoToMeter），这其中包括了播放源、播放数据整合等，然后使用binder发送，在 CarInfoBinder 接收到并解析资源后，通过 mCarVendorManager 即可完成对汽车硬件部分的控制，此处不表
  2. 开始播放，确定了以获得音乐焦点以及不是已经正在播放后，使用淡入淡出控制：
     1. 使用 VolumeUtil.fadeIn() 方法传入播放器对象和淡入回调，在 startFadeController 中先知晓开始音量和结束音量，当满足开始或结束要求时，回调函数
     2. 计算开始到结束的音频时长，创建 ValueAnimator 对象来取得平滑移动 Float 的效果（from to end），在 onAnimationUpdate 中实时地通过 mediaPlayer.setVolume() 方法来设置音量，最后设置回调开始或结束

  3. 最后用handler发送当前播放的状态，在接收中：
     1. 设置不暂停 setPauseByUser(false)
     2. 大数据记录使用情况 CarApiProxy.startProgramUsage()
     3. 通过 CarApiProxy 发送音乐信息

- **ID3显示（主界面/画廊界面）**
- **专辑图片解析显示**
- **播放进度显示**

  - 主界面

    > com.desaysv.libdsvusbmusic.proxy.ui.MusicPlayCtlProxy

    在这个view的proxy类中可以看到歌曲信息的显示方法，它们都是通过handler接受的信息来设置ID3信息到view中

    在 musicPlayerCallBack 中，除了ID3信息以外，还有包括歌曲播放位置、歌曲长度和播放状态的回调，我们去看看这个回调是如何触发的。

    在 MusicPlayerActivity 中进行MusicPlayerService 服务连接时，MUsicPlayCtlProxy 通过已连接的服务注册了上述回调，注册的方式是在服务中创建一个回调列表，当有需要时对列表中的每个回调进行调用。

    注册完了就可以在initPlayControlView()中通过发送handler信息来异步初始化信息，handler的信息数据内容是使用服务来分别获取：

            sendHandlerMsg(MusicPlayerMsg.REFRESH_DURATION_TIME, mService.getDuration());
            sendHandlerMsg(MusicPlayerMsg.REFRESH_CURRENT_TIME, mService.getPosition());
            sendHandlerMsg(MusicPlayerMsg.REFRESH_ID3_TITLE, mService.getID3Title());
            sendHandlerMsg(MusicPlayerMsg.REFRESH_ID3_ARTIST, mService.getID3Artist());
            sendHandlerMsg(MusicPlayerMsg.REFRESH_ID3_ALBUM, mService.getID3Album());
            sendHandlerMsg(MusicPlayerMsg.REFRESH_ID3_COVER, mService.getID3Cover());
            sendHandlerMsg(MusicPlayerMsg.REFRESH_FAVORITE_ICON, mService.isFavorite());

    最后在handler的信息处理中分别给每个view设置资源即可

    举例在mService.getID3Cover()中，return回来的 Bitmap 数据是在 startGetMusicID3() 方法中使用 mediaMetadataRetriever 对象进行获取的，从数据库获取到音乐文件的path后，即可通过mediaMetadataRetriever获取到各类ID3数据。mCurrentPlayFile 可以有一系列的来源，如播放上下曲和切换文件夹等，里面有不同的获取文件ID的方法。

  - 画廊

    根据项目结构，可以猜测画廊的ui处理相关类在proxy中，找到

    > com.desaysv.libdsvusbmusic.proxy.ui.MusicGalleryProxy

    可以看到里面有GalleryRecyclerView及其三大件：MusicGalleryAdapter、GalleryScaleHelper、GalleryLinearManager

    1. GalleryRecyclerView

    此处主要是定义了页面数据刷新和高亮定位的方法及各种播放控制信息的转发方法，其中onLibDataUpdate() 这个刷新数据的处理方法及onRefreshListData()来实现数据播放列表刷新过程；
    同时，还创建了一个独立线程来实时获取当前高亮位置（createGetPositionThread（）），确保能实时进行数据刷新或切换播放高亮位置；
    这里有个小细节：画廊列表中可播放列表的左右两边有填充的不可播放的图像，他是在onLibDataUpdate()中在实际播放列表前后循环加入的;

    2. MusicGalleryAdapter

    定义画廊内容填充，主要是其中有AlbumCoverLoader这个获取图片的工厂类，直接完成设置画廊中图片的任务，下面来一探究竟

    3. AlbumCoverLoader

    作为一个单例工厂类，除了设置各种变量的方法外，最重要的是其中的 loading() 方法：在线程池中加入任务，通过构造一个新的 MediaMetadataRetriever 对象来获取目标path的歌曲的图片，然后通过Bitmap.createScaledBitmap()方法来处理大小，最后用封装好了的 BitmapUtil提供的方法来修饰输出图片

    4. GalleryScaleHelper

    Helper类中使用了SnapHelper这个辅助类，用于辅助RecyclerView在滚动结束时将Item对齐到某个位置，其他包括滚动及缩放等效果都在此设置

    [SnapHelper详解]<https://www.jianshu.com/p/e54db232df62>

    该Helper还提供了OnPositionListener接口，通过在proxy实现接口来实现到达指定位置的回调

    5. GalleryLinearManager

    该类重写了部分LinearLayoutManager的scroller方法，也加入了是否允许滚动的方法

  - 播放控制（按钮）

    播放控制由

    > com.desaysv.libdsvusbmusic.proxy.ui.MusicPlayCtlProxy

    单独控制，其中的view包括了 view_usb_music_player.xml, view_usb_music_play_bar.xml 两种布局，都可以通过布局中的按钮实现控制。

    在播放控制的按钮中，以能否长按来区分了两种按钮监听，分别是 OnClickListener（单击）,OnTouchListener（可长按），在所有的按钮按下时都会创建一个ActInfo对象，使用MusicPlayerService.startMusicPlayerService(mContext, actInfo)来发送bundle封装的指令；

    在MusicPlayerService中，在handleServiceCommand中解析了所有事件，其中下一曲按钮的按下事件会在申请音频焦点后调用enableFfwFbwController()方法，在其中有个enable方法来创建一个Timer，在这个Timer中有个handleFfwFbwTimerTask()来分别对快进和快退进行处理：就是通过Timer更新seekTime自增或自减一个常量，然后调用notifyCurrentPositionChange(seekTime)和 sendHandlerMsg(MusicPlayerMsg.ACTION_SAVE_POSITION, seekTime) 来更新seekTime和刷新UI，实现快进或快退，当按钮松开时，这个Timer也即刻停止

    在上述的Timer中，定义了schedule方法使用一个常量LONG_CLICK_TIME来确定长按时间多长才算是长按，短按不会触发快进/快退效果，此时会直接return掉并紧接着出发松开效果：disableFfwFbwController中的diable方法中判断松开的按键是下一曲还是上一曲，分别进行对应方法

    下一曲：先判断当前能否进行下一曲后创建一个获取下一曲文件信息的线程，其中有startGetPlayFile方法来分别获取上一曲、下一曲、上一个文件夹、下一文件夹或指定歌曲播放；在MUSIC_ACTION_NEXT_FILE分支中将mCurrentPlayFile设为getNextOrPreFile(true)，这个方法中首先遍历播放列表的所有歌曲的uuid与当前播放的歌曲的uuid进行比对，获取当前歌曲处在列表位置的index，如果发现当前曲为列表最后一曲的化就将播放index设为0（列表最开始的曲目），否则对index进行自增返回，在遍历过程中还会对歌曲能否播放进行检查（info.isValid()），发现不能播放的就继续增加index，当遍历完整个列表后都无法播放的化则报出log.e提示。最后返回index位置的歌曲信息，上一曲同理，就是把index自减即可。

    进度条：在MusicPlayCtlProxy中有mSeekBarChangeListener监听，其中松开进度条时会调用onSeekToTime(seekBar.getProgress())，然后也是发送事件和参数到MusicPlayerService中：onSeekToTime(actInfo.getSeekTime())，我们先不管其他判断，直接看onFadeOutSeekTo(seekTime)，里面直接通过MPStateUtil.seekTo(musicPlayer, seekTime)来控制原生播放器跳转到指定播放位置。

    其他控制以此类推，均是通过构造一个ActInfo对象后，用MusicPlayerService.startMusicPlayerService(mContext, actInfo)来发送信息给MusicPlayerService进行相关操作，有部分需要注意的点：
    1. random模式时，service会清除当前播放列表，然后重新添加并shuffle，相当于构造一个内容一样但随机过后的list进行播放
    2. repeat模式时，如果不是文件夹循环则重新创建新的播放列表，如果是则进行判断循环文件夹的格式后，获取到对应的所有内容进行重建列表

  - 播放控制（线控）（未完待续）

    由软件控制可以知道，播放控制最终都是通过MusicPlayService里的对应方法来实现的，那么线控应该也是由其他地方向Service发送请求来实现控制，这里找到了
    > com.desaysv.libdsvmusic.usbExternalUsbMusicController

    ExternalUsbMusicController中实现了IMusicPlayer接口的方法，其中包括了开始、停止、下一曲等播放控制。

