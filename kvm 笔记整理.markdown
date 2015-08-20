# kvm 笔记整理



---

##安装

1，debian安装kvm
`aptitude install qemu-kvm libvirt-bin`

2,centos安装kvm
`yum -y install kvm python-virtinst libvirt tunctl bridge-utils virt-manager qemu-kvm-tools virt-viewer virt-v2v  libguestfs-tools`
也可以直接通过yum groupinstall安装
检测kvm模块是否安装
lsmod |grep kvm

3、创建br0网桥
```
vi /etc/sysconfig/network-scripts/ifcfg-br0
DEVICE=br0
TYPE=Bridge
ONBOOT=yes
NM_CONTROLLED=yes
BOOTPROTO=static
IPADDR=192.168.83.240
NETMASK=255.255.255.0
GATEWAY=192.168.83.2

vi /etc/sysconfig/network-scripts/ifcfg-eth0
DEVICE=eth0
HWADDR=00:0C:29:98:F6:00
ONBOOT=yes
BRIDGE=br0
```
通过brctl show查看网桥
## kvm虚机安装

1.使用raw格式磁盘安装
```
virt-install --name=vm001 --ram 512 --vcpus=1 --disk path=/data/test01.img,size=7,bus=virtio --accelerate --cdrom/data/iso/centos6.6.iso --vnc --vncport=5910 --vnclisten=0.0.0.0 --network bridge=br0,model=virtio --noautoconsole
```
2.使用qcow2格式安装
```
qemu-img create -f qcow2 test02.img 7G

virt-install --name=vm002 --os-type=linux --ram 512 --vcpus=1 --disk path=/data/test02.img,format=qcow2,size=7,bus=virtio --accelerate --cdrom /data/iso/centos6.6x64.iso --vnc --vncport=5910 --vnclisten=0.0.0.0 --network bridge=br0,model=virtio --noautoconsole
```
3.windows使用virtio安装
```
virt-install -n win001 -r 2048 --vcpus=2 --os-type=windows --accelerate -c /iso/windows_server_2008_R2/Win_Server_2008_R2_SP1_33in1.iso --disk path=/iso/virtio-win-0.1-81.iso,device=cdrom --disk path=/vhost/win001.img,format=qcow2,bus=virtio --network bridge=br0 --vnc --vncport=5992 --vnclisten=0.0.0.0 --force --autostart
```
>参数说明:

>--name指定虚拟机名称

>--ram分配内存大小。

>--vcpus分配CPU核心数，最大与实体机CPU核心数相同

>--disk指定虚拟机镜像，size指定分配大小单位为G。

>--network网络类型，此处用的是默认，一般用的应该是bridge桥接。

>--accelerate加速

>--cdrom指定安装镜像iso

>--vnc启用VNC远程管理，一般安装系统都要启用。

>--vncport指定VNC监控端口，默认端口为5900，端口不能重复。

>--vnclisten指定VNC绑定IP，默认绑定127.0.0.1，这里改为0.0.0.0。

>--os-type=linux,windows

>--os-variant=
>win7:MicrosoftWindows7

>vista:MicrosoftWindowsVista

>winxp64:MicrosoftWindowsXP(x86_64)

>winxp:MicrosoftWindowsXP

>win2k8:MicrosoftWindowsServer2008

>win2k3:MicrosoftWindowsServer2003

>freebsd8:FreeBSD8.x

>generic:Generic

>debiansqueeze:DebianSqueeze

>debianlenny:DebianLenny

>................

通过VNC连上就可以安装系统

##kvm管理
查询所有虚机
`# virsh list --all`
启动一个虚机
`# virsh start vm001`
关机
默认情况下virsh工具不能对linux虚拟机进行关机操作，linux操作系统需要开启与启动acpid服务。在安装KVM linux虚拟机必须配置此服务。
```
# chkconfig acpid on   
# service acpid restart
```
virsh关机
` # virsh shutdown vm001`

挂起服务器
` # virsh suspend vm001 `
恢复服务器
`# virsh  resume vm001`
kvm的克隆
`# virt-clone -o vm001 -n vm002 -f /data/vm002.img`
##kvm磁盘管理
磁盘格式与转换

1、查看磁盘格式
`# qemu-img info test01.img`

2、转换磁盘格式(关机状态)
`# qemu-img convert -f raw -O qcow2 test01.img test01.qcow2 `
-f  源镜像的格式   
-O 目标镜像的格式

3.kvm虚机增加磁盘
通过virsh attach-disk命令添加一块硬盘到系统中，即时生效，但系统重启后新硬盘会消失
`qemu-img create -f qcow2 /data/testdisk.img 20G`
`virsh attach-disk vm001 /data/testdisk.img vdb`
卸载硬盘我们可以使用virsh detach-disk命令，
`virsh detach-disk vm001 --target vdb`
可以修改配置文件使其永久生效

4、qcow2磁盘在线扩容
`qemu-img resize test01.qcow2 +2G`

##虚机快照

1、创建快照
`virsh snapshot-create vm001`
或者自定义命名
`virsh snapshot-create as vm001 snap1 `

2、查看快照
virsh snapshot-list vm001

3、恢复快照(关机状态)
`# virsh snapshot-revert vm001 14589897162`

4，删除快照
`# virsh snapshot-delete vm001 14589897162`

##添加网卡
`virsh attach-interface vm001 --type bridge --source br0`

`virsh dumpxml vm001`查看

`virsh dumpxml vm001 >vm001.xml`  永久生效

使用virsh domiflist命令可以查看虚拟机目前拥有的网卡
