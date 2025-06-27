## 目标

* 在CD阶段可以查找出哪些root DBus服务的接口没有做鉴权操作

## 准备

* 需要一份自研system dbus服务清单;

### 方案

* pkla配置默认所有action鉴权结果为no;
* 获取某个system bus的所有method和system dbus name;
* 脚本以root身份监听polkit service 的method call;
* 脚本降权为当前激活图形会话的用户调用method;
* 根据监听的内容,并通过system dbus name做过滤,判断该接口是否有鉴权;

#### 方案缺陷

1. 无法真实的判断是否有鉴权弹出,只能判断有调用鉴权接口;

### 方案增强

* 写一个polkit-agent替换dde-polkit-agent;
* TODO: check polkit-agent可以获取到哪些;

### TODO

* 用高版本的rules应该可以替换dbus监听;

