# **大纲**

* 近期漏洞案例
* 安全原则
* DBus 配置与接口设计要求(包括代码规范)
* Polkit 规范与交互方案
* Systemd service 权限管理方法
* deepin-security-loader 实现与使用

## **近期漏洞案例**

1. `deepin-offl``ine-update` https://pms.uniontech.com/bug-view-266143.html
   1. 接口入参为字符串,通过bash -c执行命令,通过命令注入,造成任意代码执行;
   2. 第一次修改使用二进制白名单方式做的修复,修复方案存在问题;
2. `deepin-offline-update` https://pms.uniontech.com/bug-view-319569.html
   1. 接口入参为字符串,通过bash -c执行命令,通过LD_PRELOAD绕过二进制路径白名单,造成任意代码执行;
   2. 接口挂载一个sfs,并执行sfs中指定位置的脚本,通过LD_PRELOAD绕过二进制路径白名单,造成任意代码执行;
   3. 接口收到非sfs文件时,会将压缩包中文件写入/etc/deepin-offline-update/update.list中,通过LD_PRELOAD绕过二进制路径白名单,造成系统配置文件被随意篡改;
3. `dde-daemon` https://pms.uniontech.com/bug-view-316595.html
   1. 设置用户自动登录的dbus接口,非本用户未鉴权管理员,导致在无管理员密码时,普通用户将管理员用户自动登录;
4. `deepin-debug-config` https://pms.uniontech.com/bug-view-288103.html
   1. 接口入参为字符串,并使用system接口运行;
   2. 接口无鉴权行为;
5. `uos-recovery` 通过二进制名称判断是否允许client的dbus调用,构建同名二进制攻击;
6. `deepin-debug-config` 使用system执行命令,命令注入;
7. `deepin-log-viewer` 命令注入、使用LD_PRELOAD绕过二进制路径判断;
8. `deepin-home-appstore-daemon` SetSystemProxy设置代理环境变量的接口没有筛选代理相关环境变量,被设置PATH环境变量后,执行二进制时导致权限泄漏;
9. `uos-recovery` 判断是否允许调用的方式,是否包含 `uos-recovery-gui`的前文字符,以及命令注入 (意外发现 context=default own service);
10. `deepin-defender` 使用LD_PRELOAD绕过二进制路径判断;导入任意压缩包内容到系统目录;使用文件名作为数据库的key,导致文件权限被修改;
11. `udcp-scms` 下载接口可以下载url内容到任意目录;
12. `dde-daemon` dbus接口从调用者获取环境变量,透传所有环境变量给子进程,调用者使用LD_PRELOAD让子进程执行的二进制链接外部so,造成root权限泄漏;
13. `dde-file-manager ` 通过dbus接口传递base64后的用户名和密码;

## **常见缺陷类型**

1. 访问控制实现方式不安全: 根据/proc/${pid}/下信息作为访问控制依据;
2. 特权接口未鉴权或鉴权无效;
3. 鉴权操作不规范:管理员鉴权和当前用户鉴权使用错误;
4. 配置不规范:root服务配置`<policy context="default">`<allow own=service name/>`</policy>`;
5. 二进制执行方式不规范:sh -c、bash -c、system等方式执行;
6. 数据传输不规范;

## **DBus****安全原则**

1. 最小权限:DBus 接口所在的进程应只拥有完成所需功能的最小权限,要求通过 systemd service 实现 DBus 系统服务的权限管理;
2. 访问控制:DBus 接口中的每个 method 都应明确访问策略和会话策略;
3. 密码管理:DBus 接口中若使用了密码功能,则应保证密钥使用的安全性。同时密码功能应以框架的形式进行实现,里面的算法、分组模式等可替换;
4. 数据保护:DBus 接口中的敏感数据应在传输时进行加密,且能够保护数据完整性;
5. 参数验证:DBus 接口在设计时应定义输入和输出参数的合法范围,实现时进行检查;
6. 失败安全:所有的错误或异常都应被正确的处理;
7. 零信任:不信任第三方,始终验证;如对调用者、文件、第三方接口;

## **DBus 配置与接口设计要求**

### **System ****DBus****启动方式**

* 所有在System bus(系统总线)上request name DBus的服务,都需要通过systemd service启动;
* System DBus 配置:
  * DBus Service配置文件内容：
    ```Plain
    Name=org.bluez
    Exec=/bin/false
    SystemdService=dbus-org.bluez.service
    ```
  * systemd service配置文件内容：
    ```Plain
    [Service]
    Type=dbus
    BusName=org.bluez
    ExecStart=/usr/libexec/bluetooth/bluetoothd
    ```

    * systemd service还有很多其他的配置用于设置服务的权限范围,此处只是讲解启动的基础配置;
    * 按照上游的习惯,一般会创建一个服务的别名,约定俗成的命名模式：dbus-{BusName}.service
    * 创建别名不作为强制要求,不过别名的使用可以提升兼容性,比如:
      ```Plain
      display-manager.service -> /lib/systemd/system/gdm3.service
      ```
    * 别名创建方式
      ```Plain
      [Install]
      Alias=dbus-org.bluez.service
      ```

### **System ****DBus****访问控制策略**

* DBus的访问策略一般基于dbus.conf实现;
* System DBus的dbus.conf文件,通常在/usr/share/dbus-1/system.d和/etc/dbus-1/system.d目录下;
* 通常分为允许所有 client 调用和仅某些 User|Group 可以调用;
* 通过conf配置文件可以根据interface、path、member明确定义访问控制策略;详细文档见 https://linux.die.net/man/1/dbus-daemon-1;
* 策略主要围绕 `<policy context="default">` `<policy group="">` `<policy user="">` `<policy context="mandatory">`展开;分别对应了默认策略,针对该组的策略,针对该用户的策略和强制策略;
* 各策略之间的生效优先级就是上述顺序,group和user之间无优先级;当策略重叠时,后应用的策略将覆盖先应用的策略;
* 通过`<deny>`和`<allow>`来控制禁用和允许;
* 常见控制配置:
  ```Plain
  send_interface="interface_name"
  send_member="method_or_signal_name"
  send_destination="service_name"
  send_type="method_call" | "method_return" | "signal" | "error"
  send_path="/path/name"
  receive_interface="interface_name"
  receive_member="method_or_signal_name"
  receive_sender="service_name"
  receive_type="method_call" | "method_return" | "signal" | "error"
  receive_path="/path/name"
  own="servicename" // 注意一般不要default配置可以own service;
  ```
* 单一`<deny>`规则可以指定 send_service、send_interface 和 send_type 等属性的组合,当所有属性都匹配上才会拒绝。例如`<deny send_interface="foo.bar" send_service="foo.blah"/>`会拒绝来自指定接口和指定服务的消息。要获得"或"的效果,需要指定多个`<deny>`规则;
* 当一个system dbus service需要对其中某些接口做仅User|Group调用以及某些接口允许所有client调用时,可以在 `<policy context="default">`中先allow send_,然后deny需要做限制的接口,最后在`<policy group="">`对接口做allow;
* 典型配置案例:

```XML
    <busconfig>
       <!-- Only root or user avahi can own the Avahi service -->
       <policy user="avahi">
          <allow own="org.freedesktop.Avahi"/>
       </policy>
       <policy user="root">
          <allow own="org.freedesktop.Avahi"/>
       </policy>

       <!-- Allow anyone to invoke methods on Avahi server, except SetHostName -->
       <policy context="default">
          <allow send_destination="org.freedesktop.Avahi"/>
          <allow receive_sender="org.freedesktop.Avahi"/>

          <deny send_destination="org.freedesktop.Avahi"
          send_interface="org.freedesktop.Avahi.Server" send_member="SetHostName"/>
       </policy>

       <!-- Allow everything, including access to SetHostName to users of the group "netdev" -->
       <policy group="netdev">
          <allow send_destination="org.freedesktop.Avahi"/>
          <allow receive_sender="org.freedesktop.Avahi"/>
       </policy>
       <policy user="root">
          <allow send_destination="org.freedesktop.Avahi"/>
          <allow receive_sender="org.freedesktop.Avahi"/>
       </policy>
    </busconfig>
```

* 使用注意

  * systemd service配置文件的User字段需要和该配置文件中允许own的用户或者group需要相匹配;
  * 在dbus conf配置中,配置了deny的操作,该操作对应的dbus消息会在dbus-daemon直接拦截,消息不会发送到服务端;
  * 通过配置调用者的group限制接口调用时,调用者的group可能来自于getent group中的静态配置,也有是可能通过systemd的SupplementaryGroups配置或Setgroups接口动态修改(V20上该机制在dbus 1.12.20.17-deepin1及以上版本支持,V23 V25都支持);
  * 如果一个项目中,有多个不同进程的system bus service,那么需要写多个conf文件;
  * 如果一个进程申请了多个dbus name,那么要写在一个conf文件中;
  * 不要和polkit机制混淆;
* 接口隐藏方案

  * 通过配置隐藏:
    ```XML
    <policy context="default">
       <deny send_destination="com.deepin.anything"   
             send_interface="org.freedesktop.DBus.Introspectable" 
             send_member="Introspect"/>
    </policy>
    ```
  * 修改代码隐藏：
    * 核心思路也是处理Introspect的内容;
    * qtdbus可以继承QDBusVirtualObject,然后override handleMessage即可,https://github.com/linuxdeepin/dde-api-proxy/blob/master/src/dbus-proxy/common/dbusproxybase.hpp#L202这个是V23已经有的例子;
    * godbus导出对象的时候不导出introspect内容即可;
  * 接口虽然实现了不可见,但是如果明确知道dbus name、path、interface和method或property,依旧是可以调用的;

### **代码规范要求**

* System DBus接口设计要求:
  * 禁止使用shell命令作为参数;
  * 禁止使用文件路径作为参数;使用文件路径作为参数的代码逻辑,全部改成使用fd传递数据;
  * 禁止参数值为敏感数据(password、token等);
  * 需要对输入的参数进行了参数类型、参数长度和参数值范围的验证;
  * 需要对输入的参数进行注入检查,检查字符: ~ < > " ' % ( ) & + \ ' " ~;
  * 需要对输入的参数进行了编码检查;
  * 应对参数的值进行了溢出检查;
  * method实现时应确保是线程安全的;
  * method中的耗时操作应实现为异步,通过信号通知执行结果;
  * method实现时应处理所有的错误,并使用系统的日志API记录日志;
  * method应正确的管理内存,确保不存在内存泄漏、double free、野指针等;
  * 应在ObjectPath中包含版本号;
* 检查调用者规范:
  * 禁止使用/proc/${pid}/下的内容作为是否允许调用的检查条件,尤其是exe、cmdline、comm;
  * 使用这些字段会存在以下风险:
    * 从dbus daemon获取到调用者的pid后,再去/proc/${pid}/下读取上述字段时,会存在pid重用攻击的可能;
    * 上述字段存在被伪造的可能:通过exec -a、修改二进制名称、namespace等手段;
    * 无法识别调用者的LD_PRELOAD等环境变量是程序设计还是恶意攻击;
  * 调用者仅为system service,通过上述的dbus conf方式通过user、group等方式限制调用者;
  * 同样不要从/proc/${pid}/status获取user和group,如果需要根据sender获取相关信息,需要通过 `org.freedesktop.DBus`的 `GetConnectionCredentials`接口获取;
  * 适当增加polkit鉴权;
* DBus接口参数为环境变量或者从调用者获取环境时,需要对环境变量做校验,以及仅获取需要用到的环境变量,禁止获取全部环境变量后直接传递给子进程或写环境变量文件;
* 使用fd传输数据可以解决下列问题:
  * 数据传输保护:防止敏感数据在传输过程中对外暴露,可以通过传递fd的方式,让被调用者通过读fd的内容完成本次数据传输;敏感数据传输通常使用memfd和pipefd完成;此时dbus monitor监听method的内容只能看到fd的inode,无法看到具体数据,建议调用方和被调用方适当增加日志,方便调试;
  * 任意文件读:传递路径容易出现软链接条件竞争导致任意文件读的问题,案例：https://pms.uniontech.com/bug-view-249773.html 和 https://pms.uniontech.com/zentao/bug-view-38623.html
* 命令执行规范:
  * 尽量避免直接执行外部命令;
  * 通过外部接口接受的数据作为命令执行参数时,需要进行验证,检查输入长度、类型、是否包含了不允许的字符(例如;&|`等,案例:https://pms.uniontech.com/zentao/bug-view-93857.html)以及文件路径是否为相对路径或软链接等(通过systemd service配置限制进程可读写范围,可以很大的避免任意地址读写问题);
  * 禁止代码中通过QProcess::start,system,popen,exec.Command等函数拼接bash/sh -c等方式拼接命令,即需要执行的二进制不能为bash或sh这些;
  * 固定命令的执行,依旧禁止上述方式使用;

## **Polkit 规范与交互方案**

* 增加鉴权是一种非常高效的避免出现安全漏洞方案;
* 通过用户身份鉴定和用户确认,避免接口被随意调用以及用户不知情时调用.

### **Polkit 策略**

* 使用policy文件来定义行为;policy通常存放在/usr/share/polkit-1/actions路径下,名称通常和dbus name相同,文件后缀为policy;
* policy文件中需要关注的字段:
  * id: 发起鉴权操作的id,一个policy文件可以配置多个action id;
  * message: 鉴权时向用户解释当前动作,可以通过配置xml:lang或gettext-domain来完成翻译;
  * defaults: 包含 allow_any,allow_inactive,allow_active 三种设置, active 对应本地活跃会话,inactive 非活跃会话,allow_any 远程登录会话;每个设置都有如下选项(通常 allow_any,allow_inactive 都应该为no):
    * no:不允许用户执行操作,无法进行身份认证;
    * yes:用户可以不进行认证就执行操作;
    * auth_self:需要用户认证当前账户,不需要属于管理员;
    * auth_admin:需要用户认证为管理员账户,不需要是当前账户,存在多个管理员账户是通过任意一个即可;
    * auth_self_keep:和 auth_self 类似,认证状态会保持一段时间,目前为15分钟;
    * auth_admin_keep:和 auth_admin 类似,认证状态会保持一段时间;
  * 通常配置为后4个选项,使用auth_self还是auth_admin取决于使用场景;
  * annotate key="org.freedesktop.policykit.owner": 如果当前进程的用户不为root,那么需要增加声明unix-user:deepin-daemon,deepin-daemon可以替换成service指定的User;增加这个声明还有一个用途:CheckAuthorization方法支持设置detail,目前常用的detail有message和exAuthFlags等,设置detail的鉴权dbus调用时,polkitd会校验调用者uid==0或者用户为声明的owner用户;
  * annotate key="org.freedesktop.policykit.imply": 目的是一个操作(action)自动继承另一个或多个操作的权限,从而简化权限配置;以及在某些场景下,可以帮助优化交互体验;
* 通常不要使用其他包policy文件中定义的action;

### **鉴权代码**

* 使用方式：
  * dbus name: org.freedesktop.PolicyKit1;
  * dbus path: /org/freedesktop/PolicyKit1/Authority;
  * dbus interface: org.freedesktop.PolicyKit1.Authority;
  * dbus method: org.freedesktop.PolicyKit1.Authority.CheckAuthorization;
* Qt进程建议使用polkit-qt库,Golang进程建议使用dde-api的CheckAuth；
* 鉴权主体(Subject):
  * 鉴权主体有三种类型:
    * unix-process:调用者pid;
    * unix-session:调用者session;
    * system-bus-name:调用者dbus unique name;
    * 目前唯一允许使用鉴权主体只有system-bus-name;
  * 鉴权主体构造场景:
    * 控制中心调用Grub2服务接口为例,dde-control-center调用com.deepin.daemon.Grub2接口,Grub2的接口逻辑发起对应action的polkit鉴权(交互式),Grub2服务构造鉴权主体时,使用调用者(即控制中心)的system-bus-name作为鉴权主体;
    * 同样以控制中心调用Grub2服务接口为例,控制中心可以先发起一个action的polkit鉴权(交互式),该action需要 imply Grub2的的action或直接使用Grub2的action,控制中心构造鉴权主体,使用自身(也是控制中心)的system-bus-name作为鉴权主体;在调用接口后,Grub2服务发起一次非交互式鉴权,鉴权主体同样还是调用者的system-bus-name;
      * 这种使用方式常见于Gnome中,action的鉴权策略一般都是*_keep;主要思路是前端控制鉴权时机,后端校验调用者权限;
* 多因子认证场景:
  * 如果当前用户增加了生物认证或者一些设备的认证,在鉴权时,默认会开启所有的认证手段,只要通过其中一个,本次认证就会通过,部分场景不能使用非密码认证方式,比如修改|重置密码时;

### **pkla和rules机制:**

* V20的polkit仅支持pkla机制,V23 V25都支持;
* pkla:用于定义用户或用户组对特定操作的权限。可以通过这些配置文件指定哪些用户或组可以执行某些操作,通常是通过定义条件(如用户名、用户组、操作名等);
* rules:使用 JavaScript 语言编写,可以实现更复杂的逻辑和条件判断.通过编写规则,可以创建动态授权决策,能够根据系统状态或其他条件来决定是否授权某个操作;
* 利用rules机制,可以避免很多鉴权场景,比如控制中心修改网络都不需要鉴权,是因为默认加了netdev组,network-manager配置了rules对netdev组放行;

## **Systemd**** service 权限管理方法**

* System DBus使用systemd service启动,就可以通过systemd的配置来管理进程权限;
* 配置应遵循权限拆分和最小权限的原则:
  * 能通过非特权用户和Linux Capabilities以及group配置解决的问题,就不应该使用root权限运行服务;
  * system service应只拥有完成所需功能的最小权限,包括Capabilities、group、user、文件|文件夹rwx权限等;

### **关键配置**

* User: 自研服务中,如果使用root作为User,需要分析是否必要；只能使用root的服务需要在service备注详细原因;

  * 要求使用deepin-daemon或deepin-admin-daemon替换;
  * 用户组创建方法结合dh和systemd-sysusers工具,在项目debian目录创建.sysusers文件,并在rules中dh_installsysusers xxx.sysusers;
  * 配置文件的格式是每行对应一个用户或组,包含如下字段: 类型, 名称, ID, 简要描述, 家目录, 登录shell:u deepin-daemon - "" - -;
  * 类型为u的使用方式:按照"名称"字段的值,创建一个系统用户以及一个同名的组,并将此用户的主组设为此同名组. 此账户创建后将处于禁用状态,不会创建其家目录,用户也将被设为禁止登录;
  * 用户创建示例:[https://gerrit.uniontech.com/c/dde-daemon/+/248813](https://gerrit.uniontech.com/c/dde-daemon/+/248813)(创建group同样需要使用该方法,类型字段修改为g即可);
  * 不能擅自新增用户,所有新增的用户和组需要走明道云并完成评审:[https://cooperation.uniontech.com/app/1c4062ac-cc50-4f6c-9d7d-52906b78a291/63082b8bd16c2047b071b6cd/6508ff230a1f50226ae41838/650a63a0cc11a7bd3db484d6](https://cooperation.uniontech.com/app/1c4062ac-cc50-4f6c-9d7d-52906b78a291/63082b8bd16c2047b071b6cd/6508ff230a1f50226ae41838/650a63a0cc11a7bd3db484d6);
* 修改User时需要同步修改dbus conf中allow own的user,以及policy文件声明的owner；
* ProtectSystem: 要求使用strict,保证整个文件系统是只读的,通过其他配置增加读写特定文件的权限;
* InaccessiblePaths: 禁止访问的路径,/etc/shadow /etc/NetworkManager/system-connection /etc/pam.d /usr/share/uadp /etc/sudoers /etc/sudoers.d这些敏感目录需要配置,不能配置的需要在serivce文件中备注说明原因;
* ReadWritePaths: 可读写文件路径;

  * 不应该写其他服务管理的配置文件,涉及到需要整改或评审;
  * 需要创建\删除文件时需要加所在文件夹路径;
  * 如果读写文件为软链接,需要将源文件添加;
  * 公共路径创建本项目文件|文件夹时,例如创建/usr/share/deepin-ab-recovery,需要/usr/share的读写权限,需要通过tmpfiles的方式创建；在项目debian目录下创建一个 包名.tmpfiles文件如下:

  ```Plain
  #Type Path                           Mode User  Group Age         Argument
  d     /usr/share/deepin-ab-recovery  0700 root  root  -           -
  ```

  ```Shell
  override_dh_auto_install:
     dh_auto_install
     dh_installtmpfiles 包名.tmpfiles
  ```

  * 有关文件路径的配置,需要在配置前增加'-',例如:ReadWritePaths=-/usr/share/uadp/,防止文件或文件夹不存在导致的服务无法启动
* RuntimeDirectory,StateDirectory,CacheDirectory,LogsDirectory,ConfigurationDirectory:

  * 可以由systemd自动创建并分配权限的文件夹,分别对应/var/run,/var/lib,/var/cache,/var/log,/etc五个目录;
  * 如果/var/run下的数据不希望在服务停止时删除,那么需要配置RuntimeDirectoryPreserve=yes;
* SupplementaryGroups:动态添加group;

  * 不能再配置PrivateUsers=yes,会导致找不到group；例如网络、输入事件、音频等都有对应的组提供相应权限,无需使用root作为User;
  * 案例:加input组会有权限调用libinput库,dde-daemon的keyevent、gesture、inputdevice模块拆分可以改为非root启动;
* CapabilityBoundingSet:用于限制能力范围,适合安全加固,防止滥用特权;
* AmbientCapabilities:用于赋予能力并传递给子进程,适合非特权服务需要部分特权的场景;追加的能力,无法超过CapabilityBoundingSet配置的能力上限;

  * 生效结果(具体数值可能因内核版本而异,建议在实际环境中验证):
    * 非root服务:默认CapBnd为完整能力集合,其他Cap参数为0,配置 `AmbientCapabilities=CAP_KILL CAP_DAC_OVERRIDE,CapabilityBoundingSet=CAP_KILL CAP_DAC_OVERRIDE`后,相关Cap参数会设置为对应的能力位掩码;
    * root服务: CapPrm CapEff CapBnd 默认为完整能力集合,其他为0;配置能力限制后会相应调整;
    * 验证命令: `cat /proc/$(pidof service_name)/status | grep Cap` 和 `capsh --decode=<十六进制值>` 来解析能力;
* 其他配置:

  * 常见配置项按照需要开启,未开启需要在service文件中对该配置进行注释,并备注关闭原因:
    * NoNewPrivileges=yes
    * ProtectHome=yes
    * PrivateTmp=yes
  * 非常用配置,根据项目具体情况配置:
    * MemoryDenyWriteExecute=yes
    * PrivateDevices=yes
    * PrivateMounts=yes
    * PrivateNetwork=yes
    * ProtectKernelTunables=yes
    * ProtectKernelModules=yes
    * ProtectControlGroups=yes
    * LockPersonality=yes
    * RestrictRealtime=yes
    * RemoveIPC=yes

## **deepin-security-loader 实现与使用**

* 核心目标:

  * 解决二进制路径白名单类型的验证方式存在的安全风险问题;
  * 解决使用LD_PRELOAD环境变量绕过身份验证的问题;
* 方案基础技术机制:

  * 每个dbus service的unique name都是唯一的;
  * dbus conf可以根据调用者的group限制调用者调用;
  * setcap cap_setgid+ep的二进制,可以修改自身的group;
  * setcap的二进制,在运行时会进入'安全模式',LD相关的环境变量无法影响到二进制;
* 使用场景:

  * system bus的接口仅在内部使用,未对外明确声明公开;
  * system bus使用二进制路径限制的方式限制仅某些进程可以调用接口;
  * system bus的接口在产品交互层面不适合增加polkit鉴权操作;
* 整体流程:

  * 整个过程涉及三个组件以控制中心调用ab-recovery接口进行备份举例:
    * deepin-security-loader：具备setgid能力的二进制,用于启动前端进程或session级进程,仅允许启动指定目录的二进制,目前暂定/usr/libexec/deepin/,后面简称为loader;
    * client：在该例子中指控制中心,需要调用system bus的后台接口;
    * server：在该例子中指ab-recovery,需要对调用者进行限制;
    * server端(即ab-recovery服务)实现SetAllowCaller接口,参数为dbus的unique name,用于设置特权接口允许哪些调用者调用,其他特权接口被调用时,通过调用者的unique name判断是否允许调用;
    * SetAllowCaller需要通过dbus.conf文件配置,限制仅指定group可以调用；--此处以deepin-daemon为例;
    * client(控制中心)通过loader启动,启动时需要将server需要的group通过启动参数告诉loader：/usr/bin/deepin-security-loader --group deepin-daemon -- /usr/libexec/deepin/dde-control-center -s;
    * loader读到数据后调用server的SetAllowCaller,将unique name设置给server端(即ab-recovery服务);
    * 设置成功后loader通过pipe2的写端写入数据,通知到client(控制中心)可以调用;
    * client(控制中心)可以开始调用server端(即ab-recovery服务)特权接口;
* 示意图:

![](https://ecnc1q4rz6nx.feishu.cn/space/api/box/stream/download/asynccode/?code=YTRlNGI2NjJhN2YxZTkwZmI2N2M5OTExYTAzM2EyODJfOURBellQMWM5R0hZdUNYdFp6eHhzQkRPYXlzU3RWWGFfVG9rZW46QjMzemJSQzJ1bzE3OG14Q3R5RGNuNDlCblRjXzE3NTA2NjM2Nzc6MTc1MDY2NzI3N19WNA)

* client需要改造的内容:

  * 需要修改二进制路径,放到/usr/libexec/deepin/下或者硬链接到该目录;
  * 需要修改自身进程启动方式(包括dbus,desktop等):deepin-security-loader --group lightdm,netdev(service允许调用SetAllowCaller的group) -- /usr/libexec/deepin/exec arg1 arg2 ...
  * 需要在system bus上export,用于获取当前进程在system bus上的唯一名称(注意只需要export,不需要request name);
  * 需要解析命令行参数,--fd1,一般该fd为3,该fd为匿名管道的write端,需要写入一个json字符串,内容如下:
    ```JSON
          {
             "UniqueName": ":1.3154", 当前进程在system bus上的唯一名称
             "ErrorMsg"   ："",    启动过程中的错误消息
             "DestList": [    需要设置允许调用的system service信息
                {
                      "DbusName": "com.deepin.service.test",
                      "DbusPath": "/com/deepin/service/test",
                      "DbusInterface": "com.deepin.service.test"
                }
             ]
          }
    ```
* client需要解析命令行参数,--fd2,一般该fd为4,该fd为另一个匿名管道的read端,用于接收来自loader的消息,是否SetAllowCaller成功,需要解析一个json字符串,内容如下:

  ```JSON
  {
     "Result": true,
     "Message":"error message"
  }
  ```
* server需要改造的内容:

  * 需要实现SetAllowCaller(String uniqueName)接口;该接口通过dbus.conf限制指定group可以调用;并保存uniqueName;
  * 之前版本server使用二进制路径限制的接口,改为判断调用者的unique name是否是允许调用的;
* 注意事项:

  * client端接口：原则上不建议client还有对外暴露的接口,通常会有一些翻页、跳转等UI操作接口,需要在代码逻辑上将这些逻辑和调用server特权接口剥离,防止外部通过这些接口作为跳板,间接攻击到server端;
  * 反面案例：应用卸载流程：dde-launcher -> dde-session-daemon -> lastore-daemon,同时dde-launcher提供了卸载app的接口;
