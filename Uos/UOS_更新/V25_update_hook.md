 
# update hook

## hook点

* 开始更新前；
* 更新安装完成后、执行关机|重启前；
* 更新后第一次开机进入桌面前以及进入桌面后(后者存疑)；

## hook脚本生命周期

* 检查更新时，请求更新平台/api/v1/package接口；
* 解析接口返回内容，获取根据json字段：preCheck midCheck postCheck获取脚本内容；
* 进行更新前，生成(更新)/var/lib/lastore/online_meta.json文件，作为系统更新工具的配置文件，该文件中包含Rules(obj array), Rules中包含preCheck midCheck postCheck字段，对应脚本内容；
* 如果存在错误系统更新工具会通过扩展错误码的方式，返回固定错误码:1 << 30

## deepin-system-update 脚本处理机制

* deepin-system-update 脚本中,通过调用deepin-system-update check precheck|midcheck|postcheck命令,执行对应hook脚本;
* 在precheck|midcheck|postcheck各阶段中,每个阶段可以根据不同的rule name,执行多个脚本；

## 自测方法

* 控制中心开始安装更新前,会生成/var/lib/lastore/online_meta.json文件,其中包含Rules字段,该字段中包含preCheck midCheck postCheck字段,对应脚本内容;
* 可以通过命令,查看输出以及检验脚本成果:
  * sudo /usr/bin/deepin-system-update check precheck  --meta-cfg /var/lib/lastore/online_meta.json --ignore-warning --debug
  * sudo /usr/bin/deepin-system-update check midcheck  --meta-cfg /var/lib/lastore/online_meta.json --ignore-warning --debug
  * sudo /usr/bin/deepin-system-update check postcheck  --meta-cfg /var/lib/lastore/online_meta.json --ignore-warning --debug

```
DEBU[0000] load config                                  
DEBU[0000] load cache                                   
DEBU[0000] load update metadata                         
WARN[0000] Base line check empty                        
DEBU[0000] update info format verify                    
DEBU[0000] check verify update metadata                 
WARN[0000] Base line check empty                        
DEBU[0000] precheck start with ignore error:false warring:true 
DEBU[0000] cmd: bash [-x /tmp/dyn_450359321/00_precheck] [] 
DEBU[0000] cmd: bash [-c dpkg -l | tail -n +6 | awk '{print $1,$2,$3}'] [] 
DEBU[0000] skip line: []              
```

## 通过deb包获取脚本

* 通过更新平台获取脚本会具有一些局限性，因此需要一种从仓库获取hook脚本的机制；
* V20上lastore-daemon在检查更新完成后，会从系统仓库下载deepin-package-list包，然后解压该包获取核心包列表；
* 由于系统安全不会对脚本的执行做权限管控，因此deepin-package-list(或许需要改个名字)包中可以放置一组脚本，由lastore-daemon完成解压后，放置到执行目录，在三个check阶段由deepin-system-update负责执行；

## lastore-daemon脚本处理

* 获取到的脚本内容，缓存至本地，以rule name为文件名，放置到/var/lib/lastore/update-hook目录下；
* online_meta.json文件中只调用脚本，不直接把脚本内容塞进去；
* 脚本调用时重定向stdout和stderr到journal；
* 每次检查更新需要清理脚本缓存；
* 如果更新失败，需要将脚本的输出同步上报；

## 脚本要求

* 使用bash、shell编写；
* debug下需要开启set -x set -v；

## 遗留讨论

* 脚本的日志输出，用什么方式？ systemd-cat -t "myapp" echo "This log has a custom identifier" ？
* 脚本执行的结果，是否需要阻塞流程； 应该视具体情况而定，从仓库deb包和平台动态控制即可；
