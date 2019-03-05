# VGA Passthrough on Solus

## Bind GPU to vfio-pci

Install dependencies:
```bash
sudo eopkg install dracut
```
Download [vfio-bind.sh](https://raw.githubusercontent.com/endumiuz/vfio-bind/master/vfio-bind.sh) from https://github.com/endumiuz/vfio-bind

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

## Performance optimizations

Enable Hugepages
```
sudo ./hugepages.sh
reboot
cat /proc/meminfo
```

## Install Windows in an virtual machine

Start Virtual Machine Manager

### Create the Virtual Machine

Click on "Create a new virtual machine" or go to "File" -> "New Virtual Machine"

On Step 1, select "Local install media"

On Step 2, click "Browse.." -> "Browse Local" -> Select your Windows ISO file

On Step 3, Choose how much memory and how many cpu threads you want to use

On Step 4, Select storage (I use a dedicated SSD) -> "Select or create custom storage" -> "Manage..." -> "Add Pool" -> Set "Name:" to "/dev/sdb" and set "Type:" to "disk"

On Step 5, Tick "Customize configuration before install", click "Finish"

### Add a CDROM device for the virtio driver iso

Download [virtio iso](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/latest-virtio/)

Click "Add hardware" -> Select "Storage"

Set "Device type" to "CDROM device"

Click "Manage..." -> Click "Browse Local" -> Select the virtio-win iso file

Click "Finish"

wip:

  Click "Add hardware"
    Storage
      Select "Select or create custom storage"
      Click "Manage"
        Click "Add Pool"
          Step 1 of 2
            Set "Name" to "sdb"?
            Set "Type" to "disk: Physical Disk Device"
            Click "Forward"
          Step 2 of 2
            Set "Target Path" to ""?
            Set "Source Path" to "/dev/sdb"?
            Click "Finish"
        Click "Create new volume"
          Set "Name" to "sdb1"?
          Click "Finish"
        Select the new volume and click "Choose Volume"
      Set "Bus type" to "VirtIO"
  Click "Add hardware"
    PCI Host Device
      Add graphic card
      Click "Finish"
      Repeat for graphic card hdmi audio
  Click "Add hardware"
    USB Host Device
      Add Keyboard
      Click "Finish"
      Repeat for Mouse
  Boot Options
    Tick "VirtIO Disk 1" and "SATA CDROM 1"
    Move "SATA CDROM 1" to the top
  SATA CDROM 1
    Set "IO mode" to "threads"
  SATA CDROM 2
    Set "IO mode" to "threads"
  VirtIO Disk 1
    Set "IO mode" to "threads"
  NIC :nn:nn:nn
    Set "Device model" to "virtio"
  Tablet
    Remove
  Sound ich6
    Remove
  Serial 1
    Remove?
  Controller USB 0
    Set "Model" to "USB 3"?
  Controller VirtIO Serial 0
    Remove?

sudo eopkg install vim
sudo virsh edit win10
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

Install Windows
Enable System Restore
Install and run ShutUp10
Reboot
Run ShutUp10 again
Install Network drivers
Run Windows Update
Run ShutUp10 again
Add GPU to VM
Install GPU Drivers
Reboot host just to be sure

-----

Nvidia error 43

virsh edit [vmname]
  <features>
    <hyperv>
            <vendor_id state='on' value='1234567890ab'/>
    </hyperv>
    <kvm>
      <hidden state='on'/>
    </kvm>
  </features>


-----

# Check CPU info in Windows
wmic cpu get caption, deviceid, name, numberofcores, status

-----

- Performance Tuning -
Virtual disk performance options = cache mode, and io mode

-----

https://passthroughpo.st/using-evdev-passthrough-seamless-vm-input/

- EVDEV Passthrough -
# Test if it is the right device
cat /dev/input/by-id/MOUSE_NAME
cat /dev/input/by-id/KEYBOARD_NAME

<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
  <qemu:commandline>
    <qemu:arg value='-object'/>
    <qemu:arg value='input-linux,id=mouse1,evdev=/dev/input/by-id/MOUSE_NAME'/>
    <qemu:arg value='-object'/>
    <qemu:arg value='input-linux,id=kbd1,evdev=/dev/input/by-id/KEYBOARD_NAME,grab_all=on,repeat=on'/>
  </qemu:commandline>
</domain>

/etc/libvirt/qemu.conf
cgroup_device_acl = [
    "/dev/null", "/dev/full", "/dev/zero",
    "/dev/random", "/dev/urandom",
    "/dev/ptmx", "/dev/kvm", "/dev/kqemu",
    "/dev/rtc","/dev/hpet",
    "/dev/input/by-id/usb-Corsair_Corsair_Gaming_K65_LUX_RGB_Keyboard_0903001BAECC0C82570EB9B1F5001940-if01-event-kbd",
    "/dev/input/by-id/usb-Logitech_USB-PS_2_Optical_Mouse-event-mouse"
]

sudo systemctl restart libvirtd.service
Switch the input devices from PS/2 to Virtio driver

-----

- Passing VM audio to host via PulseAudio -
- (Use Scream Network Audio instead) -
/etc/libvirt/qemu.conf
user = "YOUR_USERNAME"

Replace 1000 with your user id (run id)
<qemu:commandline>
  <qemu:env name='QEMU_AUDIO_DRV' value='pa'/>
  <qemu:env name='QEMU_PA_SERVER' value='/run/user/1000/pulse/native'/>
</qemu:commandline>

sudo systemctl restart libvirtd.service
sudo systemctl restart pulseaudio.service

-----

- Looking Glass -
https://looking-glass.hostfission.com/
https://forum.level1techs.com/t/looking-glass-guides-help-and-support/122387

# Install dependencies
sudo eopkg install -c system.devel
sudo eopkg install sdl2-devel sdl2-ttf-devel libnettle-devel spice-devel fontconfig-devel libx11-devel libconfig-devel libglu-devel

sudo eopkg install mesalib-devel #not needed?

# Download the latest release from github
# https://github.com/gnif/LookingGlass/releases

mkdir client/build
cd client/build
cmake ../
make

# Formula for memory size:
# width x height x 4 x 2 + 2MB = total bytes
# Round up to the nearest power of two
# 1920 x 1080 = 17.82 = 32MB

sudo virsh edit VM_NAME
<devices>
  <shmem name='looking-glass'>
    <model type='ivshmem-plain'/>
    <size unit='M'>32</size>
  </shmem>
</devices>

# Run every start of the host
sudo ./create-shared-memory-file.sh
  touch /dev/shm/looking-glass
  chown kim:kvm /dev/shm/looking-glass
  chmod 660 /dev/shm/looking-glass

In Windows
  Download the ivshmem drivers: https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/upstream-virtio/
  Device Manager -> System Devices -> PCI standard RAM Controller -> Update driver (ivshmem) https://github.com/virtio-win/kvm-guest-drivers-windows/issues/217
  Run the host application

cd client/build
./looking-glass-client -k -F -Q -o opengl:mipmap=0 -s

-----
- Create network bridge -

nm-connection-editor -> Click "Add a new connection"
  Select "Bridge" -> Click "Create"
    Bridge
      Click "Add" -> Select "Ethernet" -> Click "Create"
        Ethernet
          Set "Device" to your NIC
          Click "Save"
    Ipv4 Settings
      Set "Method"
    Click "Save"

Virtual Machine Manager
  Select "QEMU/KVM" -> Edit -> Connection Details
    Virtual Networks
      Click "Add Network"
        Step 1 of 4
          Set "Network Name" to "isolated"
          Click "Forward"
        Step 2 of 4
          Set "Network"
          Click "Forward"
        Step 3 of 4
          Click "Forward"
        Step 4 of 4
          Select "Isolated virtual network"
          Click "Finish"

In Virtual Machine Manager (virt-manager) -> win10
  Click "Add Hardware"
    Network
      Set "Network source" to "...isolated..."
      Set "Device model" to "virtio"
      Click "Finish"

-----

-  Scream Network Audio -
https://github.com/duncanthrax/scream

Create a separate NIC for audio
sudo eopkg install pulseaudio-devel alsa-lib-devel

ncpa.cpl -> right-click network adapter -> properties
  TCP/IPv4 -> Properties -> Advanced...
    Untick "Automatic metric"
    Select a value of 2 or higher
Alternative: https://sites.google.com/site/embeddedsystesting/home/how-to-modify-or-delete-windows-default-multicast-routes

cd Receivers/alsa/
make
./scream-alsa -i virbr1

cd /Receivers/pulseaudio/
make
./scream-pulse

## References
https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#Setting_up_an_OVMF-based_guest_VM
https://solus-project.com/forums/viewtopic.php?f=11&t=1479&sid=597a58c06f144e3a778bc6a1d49e7de2
