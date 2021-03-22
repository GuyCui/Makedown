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