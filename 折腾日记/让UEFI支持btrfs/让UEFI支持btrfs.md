# 让UEFI支持btrfs

装系统的时候如果要efi启动的话，启动程序文件要放在FAT格式的分区(或者有的电脑也可以放在NTFS的分区里)，所以如果系统盘不是这个格式的话还要专门划一个esp分区来存放EFI启动文件。那么能不能把EFI启动文件放在btrfs上呢？

我了解折腾了下，UEFI之所以好像叫什么可扩展固件接口，就是因为它可扩展，它跟操作系统有点类似(也许就是一个操作系统")，它有类似C:盘D:盘的盘符，不过叫FS0: FS1:，并且也有驱动程序并且可以加载驱动，其中就有文件系统的驱动程序，并且系统的EFI启动程序就相当于UEFI的可执行程序啦，并且还都是PE格式哦。

那么能不能加载btrfs的驱动呢？[rEFInd](https://sourceforge.net/projects/refind/)就有好多文件系统的驱动，(没错，EFI启动程序也可以用UEFI"系统"的接口打开文件，rEFInd直接让"系统"加载了文件系统驱动然后再打开文件的，...，嗯rEFInd好像是链式启动，quibble应该就要打开文件了吧)

那么就手动加载它的驱动试试呗

qemu 启动
> ```
> #!/bin/env zsh
> @() { __+=" $@" ; }
> 
> @ qemu-system-x86_64
> @	-m 16G -object memory-backend-file,id=mem,size=16G,mem-path=/dev/shm,share=on -numa node,memdev=mem
> @	-smp cpus=8,sockets=1,cores=8,threads=1
> @	-bios /usr/share/edk2/x64/OVMF.4m.fd
> @	-cpu host
> @	-machine q35
> @	-enable-kvm
> @	-netdev tap,id=netwan,ifname=qemu-tap-test,script=no,downscript=no -device virtio-net,netdev=netwan,mac=52:55:57:00:00:00
> @	-drive file=.qcow2,format=qcow2
> @	-boot menu=on
> @	-serial stdio
> @	-chardev socket,id=char1,path=virtiofs.socket -device vhost-user-fs-pci,queue-size=1024,chardev=char1,tag=virtiofs
> /usr/lib/virtiofsd --socket-path=virtiofs.socket -o source=virtiofs -o cache=always &
> exec eval $__ $@
> ```
> 


windows有cmd，linux下有bash,zsh，UEFI下也有个EFI Internal Shell的程序
![启动efishell.1](启动efishell.1.png)
![启动efishell.2](启动efishell.2.png)

> UEFI Interactive Shell v2.2
> 
> EDK II
> 
> UEFI v2.70 (EDK II, 0x00010000)
> 
> Mapping table
> 
>       FS0: Alias(s):F1:
>
>           PciRoot(0x0)/Pci(0x3,0x0)
>
>      BLK0: Alias(s):
>
>           PciRoot(0x0)/Pci(0x1F,0x2)/Sata(0x0,0xFFFF,0x0)
>
>      BLK1: Alias(s):
>
>           PciRoot(0x0)/Pci(0x1F,0x2)/Sata(0x2,0xFFFF,0x0)
>
> Press ESC in 1 seconds to skip startup.nsh or any other key to continue.
> 
> Shell> 
> 

`mode` 调整分辨率

> Shell> `mode`
> 
> Available modes for console output device.
> 
>   Col    80 Row    25   
> 
>   Col   100 Row    31  *
> 
> Shell> `mode 100 31`

`map`看盘符
> Shell> `map`
> 
> Mapping table
> 
>       FS0: Alias(s):F1:
>
>           PciRoot(0x0)/Pci(0x3,0x0)
>
>      BLK0: Alias(s):
>
>           PciRoot(0x0)/Pci(0x1F,0x2)/Sata(0x0,0xFFFF,0x0)
>
>      BLK1: Alias(s):
>
>           PciRoot(0x0)/Pci(0x1F,0x2)/Sata(0x2,0xFFFF,0x0)

`dir`或`ls`命令能看到里面的文件

> Shell> `ls fs0:`
> 
> Directory of: fs0:\
> 
> 01/01/1970  00:00 <DIR> r           0  .
> 
> 01/01/1970  00:00              62,840  btrfs_x64.efi
> 
> 01/01/1970  00:00              12,477  NvVars
> 
>           2 File(s)      75,317 bytes
>
>           1 Dir(s)

`load`加载驱动

> Shell> `load fs0:\btrfs_x64.efi`
> 
> Image 'FS0:\btrfs_x64.efi' loaded at 7D7F1000 - Success

`drivers`查看加载的驱动列表

> Shell> `drivers`
> 
> | DRV | VERSION | TYPE | CFG |DING | #D | #C | DRIVER NAME | IMAGE NAME |
> |----|----|----|----|----|----|----|----|----|
> | 5E | 0000000A | B | - | - | 1 | 7 | PCI Bus Driver | PciBusDxe |
> | 60 | 00000010 | ? | - | - | - | - | Virtio PCI Driver | VirtioPciDeviceDxe |
> | 61 | 00000010 | D | - | - | 1 | - | Virtio 1.0 PCI Driver | Virtio10 |
> | 62 | 00000010 | ? | - | - | - | - | Virtio Block Driver | VirtioBlkDxe |
> | 63 | 00000010 | ? | - | - | - | - | Virtio SCSI Host Driver | VirtioScsiDxe |
> | 64 | 00000010 | ? | - | - | - | - | Virtio Serial Driver | VirtioSerialDxe |
> | 65 | 0000000A | D | - | - | 2 | - | Platform Console Management Driver | ConPlatformDxe |
> | 66 | 0000000A | D | - | - | 2 | - | Platform Console Management Driver | ConPlatformDxe |
> | 67 | 0000000A | B | - | - | 2 | 2 | Console Splitter Driver | ConSplitterDxe |
> | 68 | 0000000A | ? | - | - | - | - | Console Splitter Driver | ConSplitterDxe |
> | 69 | 0000000A | ? | - | - | - | - | Console Splitter Driver | ConSplitterDxe |
> | 6A | 0000000A | B | - | - | 2 | 2 | Console Splitter Driver | ConSplitterDxe |
> | 6B | 0000000A | B | - | - | 1 | 1 | Console Splitter Driver | ConSplitterDxe |
> | 6F | 0000000A | D | - | - | 1 | - | Graphics Console Driver | GraphicsConsoleDxe |
> | 70 | 0000000A | B | - | - | 1 | 1 | Serial Terminal Driver | TerminalDxe |
> | 71 | 0000000A | D | - | - | 2 | - | Generic Disk I/O Driver | DiskIoDxe |
> | 72 | 0000000B | ? | - | - | - | - | Partition Driver(MBR/GPT/El Torito) | PartitionDxe |
> | 75 | 0000000A | B | - | - | 1 | 1 | SCSI Bus Driver | ScsiBus |
> | 76 | 0000000A | D | - | - | 1 | - | Scsi Disk Driver | ScsiDisk |
> | 77 | 0000000A | D | - | - | 1 | - | Sata Controller Init Driver | SataController |
> | 78 | 00000010 | D | - | - | 1 | - | AtaAtapiPassThru Driver | AtaAtapiPassThruDxe |
> | 79 | 00000010 | B | - | - | 1 | 1 | ATA Bus Driver | AtaBusDxe |
> | 7A | 00000010 | ? | - | - | - | - | NVM Express Driver | NvmExpressDxe |
> | 7B | 00000010 | B | - | - | 1 | 3 | OVMF Sio Bus Driver | SioBusDxe |
> | 7C | 0000000A | B | - | - | 1 | 1 | PCI SIO Serial Driver | PciSioSerialDxe |
> | 7D | 0000000A | D | - | - | 1 | - | PS/2 Keyboard Driver | Ps2KeyboardDxe |
> | 80 | 0000000A | ? | - | - | - | - | FAT File System Driver | Fat |
> | 81 | 00000010 | ? | - | - | - | - | UDF File System Driver | UdfDxe |
> | 82 | 00000010 | D | - | - | 1 | - | Virtio Filesystem Driver | VirtioFsDxe |
> | 85 | 0000000A | ? | - | - | - | - | Simple Network Protocol Driver | SnpDxe |
> | 86 | 0000000A | B | - | - | 1 | 1 | VLAN Configuration Driver | VlanConfigDxe |
> | 87 | 0000000A | B | - | - | 1 | 3 | MNP Network Service Driver | MnpDxe |
> | 88 | 0000000A | B | - | - | 1 | 1 | ARP Network Service Driver | ArpDxe |
> | 89 | 0000000A | B | - | - | 1 | 2 | DHCP Protocol Driver | Dhcp4Dxe |
> | 8A | 0000000A | B | - | - | 2 | 12 | IP4 Network Service Driver | Ip4Dxe |
> | 8B | 0000000A | B | - | - | 7 | 12 | UDP Network Service Driver | Udp4Dxe |
> | 8C | 0000000A | B | - | - | 2 | 2 | MTFTP4 Network Service | Mtftp4Dxe |
> | 8D | 0000000A | B | - | - | 1 | 2 | DHCP6 Protocol Driver | Dhcp6Dxe |
> | 8E | 0000000A | B | - | - | 2 | 12 | IP6 Network Service Driver | Ip6Dxe |
> | 8F | 0000000A | B | - | - | 6 | 10 | UDP6 Network Service Driver | Udp6Dxe |
> | 90 | 0000000A | B | - | - | 1 | 1 | MTFTP6 Network Service Driver | Mtftp6Dxe |
> | 91 | 0000000A | B | - | - | 8 | 1 | UEFI PXE Base Code Driver | UefiPxeBcDxe |
> | 92 | 0000000A | B | - | - | 7 | 1 | UEFI PXE Base Code Driver | UefiPxeBcDxe |
> | 95 | 00000000 | D | - | - | 1 | - | DNS Network Service Driver | DnsDxe |
> | 96 | 00000000 | D | - | - | 1 | - | DNS Network Service Driver | DnsDxe |
> | 97 | 0000000A | D | - | - | 1 | - | HttpDxe | HttpDxe |
> | 98 | 0000000A | D | - | - | 1 | - | HttpDxe | HttpDxe |
> | 99 | 0000000A | B | - | - | 2 | 1 | UEFI HTTP Boot Driver | HttpBootDxe |
> | 9A | 0000000A | B | - | - | 3 | 1 | UEFI HTTP Boot Driver | HttpBootDxe |
> | 9B | 0000000A | D | - | - | 1 | - | iSCSI Driver | IScsiDxe |
> | 9C | 0000000A | D | - | - | 1 | - | iSCSI Driver | IScsiDxe |
> | 9E | 00000010 | ? | - | - | - | - | Virtio Network Driver | VirtioNetDxe |
> | 9F | 00000020 | ? | - | - | - | - | Usb Uhci Driver | UhciDxe |
> | A0 | 00000030 | ? | - | - | - | - | Usb Ehci Driver | EhciDxe |
> | A1 | 00000030 | ? | - | - | - | - | Usb Xhci Driver | XhciDxe |
> | A2 | 0000000A | ? | - | - | - | - | Usb Bus Driver | UsbBusDxe |
> | A3 | 0000000A | ? | - | - | - | - | Usb Keyboard Driver | UsbKbDxe |
> | A4 | 00000011 | ? | - | - | - | - | Usb Mass Storage Driver | UsbMassStorageDxe |
> | A5 | 00000010 | B | - | - | 1 | 1 | QEMU Video Driver | QemuVideoDxe |
> | A6 | 00000010 | ? | - | - | - | - | Virtio GPU Driver | VirtioGpuDxe |
> | A7 | 00000010 | ? | - | - | - | - | Virtio Random Number Generator Driv | VirtioRngDxe |
> | A8 | 0000000A | B | - | - | 3 | 4 | TCP Network Service Driver | TcpDxe |
> | A9 | 0000000A | B | - | - | 3 | 4 | TCP Network Service Driver | TcpDxe |
> | B5 | 017D5C56 | B | - | - | 2 | 2 | 1af41000.efidrv | Offset(0x10E00,0x273FF) |
> | FF | 00000010 | D | - | - | 1 | - | rEFInd 0.14.2 btrfs File System Dri | \btrfs_x64.efi |



`map -r`重设盘符
> Shell> `map -r`
> 
> Mapping table
> 
>       FS1: Alias(s):F1:
> 
>           PciRoot(0x0)/Pci(0x3,0x0)
> 
>       FS0: Alias(s):F0a65535a:;BLK0:
> 
>           PciRoot(0x0)/Pci(0x1F,0x2)/Sata(0x0,0xFFFF,0x0)
> 
>      BLK1: Alias(s):
> 
>           PciRoot(0x0)/Pci(0x1F,0x2)/Sata(0x2,0xFFFF

可以看到文件系统被识别了 _(当然盘符也重新分配了)_


那么运行一下呢
 _(没错，这里的程序也可以有命令行参数哦)_

> Shell> `fs0:`
> 
> FS0:\> `boot\vmlinuz-linux initrd=boot\initramfs-linux-fallback.img root=UUID=59004200-5900-3100-3300-300036000000 rw splash`

启动成功了
![启动成功](启动成功.png)

edk2ovmf的设置里可以设置自动加载驱动哦
![edk2自动加载驱动1](edk2自动加载驱动1.png)
![edk2自动加载驱动2](edk2自动加载驱动2.png)
![edk2自动加载驱动3](edk2自动加载驱动3.png)
![edk2自动加载驱动4](edk2自动加载驱动4.png)
![edk2自动加载驱动5](edk2自动加载驱动5.png)
![edk2自动加载驱动6](edk2自动加载驱动6.png)
![edk2自动加载驱动7](edk2自动加载驱动7.png)


既然开机会自动识别btrfs文件系统，那么把btrfs上的EFI启动程序添加到启动列表呢

![edk2添加启动项.1](edk2添加启动项.1.png)
![edk2添加启动项.2](edk2添加启动项.2.png)
![edk2添加启动项.3](edk2添加启动项.3.png)
![edk2添加启动项.4](edk2添加启动项.4.png)
![edk2添加启动项.5](edk2添加启动项.5.png)
![edk2添加启动项.6](edk2添加启动项.6.png)
![edk2添加启动项.7](edk2添加启动项.7.png)

或者进linux后运行
`efibootmgr -c -d /dev/sda -L 'Arch Linux' -l '/boot/vmlinuz-linux' --unicode "initrd=/boot/initramfs-linux-fallback.img root=UUID=59004200-5900-3100-3300-300036000000 rw splash" # -p 1 ` 
> - `-c`代表创建
> - `-d`后面是硬盘
> - `-p`是分区数，如果没有分区就没有p参数
> - `-L`名称
> - `-l`文件路径(对于分区而言)，会自动转换斜杠
> - `--unicode` 给EFI启动程序的参数


正常启动

![启动成功](启动成功.png)

但是这个驱动还是要在别的分区上的才能被加载

那么为什么FAT文件系统就可以直接识别呢？

用[UEFITool](https://github.com/LongSoft/UEFITool)打开固件，发现原来内嵌了FAT的驱动

![UEFITool发现原来内嵌了FAT驱动](UEFITool发现原来内嵌了FAT驱动.png)

那么怎么把btrfs驱动也内嵌进入呢？

发现这里相当于结构体，有属性和内容

![UEFITool右键菜单](UEFITool右键菜单.png)

其中这些是操作"结构体"
- Extract as is...	Ctrl+E	
- Insert into...	Ctrl+l	
- Insert before...	Ctrl+Alt+|	
- Insert after...	Ctrl+Shift+|	
- Replace as is...	Ctrl+R	
- Remove	Ctrl+Del

其中这些是操作"内容"
- Extract body...	Ctrl+Shift+E	
- Replace body...	Ctrl+Shift+R	

然后这又相当于List与元素的区别
![UEFITool_ffs与sct](UEFITool_ffs与sct.png)

- Type显示为File(导出导入时默认为ffs后缀)的为List
- Type显示为Section(导出导入时默认为sct后缀)的为元素



那就把btrfs驱动也嵌入呗


1. 先把ffs结构体导出，用16进制编辑器改个GUID(前面的字节)，再找个地方插入
2. 把 PE32 image section 的内容替换成btrfs驱动
3. User interface section对应右边的Text，还有Version section，好像是给自己看的（嗯好像drivers也会显示，不过那也是给自己看的嘛～），好像改不改都行，(好像直接删掉也行)，可以导出内容用16进制编辑器修改再导入
4. 如果有DXE dependency section的要移除，否则好像不会自动加载，具体的我也不懂

![UEFITool修改ffssct.1](UEFITool修改ffssct.1.png)
![UEFITool修改ffssct.2](UEFITool修改ffssct.2.png)
![UEFITool修改ffssct.2](UEFITool修改ffssct.3.png)
![UEFITool修改ffssct.4](UEFITool修改ffssct.4.png)
![UEFITool修改ffssct.5](UEFITool修改ffssct.5.png)
![UEFITool修改ffssct.6](UEFITool修改ffssct.6.png)
![UEFITool修改ffssct.7](UEFITool修改ffssct.7.png)
![UEFITool修改ffssct.8](UEFITool修改ffssct.8.png)
![UEFITool修改ffssct.9](UEFITool修改ffssct.9.png)
![UEFITool修改ffssct.10](UEFITool修改ffssct.10.png)



然后保存并把qemu的bios参数选择为修改的固件就可以把esp分区删了，btrfs就能当esp分区啦！连分区都不用啦！



注：
- 这个驱动对于btrfs是只读的，不像FAT那样是可读写的。
- 如果加载多个同一种格式的文件系统驱动也只会有一个驱动接管分区。[rEFInd](https://sourceforge.net/projects/refind/)的btrfs驱动与[quibble](https://github.com/maharmstone/quibble)暂不兼容，导致[quibble](https://github.com/maharmstone/quibble)无法启动btrfs上的windows。[quibble的btrfs驱动](https://github.com/maharmstone/btrfs-efi)与Linux内核的UKI暂不兼容，导致Linux内核的UKI功能无法找到initrd参数的initrd。

<details markdown='1'><summary></summary>
注：

我在华硕和技嘉的电脑上试了，无法通过校验，无法正常刷入，需要通过BIOS FlashBack 或者用编程器刷入。

我在华硕的电脑上试了，直接开启安全启动后也能识别到btrfs分区。

</details>
