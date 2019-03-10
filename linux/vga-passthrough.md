# VGA Passthrough on Solus

Note: Solus might need to be installed in UEFI mode for this guide to work

## Pre-requirements

Read the [Prerequisites section](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#Prerequisites) in the ArchWiki

- CPU must support hardware virtualization and IOMMU
- Motherboard must support IOMMU
- Guest GPU must support UEFI


## Isolating the GPU

Install dependencies:
```bash
sudo eopkg install dracut
```

Download [vfio-bind.sh](https://raw.githubusercontent.com/gmol1/vfio-bind/master/vfio-bind.sh) from https://github.com/gmol1/vfio-bind

Run vfio-bind.sh
```bash
sudo ./vfio-bind.sh
```
Reboot

Check that the device(s) now are using vfio-pci
```
lspci -k
```

## Install Virtual Machine Manager

```
sudo eopkg it qemu virt-manager ovmf virt-viewer
sudo systemctl enable libvirtd
sudo gpasswd -a $USERNAME kvm
sudo gpasswd -a $USERNAME libvirt
```
reboot


## Install Windows in an virtual machine

Start Virtual Machine Manager

### Create the Virtual Machine

Click on "Create a new virtual machine" or go to "File" -> "New Virtual Machine"

On Step 1, select "Local install media"

On Step 2, click "Browse.." -> "Browse Local" -> Select your Windows ISO file

On Step 3, Choose how much memory and how many cpu threads you want to use

On Step 4, Select storage (I use a dedicated SSD) -> "Select or create custom storage" -> "Manage..." -> "Add Pool" -> Set "Name:" to "/dev/sdb" and set "Type:" to "disk"

If you want to use a dedicated SSD click here. see this section.

---

Select "Select or create custom storage" -> Click "Manage..." -> Click "Add Pool"

Step 1, Set "Name:" to "sdb" and "Type:" to "disk"

Step 2, Set "Target Path" to ""? and "Source Path" to "/dev/sdb"

Click "Create new volume"

Set "Name" to "sdb1"

Select the new volume and click "Choose Volume"

Set "Bus type" to "VirtIO"

---

On Step 5, Tick "Customize configuration before install", click "Finish".

### Configure the Virtual machine

In the "Overview" section, set "Firmware" to "UEFI" and click "Apply".

In the "CPUs" section, untick "Copy host CPU configuration" and set "Model" to "host-passthrough" (If it's not on the list, type it in).

In the "Boot Options" section

Tick "VirtIO Disk 1" and "SATA CDROM 1"

Move "SATA CDROM 1" to the top

In the "SATA CDROM 1" section, set "IO mode" to "threads"

In the "VirtIO Disk 1" section, set "IO mode" to "threads"

In the "NIC" section, set "Device model" to "virtio"


#### Add a CDROM device for the virtio driver iso

Download [virtio iso](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/latest-virtio/)

Click "Add hardware" -> Select "Storage"

Set "Device type" to "CDROM device"

Click "Manage..." -> Click "Browse Local" -> Select the virtio-win iso file

Click "Finish"

In the "SATA CDROM 2" section, set "IO mode" to "threads"


### Pass-through a graphics card to the virtual machine

Click "Add hardware" -> Select "PCI Host Device"

Select the graphic card -> Click "Finish"

Repeat for graphic cards hdmi audio device


### Nvidia error 43

```
virsh edit <vm_name>
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
sudo virsh edit <vm_name>
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
sudo virsh edit VM_NAME
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
sudo ./create-shared-memory-file.sh
  touch /dev/shm/looking-glass
  chown root:kvm /dev/shm/looking-glass
  chmod 660 /dev/shm/looking-glass

In Windows

Download the ivshmem drivers: https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/upstream-virtio/

Device Manager -> System Devices -> PCI standard RAM Controller -> Update driver (ivshmem) https://github.com/virtio-win/kvm-guest-drivers-windows/issues/217

Run the host application
```
cd client/build
./looking-glass-client
```

## Performance optimizations

Enable Hugepages
```
sudo ./hugepages.sh
reboot
cat /proc/meminfo
```


### To do

- Create network bridge
- [Scream Network Audio](https://github.com/duncanthrax/scream) - Sound ich9 -> Remove
- More performance optimizations


## References
https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF
https://solus-project.com/forums/viewtopic.php?f=11&t=1479&sid=597a58c06f144e3a778bc6a1d49e7de2
