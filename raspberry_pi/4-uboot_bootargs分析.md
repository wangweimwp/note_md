### 分析启动命令bootcmd

### 1. 分析启动命令bootcmd

首先要在在uboot界面终止引导到linux中。

```bash
U-Boot> pri bootcmd
bootcmd=run distro_bootcmd
```

可见，bootcmd实际是执行distro_bootcmd命令，如下：

```bash
U-Boot> pri U-Boot> pri distro_bootcmd
distro_bootcmd=for target in ${boot_targets}; do run bootcmd_${target}; done=for target in ${boot_targets}; do run bootcmd_${target}; done
```

如上，distro_bootcmd遍历执行boot_targets，如下：

```bash
U-Boot> pri boot_targets
boot_targets=mmc0 mmc1 usb0 pxe dhcp
```

可见，最终是执行bootcmd_{mmc0、mmc1、usb0、pxe、dhcp}中的一个。对于rpi4b，只有mmc0有效，如下：

```bash
U-Boot> mmc list
mmcnr@7e300000: 1
emmc2@7e340000: 0 (SD)
U-Boot> pri bootcmd_mmc0
bootcmd_mmc0=devnum=0; run mmc_boot
U-Boot> pri bootcmd_mmc1
bootcmd_mmc1=devnum=1; run mmc_boot
```



如上，无论bootcmd_mmc0还是bootcmd_mmc1来启动内核，都是调用mmc_boot，如下：

```bash
U-Boot> pri mmc_boot
mmc_boot=if mmc dev ${devnum}; then devtype=mmc; run scan_dev_for_boot_part; fi
U-Boot> pri scan_dev_for_boot_part
scan_dev_for_boot_part=part list ${devtype} ${devnum} -bootable devplist; env exists devplist || setenv devplist 1; for distro_bootpart in ${devplist}; do if fstype ${devtype} ${devnum}:${distro_bootpart} bootfstype; then run scan_dev_for_boot; fi; done; setenv devplist
```

可见，第一步是探测mmc（SD）是否存在`if mmc dev ${devnum}`，第二步则执行scan_dev_for_boot_part命令来获取分区信息，如下：

```cobol
U-Boot> part list mmc 0
 
Partition Map for MMC device 0  --   Partition Type: DOS
 
Part    Start Sector    Num Sectors     UUID            Type
  1     2048            524288          87c6153d-01     0c Boot
  2     526336          14997471        87c6153d-02     83
```

如上，对于可引导分区是有标识的，因此首先获取哪个分区可以引导，并将结果放在devplist变量中。然后执行`if fstype ${devtype} ${devnum}:${distro_bootpart} bootfstype;`来确认是否是可引导分区，如下：

```bash
U-Boot> fstype mmc 0:1 bootfstype
U-Boot> echo $?
0
```

最后使用scan_dev_for_boot来查找启动目录和文件，如下：

```bash
U-Boot> pri scan_dev_for_boot
scan_dev_for_boot=echo Scanning ${devtype} ${devnum}:${distro_bootpart}...; for prefix in ${boot_prefixes}; do run scan_dev_for_extlinux; run scan_dev_for_scripts; done;run scan_dev_for_efi;
```

首先认为启动文件在`/`目录或者`/boot/`，其它目录不考虑，如下：

```bash
U-Boot> pri boot_prefixes
boot_prefixes=/ /boot/
```

然后，使用`scan_dev_for_extlinux/scan_dev_for_scripts/scan_dev_for_efi`检测启动方式，如下：

```bash
U-Boot> pri scan_dev_for_extlinux
scan_dev_for_extlinux=if test -e ${devtype} ${devnum}:${distro_bootpart} ${prefix}${boot_syslinux_conf}; then echo Found ${prefix}${boot_syslinux_conf}; run boot_extlinux; echo SCRIPT FAILED: continuing...; fi
```

查看boot_syslinux_conf文件，如下：

```cobol
U-Boot> printenv boot_syslinux_conf
boot_syslinux_conf=extlinux/extlinux.conf
```

由于并没有`extlinux.conf`这个文件，因此执行到boot_a_script命令，如下：

```bash
U-Boot> printenv scan_dev_for_scripts
scan_dev_for_scripts=for script in ${boot_scripts}; do if test -e ${devtype} ${devnum}:${distro_bootpart} ${prefix}${script}; then echo Found U-Boot script ${prefix}${script}; run boot_a_script; echo SCRIPT FAILED: continuing...; fi; done
```

由于，boot_scripts如下：

```cobol
U-Boot> printenv boot_scripts
boot_scripts=boot.scr.uimg boot.scr
```

可见，第一个分区中的boot.scr最终在这里被识别到，并赋值给script变量，然后使用boot_a_script将脚本加载到内存，并执行，如下：

```bash
boot_a_script=load ${devtype} ${devnum}:${distro_bootpart} ${scriptaddr} ${prefix}${script}; source ${scriptaddr}
```

### 2. 启动脚本

最后，使用boot.scr来做内核启动引导，如下：

```bash
# Ubuntu Classic RPi U-Boot script (for armhf and arm64)
 
# Expects to be called with the following environment variables set:
#
#  devtype              e.g. mmc/scsi etc
#  devnum               The device number of the given type
#  distro_bootpart      The partition containing the boot files
#                       (introduced in u-boot mainline 2016.01)
#  prefix               Prefix within the boot partiion to the boot files
#  kernel_addr_r        Address to load the kernel to
#  fdt_addr_r           Address to load the FDT to
#  ramdisk_addr_r       Address to load the initrd to.
 
# 设置设备树地址
# Take fdt addr from the prior stage boot loader, if available
if test -n "$fdt_addr"; then
  fdt addr ${fdt_addr}
  fdt move ${fdt_addr} ${fdt_addr_r}  # implicitly sets fdt active
else
  fdt addr ${fdt_addr_r}
fi
fdt get value bootargs /chosen bootargs
 
setenv bootargs " ${bootargs} quiet splash"
 
 
 
 
if test -z "${fk_image_locations}"; then
  setenv fk_image_locations ${prefix}
fi
 
for pathprefix in ${fk_image_locations}; do
  # Store the gzip header (1f 8b) in the kernel area for comparison to the
  # header of the image we load. Load "vmlinuz" into the portion of memory for
  # the RAM disk (because we want to uncompress to the kernel area if it's
  # compressed) and compare the word at the start
  mw.w ${kernel_addr_r} 0x8b1f  # little endian
 
  # 加载内核
  if load ${devtype} ${devnum}:${distro_bootpart} ${ramdisk_addr_r} ${pathprefix}vmlinuz; then
    kernel_size=${filesize}
 
    # 判断内核镜像是否被压缩
    if cmp.w ${kernel_addr_r} ${ramdisk_addr_r} 1; then
      # gzip compressed image (NOTE: *not* a self-extracting gzip compressed
      # kernel, just a kernel image that has been gzip'd)
      echo "Decompressing kernel..."
      unzip ${ramdisk_addr_r} ${kernel_addr_r}
      setenv kernel_size ${filesize}
      setenv try_boot "booti"
    else
      # Possibly self-extracting or uncompressed; copy data into the kernel area
      # and attempt launch with bootz then booti
      echo "Copying kernel..."
      cp.b ${ramdisk_addr_r} ${kernel_addr_r} ${kernel_size}
      setenv try_boot "bootz booti"
    fi
 
    # 加载根文件系统
    if load ${devtype} ${devnum}:${distro_bootpart} ${ramdisk_addr_r} ${pathprefix}initrd.img; then
      setenv ramdisk_param "${ramdisk_addr_r}:${filesize}"
    else
      setenv ramdisk_param "-"
    fi
 
    # 启动内核
    for cmd in ${try_boot}; do
        echo "Booting Ubuntu (with ${cmd}) from ${devtype} ${devnum}:${partition}..."
        ${cmd} ${kernel_addr_r} ${ramdisk_param} ${fdt_addr_r}
    done
  fi
done
```



如上，在boot.scr中，加载内核镜像以及根文件系统镜像，之后使用'booti bootm'等启动内核。

### 3. 设备树的加载

但有疑问，设备树是何时被加载到内存的呢？

此时，`start4.elf`文件是有些莫名其妙，由于带有".elf"结尾，因此可猜测是执行文件，如下：

```cobol
$ file start4.elf
start4.elf: ELF 32-bit LSB executable, Broadcom VideoCore III, version 1 (SYSV), statically linked, stripped
```

果然是执行文件，同时，在uboot的环境变量中可看到，如下：

```bash
fdtfile=broadcom/bcm2711-rpi-4-b.dtb
```

根据博通官方说明，芯片启动时是会自动执行`start4.elf`文件的，找到一丝线索如下：

```cobol
$ strings start4.elf | grep config.txt
config.txt
Found SD card, config.txt = %d, start.elf = %d, recovery.elf = %d, timeout = %d
config.txt
```

可见，`start4.elf`的确会读取并分析config.txt中的内容。

同样，查找设备树加载地址device_tree_address，也能找到蛛丝马迹。

```bash
$ grep device_tree_address . -nR
Binary file ./start4.elf matches
$ strings start4.elf | grep device_tree_address
device_tree_address
device_tree_address
```

由此断定，树莓派4B启动后，运行`start4.elf`程序，由`start4.elf`来提前加载设备树，执行：

```cobol
[pi4]kernel=uboot_rpi_4.bin
```

kernel标签所指定的程序。此节仅猜测，无相关编译器，无法逆向。


