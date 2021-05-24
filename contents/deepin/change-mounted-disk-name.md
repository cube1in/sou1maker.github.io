# Deepin Linux 重命名挂载磁盘

#### 1. 列出以挂载磁盘信息

```bash
df -h
```

#### 2. 卸载需要修改名称的分区

```bash
sudo umount /dev/sda1
```

#### 3. 修改名称

```bash
sudo ntfslabel /dev/sda1 HDD
```

> 注：ntfslabel会在修改名称后自动重新加载，不需要再执行mount命令
