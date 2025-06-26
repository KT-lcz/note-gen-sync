# 基于v25 polkit 的非root执行方案

* 核心思路：通过polkit和systemd,给指定用户和组在某些固定场景提权，避免整个进程权限提升；

## 验证

- unshare 环境下无法进行CheckAuth,可以用`<annotate key="org.freedesktop.policykit.exec.path">和<annotate key="org.freedesktop.policykit.owner">`
- rules 功能验证：
  
    ```javascripts
    polkit.addRule(function(action, subject) {
        if (action.id == 'org.freedesktop.timedate1.set-timezone' && subject.user == "deepin-daemon"){
            return polkit.Result.YES;
        }
        return polkit.Result.NOT_HANDLED;
    })

    polkit.addRule(function(action, subject) {
        if (action.id == "org.freedesktop.systemd1.manage-units" && action.lookup("unit") == "test.service" && action.lookup("verb") == "start" && subject.isInGroup("deepin-daemon") ){
                return polkit.Result.YES;
        }
        return polkit.Result.NOT_HANDLED;
    })

    polkit.addRule(function(action, subject) {
        if (action.id == "org.freedesktop.systemd1.manage-units" && action.lookup("unit") == "test.service" && action.lookup("verb") == "start" && subject.user == "deepin-daemon" ){
                return polkit.Result.YES;
        }
        return polkit.Result.NOT_HANDLED;
    })
    ```

  - 可以根据进程的user和group进行权限控制;
  - 只针对指定的service和polkit actions控制;

## 目前使用root的情况

1. 调用的dbus接口需要鉴权,只有root可以默认通过鉴权;但是产品设计要求场景无需鉴权交互
2. 调用的命令需要root权限,比如update-grub、apt dist-upgrade、dmidecode等;
3. 操作外设、cgroup、iptables、病毒扫描等；

## 修改方

## 保障方案

1. 定时扫描所有systemd service,禁用非UOS组件使用deepin-daemon和deepin-admin-daemon;
2. 修改用户组时禁止添加deepin-daemon和deepin-admin-daemon;