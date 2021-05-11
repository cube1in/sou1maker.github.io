# Centos

Centos 默认安装无法上网

**1. 查看`ifcfg-ens33`文件**

```bash
cd /etc/sysconfig/network-scripts
```

 ![Snipaste_2021-05-11_11-06-46](https://user-images.githubusercontent.com/58240137/117752243-061f3f80-b249-11eb-92b7-25e8c19f13c2.png)
 
 **2. 编辑`ifcfg-ens33`文件**

```bash
vi ./ifcfg-ens33
```

**3. 输入`i`进入编辑模式，将此处改为`yes`**

![image](https://user-images.githubusercontent.com/58240137/117752478-66ae7c80-b249-11eb-814a-a9fda921b9ba.png)

**4. 点击`Esc`后，输入`:wq`写入并退出**


**5. 重启网络**

```bash
service network restart
```

**6. ping 一下百度**

![image](https://user-images.githubusercontent.com/58240137/117753984-02d98300-b24c-11eb-9096-f43fdc129165.png)
