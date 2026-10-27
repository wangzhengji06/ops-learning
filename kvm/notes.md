## Virtualization technology

Full virtualization techonology: KVM

Half virtualization technology: Xen

KVM itself is a single server software.

After you enable the cpu accelerated virtualization, KVM doesn't replace or take over the Linux kernel. It extends the Linux kernel with hypervisor capabilities.

### How do we manage virtual machine?
1. by using service like libvirt and a bunch of command lines
2. By using software like qemu

### Cloud Computing

By introducing kvm and mutiple servers and a software to create the resource pool, you bascially create a cloud computing environment. 

### Checking whether cpu supports the kvm

```bash
grep -i kvm /boot/config...
```

Because I am using windows, I have to use

```cmd
bcdedit /set hypervisorlaunchtype off
bcdedit /set hypervisorlaunchtype auto
```

## KVM

### terminology

vm is named as domain in KVM.

### libvert

libvert relies on a service called libvertd



### Check network

```bash
brctl show
```

You are supposed to see virbr0, which is under 192.168.122.1/24 

### Manage the KVM using libvirt

```bash
virt install # create virtual machine
virsh # used to manage environment
virt-manager # the GUI interface for virutal machine management
```



Using `virt install`

```bash
virt-install --virt-type kvm --name CentOS-7-x86_64 --memory 1024 --cdrom=/data/images/CentOS-7-x86_64-Minimal-2009.iso --disk=/opt/CentOS-7-x86_64.raw --network network=default --graphics vnc,listen=0.0.0.0 --noautoconsole
```

List the virtual machine / close them

```bash
virsh list
virch start xx
virsh suspend xx 
virsh resume xxx
virsh shutdown xx
virsh destroy xx # force shutdown
virsh create # temporary
virsh define # permanent
virsh undefine # remove it from list
virsh cat
virsh edit
```

### vnc 

by default vnc connects to kvm's virtual desktop, and it is on 5900.

### ssh

`virch console`  provide a sshish way of connection to virtual machine's console. But you need to log into the virtual machine and allow tty setting etc.

## KVM network

By default, KVM domain use nat.  like vm is on 192.168.122.0/24, and it uses nat to transform to rocky, which is 10.0.0.0/8, which is further transferred to vmware workstation which is 172.22.0.0/16. 

With such nat transformations, of course performance cannot be that good.

Suppose now you have two virtual machines. You will find by using `ip -a` and `brctl show` that, new net interface has been created for you already.

![image-20261024184423603](C:\Users\lzabry\AppData\Roaming\Typora\typora-user-images\image-20261024184423603.png)

## Infrastructure as code

You can modify the xml file, then use `virch define xxx.xml` to create a permanent 



## Virtual Machine management

### Disk management

![image-20261024192907601](C:\Users\lzabry\AppData\Roaming\Typora\typora-user-images\image-20261024192907601.png)

You want to increase size for /opt, you mount it to a virtual disk, that virtual disk is attached to the vm machine.

Where do I get the virtal disk?  1. First I create a storage pool 2. Create virtual disk inside storage pool 3. attach the disk into virtual machine 4. mount the disk to /opt

```bash
virch pool-list
virch pool-define-as vmware_pool --type dir --target /kvm/images
```

```bash
mkdir /kvm/images -p
virsh pool-define-as sswang_dpool --type dir --target /kvm/images
virsh pool-build sswang_dpool
virsh pool-list --persistent
virsh pool-start sswang_dpool
virsh pool-autostart sswang_dpool
virsh pool-autostart sswang_dpool


## Now we need to create volume
virsh vol-create-as --pool sswang_pool --name sswang.img --capacity 2G --allocation 1G --format raw
virsh vol-info sswang.img sswang_pool
virsh vol-dumpxml sswang.img sswang_pool #use to check information

## attach needs to be applied on running vm
virsh start CentOS-7-x86_64
virsh attach-disk CentOS-7-x86_64 /kvm/images/sswang.img vdb

## after that just normal linux
mkfs.ext4 /dev/vdb
mkdir /sswang
mount /dev/vdb /sswang/

dd if=/dev/zero of=/sswang/test.1.txt bs=1M count=200

```

### Network Management

There are in general 4 network modes:  Bridge,  Route(Host only),  Nat, seperate.

#### Bridge Mode

This is different from VmWare's bridge mode.  vmware's bridge means that the vmware pc will directly connect to the outside router. In KVM it means that it will set up a virtual bridge so that the vms can talk to each other. The virtual bridge needs to be binded to the real net drive.  

#### Route Mode

Kind of like soft route, when you want to connect to outside network, you use the physical net drive as the router.

#### Nat Mode

Just normal ip rewrite

#### Command

```bash
virsh net-list --all #to check all the net list
virsh net-info default # to cehck the specific net information

vim /etc/libvirt/qemu/networks/sswang.xml
<!--
WARNING: THIS IS AN AUTO-GENERATED FILE. CHANGES TO IT ARE LIKELY TO BE
OVERWRITTEN AND LOST. Changes to this xml configuration should be made using:
  virsh net-edit default
or other application using the libvirt API.
-->

<network>
  <name>sswang-yongjiu</name>
  <uuid>ac26dc57-8c1b-4b1e-8aa6-93e1f699dae9</uuid>
  <forward mode='nat'/>
  <bridge name='virbr1' stp='on' delay='0'/>
  <mac address='52:55:00:ec:32:eb'/>
  <ip address='192.168.144.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.144.2' end='192.168.144.254'/>
    </dhcp>
  </ip>



# Now we define the network
virsh net-define sswang.xml
# Make the network start automatically
virsh net-autostart sswang-yongjiu 

```

#### Custom Network Practice

1. first create virtual network bridge.
2. Bind network bridge to physical drive.
3. Remove the ip address on physical interface, make ip address appear on virtual network bridge
4. let virtual machine use bridge

Bridge mode will use host server's local network router's ability to assign DHCP address directly to the virtual machine.

```bash
#!/bin/bash
# *************************************
# * 功能: 脚本方式定制虚拟网卡
# * 作者: 王树森
# * 联系: wangshusen@sswang.com
# * 版本: 2025-03-26
# *************************************
# 定义网桥名称
BRIDGE_NAME="br0"
# 定义网络接口名称
INTERFACE_NAME="ens160"
# 定义网桥的 IP 地址
BRIDGE_IP="10.0.0.12/24"
# 定义默认网关
DEFAULT_GATEWAY="10.0.0.2"
# 检查命令是否可用
check_command() {
    local cmd=$1
    if ! command -v "$cmd" &> /dev/null
    then
        echo "$cmd 命令未找到，请安装相关包。"
        exit 1
    fi
}
# 创建网桥
create_bridge() {
    if brctl addbr "$BRIDGE_NAME" 2>/dev/null; then
        echo "成功创建网桥 $BRIDGE_NAME"
    else
        echo "创建网桥 $BRIDGE_NAME 失败"
        cleanup
        exit 1
    fi
}
# 开启网桥的生成树协议
enable_stp() {
    if brctl stp "$BRIDGE_NAME" on 2>/dev/null; then
        echo "成功开启 $BRIDGE_NAME 的生成树协议"
    else
        echo "开启 $BRIDGE_NAME 的生成树协议失败"
        cleanup
        exit 1
    fi
}
# 关闭网络接口的 IP 地址并启动接口
prepare_interface() {
    if ifconfig "$INTERFACE_NAME" 0 up 2>/dev/null; then
        echo "成功关闭 $INTERFACE_NAME 的 IP 地址并启动接口"
    else
        echo "关闭 $INTERFACE_NAME 的 IP 地址并启动接口失败"
        cleanup
        exit 1
    fi
}
# 将网络接口添加到网桥
add_interface_to_bridge() {
    if brctl addif "$BRIDGE_NAME" "$INTERFACE_NAME" 2>/dev/null; then
        echo "成功将 $INTERFACE_NAME 添加到 $BRIDGE_NAME"
    else
        echo "将 $INTERFACE_NAME 添加到 $BRIDGE_NAME 失败"
        cleanup
        exit 1
    fi
}
# 为网桥设置 IP 地址并启动网桥
configure_bridge_ip() {
    if ifconfig "$BRIDGE_NAME" "$BRIDGE_IP" up 2>/dev/null; then
        echo "成功为 $BRIDGE_NAME 设置 IP 地址 $BRIDGE_IP 并启动网桥"
    else
        echo "为 $BRIDGE_NAME 设置 IP 地址 $BRIDGE_IP 并启动网桥失败"
        cleanup
        exit 1
    fi
}
# 添加默认网关
add_default_gateway() {
    if route add default gw "$DEFAULT_GATEWAY" 2>/dev/null; then
        echo "成功添加默认网关 $DEFAULT_GATEWAY"
    else
        echo "添加默认网关 $DEFAULT_GATEWAY 失败"
        cleanup
        exit 1
    fi
}
# 清理操作
cleanup() {
    brctl delbr "$BRIDGE_NAME" 2>/dev/null
}
# 主函数
main() {
    check_command "brctl"
    create_bridge
    enable_stp
    prepare_interface
    add_interface_to_bridge
    configure_bridge_ip
    add_default_gateway
    echo "网桥配置完成"
}
# 执行主函数
main

```

```
[root@rocky10-12 ~]# virsh edit CentOS-7-x86_64
域'CentOS-7-x86_64' XML 配置已被编辑。
# 源内容
    <interface type='network'>
      <mac address='52:54:00:a1:60:08'/>
      <source network='default'/>
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x01' slot='0x00'
function='0x0'/>
    </interface>
# 修改后内容
    <interface type='bridge'>
      <mac address='52:54:00:a1:60:08'/>
      <source bridge='br0'/>
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x01' slot='0x00'
function='0x0'/>
    </interface>
注意：
 修改了三处地方：
 type后面的network修改为了bridge
 source后面的network修改为了bridge,network后面的default修改为了br0

```

### Image Management

```bash
#create disk
qemu-img create [-q] [-f fmt] [-o options] filename [size]

# convert format
qemu-img convert -f 
```

QCOW2 supports **Copy-on-Write (CoW)**:

- **Backing file:** Stores the original disk data.
- **QCOW2 file:** Stores new or modified data.
- **Read:** Check the QCOW2 layer first, then the backing file.
- **Write:** Write changes to the QCOW2 layer without modifying the backing file.

```
         base.raw
        /        \
   vm1.qcow2   vm2.qcow2
```

Multiple VMs can share the same read-only base image while maintaining independent changes.

**QEMU Disk Images: Raw vs QCOW2**

- **Raw:** Simple disk image format with minimal overhead.
- **Sparse File:** Virtual size does not equal actual disk usage. Physical storage is allocated as data is written.
- **QCOW2:** Supports Copy-on-Write (CoW), backing files, and snapshots.
- **Backing File:** A base image shared by multiple QCOW2 images. Reads fall back to the base image; writes are stored in the child image.

```
# Create a 3 GiB raw disk
qemu-img create -f raw disk.raw 3G

# Inspect virtual size and actual disk usage
qemu-img info disk.raw
ls -lh disk.raw
du -h disk.raw

# Create a QCOW2 image based on an existing raw disk
qemu-img create -f qcow2 \
  -o backing_file=base.raw,backing_fmt=raw \
  child.qcow2

# Inspect the backing chain
qemu-img info --backing-chain child.qcow2
```

**Key differences:**

- Raw offers simpler storage and generally lower overhead.
- QCOW2 supports snapshots and space-efficient VM clones.
- Multiple QCOW2 images can share one read-only backing file.
- Never modify or delete a backing file while dependent images are in use.

#### Convert and Clone

Please reference the document, kind of inutitive.



### Resource Management

```bash
virsh setvcpus <name> <cpu count> # You have to set it when it is closed(if you want to reduce)

# Here it means you can modify the cpu count, and the max is 2
virsh edit CentOS-7-x86_64
 ...
  <vcpu placement='static'>2
  
# Before
 <vcpu placement='static'>2</vcpu>
# After
 <vcpu placement='static' current='1'>4</vcpu>

注意：
 意思就是，当前的cpu可以动态调整，当前的cpu个数为1，可调整的最大个数为4
```

```bash
virsh dominfo CentOS-7-x86_64 # check the cpu info
virsh vcpupin CentOS-7-x86_64 0 1 # bind kvm 0's cpu to host's 1st cpu. THis binding is likely to provide some optimizaiton
```

## Windows virtual Machine Creation

Because windows system and linux system do not have the same kernel,  os relies on different hardware drivers. Therefore, when doing windows virtualization, we need extra hardware drivers.

We need to download 1. windows system iso image 2. driver software.

