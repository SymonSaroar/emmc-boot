# Booting from EMMC 
## Building a Petalinux Project
1. Create petalinux project 
```petalinux-create -t project --template zynqMP --name emmc```
2. Change directory to project root and import hardware description
	```cd emmc```
	```petalinux-config --get-hw-description /path/to/emmc.xsa```
3. Configure petalinux to use downloaded sstate mirror and disable network download during build.
4.  Set *Image Packaging Configuration > Root filesystem type* to `EXT4`
5.  Set *Device node of SD device* to `/dev/mmcblk0p2`
6.  Use `petalinux-config -c rootfs` to enable and disable following packages  
- **Essential**
	- [X] e2fsprogs --- `filesystem packages > base > e2fsprogs`
	- [X] util-linux --- `filesystem packages > base > util-linux`
- **Enable SSH server**
	- [X] imagefeature-ssh-server-openssh --- `Image Features`
	- [X] openssh-ssh --- `filesystem packages > console > network > openssh`
	- [X] openssh-sshd --- `filesystem packages > console > network > openssh`
	- [x] openssh-misc --- `filesystem packages > console > network > openssh`
	- [X] openssh-keygen --- `filesystem packages > console > network > openssh`
	- [X] openssh-scp --- `filesystem packages > console > network > openssh`
	- [X] openssh-sftp-server --- `filesystem packages > console > network > openssh`
	**Disable these packages** 
	- [ ] imagefeature-ssh-server-dropbear-- --- `Image Features`
	- [ ] packagegroup-core-ssh-dropbear --- `filesystem packages > misc > packagegroup-core-ssh-dropbear`
	- [ ] dropbear --- `filesystem packages > console > network > dropbear`
- **Optional**
	- [X] busybox --- `filesystem packages > base > busybox`
	- [X] tar --- `filesystem packages > base > tar`
	- [X] usbutils --- `filesystem packages > base > usbutils`
	- [X] blktool --- `filesystem packages > misc > blktool`
	- [X] blktrace --- `filesystem packages > misc > blktrace`
	- [X] net-tools --- `filesystem packages > misc > net-tools`
	- [X] netcat --- `filesystem packages > net > netcat`
- **Enable package manager**
	- [X] package-management --- `Image Features`
	- [X] package-feed-uris [ http://petalinux.xilinx.com/sswreleases/rel-v2020.2/feeds/ultra96-zynqmp ]
	- [X] package-feed-archs [ aarch64 noarch ultra96_zynqmp zynqmpeg zynqmp ] 
7. Add the following in *project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi*
	```
	bootargs = "earlycon clk_ignore_unused console=ttyPS0,115200 cpuidle.off=1 root=/dev/mmcblk0p2 rw rootfstype=ext4 rootwait"

	amba {
		ethernet@ff0c0000 {
			is-internal-pcspma;
			status = "okay";
		};
		ethernet@ff0d0000 {
			is-internal-pcspma;
			status="okay";
			fixed-link {
				speed = <1000>;
				full-duplex;
	        };
		};
	};
	        
	&uart0 {
		status="okay";
	};

	&uart1 {
		status="okay";
	};

	&gem1 {	
		phy-handle = <&phy0>;
	             phy0: phy@0 {
	                       #compatible = "marvell, 88E1518";
	                       #device-type = "ethernet-phy";
	                       reg = <0>;
	               };
	};
	```
8.  Build the project using `petalinux-build`
9. Create BOOT.BIN file and include u-boot as well as the fsbl using 
`petalinux-package --boot --u-boot --fsbl images/linux/zynqmp_fsbl.elf`
10. Create prebuilt package using `petalinux-package --prebuilt`
## Booting the Board
11. Set the boot mode of the board to JTAG_MODE [see mode changing]
12. boot the project in JTAG mode `petalinux-boot --jtag --prebuilt 3`
13.  Minicom to listen to JTAG serial port `sudo minicom -D /dev/ttyUSB2 -b 115200 -s`
14. Turn off Hardware control and Software control in minicom.  
## Preparing the eMMC
15.  After getting the root prompt in board. use `lsblk` to check if any partition of the mmc */dev/mmcblk0* is mounted or not. unmount them if mounted.
16. Use `df /dev/mmcblk0` to delete all partitions and create 2 primary partition
	- Partition 1 - /dev/mmcblk0p1 - 250 MB
	- Partition 2 - /dev/mmcblk0p2 - Remaining space
17. `mkfs.vfat -F 32 -n boot /dev/mmcblk0p1` to  format partition 1 as fat32
18. `mkfs.ext4 -L root /dev/mmcblk0p2` to format partition 2 as ext4
19.  Mount the formatted partitions

	root@emmc:~# mkdir -p /media/sd-mmcblk0p1
	root@emmc:~# mkdir -p /media/sd-mmcblk0p2
	root@emmc:~# mount /dev/mmcblk0p1 /media/sd-mmcblk0p1
	root@emmc:~# mount /dev/mmcblk0p2 /media/sd-mmcblk0p2
## File transfer using SSH
20. Set IP address of eth0 to local LAN. In our case its 172.32.8.201
```ifconfig eth0 172.32.8.201 netmask 255.255.255.0```
21. Ping the BJ server to check LAN connection
```ping 172.32.8.10```
22.  Use ssh secure copy protocol (scp) to transfer BOOT.BIN, boot.scr and Image file to partition 1 of the eMMC.

	root@emmc:~# scp Mazharul@172.32.8.10:/path/to/project/images/linux/BOOT.BIN /media/sd-mmcblk0p1
	root@emmc:~# scp Mazharul@172.32.8.10:/path/to/project/images/linux/boot.scr /media/sd-mmcblk0p1
	root@emmc:~# scp Mazharul@172.32.8.10:/path/to/project/images/linux/Image /media/sd-mmcblk0p1
23. Transfer rootfs.tar.gz to partition 2 of the eMMC and extract it there
	```
	root@emmc:~# scp Mazharul@172.32.8.10:/path/to/project/images/linux/rootfs.tar.gz /media/sd-mmcblk0p2
	root@emmc:~# cd /media/sd-mmcblk0p2
	root@emmc:/media/sd-mmcblk0p2# tar -xzvf rootfs.tar.gz
	```
## eMMC Boot
24. Change boot mode to EMMC_MODE
25. Turn on the board.

---
---
# Changing Boot Mode
1. Run xsdb from vivado
2. connect with the board
```xsdb% connect```
3. verify connection by running `target`
### Changing to JTAG Mode
```
xsdb% targets -set -filter {name =~ "PSU"}
xsdb% stop
xsdb% mwr 0xffca0010 0x0
xsdb% mwr 0xff5e0200 0x0100
xsdb% rst -system
```
### Changing to EMMC Mode
```
xsdb% targets -set -filter {name =~ "PSU"}
xsdb% stop
xsdb% mwr 0xffca0010 0x0
xsdb% mwr 0xff5e0200 0x6100
xsdb% rst -system
```
### Running the board
```
xsdb% targets -set -filter {name =~ "PSU"}
xsdb% after 2000
xsdb% con
```
