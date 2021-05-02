### 设置国家
```bash
sudo raspi-config
```

### 默认账密

```bash
pi/raspberry
```

### 解锁root用户

```bash
sudo passwd root
```

### 多WiFi配置(通过`priority`指定顺序)

```bash
nano /etc/wpa_supplicant/wpa_supplicant.conf
```

> 复制以下:

```text
country=CN
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
 
network={
    ssid="无线网络名称1"
    key_mgmt=WPA-PSK
    psk="无线网络密码1"
    priority=1
}

network={
    ssid="无线网络名称2"
    key_mgmt=WPA-PSK
    psk="无线网络密码2"
    priority=2
}
```
