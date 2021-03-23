# Mac 配置
### 查看 WIFI 密码
```shell
current_wifi=`/System/Library/PrivateFrameworks/Apple80211.framework/Versions/Current/Resources/airport -I | \
awk '/ SSID/ {print substr($0, index($0, $2))}'`  
security find-generic-password -a "$current_wifi" -g | tail -0
```

### 卸载 Java

```shell
sudo rm -rf /Library/Internet\ Plug-Ins/JavaAppletPlugin.plugin 
sudo rm -rf /Library/PreferencesPanes/JavaControlPanel.prefPane 
sudo rm -rf ~/Library/Application\ Support/Java 

ls /Library/Java/JavaVirtualMachines/ 查询 jdk 名称
sudo rm -rf /Library/Java/JavaVirtualMachines/查询到 jdk 的名称
```



### Mac 目录

#### Launchd 配置

##### 1.目录配置

Mac 下 Launchd 指定目录有：

- ~/Library/LaunchAgents
- /Library/LaunchAgents
- /Library/LaunchDaemons
- /System/Library/LaunchAgents
- /System/Library/LaunchDaemons

其中的区别：

- /System/Library 目录下存放的是系统文件
- /Library 、~/Library/ 目录是用户存放的第三方软件
- LaunchDaemons 是用户未登陆前就启动的服务
- LaunchAgents 是用户登陆后启动的服务

##### 2.Plist 配置

这里列举几个比较有用的配置关键字：

- Label - 标识符，用来表示该任务的唯一性
- Program - 程序名称，用来说明运行哪个程序、脚本
- ProgramArguments - 数组程序名，同上，只是可以运行多个程序
- WatchPaths - 监控路径，当路径文件有变化是运行程序，也是数组
- RunAtLoad - 是否在加载的同时启动

