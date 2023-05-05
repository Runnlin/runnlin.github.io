# Android 四大组件综述



## 四大组件综述

    frameworks/base/services/core/java/com/android/server/am/
      - ActivityManagerService.java
      - ProcessRecord
      - ActivityStackSupervisor.java
      - ActivityStack.java
      - ActiveServices
      - BroadcastQueue


### 1. 进程管理

描述进程的数据结构：ProcessRecord
管理进程的核心模块：AMS

四大组件在AndroidManifest.xml 中可以用 android:process 指定运行的进程，同一个app可以运行在同一个、多个甚至多个app共享同一个进程

![AMS 管理进程的相关成员变量以及ProcessRecord对象](http://gityuan.com/images/ams/process_record.jpg)



1. mProcessNames 
    
    数据类型为ProcessMap，以进程名和userId为key来记录ProcessRecord
    - 添加进程：addProcessNameLocked
    - 删除进程：removeProcessNameLocked

2. mPidsSelfLocked

    

