# USB 讲解笔记


> U盘内容按媒体资源类型可以分为 Music、Picture、Video三类，当读取U盘时，整个系统可以分为 Service 和 Client 两端

## Service

- USB-IF

- Indexing / Parsing

- Database

  将解析后的数据存放在数据库中备用

## Client

- UI
- Logic
- Data

Service 和 Client 通过 Android 的 IPC 来互相连通，主要是使用 AIDL 及 ContentProvider

## 详细说明

### USB-IF

Devices --[Insert]--> Kernal --[UEvent]--> User

USB设备插入后，内核会进行分析其设备类型，可以分为

- HUB
- U盘
- Apple 设备
- Android 或 Unknow

完成对设备类型的分析后，将通过 UEvent 的方式发送给 Service

**【避坑】**
整个程序项目在系统启动后才能启动，而在启动前就已插入的USB设备会因为无法接收到插入/拔出 Event 而无法识别，必须在 Service 启动后主动检索一次系统当前已加载的 USB 设备

### Indexing / Parsing

Index：检索所有文件，找到其中的媒体文件

**【说明】**
项目主要采用深度遍历的方式进行文件检索，这是一个先进后出的栈：从根目录开始将所有文件以先文件夹后文件的方式入栈，当前目录所有文件入栈后依次出栈分析，若出栈的是文件则加入索引，若出栈的是文件夹则索引该文件夹后，进入该文件夹内并再次以相同逻辑将内容所有文件入栈，如此反复即可递归地完成深度遍历所有文件

**【思考】**
若能以动态解析的方式进行索引，会大大提高用户体验

Parsing：解析文件，分为 Music、Picture、Video 等，并读取其详细内容，如

- music：取得 ID3 等数据，获得音乐的专辑、流派、艺术家等信息
- video：取得某关键帧作为缩略图，依需求取得视频长度等
- picture：取得宽高大小等信息

### Database

使用 Service 中封装好的 SQLiteOpenHelper 方法以及 ContentProvider 来对索引解析好的数据进行存储，这样在 Client 中即可通过 ContentResolver 来读取，具体是使用 HandleThread 的方式

### UI

响应控制及刷新UI：在 ui 获取到用户控制后，依业务情况通过 Proxy 发送到 Service，在其中逻辑完成后进行回调并刷新 UI，UI 层与 Service 层的联系方式是通过 StartService 或 BindService 方法

### Logic(Service)

获取并处理逻辑，提供 Binder 来数据交互，提供一系列 Interface 供使用

### Data

封装 SQLite

### U盘记忆

USB记忆有不同的方案，目前采用的方案是：存储当前U盘的容量信息，若容量与新设备相同则认作相同U盘，该方案有以下特点：

1. 不能辨别文件更改的情况，此时文件大小不变，但文件名变动不能辨认，该情况出现概率较小

2. 当U盘插入Android设备或其他设备时，可能会自动生成LOST.DIR或Android文件夹，其内容可能导致辨认错误，解决方案是在统计U盘内容大小时排除相关文件夹

