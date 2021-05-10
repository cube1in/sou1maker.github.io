# OpenWrt

**:unicorn: 软路由配置教程**

这里使用的是`LEDE`系统

**1. 准备工作**

刷入`image`的工具`win32diskimager`下载地址：[Win32 Disk Imager](https://sourceforge.net/projects/win32diskimager/)

恩山论坛下载地址：[OpenWrt ipv6/docker/大全版/精简版/旁路由版 丰富插件免费使用](https://www.right.com.cn/forum/thread-4053752-1-1.html)

![image](https://user-images.githubusercontent.com/58240137/117666361-5dd09300-b1d6-11eb-8562-7b30921c8b5b.png)


**2. 制作U盘启动**

下载完压缩包之后解压

![image](https://user-images.githubusercontent.com/58240137/117667266-4fcf4200-b1d7-11eb-909d-06db0231d111.png)

插入U盘，安装`win32diskimager`并制作U盘启动盘

![image](https://user-images.githubusercontent.com/58240137/117667198-3d550880-b1d7-11eb-909f-569da755af9e.png)

之后软路由设备设置U盘为第一启动，等待`image`刷入完成即可


**3. 设置`OpenWrt`**

默认的IP地址：192.168.1.1

用户名：root
密码：password

![Uploading Snipaste_2021-05-10_21-52-09.png…]()

![Uploading Snipaste_2021-05-10_21-51-39.png…]()

设置`WAN`口为拨号`PPPoE`，如果已经使用了光猫拨号，这里不需要设置

`LAN`口默认是静态地址+DHCP功能，所以默认就好

**4. 设置无线路由器为AP模式**

不要用光猫PPPOE拨号+固定IP，软路由=主路由=PPPOE+网关+DHCP服务器，WIFI路由器指定固定IP+不开DHCP服务

------你所有电脑均可访问光猫+软路由+WIFI路由器

**注意：**这里软路由`LAN`口出来，接WIFI路由器的**其中一个`LAN`** ，WIFI路由器的`WAN`不需要

其它设备(如有线连接的电脑)后续只需要接入WIFI路由器剩余的`LAN`口即可

然后设置WIFI路由器的`LAN`口IP和软路由相同网段

![Snipaste_2021-05-10_21-50-20](https://user-images.githubusercontent.com/58240137/117669691-c2d9b800-b1d9-11eb-877c-7ef90695d34b.png)

到这里为止，所有的IP地址均在软路由处进行分配，WIFI路由器只作为AP模式使用(交换机+开WIFI的)

