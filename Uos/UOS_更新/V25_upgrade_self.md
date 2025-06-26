# upgrade self

## lastore-daemon更新前完成自己更新

* 时机:检查更新阶段,apt update执行完成,但是实际job不会结束;
* dh_installsystemd --no-restart-on-upgrade 不去掉禁用重启的配置;
* 由lastore-daemon完成安装后自行重启;

### 处理自动下载

* 安装完成后,重启服务,加download参数


### 部分从更新平台获取的数据,需要记录 
