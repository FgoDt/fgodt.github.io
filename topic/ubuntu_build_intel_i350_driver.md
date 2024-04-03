# ubuntu 22.04 安装 Intel I350 网卡驱动

   最近公司购买了一台服务器来炼丹，迫于价格优势这个供应商只管硬件不管软件。我们的运维在网卡驱动耗了2天都没成功，出于好奇我也去瞅了瞅（本人多年安装Linux的经验应该能分分钟秒杀）
### 让机器先连上网
最简单的办法是拿台安卓手机和一个USB线，使用安卓的USB给服务器提供网络（老装机的都应该知道）

连上USB手机选好共享后查看网络使用下面的命令
```shell
$ ip addr
应该会有一个USB0的地址出来
```
如果状态为DOWN可以使用netplan来配置网卡
``` shell
$ sudo vim /etc/netplan/0-netcfg.yaml
```
在文件中添加USB0的配置，配置文件看起来应该是这样

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    USB0:
      dhcp4: true
```
然后使用netplan更新网络
```shell
$ netplan apply
```

### 下载网卡驱动包和编译环境
能上网了第一步当然是下载gcc 和 make
```shell
$ sudo apt update
$ sudo apt install gcc make
```

然后去下intel的网卡[驱动](https://www.intel.com/content/www/us/en/download/14098/intel-network-adapter-driver-for-82575-6-82580-i350-and-i210-211-based-gigabit-network-connections-for-linux.html)

下载后解压
```shell
$ tar -xvf igb-5.13.16.tar.gz
```

### 编译和安装驱动
编译一般来说很简单， 最后在接下来使用root用户来操作
```shell
# cd igb-5.13.16
# cd src
# make install
```
没什么意外的话应该可以看到提示已经编译完成了igb.ko

接下来就是安装驱动
```shell
# rmmod igb; modprobe igb
```
这个时候如果没提示错误那么就应该完成安装了

我们还是使用ip命令查看下网络
```shell
$ ip addr
```
如果显示了4个新的网络那恭喜安装成功了🎉

如果状态为DOWN可以同样适用netplan来配置网络，这里我们就不在说明了

### The NVM Checksum Is Not Valid 错误

但是我们的机器在安装驱动的时候提示 The NVM Checksum Is Not Valid 。我网上找了好多方案都解决不了，最后我们选择了最差的方案修改驱动源码跳过检查。
这个NVM checksum方法在igb_main.c中，将下面的代码注释掉即可跳过检测
![2023-03-31T10:02:39.png][1]


  [1]: https://blog.nintendo-fans.com/usr/uploads/2023/03/2717924493.png

**但是这是最后一个选择我还是推荐大家去找其他方法**

最后希望大家炼丹愉快
