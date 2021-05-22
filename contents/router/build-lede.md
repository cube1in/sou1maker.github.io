# 构建自己的[`LEDE`](https://github.com/coolsnowwolf/lede)包

### 编译

##### 1. 更新包管理工具

```bash
sudo apt-get update
```

##### 2. 安装编译所需的工具依赖

```bash
sudo apt-get -y install build-essential \
asciidoc binutils bzip2 gawk gettext git \
libncurses5-dev libz-dev patch python3 python2.7 \
unzip zlib1g-dev lib32gcc1 libc6-dev-i386 subversion \
flex uglifyjs git-core gcc-multilib p7zip p7zip-full msmtp \
libssl-dev texinfo libglib2.0-dev xmlto qemu-utils upx libelf-dev \
autoconf automake libtool autopoint device-tree-compiler g++-multilib \
antlr3 gperf wget curl swig rsync
```

##### 3. `Clone`官方项目

```bash
git clone https://github.com/coolsnowwolf/lede
```

##### 4. `cd lede`到项目下，修改`feeds.conf.default`文件

![image](https://user-images.githubusercontent.com/58240137/119229039-92ced500-bb48-11eb-9e90-a1193e46d2d2.png)

把这项注释去掉(梯子程序)

![image](https://user-images.githubusercontent.com/58240137/119229218-7e3f0c80-bb49-11eb-9cac-54c817e00aa0.png)

##### 5. 升级安装`feeds`

```bash
./scripts/feeds update -a && ./scripts/feeds install -a
```

##### 6. 配置相应的编译项

```bash
make menuconfig
```

![image](https://user-images.githubusercontent.com/58240137/119229298-e7bf1b00-bb49-11eb-9627-099498dccea6.png)

这里主要讲一下开启`ipv6`支持/主题/梯子

> 移动到`Expra packages`，选择`ipv6helper`

![image](https://user-images.githubusercontent.com/58240137/119229339-205ef480-bb4a-11eb-9f7d-71aaa9955f6c.png)

> 移动到`LuCI`，选择`Themes`，选上自己喜欢的主题

![image](https://user-images.githubusercontent.com/58240137/119229386-6916ad80-bb4a-11eb-9c2b-134e37013511.png)

> 然后继续进入`LuCI`下的`Applications`下，选上`luci-app-ssr-plus`

![image](https://user-images.githubusercontent.com/58240137/119229454-c01c8280-bb4a-11eb-9ce8-3d10fb93bc7b.png)

点击`Save`保存配置到`.config`后，一直`Exit`退出配置界面

##### 7. 下载dl库（国内请尽量全局科学上网）

```bash
make -j8 download V=s 
```

##### 8. 执行编译

```bash
make -j$(($(nproc) + 1)) V=s
```

##### 9. 生成文件

最后生成文件在`lede/bin/targets/具体环境`目录下

![image](https://user-images.githubusercontent.com/58240137/119229706-cd863c80-bb4b-11eb-95ce-d0a9948f109e.png)

##### 10. 后记

具体详情请访问：[`LEDE`](https://github.com/coolsnowwolf/lede)


### 虚拟机测试

##### 1. 创建虚拟机

> 稍后安装操作系统

![image](https://user-images.githubusercontent.com/58240137/119234649-9d499880-bb61-11eb-96cd-6947375ca20f.png)

> 其他 Linux 5.x 及更高版本内核 64 位

![image](https://user-images.githubusercontent.com/58240137/119234680-b9e5d080-bb61-11eb-8ebc-1fd1b03d512d.png)

> 命名

![image](https://user-images.githubusercontent.com/58240137/119234703-d41fae80-bb61-11eb-8d66-521ade579d80.png)

> 使用网络地址转换(NAT)

![image](https://user-images.githubusercontent.com/58240137/119234751-07623d80-bb62-11eb-9647-9908ef17e727.png)

> 使用现有虚拟磁盘

![image](https://user-images.githubusercontent.com/58240137/119234779-282a9300-bb62-11eb-92c9-f80bff81049e.png)

选择此前生成的`.vmdk`文件

![image](https://user-images.githubusercontent.com/58240137/119234599-63789200-bb61-11eb-9641-dbbf7a737dbb.png)

##### 2. 修改为`lan`口`ip`地址为`dhcp`获取

```bash
uci set network.lan.proto=dhcp
```

##### 3. 保存网络配置

```bash
uci commit network
```

##### 4. 重启网络

```bash
/etc/init.d/network restart
```

##### 5. 查看状态

```bash
uci show network
```

##### 6. 通过`ifconfig`查看 br-lan 的`ip`地址

在浏览器中输入相应的`ip`即可进入OpenWrt管理
