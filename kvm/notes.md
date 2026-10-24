## Virtualisation technology

Full virtualisation techonology: KVM

Half virtualisation technology: Xen

KVM itself is a single server software.

After you enable the cpu accelerated virtualisation, KVM doesn't replace or take over the Linux kernel. It extends the Linux kernel with hypervisor capabilities.

### How do we manage virtual machine?
1. by using service like libvirt and a bunch of command lines
2. By using software like qemu

### Cloud Computing

By intoducing kvm and mutiple servers and a software to create the resource pool, you bascially create a cloud computing environment. 

### Checking whether cpu supports the kvm

```bash
grep -i kvm /boot/config...
```

