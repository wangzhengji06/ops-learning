# Kernel

Kernel is split into modules: THe core module(load on boot) + Optional module(load when you need it)

The format is .ko.


## Compile
ldd to reveal the dependency on system library. You will see file with .so, which is software modules.

Dynamic compile: Add link to system library software, highly dependent on system.

Static compile: Copy those also into software itself, independent of system.

## Module Management

lsmod: list the current kernel module information

modprobe: automatically install and uninstall kernel module

insmod: manually insert the module

rmmod: remove the kernel module 


## Parameter setting

sysctl -a: Check all the parameters


temporary: 

echo  

sysctl -w 'key=value'

permamnent:

vim /etc/sysctl.conf

vim /etc/sysctl.d/xxx.conf

sysctl -p 
