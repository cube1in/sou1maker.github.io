# Deepin 相关设置

### 重命名挂载磁盘

```bash
# 列出已挂载磁盘信息
df -h

# 卸载需要修改名称的分区
sudo umount /dev/sda1

# 修改名称
sudo ntfslabel /dev/sda1 HDD
```

> 注：ntfslabel会在修改名称后自动重新加载，不需要再执行mount命令

***

### 修改系统监控文件上限 

```bash
# 查看上限
sudo sysctl -a | grep fs.inotify.max_user_watches

# 修改上限
sudo sysctl -w fs/inotify/max_user_watches=10000000
```

***

### 输入法配置

```bash
fcitx-config-gtk3
```

***
