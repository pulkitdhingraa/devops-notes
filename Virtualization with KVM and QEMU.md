
# Virtualization with KVM and QEMU

These notes are derived from the [LinkedIn Learning course on Virtualization with KVM and QEMU](https://www.linkedin.com/learning/virtualization-with-kvm-and-qemu/virtualization-with-kvm-and-qemu?u=0), along with additional information collected from Google searches, offical documentation and ChatGPT.

## QEMU Introduction

- QEMU, which is short for Quick Emulator, is software that runs on a host computer and acts as a hardware emulator for guest virtual machines.

- It presents a guest with emulated hardware it needs in order to operate as a computer, for example, a processor, memory, and other hardware, like network interfaces and video cards.

- The software then takes whatever commands or data are sent to those emulated devices by the guest and translates them into something that can be processed by the host system.

- These emulated devices fall into two primary categories: processors, or CPUs, and everything else.

- Commands sent by the guest to its emulated processor are translated to the host system's architecture and executed on the host's CPU, and the results are sent back to the guest operating system.

- QEMU can operate in two different modes. 
    - One is full system emulation mode, where a whole system is emulated. 
    - Another mode, only available on the Linux host, is user-mode emulation, where we can use QEMU to run just one program or process through emulation instead of running a whole system to do that.
        - E.g. Game Emulators, applications where legacy software needs to run on modern hardware.


## KVM Introduction

- KVM, or kernel-based virtual machine, is a feature of the Linux kernel that allows it to act as a type one hypervisor. 

- This feature allows virtual machines direct access to a host's processor, instead of relying on an emulated processor, providing native performance, without the overhead of emulation. 

- Other platforms offer their own hypervisor features, including Apple's Hypervisor Framework and Windows HyperV. 

- KVM is available on systems that have a CPU that supports virtualization, which is common on most Intel and AMD processors. Intel calls its virtualization feature VTX, and AMD calls it AMDV.

- Check if system supports KVM

    - Entry dev kvm exists
    ```
    ls /dev/kvm
    ```

    - Module exists
    ```
    lsmod | grep kvm
    ```

- Virtual machines using KVM must run an operating system built for the same architecture as the host.

- Paravirtualziation (VirtIO) provides faster and more direct access to host's memory, storage, GPU and other devices. The guest is aware that it's virtualizated, and makes use of special hypervisor features unlike full virtualization where a guest behaves as though it's in an isolated box.


## KVM and QEMU together

KVM provides access to the host CPU and memory and offers fast, efficient vert IO access to some devices. And QMU provides all the other parts that a guest computer might need that aren't available with para-virtualization.


## QEMU Command Line Tools

- Install qemu on Ubuntu
```
apt install qemu-kvm
```

| Command  | Description |
| ------------- |:-------------:|
| qemu-img      | Manage and manipulate disk image files (create, convert, resize, check, etc).     |
| qemu-io | Debug and test block devices or disk images with low-level I/O commands. |
| qemu-nbd | Expose a QEMU disk image as a Network Block Device (NBD) to the host. |
| qemu-pr-helper | Handle SCSI Persistent Reservations (PR) passthrough for shared storage setups. |
| qemu-storage-daemon | Run QEMU’s block layer services as a standalone daemon (no VM needed). |
| qemu-system-i386 | Full-system emulator for 32-bit x86 guest machines. |
| qemu-system-amd64 | Alias for qemu-system-x86_64 (64-bit x86 virtualization/emulation). |
| qemu-system-x86_64 | Full-system emulator for 64-bit x86 guest machines. |
| qemu-system-x86_64-spice | Same as qemu-system-x86_64 but with SPICE protocol support (better remote graphics). |
| qemu-system-x86_64-microvm | Lightweight x86_64 machine type optimized for minimal VMs (like Firecracker). |


## Planning a virtualization solution

- Need to plan out what resources our virtual machine will have. This includes disk space, memory, and processor resources. It also includes emulated hardware like video and network adapters, and even real hardware we might pass to a guest from the host.

- It's usually best to only think of part of the host's overall memory as being available to allocate to guests. Perhaps in your lab that's 75%.

- Determine what processor model and how many processor cores a guest should have. 

- The processor, however, might vary between different virtualization hosts, and so instead of using a direct pass-through to whatever model processor a particular host has, which is common practice on a single machine or in a lab, we may need to choose an emulated processor model that will allow guests to have the same CPU across a fleet of hosts.

- Depending on the role of the guest will decide which video hardware it has, what its network topology is, and whether it needs additional devices connected, either emulated ones or real hardware from the host.

- We'll also need to decide how we'll be accessing the guest. We can connect to the guest's display and send it mouse and keyboard input either through a local window or through VNC across the network. And as we configure our guest, we might choose to establish other kinds of remote access, such as with SSH.

-  It's useful to think of QEMU guests as having two primary parts, the disk image from which they run, and the configuration that determines the virtual hardware and configuration of the guest system.


## Disk Images

- Disk image files are file systems that exist within a standard file on the host's hard drive.

- To do this, we'll use the QEMU image utility. We'll need to tell the utility three primary pieces of information 
    - The format of the file
        - raw → Plain binary disk image, no metadata, fastest but no advanced features. `Takes up all their allocated space at once`
        - qcow2 → QEMU’s native format, supports snapshots, compression, encryption, thin provisioning. `Copy-on-write only take up space as the disk is filled`
        - qcow → Older version of qcow2, rarely used now.
        - vmdk → VMware disk image format, for compatibility with VMware products.
        - vdi → VirtualBox disk image format, for compatibility with Oracle VirtualBox.
        - vhd / vpc → Microsoft Virtual PC / Hyper-V disk format, limited features.
        - vhdx → Newer Hyper-V format, supports larger disks and resiliency.
        - cloop → Compressed loopback format, mostly used for live CDs.
    - The path and name of the file 
    - The size of the virtual disk.

- Create disk image
    ```
    qemu-img create -f qcow2 disk1.qcow2 50G
    ```

## Virutal Machine

- Guests are made up primarily of two things, their configuration and their disk image(s).

- Run guests commands in tmux/screen so closing terminals or disconnecting ssh sessions cause commands running in guests to terminate abruptly 

- Create guest
    ```
    qemu-system-x86_64 -enable-kvm -cpu host -smp 4 -m 8G -k en-us -vnc :0 -usbdevice tablet -drive file=disk1.qcow2,if=virtio -cdrom ubuntu-22.04-desktop-amd64.iso -boot d
    ```

| Options  | Explanation |
| ------------- |:-------------:|
| qemu-system-x86_64 | Run the 64-bit x86 system emulator (for Linux/Windows/BSD guests). |
| -enable-kvm | Use KVM instead of CPU emulation |
| -cpu host/qemu64/Skylake-Client | Pass the host CPU model directly to the VM (guest sees same CPU as host). |
| -smp 4 | smp short for Symmetric Multi Processing Set number of virtual CPUs (vCPUs). Example: -smp 2 = 2 cores, -smp 4 = 4 cores. Can also specify sockets/cores/threads: -smp 4,sockets=2,cores=2,threads=1. |
| -m 8G | Allocate 8 GB of RAM to the guest. |
| -k en-us/fr/de/ja | Set keyboard layout for VNC/SDL window. |
| -vnc :0 | Enable VNC remote display on display :0 (which usually maps to TCP port 5900). -vnc :1 → port 5901, -vnc :2 → port 5902, etc. Can connect using vncviewer localhost:0 |
| -device usb-tablet/qxl-vga/virtio-blk-pci/virtio-net-pci | Adds a USB tablet input device, which improves mouse pointer accuracy in VNC/SPICE sessions. |
| -drive file=disk1.qcow2,if=virtio | Attach a disk image (disk1.qcow2) to the guest using virtio interface. file= → the disk image file. if= → interface type (common values): **virtio → best performance (paravirtualized)**. **ide → old but compatible.** **scsi → for SCSI disks.** **sata → emulates SATA.**
| -cdrom ubuntu-22.04-desktop-amd64.iso | Mount an ISO image as a CD-ROM (installation media). |
| -boot d | Set the boot order. Common values: **a → floppy**, **c → first hard disk**, **d → CD-ROM**, **n → network (PXE boot)** Example: -boot order=dc (boot from CD first, then disk). |

- After installing OS on the guest using VNC, power it off, remove the installer options and boot the guest again.

- Run the following commands
    ```
    hostnamectl → Show or change the system hostname and related OS/kernel info.
    lscpu → Display CPU architecture details (cores, threads, sockets, flags).
    lspci → List PCI devices (e.g., NICs, storage controllers, GPUs).
    lsusb → List USB devices recognized by the guest.
    lsmem → Show memory block layout and hotplug information.
    ```

## QEMU monitor

- The QEMU Monitor is basically a command-line interface (CLI) built into QEMU that lets you interact with a running virtual machine at a low level. Think of it like a “debug/management console” for the VM.

- It’s separate from the guest OS — commands go directly to QEMU, not the VM’s shell.

- Available via:
    - A local console (when starting QEMU with -monitor stdio or -monitor telnet::port,server,nowait)
    - Libvirt / virt-manager often exposes it as the QEMU Monitor Console.


## Modifying Disk images

- An overlay is where we start with the one disk image that we call the backing image and lay another disk image on top of it. So any changes made to the guest's file system are stored in that overlay file and not in the original backing image. 

- Overlays make it easy to roll back to a known or original state quickly by creating a new overlay with the original backing image. It also allows us to use one copy of a disk image with a base operating system or preconfigured template installed as the backing store for more than one guest, saving disk space on the host if we have multiple, similar, or identical guests, running at the same time.

- Create overlay
    ```
    qemu-img create -o backing
    ```
- qemu-img Common usage and Commands
    ```
        qemu-img info disk1.qcow2
        qemu-img convert -f qcow2 -O raw disk1.qcow2 disk1.raw
        qemu-img resize disk.qcow2 +10G
        qemu-img check disk.qcow2
        qemu-img create -f qcow2 -b base.qcow2 overlay.qcow2
        qemu-img rebase -b newbase.qcow2 overlay.qcow2
        qemu-img commit overlay.qcow2
    ```

## Disk Image as Network Block Device

- qemu-nbd is a QEMU utility that lets you expose a QEMU disk image as a Network Block Device (NBD) on your host. This allows the host OS to access the guest disk as if it were a physical block device.

- Before using qemu-nbd, ensure the NBD kernel module is loaded:
sudo modprobe nbd max_part=8
(optional) max_part=8 → allows 8 partitions per NBD device (/dev/nbd0, /dev/nbd1, …).

- qemu-nbd Common usage and commands
    - Connect a disk image to an NBD device. After this, you can use standard Linux commands like fdisk -l /dev/nbd0 or lsblk -f /dev/nbd*
    ```
    qemu-nbd -c /dev/nbd0 disk1.qcow2 
    ```
    - Disconnect the NBD device
    ```
    qemu-nbd -d /dev/nbd0
    ```
    - Read-only mount
    ```
    qemu-nbd -r -c /dev/nbd0 disk1.qcow2
    ```
    - Inspect partitions or mounts 
    ```
    fdisk -l /dev/nbd0
    mount /dev/nbd0p1 /mnt
    ```


## Guest graphics and display options

- To change the way that QEMU displays the guest, we will use the -display option, and we'll often combine that with the -vga option, which allows us to set which emulated or paravirtualized video device the guest will have.

- -display determines where we see the guest's screen, and we can think of that as the actual display or monitor, and -vga determines what video adapter is used, that's like the guest's video card. 

#### -display < type >
Specifies the display backend QEMU should use. Common values:

| Options  | Meaning |
| ------------- |:-------------:|
| sdl	|    Use SDL (Simple DirectMedia Layer) library for graphical window. |
| gtk	 |   Use GTK window (Gnome Toolkit) (better for modern Linux desktops) default on qemu. |
| none |	No graphical output; headless VM (useful with -vnc or SSH). |
| curses  |	Text-based console in the terminal (good for servers). |
| vnc=:0  |	Expose VM display via VNC on port 5900 (:0 → 5900, :1 → 5901). |

**Note: gtk and sdl can include ,gl=on to specify OpenGL support (OpenGL support allows the guest VM graphics to be hardware-accelerated using the host GPU.)**

#### -vga < type >
Specifies the emulated video card the guest OS will see. Common values:

| Options  | Description |
| ------------- |:-------------:|
| std	|    Standard VGA (safe, basic support, moderate resolution, low VRAM). |
| cirrus  |	Cirrus Logic VGA (good legacy support for Linux). |
| vmware | 	VMware SVGA II, compatible with guest drivers. |
| qxl	|    Optimized for SPICE, supports high-resolution graphics & multiple monitors. |
| virtio |	Paravirtualized GPU (requires guest driver, best performance with Linux). |
| none |	No video card (headless, combined with -nographic). |

- Add more video memory to some emulated devices
    ```
    qemu-system-x86_64 -m 4G -vga std -device VGA,vgamem_mb=16 -drive file=disk.qcow2
    qemu-system-x86_64 -m 4G -vga qxl -global qxl-vga.vram_size=67108864 -global qxl-vga.vgamem_size=16777216 -drive file=disk.qcow2
    (You can also add multiple displays: -device qxl-vga,vram_size=64M,vgamem_mb=16)
    qemu-system-x86_64 -m 4G -vga virtio -device virtio-vga,ram_size=134217728 -display gtk,gl=on -drive file=disk.qcow2
    ```

##### Troubleshooting Tip : Black screen

- A gray or black screen can be caused by an unsupported display resolution
- Safely shut down the guest with system_powerdown in the QEMU monitor
- Attach a more capable adapter (like VirtIO) and set the resolution of the guest to something compatible (1920 x 1080).


## Sharing files between host and guest

- QEMU supports Plan 9 Filesystem Protocol (9P) for file sharing b/w guests and host

- Share a folder on the host and mount it from within guests

- Syntax 
    ```
    -virtfs local,path=<host_dir>,mount_tag=<tag>,security_model=<mode>,id=<id>
    ```

| Options  | Description |
| ------------- |:-------------:|
| local	     |           Type of filesystem export (always local for host directory). |
| path=< host_dir >	|        Absolute path of the directory on the host to share. |
| mount_tag=< tag >	 |       Identifier used in the guest to mount this directory. |
| security_model=< mode > |	How guest permissions map to host: mapped (default), passthrough, none. |
| id=< id >	             |   Optional identifier for the virtfs device (useful when multiple exports). |


**Example** 

#### Host
```
qemu-system-x86_64 -m 2G -drive file=disk.qcow2,if=virtio -virtfs local,path=/home/user/shared,mount_tag=hostshare,security_model=mapped,id=hostshare
```

#### Guest
```
mkdir /mnt/hostshare
mount -t 9p -o trans=virtio,version=9p2000.L hostshare /mnt/hostshare
```

## Using Host USB devices in a guest

- Find your USB device on the host
    ```
    lsusb 
    ```
**Output:** Bus 001 Device 005: ID 0781:5591 SanDisk Corp. Ultra Flair

Bus: 001
Device: 005
Vendor ID: 0781
Product ID: 5591

**Note: Need to run the commands with superuser priveleges(dev) or set custom udev rules to give appropriate permissions to guest to use particular hardware (prod).**

- Attach by Bus/Device
    ```
    qemu-system-x86_64 -m 2G -hda disk.qcow2 -device usb-host,hostbus=1,hostaddr=5
    ```
- Attach by Vendor/Product ID
    ```
    qemu-system-x86_64 -m 2G -hda disk.qcow2 -device usb-host,vendorid=0x0781,productid=0x5591
    ```
- For USB 3.0+ features, because of a different host controller architecture for USB 3.x (xHCI), need to create a host controller first and then attach devices to it.
    ```
    qemu-system-x86_64 -m 2G -hda disk.qcow2 -device usb-xhci,id=xhci -device usb-host,hostbus=1,hostaddr=5,bus=xhci.0
    ```

## User-Mode Networking

- User-mode networking (a.k.a. SLiRP) is a built-in network backend in QEMU that lets the guest OS use the host machine’s TCP/IP stack without requiring special privileges (no root needed).

- It emulates a NAT (Network Address Translation) between the guest and the outside world.

- The guest can access the Internet (e.g., download packages, browse web). But the outside world (and even the host) usually cannot initiate connections to the guest (unless you explicitly forward ports).

- Networking mode by Default. Explicit form:
    ```
    qemu-system-x86_64 -netdev user,id=net0 -device virtio-net-pci,netdev=net0
    ```

| Options  | Description |
| ------------- |:-------------:|
| -netdev user,ipv6=off,id=net0    |             defines a user-mode network backend (NAT). |
| -device virtio-net-pci/rtl8139/e1000,netdev=net0 |   adds a virtual NIC attached to that backend. |

- Inside the guest, you’ll see a NIC (e.g. eth0), which gets an IP like:
    - Guest IP: 10.0.2.15
    - Gateway: 10.0.2.2 (QEMU acts as gateway)
    - DNS: 10.0.2.3 (QEMU’s internal DNS proxy)


## Port Forwarding 

- Since guest is NAT’d, external connections can’t reach it directly. To allow it, you add hostfwd rules:

**Syntax**
```
-netdev hostfwd=protocol:hostaddr:hostport-guestaddr:guestport

-netdev user,id=net0,hostfwd=tcp::2222-:22
-netdev user,id=net0,hostfwd=tcp:0.0.0.0:2222-10.0.2.15:22
-netdev user,id=net0,hostfwd=tcp::2222-:22,hostfwd=tcp::8080-:80
```


## Remove network connectivity

- QEMU doesn't provide any emulated network interface to the guest. We could, however, still attach a USB network adapter, or something like that, and establish connectivity for the guest. 
    ```
    -nic none 
    ```
*nic is a shortcut command that combines both -netdev and -device*


## Bridge Networking

- Bridge networking in QEMU is what gives the guest VM a real IP address on the same LAN as the host, instead of hiding behind NAT like in user mode.

- A bridge is a virtual switch on the host that forwards packets between the guest VM's virtual NIC, and the host's physical NIC (eth0, wlan0, etc).

- Guests get IP addresses from the same DHCP server as the host (router, corporate DHCP, etc).

- When we're using a bridge set up, our guest network interfaces are represented on the host as TAPs, a term for a virtual network adapter that we can plug into our bridge.

- **Bridge Network Manual method**

    - **Set up the bridge br0**
        ```
        ip link add br0 type bridge && ip link set br0 up
        ```
    - **Create tap interfaces tap0 and tap1**
        ```
        ip tuntap add tap0 mode tap && ip tuntap add tap1 mode tap
        ```
    - **Bridge up tap interfaces**
        ```
        ip link set tap0 up promisc on && ip link set tap1 up promisc on
        ```
    - **Connect tap interfaces to the bridge**
        ```
        ip link set tap0 master br0 && ip link set tap1 master br0
        ```
    - **Start 2 guests using tap0 and tap1**
        ```
        qemu-system-x86_64 .. -netdev tap,id=t0,ifname=tap0,script=no,downscript=no -device e1000,netdev=t0,id=nic0
        qemu-system-x86_64 .. -netdev tap,id=t1,ifname=tap1,script=no,downscript=no -device e1000,netdev=t1,id=nic1
        ```

- **Bridge Helper method**

    - QEMU provides qemu-bridge-helper to handle creation and managing the TAPs for us.

    - It creates and manages the TAP interfaces for guests that use them and runs the QEMU ifup and QEMU ifdown scripts in the host's etc directory to set the TAP interfaces up and down when the guest boots and is shut off.

    - **setuid root, meaning always run with root privileges**
        ```
        sudo chmod u+s /usr/lib/qemu/qemu-bridge-helper
        # Safer alternative
        sudo setcap cap_net_admin+ep /usr/lib/qemu/qemu-bridge-helper
        ```

    - **Edit bridge.conf and add the line 'allow br0'**
        ```
        sudo vi /etc/qemu/bridge.conf
        ```


## Creating a private network

- Set up the bridge br0 and qemu-bridge-helper to manage the TAP interfaces

- Run the following command to join the guest to bridge network
    ```
    qemu-system-x86_64 -drive file=disk1.qcow2,if=virtio -netdev bridge,br=br0,id=net1 -device virtio-net-pci,netdev=net1
    ```

**Note: Running same command guest2 will result in both having the same mac address, so we will increment the address by 1 and assign to guest2**
    ```
    qemu-system-x86_64 -drive file=disk2.qcow2,if=virtio -netdev bridge,br=br0,id=net1 -device virtio-net-pci,netdev=net1,mac=52:54:00:12:34:57
    ```

1. *QEMU’s recommended OUI prefix for virtual NICs is 52:54:00 (Red Hat’s block).*
2. *The last 3 octets should be unique per VM: 12:34:56, 12:34:57, etc.*
3. *MACs must be globally unique inside the same L2 network segment (your bridge).*

- Run **ip link** in host to see both bridge and tap interfaces created

- With this setup, no DHCP server exists. Therefore need to provide guests with static ips. 
e.g. 10.10.10.2 and 10.10.10.3 , gateway 10.10.10.1

- Successfully ping between guests


## Host only network

- Add host to the private network created above

- Assign the bridge on the host an address in the private network.
    ```
    ip addr add 10.10.10.1/24 dev br0
    ```
- Now guests can communicate to the host 

- OR configure DHCP on the host to provide guests with IP addresses automatically. 

- To do that remove the manually created ip addresses from the guest and run the command
    ```
    dnsmasq --interface=br0 --bind-interfaces --dhcp-range=10.10.10.2,10.10.10.100
    ```


## NAT Network

- Provide our host-only network guests access to the outside world by configuring our host as a router.

- Set the kernel parameter for ipv4 Forwarding
    ```
    sysctl -w net.ipv4.ip_forward=1
    ```
- Set a firewall rule that will accept and forward traffic from the bridge.
    ```
    iptables -A FORWARD -i br0 -j ACCEPT
    ```
- Add another rule to perform NAT, from the bridge's address out to the rest of the host's network and to the world.
    ```
    iptables -t nat -I POSTROUTING -s 10.10.10.0/24", the bridge's address, "-j MASQUERADE
    ```
- Make changes permanent by adding to the script.


## Public Network

- Guests share the address space of the host's network, and can have their addresses assigned via that network's DHCP server.

- Before setting up, make sure hosts has more than 1 network interface otherwise host will loose the connectivity

- Set up the bridge br0 and qemu-bridge-helper to manage the TAP interfaces
    ```
    ip link add br0 type bridge && ip link set br0 up
    allow br0 in /etc/qemu/bridge.conf
    ```

- Enslave the hosts physical NIC into the bridge 
ip link set eth0 master br0

- To remove host's network interface from the bridge
ip link set dev eth0 nomaster


## Libvirt

- Libvirt provides an API that client software like VIRSH and VIRT Manager can use to manage QEMU, KVM and other hypervisors.

- Libvirt stores configurations for guest machines and other entities like networks, storage and so on as XML files.

- Libvirt uses the name domains for guest machine definitions and resources.

- Install libvirt 
    ```
    apt install libvirt-daemon-system libvirt-clients
    ```
- Configuration files are stored in __*/etc/libvirt/qemu*__

- It's handling all the communication with QEMU and it even sets up its own network bridge and routing so that guests can participate in a shared NAT network with the host.

- Out of the box, that network uses the 192.168.122 space, and the host is available at 192.168.122.1


## Virsh

- Virsh allows us to manage libvirt guests and other resources from the command line. It's installed as part of the libvirt-clients package.

**Common Tools**

| Tools  | Description |
| ------------- |:-------------:|
| virsh	        | CLI tool to manage libvirt VMs |
| virt-manager	| GUI tool for managing VMs |
| virt-install	| Create VMs from command line |
| virt-viewer   | 	Connect to VM console |
| virt-clone	|    Clone VMs easily |

**Common commands**
```
virsh start my-vm
virsh list --all
virsh shutdown my-vm
```
