# GPU Passthrough on Solus

Notes:
- WORK IN PROGRESS: This guide is not done yet!
- Solus might need to be installed in UEFI mode for this guide to work
- This guide does not (currently) work with identical guest and host GPUs

Todo:
- Add [Using identical guest and host GPUs](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#Using_identical_guest_and_host_GPUs) to the guide
- Add how to create a network bridge
- [Scream Network Audio](https://github.com/duncanthrax/scream) - Sound ich9 -> Remove
- More performance optimizations
- Add troubleshooting section and move nvidia error 43 there


## Pre-requirements

Read the [Prerequisites section](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#Prerequisites) in the ArchWiki

Summary:
- CPU must support hardware virtualization and IOMMU
- Motherboard must support IOMMU
- Guest GPU must support UEFI
- Guest GPU must be connected to a display


## Isolating the GPU

Install dependencies:
```bash
sudo eopkg install dracut
```

Download [vfio-bind.sh](https://raw.githubusercontent.com/gmol1/vfio-bind/master/vfio-bind.sh) from https://github.com/gmol1/vfio-bind

Run vfio-bind.sh:
```bash
chmod +x vfio-bind.sh
sudo ./vfio-bind.sh
```
Reboot

Verify that the GPU and its audio controller now are using vfio-pci:
```
lspci -nnk
```

## Install Virtual Machine Manager

```
sudo eopkg install qemu virt-manager ovmf
sudo systemctl enable libvirtd
sudo gpasswd -a $USERNAME kvm
sudo gpasswd -a $USERNAME libvirt
```
Reboot


## Install Windows in a virtual machine

Start Virtual Machine Manager

### Create the Virtual Machine

Click on "Create a new virtual machine" or go to "File" -> "New Virtual Machine".

In step 1, select "Local install media" and click "Forward".

In step 2, click "Browse.." -> "Browse Local" -> Select your Windows ISO file.

In step 3, choose how much memory and how many cpu threads you want to use (both of these can be changed later).

wip: In step 4, If you have a dedicated SSD you want to use, see [Use a dedicated SSD](Use-a-dedicated-SSD) -> "Select or create custom storage" -> "Manage..." -> "Add Pool" -> Set "Name:" to "/dev/sdb" and set "Type:" to "disk"

In step 5, tick "Customize configuration before install" and click "Finish".


### Configure the Virtual machine

In the "Overview" section, set "Firmware" to "UEFI" and click "Apply".

In the "CPUs" section, untick "Copy host CPU configuration" and set "Model" to "host-passthrough" (If it isn't on the list, type it in). Tick "Manually set CPU topology", what you want here depends on your hardware.

In the "Boot Options" section, tick "SATA Disk 1" and "SATA CDROM 1" and move "SATA CDROM 1" to the top.

In the "SATA CDROM 1" section, set "IO mode" to "threads".

In the "SATA Disk 1" section, set "IO mode" to "threads" and "Disk bus" to "VirtIO" (or SCSI if you use a dedicated SSD for the VM).

In the "NIC" section, set "Device model" to "virtio".


#### Add a CDROM device for the virtio driver iso

Download the latest [virtio iso](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/latest-virtio/)

Click "Add hardware" -> Select "Storage" and set "Device type" to "CDROM device"

Click "Manage..." -> Click "Browse Local" -> Select the virtio-win iso file and click "Finish"


### Pass-through a graphics card to the virtual machine

Click "Add hardware" -> Select "PCI Host Device"

Select the graphics card -> Click "Finish"

Select the graphics cards audio controller -> Click "Finish"


### Install Windows

Install Windows like you normally would, but at the disk selection screen, click "Load driver" -> "Browse" -> 

Navigate to the "virtio-win" CD drive -> viostor

Select to the right sub-folder for your Windows version and cpu architecture (example: E:\viostor\w10\amd64) then click "OK"

Select "Red Hat VirtIO SCSI controller" and click "Next"


### Configure Windows

Start the Device Manager

Righ-click on "Ethernet Controller" -> "Update driver" -> "Browse my computer for driver software" -> Browse -> Select the virtio-win CD drive and click "OK" -> Click "Next" -> Click "Install"

Repeat the previous step for "PCI Device"

Install the drivers for your graphics card (If you have an nVidia card see [Nvidia error 43](Nvidia-error-43)).


## Troubleshooting

### Nvidia error 43

Fix error 43 with nVidia GPU

```
sudo EDITOR=nano virsh edit <vm_name>
```

```xml
  <features>
    <hyperv>
            <vendor_id state='on' value='1234567890ab'/>
    </hyperv>
    <kvm>
      <hidden state='on'/>
    </kvm>
  </features>
```


### EVDEV Passthrough

https://passthroughpo.st/using-evdev-passthrough-seamless-vm-input/

Test if it is the right device
```bash
cat /dev/input/by-id/MOUSE_NAME
cat /dev/input/by-id/KEYBOARD_NAME
```

```xml
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
  <qemu:commandline>
    <qemu:arg value='-object'/>
    <qemu:arg value='input-linux,id=mouse1,evdev=/dev/input/by-id/MOUSE_NAME'/>
    <qemu:arg value='-object'/>
    <qemu:arg value='input-linux,id=kbd1,evdev=/dev/input/by-id/KEYBOARD_NAME,grab_all=on,repeat=on'/>
  </qemu:commandline>
</domain>
```

```
/etc/libvirt/qemu.conf
```

```
cgroup_device_acl = [
    "/dev/null", "/dev/full", "/dev/zero",
    "/dev/random", "/dev/urandom",
    "/dev/ptmx", "/dev/kvm", "/dev/kqemu",
    "/dev/rtc","/dev/hpet",
    "/dev/input/by-id/usb-Corsair_Corsair_Gaming_K65_LUX_RGB_Keyboard_0903001BAECC0C82570EB9B1F5001940-if01-event-kbd",
    "/dev/input/by-id/usb-Logitech_USB-PS_2_Optical_Mouse-event-mouse"
]
```

```
sudo systemctl restart libvirtd.service
```


#### Add keyboard and Mouse to the virtual machine

Click "Add hardware" -> Select "USB Host Device"

Select your keyboard -> Click "Finish"

Repeat for your mouse

Switch the input devices from PS/2 to Virtio driver


### Looking Glass

```sh
sudo EDITOR=nano virsh edit <vm_name>
```

```xml
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
  <disk type='block' device='disk'>
    <source dev='/dev/sdb'/>
  </disk>
  <qemu:commandline>
    <qemu:arg value='-device'/>
    <qemu:arg value='ivshmem-plain,memdev=ivshmem'/>
    <qemu:arg value='-object'/>
    <qemu:arg value='memory-backend-file,id=ivshmem,share=on,mem-path=/dev/shm/looking-glass,size=32M'/>
  </qemu:commandline>
</domain>
```

https://looking-glass.hostfission.com/
https://forum.level1techs.com/t/looking-glass-guides-help-and-support/122387

Install dependencies
```
sudo eopkg install -c system.devel
sudo eopkg install sdl2-devel sdl2-ttf-devel libnettle-devel spice-devel fontconfig-devel libx11-devel libconfig-devel libglu-devel
sudo eopkg install mesalib-devel #not needed?
```

Download the [latest release](https://github.com/gnif/LookingGlass/releases)

Compile the client
```
mkdir client/build
cd client/build
cmake ../
make
```

Formula for memory size:
```
width x height x 4 x 2 + 2MB = total bytes
Round up to the nearest power of two (n^2)
1920 x 1080 = 17.82 = 32MB
```

```
sudo EDITOR=nano virsh edit VM_NAME
```

```xml
<devices>
  <shmem name='looking-glass'>
    <model type='ivshmem-plain'/>
    <size unit='M'>32</size>
  </shmem>
</devices>
```

Run every start of the host


```
/etc/systemd/system/create-shared-memory-file.service
```

```
[Service]
ExecStart=/opt/looking_glass/create-shared-memory-file.sh
```

```
/etc/systemd/system/create-shared-memory-file.timer
```

```
[Unit]
Description=Create shared memory file for Looking Glass

[Timer]
OnBootSec=0min

[Install]
WantedBy=timers.target
```

```
/opt/looking_glass/create-shared-memory-file.sh
```

```
#!/bin/bash
# -*- coding: utf-8 -*-

if [ "$EUID" -ne 0 ]; then
    echo "This script must be run as root"
    exit 1
fi

echo "Creating shared memory file..."
touch /dev/shm/looking-glass
chown root:kvm /dev/shm/looking-glass
chmod 660 /dev/shm/looking-glass
echo "Done"
```

In Windows

Download the ivshmem drivers: https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/upstream-virtio/

Open "Device Manager" -> System Devices -> PCI standard RAM Controller -> Update driver (ivshmem) https://github.com/virtio-win/kvm-guest-drivers-windows/issues/217

Run the host application in Windows

Run the client application in Solus
```
cd client/build
./looking-glass-client
```

## Performance tuning

Todo:
- CPU pinning
- Transparent huge pages
- Static huge pages

Enable Hugepages
```
sudo ./hugepages.sh
reboot
cat /proc/meminfo
```

### Use a dedicated SSD

wip: Find out the path to the drive you want to use (ex. /dev/sdb)
```
sudo parted --list
```

Select "Select or create custom storage" -> Click "Manage..." -> Click "Add Pool"

In step 1, Set "Name:" to "win_ssd" and "Type:" to "disk"

In step 2:
- Click "Browse" next to "Source Path", navigate to "/dev/disk/by-id/" and select the drive you want to use.
- Tick "Build Pool"? and click "Finish"

Select the new pool and click "Create new volume"

Set "Name" to "sdb1"

Select the new volume and click "Choose Volume"

```
sudo EDITOR=nano virsh edit VM_NAME
```

```xml
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
  <disk type='block' device='disk'>
    <source dev='/dev/disk/by-id/'/>
  </disk>
</domain>
```

Replace /dev/disk/by-id/ with the path to your drive


## References
https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF

https://solus-project.com/forums/viewtopic.php?f=11&t=1479&sid=597a58c06f144e3a778bc6a1d49e7de2