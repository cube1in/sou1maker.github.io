# 安装Aria2

#### :unicorn: 让树莓派当作下载机

**1. 安装`Aria2`:**

> :point_right: [`Aria2`](https://github.com/aria2/aria2)

```bash
apt-get install aria2
```

**2. 创建 Aria2 的配置文件夹:**

```bash
mkdir /etc/aria2
```

**3. 创建`aria2.session`和`aria2.congig`配置文件:**

```bash
touch /etc/aria2/aria2.session
touch /etc/aria2/aria2.conf
```

**4. 编辑`/etc/aria2/aria2.conf`:**

```bash
nano /etc/aria2/aria2.conf
```

> 复制以下内容:

```text
## 文件保存相关 ##

# 文件保存目录
dir=/mnt/storage/download
# 启用磁盘缓存, 0为禁用缓存, 需1.16以上版本, 默认:16M
disk-cache=32M
# 断点续传
continue=true

# 文件预分配方式, 能有效降低磁盘碎片, 默认:prealloc
# 预分配所需时间: none < falloc ? trunc < prealloc
# falloc和trunc则需要文件系统和内核支持
# NTFS建议使用falloc, EXT3/4建议trunc, MAC 下需要注释此项
#file-allocation=falloc

## 下载连接相关 ##

# 最大同时下载任务数, 运行时可修改, 默认:5
#max-concurrent-downloads=5
# 同一服务器连接数, 添加时可指定, 默认:1
max-connection-per-server=15
# 整体下载速度限制, 运行时可修改, 默认:0（不限制）
#max-overall-download-limit=0
# 单个任务下载速度限制, 默认:0（不限制）
#max-download-limit=0
# 整体上传速度限制, 运行时可修改, 默认:0（不限制）
#max-overall-upload-limit=0
# 单个任务上传速度限制, 默认:0（不限制）
#max-upload-limit=0
# 禁用IPv6, 默认:false
disable-ipv6=true

# 最小文件分片大小, 添加时可指定, 取值范围1M -1024M, 默认:20M
# 假定size=10M, 文件为20MiB 则使用两个来源下载; 文件为15MiB 则使用一个来源下载
min-split-size=10M
# 单个任务最大线程数, 添加时可指定, 默认:5
split=10

## 进度保存相关 ##

# 从会话文件中读取下载任务
input-file=/etc/aria2/aria2.session
# 在Aria2退出时保存错误的、未完成的下载任务到会话文件
save-session=/etc/aria2/aria2.session
# 定时保存会话, 0为退出时才保存, 需1.16.1以上版本, 默认:0
save-session-interval=60

## RPC相关设置 ##

# 启用RPC, 默认:false
enable-rpc=true
# 允许所有来源, 默认:false
rpc-allow-origin-all=true
# 允许外部访问, 默认:false
rpc-listen-all=true
# RPC端口, 仅当默认端口被占用时修改
# rpc-listen-port=6800
# 设置的RPC授权令牌, v1.18.4新增功能, 取代 --rpc-user 和 --rpc-passwd 选项
#rpc-secret=<TOKEN>

## BT/PT下载相关 ##

# 当下载的是一个种子(以.torrent结尾)时, 自动开始BT任务, 默认:true
#follow-torrent=true
# 客户端伪装, PT需要
peer-id-prefix=-TR2770-
user-agent=Transmission/2.77
# 强制保存会话, 即使任务已经完成, 默认:false
# 较新的版本开启后会在任务完成后依然保留.aria2文件
#force-save=false
# 继续之前的BT任务时, 无需再次校验, 默认:false
bt-seed-unverified=true
# 保存磁力链接元数据为种子文件(.torrent文件), 默认:false
bt-save-metadata=true
```

**5. 启动`Aria2`后台运行命令(无输出代表成功):**

```bash
aria2c --conf-path=/etc/aria2/aria2.conf -D
```

**6. 编辑`/etc/init.d/aria2c`, 以添加开机自启:**

```bash
nano /etc/init.d/aria2c
```

> 复制以下内容:

```text
#!/bin/sh
### BEGIN INIT INFO
# Provides: aria2c
# Required-Start:    $network $local_fs $remote_fs
# Required-Stop:     $network $local_fs $remote_fs
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: aria2c RPC init script.
# Description: Starts and stops aria2 RPC services.
### END INIT INFO

USER=root
RETVAL=0

case "$1" in  
    start)  
        echo "Starting service Aria2..."
        aria2c --conf-path=/etc/aria2/aria2.conf -D  
        echo "Start service done."  
    ;;  
    stop)  
        echo "Stoping service Aria2..."  
        killall aria2c   
        echo "Stop service done."  
    ;;  
esac  

exit $RETVAL
```

**7. 安装`nginx`, 以使用`Aria2`前端界面:**

```bash
apt-get -y install nginx
```

**8. 下载前端第三方程序`webui-aria2`:**

> :point_right: [`webui-aria2`](https://github.com/ziahamza/webui-aria2)

```bash
rm -rf /var/www/html
git clone https://github.com/ziahamza/webui-aria2.git /var/www/html
```

**9. 启动`nginx`:**

```bash
/etc/init.d/nginx start
```

**10. 添加自启动(`nginx`和`Aria2`):**

```bash
nano /etc/rc.local
```

> 将以下复制到`exit 0`之前

```text
aria2c --conf-path=/etc/aria2/aria2.conf -D
/etc/init.d/nginx start
```


