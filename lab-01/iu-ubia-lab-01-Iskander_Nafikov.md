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
		![[Pasted image 20250624204721.png]]
		![[Pasted image 20250624204741.png]]
		![[Pasted image 20250624204800.png]]
3. **`pxe-server-uefi`** has pretty similar network configuration to `pxe-server-bios`
	- This node has a statically set IP address which is `10.0.0.3` that is configured using Netplan
		![[Pasted image 20250624195318.png]]
	- This VM is created right inside the GNS3 using QEMU/KVM virtualisation
		![[Pasted image 20250624204859.png]]
		![[Pasted image 20250624204843.png]]
		![[Pasted image 20250624204823.png]]
		![[Pasted image 20250624204823.png]]
4. **`pxe-client`**
	- This VM is created right inside the GNS3 using QEMU/KVM virtualisation
		![[Pasted image 20250624203831.png]]
	- As we can see this node has some differences:
		- For instance, its boot priority is set to Network since it will boot from network while its "HDD" (virtual disk image) is empty.
		- Also, its console type is `spice` instead of `telnet` since we need to have a graphical interface
	- What is more, to "make HDD empty" I recreated a new virtual disk image in QCOW2 format
		![[Pasted image 20250624204211.png]]
	- Finally, I removed `-nographich` option in QEMU/KVM additional settings
		![[Pasted image 20250624204455.png]]
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
	![[Pasted image 20250624204555.png]]
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
	![[Pasted image 20250302224125.png]]
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
	![[Pasted image 20250624202847.png]]
- Finally, I started it again and logged into OS successfully with credential I used
	![[Pasted image 20250624214238.png]]
- We can see that this node got `10.0.0.253` address from our DHCP server	![[Pasted image 20250624214359.png]]
#### **UEFI-based approach**
- UEFI-based approach is almost the same, so I will skip a lot of steps focusing on differences between the approaches
- Firstly, I stopped `pxe-server-bios` and started `pxe-server-uefi` node to save hardware resources :)
	<img src="Pasted image 20250624214938.png" width=600 />
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
	![[Pasted image 20250624234014.png]]
- And what is most important I enabled UEFI boot mode in node's additional settings
	![[Pasted image 20250624234121.png]]
- Here we can see that interface is slightly different, but the content is pretty the same. We can see that it downloaded `syslinux.efi` successfully
	![[Pasted image 20250625154812.png]]
- However, the menu was the same
	![[Pasted image 20250302224125.png]]
- So, as everything else
	![[Pasted image 20250625155749.png]]
- Finally, I had successfully reinstalled the same OS on `pxe-client` but this time using UEFI
### 1.3
> [!Task Description]
> **1.3.** Question: why not run your DHCP service on the SNE network directly?

- Since in the lab there are several routers which of course have their own DHCP servers configured. So, a conflict might occur if we have done everything in the same public network. Our PXE client might get an address from another DHCP server or our DHCP server might try to offer and address to a SNE network router.

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

#### 1. Briefly explain UEFI with secure boot enabled, UEFI without secure boot, and BIOS PXE booting approaches.

> [!Task Description]
> **1.1.** How do they work? Explain with a simple diagram. 


#### 2. What is a GPT?


> [!Task Description]
> **2.1.** What is its general layout? Explain each element.

>  [!Task Description]
> **2.2.** What is the role of a partition table?

#### 3. What is gdisk?

> [!Task Description]
> **3.1.** How does it work?


> [!Task Description]
> **3.2.** What can you do with it?


> [!Task Description]
> **3.3.** Provide a simple practice.

#### 4. What is a Protective MBR and why is it in the GPT?

---
# Task 3 - Partitions

## 1.
> [!Task Description]
> **1.** Verify the GPT schema of your Ubuntu machine.

## 2.
> [!Task Description]
> **2.** Use the dd utility to dump the Protective MBR and GPT into a file in your home directory. The dump should contain up to first partition entry (InUEFI bootloaderclusive).

## 3.

> [!Task Description]
> **3.** Load the dump file into a hex dump utility (e.g. 010 editor) to look at the raw data in the file.

## 4.

> [!Task Description]
> **4.** Understand and fully annotate the Protective MBR, GPT header and first partition entry in the report. This means you must describe the purpose of every field, and translate all fields that have a numerical value into human-readable, decimal format. Hint: make a table to be clear. Show the byte indess address.

### 4.1
> [!Task Description]
> **4.1.** At what byte index from the start of the disk do the partition table entries start?


### 4.2

> [!Task Description]
> **4.2.** At what byte index would the partition table start if your server had a so-called “4K native” (4Kn) disk?

## 5.
> [!Task Description]
> **5.** Name two differences between primary and logical partitions in an MBR partitioning scheme.

UEFI bootloader
