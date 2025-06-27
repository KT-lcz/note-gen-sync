## SUSE Polkit

SUSE 使用 polkit-default-private做polkit action管控

### 管控方式

* 对所有的action做overrride配置;
* override的实现方式是通过rules;
* 包安装时会根据模版生成3套配置;
* 3套override配置可以在不同安全场景下切换;

### V25借鉴

* 使用rules机制,实现某个用户某个action不再鉴权;可以放到polkit-agent中,让用户选择;
* 以及可以实现全局action统一管理,仿照windows调整级别:![image.png](https://cdn.jsdelivr.net/gh/KT-lcz/note-gen-image-sync@main/2025-06/4f4f94a1-6aa5-4da6-a192-026e33ede01d.png)
* 默认级别应设置为严格或标准,以及可以在polkit-agent中可以跳转到级别调整;

