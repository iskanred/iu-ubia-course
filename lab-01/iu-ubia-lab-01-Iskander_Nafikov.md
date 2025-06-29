* **Name**: Iskander Nafikov
* **E-mail**: i.nafikov@innopolis.university
* **Username**: [iskanred](https://github.com/iskanred)
---
# Task 1 - PXE Installation

## 1. PXE Server Setup

### 1.1
>[!Task Description]
>**1.1.** Create the first virtual machine using VirtualBox and isolate a private network on your workstation: do not pollute our shared network with your own DHCP service. There are several network settings offered by VirtualBox, choose the right one accordingly and Install your PXE server there.
> Hint: use two network adapters
#### **GNS3**
To make the process of network setup more **simple and explicit** I decided to use [GNS3](https://www.gns3.com/) because it gives a visual representation of a virtual network.
* Below is the configuration of network topology that I created for this task:
	![[Pasted image 20250624203627.png]]
#### **Brief overview of nodes**
Let me briefly explain what does each node is used for:
* **`pxe-server-bios`** is a QEMU/KVM-virtualised Ubuntu Cloud machine that runs PXE server for BIOS-based boot loading.
* **`pxe-server-uefi`** is a QEMU/KVM-virtualised Ubuntu Cloud machine that runs PXE server for UEFI-based boot loading.
* **`pxe-client`** is a QEMU/KVM-virtualised machine that behaves as a PXE client with no OS setup (empty disk image) because we will install OS with the help of PXE.
* **`Switch`** is a simple network switch to allow multiple nodes to be connected to the same router inside the same LAN.
* **`Router`** is a QEMU/KVM-virtualised [Mikrotik](https://en.wikipedia.org/wiki/MikroTik) router that allows me to create LAN for my nodes easily and to provide DCHP service (DHCP server).
* **`Internet`** is the node that allows all other nodes to interact with the internet. My router uses NAT and DHCP (DHCP client) to allow this. My host machine acts as a gateway in this configuration.
* I didn't use VirtualBox since it works slower on Linux machines especially in GNS3, while Qemu/KVM is remains the best option
#### **Deep overview of nodes**
Now let's dive deeper in configuration of nodes
1. **`Router`**
	- Router is virtualised using [QEMU](https://ru.wikipedia.org/wiki/QEMU)/ [KVM](https://ru.wikipedia.org/wiki/KVM) and was created right inside the GNS3 software. The OS used is [Mikrotik CHR](https://help.mikrotik.com/docs/spaces/ROS/pages/18350234/Cloud+Hosted+Router+CHR) firmware.
	- The local network is `10.0.0.0/24` on the port `ether2`, while router's address is `10.0.0.1/24`
		![[Pasted image 20250624161147.png]]
	* It maintains DHCP server with the following pool of addresses: `10.0.0.5 - 10.0.0.254` on the port `ether2`
		![[Pasted image 20250624161240.png]]
		![[Pasted image 20250624165838.png]]
	* It maintains DHCP client to allow my nodes to be connected to the Internet on the port `ether1`
		![[Pasted image 20250624161303.png]]
	* It maintains NAT to to allow my nodes to be connected to the Internet on the port `ether1`. I created only `source` NAT because I don't need to connect to my nodes outside the topology.
		![[Pasted image 20250624162706.png]]
2. **`pxe-server-bios`**
	- This node has a statically set IP address which is `10.0.0.2` that is configured using [Netplan](https://ubuntu.com/server/docs/about-netplan)
		![[Pasted image 20250624195242.png]]
	- This VM is created right inside the GNS3 using QEMU/KVM virtualisation
		<img src="Pasted image 20250624204721.png" width=500 />
		<img src="Pasted image 20250624204741.png" width=500 />
	<img src="Pasted image 20250624204800.png" width=500 />
3. **`pxe-server-uefi`** has pretty similar network configuration to `pxe-server-bios`
	- This node has a statically set IP address which is `10.0.0.3` that is configured using Netplan
		![[Pasted image 20250624195318.png]]
	- This VM is created right inside the GNS3 using QEMU/KVM virtualisation
		<img src="Pasted image 20250624204859.png" width=500 />
		<img src="Pasted image 20250624204843.png" width=500 />
		<img src="Pasted image 20250624204823.png" width=500 />
4. **`pxe-client`**
	- This VM is created right inside the GNS3 using QEMU/KVM virtualisation
		<img src="Pasted image 20250624203831.png" width=500 />
	- As we can see this node has some differences:
		- For instance, its boot priority is set to Network since it will boot from network while its "HDD" (virtual disk image) is empty.
		- Also, its console type is `spice` instead of `telnet` since we need to have a graphical interface
	- What is more, to "make HDD empty" I recreated a new virtual disk image in QCOW2 format
		<img  width=500 src="Pasted image 20250624204211.png" />
	- Finally, I removed `-nographich` option in QEMU/KVM additional settings
		<img  width=500 src="Pasted image 20250624204455.png" />
### 1.2
> [!Task Description]
> **1.2.** Hint: you need to set up some of the services such as DHCP, TFTP, HTTP, NFS, and some boot loader (e.g. PXELINUX or GRUB) based on your approach. Also, take care of UEFI and BIOS based approaches.
> Hint: understand BIOS based approach but implement UEFI approach.
> **1.2.1.** Write about each service’s role in the PXE environment.
> **1.2.2.** Your PXE server should serve the operating system of your choosing.
#### **Description of a PXE process**
> [!Task Description]
> **1.2.1.** Write about each service’s role in the PXE environment.

In simple words the process works as following:
1. When some machine wants to boot from the network (rather than local drive such as CD/DVD or USB) it runs **[DHCP](https://en.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol)** process in the local network to receive its IP address and an address of a [PXE](https://en.wikipedia.org/wiki/Preboot_Execution_Environment) server.
2. DHCP server provides it an IP address together with the IP address of the PXE server and a name of a file with a [bootloader](https://en.wikipedia.org/wiki/Bootloader).
3. Then the machine asks the PXE server to download the file of a bootloader. For this purpose it uses **[TFTP](https://en.wikipedia.org/wiki/Trivial_File_Transfer_Protocol)**, that's why **PXE server is actually a TFTP server that serves a bootloader**.
4. After downloading a bootloader, the machine runs it and a user can then select different options to boot: tools such as [Memtest](https://www.memtest86.com/), [Live CD](https://en.wikipedia.org/wiki/Live_CD) or OS installers. Let's assume this user wants to install some OS, so they select the corresponding option and the bootloader starts downloading necessary files for installation process (OS's kernel and image) from the PXE server or from the internet.
5. Since TFTP is based on UDP (in contrast to FTP or HTTP that are based on TCP), it can be really slow for a reliable data transfer. Therefore, often **HTTP** or **FTP** are used instead of TFTP for downloading necessary files.
6. At the same time for diskless installations where the root filesystem is mounted from a common place **NFS** may be used.

Thus, in my case I will deploy the following services:
1) **DHCP server**: For assigning IP and transferring info about PXE server such IP address and name of bootloader.
2) **TFTP server**: For serving a bootloader.
3) **Bootloader**: For selecting and option to install OS and starting this process.
4) **HTTP server**: For serving files that are necessary to install and boot OS since TFTP is slow.
5) **Operating System**: to install it on **`pxe-client`** node.

>❗️I implemented both **BIOS**- and **UEFI**-based approaches to understand PXE more deeply.
#### **DHCP server setup**
 **Both approaches** have setup of DHCP server in common.
- I have already shown configuration of my DHCP server that is set up on the `router` above.
- However, let me show it in more detail.
- Again here is my DHCP server configuration on the `ether2` interface of my router . We see that the pool of addresses includes all the addresses inside the LAN starting from `10.0.0.5`. This was made to reserve IP-addresses for PXE servers and a little bit more just in case.
	![[Pasted image 20250624165948.png]]
- Then I want to show the configuration that is related to PXE process.
	![[Pasted image 20250624165106.png]]
- Above we see two important options that are hidden by now:
	- **`next-server`**: stores an address of a PXE server (which is actually a TFTP server that servers a bootloader). When a machine boots from network it receives this address and requests a server by this IP for a bootloader file.
	- **`boot-file-name`**: is exactly the name of a bootloader file.
#### **BIOS-based approach**
First, let me explain BIOS-based approach. In this section I will primarily use **`pxe-server-bios`** node which is not surprising I believe :)
##### 1) DHCP server configuration
- Let's check what are the parameters `next-server` and `boot-file-name` of the DHCP server for the BIOS-based approach
	![[Pasted image 20250624165149.png]]
* We see that for the BIOS-based approach DHCP server's options are the following:
	- **`next-server`** $=$ `10.0.0.2` which is an address of a **`pxe-server-bios`**
	- **`boot-file-name`** $=$ `pxelinux.0` which is a name of a file with a bootloader. ❗️Later we will look deeply at this file and what does this bootloader provide for us.
##### 2) TFTP server setup
- TFTP server is necessary for file transfer from PXE server to PXE client. It is used to transfer a bootloader and also can be used to transfer a software that was selected in the bootloader.
- I set up my TFTP server on the **`pxe-server-bios`** machine using [tftp-hpa](https://packages.debian.org/ru/sid/tftpd-hpa) which is "an enhanced version of the BSD TFTP client and server".
- I downloaded `tftp-hpa` package on my **`pxe-server-bios`** machine.
	```shell
	sudo apt install tftp-hpa tftpd-hpa
	```
- Below is the state of my TFTP server that is hosted on **`pxe-server-bios`**
	![[Pasted image 20250624175108.png]]
- I created `/var/lib/tftpboot` directory for my TFTP server to serve bootloader files there.❗️Further we will look deeply at it.
	![[Pasted image 20250624175224.png]]
- Below is the configuration of my TFTP server. I changed `TFTP_DIRECTORY` to my newly created `/var/lib/tftpboot/`
	![[Pasted image 20250624175346.png]]
##### 3) PXELINUX setup
- [SYSLINUX](https://en.wikipedia.org/wiki/SYSLINUX) is "a suite of five different bootloaders for starting up Linux distros on computers".
- We need a variant of **SYSLINUX** that is called **PXELINUX**, which is a lightweight **bootloader used in PXE** environments for booting from a network.
- **PXELINUX** provides a **bootstrap program** that loads and configures an operating system kernel that puts user in control of the computer.
- I downloaded `pxelinux` package (which depends on `syslinux-common` package that keeps necessary [library modules](https://wiki.syslinux.org/wiki/index.php?title=Library_modules) for all syslinux variants).
	```shell
	sudo apt install pxelinux
	```
	![[Pasted image 20250624175511.png]]
	![[Pasted image 20250624175542.png]]
- Firstly, I copied the necessary [Library modules](https://wiki.syslinux.org/wiki/index.php?title=Library_modules)  for booting in BIOS that are provided by `syslinux-common` package to the directory of my TFTP server. These modules are a part of [COMBOOT API](https://wiki.syslinux.org/wiki/index.php?title=Comboot_API).
UEFI bootloader	```shell
	sudo cp /usr/lib/syslinux/modules/bios/{ldlinux.c32,libcom32.c32,libutil.c32,vesamenu.c32} /var/lib/tftpboot
	```
	![[Pasted image 20250624195606.png]]
- Secondly, I copied the bootloader itself (file `pxelinux.0`) that is provided by `pxelinux` package to the directory of my TFTP server.
	```shell
	sudo cp /usr/lib/PXELINUX/pxelinux.0 /var/lib/tfptboot
	```
	![[Pasted image 20250624195927.png]]
##### 4) OS setup
* I chose the latest Ubuntu Server 24.04.02 as the operating system to install on **`pxe-client`**.
* Firstly, I downloaded the ISO image from the official [Yandex Mirror](https://mirror.yandex.ru/ubuntu-releases) for Ubuntu.
	```shell
	wget https://mirror.yandex.ru/ubuntu-releases/24.04.2/ubuntu-24.04.2-live-server-amd64.iso
	```
	![[Pasted image 20250624200023.png]]
* Afterwards, I mounted it to `/mnt`
	```shell
	 sudo mount --read-only ubuntu-24.04.2-live-server-amd64.iso /mnt
	```
	![[Pasted image 20250624183912.png]]
	![[Pasted image 20250624183955.png]]
* Finally, I copied Linux kernel (`vmlinuz`) and file system for it (`initrd`) to the `/var/lib/tftpboot` from `mnt/casper` directory that helps to use a Live image.
	![[Pasted image 20250624184411.png]]
- Afterward, I unmounted `/mnt` partition
	![[Pasted image 20250624184612.png]]
##### 5) HTTP server setup
- Since TFTP is slow I preferred to transfer OS installation files through **HTTP**.
- For this purpose I had to deploy HTTP server on my **`pxe-server-bios`**
- I selected [Apache](https://httpd.apache.org/) HTTP server due to its popularity for PXE installations
- I downloaded `apache2` package
	```shell
	sudo apt install apache2
	```
	![[Pasted image 20250624184812.png]]
- Now when the Live kernel's image is ready to be downloaded through TFPT and launched on the machine we can configure to make the installation of the whole Ubuntu OS through HTTP. To do this we can just put the ISO image to the appropriate Apache's directory. Let's create it:
	```shell
	sudo mkdir /var/www/html/ubuntu
	```
	![[Pasted image 20250624184850.png]]
- And then copy the ISO image to it
	```shell
	sudo cp ubuntu-24.04.1-live-server-amd64.iso /var/www/html/ubuntu/
	```
	![[Pasted image 20250624184940.png]]
- Let's check if it actually works
	![[Pasted image 20250624185231.png]]
- And yes, we see that our HTTP server responded us with the ISO image for a request. Moreover, this image is the same as we moved to `/var/www/html/ubuntu` since the sizes are the same.
- Finally, I can remove unnecessary images from my home directory to save disk space
	![[Pasted image 20250624190153.png]]
##### 6) Configuring PXELINUX menu
- Firstly, let's create a directory `pxelinux.cfg` and default configuration for PXE menu.
	![[Pasted image 20250624191335.png]]
- I created the following simple menu config:
	```
	UI vesamenu.c32 # graphical menu
	
	# Default boot configuration that will be selected after a timeout
	DEFAULT ubuntu
	TIMEOUT 600 # 60 seconds
	
	MENU TITLE Booting by PXE
	
	SAY Please, select the boot option...
	
	# Ubuntu boot configuration
	LABEL ubuntu
	  MENU LABEL Install ^Ubuntu Server 24.04
	
	  KERNEL vmlinuz
	  INITRD initrd
	  # Kernel options
	  APPEND root=/dev/ram0 ramdisk_size=4000000 ip=dhcp url=http://10.0.0.2:80/ubuntu/ubuntu-24.04.2-live-server-amd64.iso
	
	  TEXT HELP
	  Ubuntu Server 24.04 will be downloaded via TFTP+HTTP and installed
	  ENDTEXT
	```
- Below is an explanation of the options:
	- `UI`: refers to menu COM module; `vesamenu` is a graphical menu
	- `DEFAULT`: is a default label to be selected
	- `TIMEOUT`: is a number of 1/10 seconds before the selected option will be booted
	- `MENU TITLE`: is a title of SYSLINUX menu
	- `LABEL`: is the menu option label
		- `MENU LABEL`: is the displayed menu option name
		- `KERNEL`: refers to kernel's binary
		- `INITRD`: refers to `initrd`
		- `APPEND`: allow to set options to run the kernel with. I set 4Gb for a RamDisk in order to download the whole ISO image from the server and keep it in memory before installation.
		- `TEXT HELP` {message} `ENDTEXT`: is the helping text to be displayed when an option is selected
##### 7) Testing PXELINUX
- Finally, we can test our configuration. Let's run **`pxe-client`** and boot from network.
	<img  width=550 src="Pasted image 20250624204555.png" />
- After starting we see such a menu where iPXE is starting and configures network interface 
	![[Pasted image 20250624205315.png]]
- After network configuration being done we can see several more messages from which we can imply that the machine got an IP address on its `net0` interface $=$ `10.0.0.254/24` from the DCHP server together with the PXE server address and PXE file name.
	![[Pasted image 20250624205508.png]]
- In Wireshark we can immediately see DHCP packets started being between the **`pxe-client`** and the **`Router`**:
	![[Pasted image 20250624210251.png]]
- Here we can also notice that the **`pxe-server-bios`** got IP address $=$ `10.0.0.254`
- Let's check the `DHCP Offer` message from the **`Router`**. We can see that our DHCP server responded with the address of our PXE server (`10.0.0.2` which is the **`pxe-server-bios`**) and the name of the boot file (`pxelinux.0`)
	![[Pasted image 20250624210501.png]]
- Then we can see a lot of TFTP traffic between the **`pxe-server-bios`** and the **`pxe-client`** that keeps `pxelinux.0` and other necessary PXE files
	![[Pasted image 20250624210553.png]]
- These packets contain the PXELINUX bootloader. Finally, we can see the menu on the screen:
	<img  width=700 src="Pasted image 20250302224125.png" />
- We see that this menu looks exactly as we configured it!
- Let's select and run the only one option available "*Install Ubuntu Server 24.04*"
- And we can notice the same in Wireshark: a lot of TFTP packets started transferring. This time the requested files were `vmlinuz` and `initrd`
	![[Pasted image 20250624211850.png]]
- After it we can notice that the `initrd` was loaded to the memory and `vmlinuz` kernel was started.
	![[Pasted image 20250624211058.png]]
- As wee see the kernel started **[casper](https://manpages.ubuntu.com/manpages/focal/en/man7/casper.7.html)** to download the ISO image from our HTTP server which was still the same **`pxe-server-bios`**
- And it is again noticeable in Wireshark because there were bunch of HTTP and TCP packets requesting `/ubuntu/ubuntu-24.04.2-live-server-amd64.iso` from **`pxe-server-bios`**.
	![[Pasted image 20250624212113.png]]
- After the download was done successfully I can finally enter the installation menu properties
	![[Pasted image 20250624201255.png]]
	![[Pasted image 20250624201317.png]]
- Then I can easily install OS onto my VM's disk
	![[Pasted image 20250624202345.png]]
	![[Pasted image 20250624202403.png]]
- I entered credentials 
	![[Pasted image 20250624202520.png]]
- And installation process started
	![[Pasted image 20250624212842.png]]
- After some time, installation completed successfully
	![[Pasted image 20250624214052.png]]
- Then I stopped the `pxe-client` node and changed boot priority from Network to HDD
	<img  width=500 src="Pasted image 20250624202847.png" />
- Finally, I started it again and logged into OS successfully with credential I used
	![[Pasted image 20250624214238.png]]
- We can see that this node got `10.0.0.253` address from our DHCP server	![[Pasted image 20250624214359.png]]
#### **UEFI-based approach**
- UEFI-based approach is almost the same, so I will skip a lot of steps focusing on differences between the approaches
- Firstly, I stopped `pxe-server-bios` and started `pxe-server-uefi` node to save hardware resources :)
	<img src="Pasted image 20250624214938.png" width=550 />
##### 1) DHCP server configuration
- The difference here is that I changed `next-server` option in Mikrotik DHCP server to the IP address of the **`pxe-server-uefi`** (`10.0.0.3`)
	![[Pasted image 20250624215558.png]]
- However, it is also important to change bootloader file name since for EFI it is different compared to BIOS. So I changed it to `syslinux.efi` instead of `pxelinux.0`
	![[Pasted image 20250624223138.png]]
##### 2) TFTP server setup
- TFTP server setup is absolutely the same
	![[Pasted image 20250624223511.png]]
##### 3) PXELINUX server setup
- Here the difference is that I downloaded `syslinux-efi` package instead of `pxelinux`
	```shell
	sudo apt install syslinux-efi
	```
	![[Pasted image 20250624223728.png]]
- So I copied the `syslinux.efi` file to the `/var/lib/tftpboot` directory
	```shell
	sudo cp /usr/lib/SYSLINUX.EFI/efi64/syslinux.efi /var/lib/tftpboot
	```
	![[Pasted image 20250624224402.png]]
- Then I copied all the EFI64 Comboot library modules to the same place
	![[Pasted image 20250625001546.png]]
- As you can see this time I decided not to figure out which modules may not be used and copied them all
##### 4) OS setup
- Here is the process is absolutely the same: I downloaded `ubuntu-24.04.2-live-server-amd64.iso`, mounted it to get `vmlinuz` and `initrd`, and copied to the `/var/lib/tftpboot`.
	![[Pasted image 20250624225953.png]]
##### 5) HTTP server setup
- Again the process is the same as for BIOS-based approach using `apache2`
	![[Pasted image 20250624230347.png]]
	![[Pasted image 20250624230320.png]]
##### 6) Configuring PXELINUX menu
- There is no any differences here too. I created a configuration for a boot menu using the `pxelinux.cfg/default` file which I left almost the same as for BIOS-based approach. The only thing I changed is the address from **`pxe-server-bios`** to **`pxe-server-uefi`**
	![[Pasted image 20250625155911.png]]
##### 7) Testing PXELINUX
- After configurations had been done, I decided to quickly check that everything worked
- But first I changed boot priority back to Network on **`pxe-client`** node
	<img  width=500 src="Pasted image 20250624234014.png" />
- And what is most important I enabled UEFI boot mode in node's additional settings
	<img  width=500 src="Pasted image 20250624234121.png" />
- Here we can see that interface is slightly different, but the content is pretty the same. We can see that it downloaded `syslinux.efi` successfully
	![[Pasted image 20250625154812.png]]
- However, the menu was the same
	<img  width=700 src="Pasted image 20250302224125.png" />
- So, as everything else
	![[Pasted image 20250625155749.png]]
- Finally, I had successfully reinstalled the same OS on `pxe-client` but this time using UEFI ✅
### 1.3
> [!Task Description]
> **1.3.** Question: why not run your DHCP service on the SNE network directly?

- There are router in the lab that of course have their own DHCP servers configured. So, a conflict might occur if we have done everything in the same public network. Our PXE client might get an address from another DHCP server or our DHCP server might try to offer and address to a SNE network router. That's why we need to use virtual network with virtual interfaces on which our DHCP server can be configured.

---
## 2. PXE Client Setup
### 2.1
> [!Task Description]
> **2.1.** Create the second virtual machine using VirtualBox in order to test the PXE service.

✅ Done in the previous task
### 2.2
> [!Task Description]
> **2.2.** Change the boot order

✅ Done in the previous task
### 2.3
> [!Task Description]
> **2.3.** Show that your PXE client takes the IP.

✅ Done in the previous task
### 2.4
> [!Task Description]
> **2.4.** Boot and install a new system with it and show the proof in the report.

✅ Done in the previous task

---
# Task 2 - Questions to answer

## 1. 
> [!Task Description]
> **1.** Briefly explain UEFI with secure boot enabled, UEFI without secure boot, and BIOS PXE booting approaches.
- **UEFI with Secure Boot** is a modern firmware interface for booting system software that enhances security by allowing only signed bootloaders and operating systems to boot. During the boot process, the UEFI firmware checks the digital signature of the bootloader and any subsequent components (like OS kernel) against a database of trusted signatures. If the signature is valid, the system proceeds to boot; otherwise, it halts. Such an approach prevents unsigned or untrusted code from executing, thus reducing the risk of malware and bootkit attacks.
- **UEFI without Secure Boot** offers the same benefits of UEFI (such as faster boot times and support for larger hard drives) but does not enforce signature checks during booting. The UEFI firmware simply loads the bootloader, regardless of whether it is signed. As a result, the boot process can proceed with any kernel and OS, regardless of the source or integrity. Such an approach makes it easier to use for development, testing, and installation of custom or open-source operating systems but increased risk of booting malicious software.
- **BIOS PXE booting** refers to the traditional method of network booting used mainly in older systems that rely on the BIOS. The BIOS firmware initializes hardware during the boot process and checks for PXE support. It retrieves the network boot program (typically `pxelinux.0`) from a TFTP server, which then processes a configuration file to load an OS image. Such an approach does not imply any security and is needed for a network boot without hard disks.
### 1.1
> [!Task Description]
> **1.1.** How do they work? Explain with a simple diagram. 
- **BIOS PXE booting**
	<img src="Pasted image 20250626200220.png" width=600 />
- **UEFI without Secure Boot**
  <img src="Pasted image 20250625230400.png" width=600 />
- **UEFI with Secure Boot**
	<img src="Pasted image 20250626001139.png" width=600 />

## 2.

> [!Task Description]
> **2.** What is a GPT?
- **GPT** stands for **GUID Partition Table**, which is a standard for the layout of the partition table on a physical storage device, such as a hard drive or SSD. It is part of the UEFI specification and is designed to replace the older Master Boot Record (MBR) partitioning scheme for BIOS.
- Key advantages of GPT over MBR:
	1. **Larger Disk Size Support**: GPT allows for disk sizes larger than 2 TB, which is a limitation of MBR. With GPT, you can work with disks that are multiple petabytes in size.
	2. **More Partitions**: MBR supports a maximum of four primary partitions (or three primary partitions and one extended partition). In contrast, GPT allows for 128 partitions by default without the need for extended partitions.  
	3. **Redundancy and Reliability**: GPT stores multiple copies of the partitioning and boot data across the disk. This redundancy improves reliability and allows recovery from corruption or data loss.  
	4. **CRC Protection**: GPT uses checksums (Cyclic Redundancy Check, CRC) to verify the integrity of its data. This means that the system can detect errors in the partition table and respond accordingly. 
	5. **Globally Unique Identifier (GUID)**: Each partition is assigned a unique GUID, which helps in identifying partitions on a global scale, avoiding conflicts and ensuring better media management.

### 2.1
> [!Task Description]
> **2.1.** What is its general layout? Explain each element.

- I took a look at [official UEFI specification](https://uefi.org/specs/UEFI/2.10/05_GUID_Partition_Table_Format.html). This image demonstrates a GPT disk's general layout:
	![[Pasted image 20250626113447.png]]
- Below is the description of each element:
	- **Protective MBR** (Master Boot Record) is a traditional MBR with a special signature. It allows to use GPT disk with legacy systems that support only MBR. It informs older systems that the disk is in use and prevents them from mistakenly overwriting the GPT data. The protective MBR has a single partition that is called GPT Protective partition.
	- **GPT Protective partition** is a feature of the GPT designed to safeguard the GPT data structure from being overwritten or mismanaged by software or operating systems that do not support GPT. It serves an important role in ensuring the integrity and proper identification of a GPT disk, especially in environments where older systems or legacy software may be in use. This partition indicates the whole disk or a part of $2$ Tb which BIOS can recognise if the disk size $> 2$ Tb.
	- **Primary GPT**  is the second sector of the disk which contains a partition table. It consists of two parts:
		1) **GPT Header** contains essential metadata about the GPT itself, including the number of partition, their size and location, the size of the header structure, the disk GUID, and etc.
		2) **Partition Entries** contains entries that describe each partition on the disk. Each entry holds information about a single partition, such as its size, type, and unique identifier.
	- **Partitions** are just partitions of the disk that can hold different filesystems or even OS (for primary partitions).
	- **Backup GPT** is located at the end of the disk and is necessary to maintain redundancy, ensuring that if the primary GPT becomes corrupted, the backup can be used to recover information about the partitions.

### 2.2
>[!Task Description]
> **2.2.** What is the role of a partition table?
- A partition table is a critical structure on storage devices (such as hard drives, SSDs, and USB drives) that defines how the storage is organized. It holds information about the arrangement of partitions on the disk, which are distinct sections that can each contain a file system.
- Below are the key roles and functions of a partition table:
	- **Defines Partitions**: The partition table specifies the boundaries and sizes of partitions on the disk.
	- **Identifies File Systems**: Each entry in the partition table typically includes the type of file system used (e.g., NTFS, FAT32, ext4), which helps the operating system understand how to read and manage the contents of each partition.
	- **Boot Information**: In many cases, the partition table contains information necessary for the boot process. For example, it can indicate which partition is bootable, allowing the computer to load the operating system from the specified partition during startup.
	- **Logical Organization**: The partition table enables logical organisation of data, separating it into manageable sections. This is useful for organising files into different categories or purposes (e.g., one partition for the operating system, another for user data, etc.).
	- **Dynamic Resizing and Management**: Many modern operating systems provide tools for managing partitions (resizing, creating, deleting). The partition table must be updated accordingly to reflect these changes, allowing for dynamic adjustments to how space is allocated on the disk.
## 3.

> [!Task Description]
> **3.** What is **`gdisk`**?
- **`gdisk`** is a command-line utility used for managing GPT disks in Unix-like OS. It provides functionality for creating, deleting, modifying, and managing partitions on disks that use the GPT partitioning scheme.
- Below are key features of `gdisk`:
	- **Partition Creation and Deletion**: Users can create new partitions, delete existing ones, resize partitions, and modify partition types.
	- **Backup and Restore**: Allows users to back up the GPT structure to a file and restore it, which is useful for data recovery and redundancy.
	- **Conversion**: It can convert MBR disks to GPT without data loss, making it useful when migrating to newer systems or larger disks.
	- **Error Checking**: Can check the integrity of the GPT structures and also perform repairs if needed.
### 3.1
> [!Task Description]
> **3.1.** How does it work?
#### From user's perspective
1. **Loading the Disk**:
    - When you run `gdisk` with the path to a disk (e.g., `/dev/sda`), the program reads the GPT from the disk. It examines primary and backup GPT headers and partition entries. The data read includes crucial information such as partition sizes, types, and locations.
2. **User Interface**:
    - Once the GPT has been successfully loaded, `gdisk` provides a command-line interface (CLI) where users can enter commands to perform various operations on the partitions. The interface is interactive, allowing users to see a list of available commands and to output the partition table.
3. **Commands and Operations**:
    - Users can execute a variety of commands. Here are some common ones and their functions:
        - **`p`**: Print the current partition table, displaying details about each partition (number, type, size, and status).
        - **`n`**: Create a new partition. The user will be prompted for details such as size, type code, and partition name.
        - **`d`**: Delete an existing partition. The user specifies which partition to remove.
        - **`t`**: Change the type of a partition by selecting a type code.
        - **`r`**: Recover a lost GPT by rebuilding the partition table from the backup.
        - **`w`**: Write changes to the disk. Before executing this command, users should verify their changes.
        - **`q`**: Quit without saving changes, allowing users to exit without altering the disk.
4. **Editing Partition Entries**:
    - When creating or modifying partitions, `gdisk` allows users to specify parameters.
5. **Handling GPT Headers**:
    - `gdisk` manages both primary and backup GPT headers found at the beginning and end of the disk, respectively.
6. **Error Checking and Repair**:
    - Upon loading the disk, `gdisk` performs integrity checks on the GPT. If it finds inconsistencies (e.g., invalid checksums), it will alert the user and might offer repair options.
7. **Writing Changes**:
    - After making changes (creating or deleting partitions), `gdisk` requires the user to explicitly write these changes to the disk with the `w` command. This prevents accidental data loss from unintended modifications.
    - If the user decides not to write changes, they can simply exit with the `q` command.
#### Internally
1. **Disk Access**:
	- `gdisk` accesses the disk directly through system calls provided by the operating system, typically using block device interfaces.
2. **Reading Raw Data**:
    - When `gdisk` starts, it reads the raw bytes from the disk at specific offsets using system calls like `read()`. The GPT is stored in specific sectors at the start of the disk (primary header at LBA 1) and the end of the disk (backup header at the last sectors).
3. **Interpreting Data Structures**:
    - The data read from the disk is interpreted according to the GPT specification. `gdisk` parses the binary data to construct in-memory representations of the partition table.
4. **Error Handling**:
    - `gdisk` includes functionality to validate the integrity of the GPT by checking checksums and verifying that partition entries point to valid locations. If inconsistencies are detected, it may prompt the user for corrective actions, or it may self-repair using backup GPT information.
5. **Generating User-Friendly Output**:
    - Once the GPT is read and processed, `gdisk` formats the information into a readable format for the user.
### 3.2
> [!Task Description]
> **3.2.** What can you do with it?
- **`p`**: Print the partition table. Displays current partitions, types, sizes, and status.
- **`n`**: Add a new partition. You will be prompted for details such as partition number, size, type, and starting sector.
- **`d`**: Delete a partition. You specify which partition number to delete.
- **`t`**: Change a partition's type. You'll be prompted to enter the partition number and then the new type code.
- **`r`**: Recover a lost GPT from the backup. Useful in case the primary GPT is corrupted.
- **`x`**: Extra features. This command provides access to additional options, such as changing the partition GUIDs or viewing diagnostic information.
	- **`b`**: Create a backup of the GPT data structures to a file.
	- **`e`**: Change the backup GPT header location (not commonly needed).
	- **`l`**: List the firmware-identified device name (useful in specific contexts).
	- **`i`**: Show detailed information about the GPT itself, including block size and other parameters.
	- **`s`**: Rescan the disk for partitions.
	- **`m`**: Modify the size of the GPT structures if necessary.
- **`v`**: Verify current partition table structures for integrity and correctness. It checks checksums and other critical parameters.
- **`w`**: Write the changes to disk. This command commits any changes made during the session to the physical disk. Requires user confirmation.
- **`q`**: Quit without saving changes. Exits `gdisk` without writing any modifications to the disk.
- **`?`**: Print help. Displays a list of available commands with brief descriptions.
### 3.3
> [!Task Description]
> **3.3.** Provide a simple practice.
- First, I explored names of disks and partitions
	```shell
	lsblk
	```
	![[Pasted image 20250629150416.png]]
- Then I checked information about partitions on my `/dev/nvme0n1` disk using `gdisk` `-l` option
	![[Pasted image 20250629151117.png]]
- Above we can see that our partitioning scheme is in fact GPT and it contains protective MBR
- Then I ran `gdisk` in interactive mode and requested `help`
	![[Pasted image 20250629151944.png]]
- So, with `p` command in interactive menu I received the same information
	![[Pasted image 20250629152031.png]]
- We see that are 5 partitions in total, so I requested detail information about the first one
	![[Pasted image 20250629152143.png]]
- The 1st partition is EFY system partition which is 100MiB of size. 
- Afterwards, I verified integrity of the GPT using `v` command and found no problems
	![[Pasted image 20250629152428.png]]
- Using `o` command I checked protective MBR info
	![[Pasted image 20250629152809.png]]
- We see that the disk size is `2000409264`, while protective MBR partition is `2000409263` sectors of length, so it indeed includes the whole disk if disk size if less than 2Tb
- Finally, I exited `gdisk` using `q` command which does not save any modifications
	![[Pasted image 20250629153523.png]]
## 4.
> [!Task Description]
> **4.** What is a Protective MBR and why is it in the GPT?
- **Protective MBR (Master Boot Record)** is a critical component of the GPT partitioning scheme. Its primary role is to safeguard GPT disks against being misrecognised and improperly modified by older systems or software that only understand the MBR partitioning scheme.
- **Purpose of the Protective MBR**:
    - **Prevent Confusion**: The Protective MBR is designed to prevent legacy systems from incorrectly interpreting a GPT disk as an unpartitioned drive.
    - **Ensure Data Integrity**: It helps to ensure that older software does not attempt to make destructive changes to a GPT disk by mistakenly assuming it is unallocated space.
- **Structure of the Protective MBR**:
	1. **Boot Indicator**:
	    - The first byte of the MBR is the boot indicator, typically set to `0x00`.
	2. **CHS (Cylinder-Head-Sector) Address**:
	    - The next three bytes (bytes 1 to 3) are set to `0x000000`, which indicates that there is no usable CHS addressing.
	3. **Partition Entry**:
	    - The Protective MBR contains a single partition entry, which takes up 16 bytes (the traditional size of a single MBR partition entry).
	    - This entry lists the entire disk as one single partition that occupies all available space, extending to the maximum capacity of the disk.
	4. **Partition Type**:
	    - The partition type for the Protective MBR entry is generally set to `0xEE`, which is **the identifier for a GPT partition**.
	5. **Ending CHS**:
	    - The next 3 bytes are set to `0x000000` or correspond to the last sector of the disk, again indicating that there are no partitions to boot from.
	6. **Signature**:
	    - The last two bytes of the MBR (`0x55AA`) are the boot signature, signalling that the structure is a valid MBR.

---
# Task 3 - Partitions

## 1.
> [!Task Description]
> **1.** Verify the GPT schema of your Ubuntu machine.
- I did it with `gdisk` utility
	![[Pasted image 20250629153937.png]]
## 2.
> [!Task Description]
> **2.** Use the **`dd`** utility to dump the Protective MBR and GPT into a file in your home directory. The dump should contain up to first partition entry (Inclusive). Note: upload the dump file to your moodle
- For this task I computed how much bytes I need to dump for the start of the disk:
	- LBA block size $= 512$ bytes, so
		- Protective MBR size $=512$ bytes
		- GPT header size = $512$ bytes
	- Partition entry size $=128$ bytes
- Therefore, to dump the Protective MBR and GPT up to first partition entry inclusively I need to dump first $512 + 512 + 128 = 1152$ bytes
- In order for dump to have such a precision in bytes we need to set block size to the value that would divide $1152$ without a reminder. To make things simply I just used block size $=1$
- So, using `dd` I made a dump to `~/mbr_gpt_dump.bin` file. The dump size is $1152$ blocks of size $1$ byte.
	```shell
	sudo dd if=/dev/nvme0n1 of=mbr_gpt_dump.bin bs=1 count=1152
	```
	![[Pasted image 20250629160546.png]]
## 3.

> [!Task Description]
> **3.** Load the dump file into a **hex dump utility** (e.g. 010 editor) to look at the raw data in the file.
- I verified the dump using `hexdump` utility
	```shell
	hexdump -C ~/mbr_gpt_dump.bin
	```
	![[Pasted image 20250629160754.png]]
- We see the the first partition entry has its name `Basic data partition` which we have already seen in `gdisk` utility
- Below is the dump in hexadecimal format
	```
	00000000  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
	*
	000001c0  02 00 ee ff ff ff 01 00  00 00 af d2 3b 77 00 00  |............;w..|
	000001d0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
	*
	000001f0  00 00 00 00 00 00 00 00  00 00 00 00 00 00 55 aa  |..............U.|
	00000200  45 46 49 20 50 41 52 54  00 00 01 00 5c 00 00 00  |EFI PART....\...|
	00000210  e6 8c 15 11 00 00 00 00  01 00 00 00 00 00 00 00  |................|
	00000220  af d2 3b 77 00 00 00 00  22 00 00 00 00 00 00 00  |..;w....".......|
	00000230  8e d2 3b 77 00 00 00 00  1b d2 0e dc 91 e7 a2 4b  |..;w...........K|
	00000240  96 cb 3a 85 f0 06 37 5e  02 00 00 00 00 00 00 00  |..:...7^........|
	00000250  80 00 00 00 80 00 00 00  18 e2 31 2f 00 00 00 00  |..........1/....|
	00000260  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
	*
	00000400  28 73 2a c1 1f f8 d2 11  ba 4b 00 a0 c9 3e c9 3b  |(s*......K...>.;|
	00000410  6c c3 82 e8 69 1e bc 44  96 e5 bd de 03 1f 35 ab  |l...i..D......5.|
	00000420  00 08 00 00 00 00 00 00  ff 27 03 00 00 00 00 00  |.........'......|
	00000430  00 00 00 00 00 00 00 80  42 00 61 00 73 00 69 00  |........B.a.s.i.|
	00000440  63 00 20 00 64 00 61 00  74 00 61 00 20 00 70 00  |c. .d.a.t.a. .p.|
	00000450  61 00 72 00 74 00 69 00  74 00 69 00 6f 00 6e 00  |a.r.t.i.t.i.o.n.|
	00000460  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
	*
	00000480
	```
## 4.

> [!Task Description]
> **4.** Understand and fully annotate the Protective MBR, GPT header and first partition entry in the report. This means you must describe the purpose of every field, and translate all fields that have a numerical value into human-readable, decimal format. Hint: make a table to be clear. Show the byte index address.
- I made the following table with all fields translation and description:

| Field Name                   | Size (Bytes) | Byte Range (Decimal) | Hexadecimal Value                                                                                                                            | Decimal/ASCII Value    | Description                                               |
| ---------------------------- | ------------ | -------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------- | --------------------------------------------------------- |
| **Protective MBR**           | =====        | =====<br>            | =====<br>                                                                                                                                    | =====<br>              | =====<br>                                                 |
| **Bootloader**               | 446          | 0–445                | `00...00`                                                                                                                                    | N/A                    | Reserved for bootloader, filled with zeros.               |
| **Partition Status**         | 1            | 446                  | `00`                                                                                                                                         | 0                      | Indicates inactive partition (for legacy purposes).       |
| **CHS Start**                | 3            | 447–449              | `00 02 00`                                                                                                                                   | N/A                    | CHS start address (not used in GPT, but kept for legacy). |
| **Partition Type**           | 1            | 450                  | `EE`                                                                                                                                         | N/A                    | Indicates GPT protective MBR.                             |
| **CHS End**                  | 3            | 451–453              | `FF FF FF`                                                                                                                                   | N/A                    | CHS end address (not used in GPT, but kept for legacy).   |
| **Start LBA**                | 4            | 454–457              | `01 00 00 00`                                                                                                                                | 1                      | First LBA of the protective partition.                    |
| **Partition Size**           | 4            | 458–461              | `AF D2 3B 77`                                                                                                                                | 2013265920             | Size of the partition in LBA.                             |
| **Padding**                  | 48           | 462–509              | `00...00`                                                                                                                                    | 0                      | Padding bytes in the partition table.                     |
| **Boot Signature**           | 2            | 510–511              | `55 AA`                                                                                                                                      | N/A                    | Indicates a valid MBR.                                    |
| **GPT Header**               | =====<br>    | =====<br>            | =====<br>                                                                                                                                    | =====<br>              | =====<br>                                                 |
| **Signature**                | 8            | 512–519              | `45 46 49 20 50 41 52 54`                                                                                                                    | `EFI PART`             | Identifies this as a GPT header.                          |
| **Revision**                 | 4            | 520–523              | `00 00 01 00`                                                                                                                                | N/A                    | GPT header version (1.0).                                 |
| **Header Size**              | 4            | 524–527              | `5C 00 00 00`                                                                                                                                |                        | Size of the GPT header in bytes.                          |
| **CRC32**                    | 4            | 528–531              | `E6 8C 15 11`                                                                                                                                | N/A                    | CRC32 checksum of the GPT header.                         |
| **Reserved**                 | 4            | 532–535              | `00 00 00 00`                                                                                                                                | 0                      | Reserved field, must be zero.                             |
| **Current LBA**              | 8            | 536–543              | `01 00 00 00 00 00 00 00`                                                                                                                    | 1                      | LBA of the GPT header itself.                             |
| **Backup LBA**               | 8            | 544–551              | `AF D2 3B 77 00 00 00 00`                                                                                                                    | 2013265920             | LBA of the backup GPT header.                             |
| **First Usable LBA**         | 8            | 552–559              | `22 00 00 00 00 00 00 00`                                                                                                                    | 34                     | First usable LBA for partitions.                          |
| **Last Usable LBA**          | 8            | 560–567              | `8E D2 3B 77 00 00 00 00`                                                                                                                    | 2013265894             | Last usable LBA for partitions.                           |
| **Disk GUID**                | 16           | 568–583              | `1B D2 0E DC 91 E7 A2 4B 96 CB 3A 85 F0 06 37 5E`                                                                                            | N/A                    | Unique identifier for the disk.                           |
| **Partition Table LBA**      | 8            | 584–591              | `02 00 00 00 00 00 00 00`                                                                                                                    | 2                      | Starting LBA of the partition table.                      |
| **Num of Partition Entries** | 4            | 592–595              | `80 00 00 00`                                                                                                                                | 128                    | Number of partition entries in the table.                 |
| **Partition Entry Size**     | 4            | 596–599              | `80 00 00 00`                                                                                                                                | 128                    | Size of each partition entry in bytes.                    |
| **Partition Table CRC32**    | 4            | 600–603              | `18 E2 31 2F`                                                                                                                                | N/A                    | CRC32 checksum of the partition table.                    |
| **Reserved Space**           | 420          | 604-1023             | `00...00`                                                                                                                                    | N/A                    | Reserved area of zeros to fill the entire logical block   |
| **1st Partition Entry**      | =====<br>    | =====<br>            | =====<br>                                                                                                                                    | =====<br>              | =====<br>                                                 |
| **Partition Type GUID**      | 16           | 1024–1039            | `28 73 2A C1 1F F8 D2 11 BA 4B 00 A0 C9 3E C9 3B`                                                                                            | N/A                    | Type of the partition (e.g., basic data).                 |
| **Unique Partition GUID**    | 16           | 1040–1055            | `6C C3 82 E8 69 1E BC 44 96 E5 BD DE 03 1F 35 AB`                                                                                            | N/A                    | Unique identifier for the partition.                      |
| **First LBA**                | 8            | 1056–1063            | `00 08 00 00 00 00 00 00`                                                                                                                    | 2048                   | First LBA of the partition.                               |
| **Last LBA**                 | 8            | 1064–1071            | `FF 27 03 00 00 00 00 00`                                                                                                                    | 2047999                | Last LBA of the partition.                                |
| **Attributes**               | 8            | 1072–1079            | `00 00 00 00 00 00 00 80`                                                                                                                    | N/A                    | Partition attributes (e.g., bootable flag).               |
| **Partition Name**           | 72           | 1080–1151            | Unicode string:<br>`80 42 00 61 00 73 00 69 00 63 00 20 00 64 00 61 00 74 00 61 00 20 00 70 00 61 00 72 00 74 00 69 00 74 00 69 00 6f 00 6e` | `Basic data partition` | Name of the partition, stored in UTF-16LE.                |
### 4.1
> [!Task Description]
> **4.1.** At what byte index from the start of the disk do the partition table entries start?
- In **MBR** at the index $446$ ($447$-th byte). There can be 4 partitions, each with a length of 16 bytes.
- In **GPT** it depends on the LBA block size. Both Protective MBR and GPT Header should occupy the whole block and start from LBA-0 and LBA-1 correspondingly. Therefore, the partition table must start from LBA-2, so the exact index depends on the size of LBA block. For example, in my case the size is 512, so the index is $512 + 512 = 1024$ ($1025$-th byte).
### 4.2

> [!Task Description]
> **4.2.** At what byte index would the partition table start if your server had a so-called "4K native" (4Kn) disk?
- In GPT scheme the following rules are applied:
	- Protective MBR occupies the whole LBA-0
	- GPT Header occupies the whole LBA-1
	- Partition table start from LBA-2
- Therefore, for LBA block size $=4K=4096$ bytes the index would be $8192$ ($8193$-th byte).
## 5.
> [!Task Description]
> **5.** Name two differences between primary and logical partitions in an MBR partitioning scheme.
- **Bootability**: 
	- A **primary** partition can be marked as active, making it bootable.
	- **Logical** partitions cannot be marked as active, therefore, cannot directly be booted from.
- **Number of partitions allowed**:
	- The MBR partitioning scheme allows for a maximum of 4 **primary** partitions on a single hard drive.
	- **Logical** partitions are created within an extended partition to bypass the limitation on the number of primary partitions. One can have multiple logical partitions inside a single extended partition, allowing for up to 127 logical partitions.