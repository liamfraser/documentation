# Network Boot Your Raspberry Pi
This tutorial is written to explain how to set up a simple DHCP / TFTP server that will allow you to boot a Raspberry Pi 3 from the network. The tutorial assumes you have an existing home network, and want to use a Raspberry Pi for the SERVER. You will need a second Pi 3 as a CLIENT to be booted. Only one SD card is needed because the CLIENT will be booted from the SERVER after the initial client configuration.

## Client configuration
Before a Pi will network boot, it needs to be booted with a config option to enable USB Boot Mode. Enabling this config option requires a special `start.elf` and `bootcode.bin` file. 

Install Raspbian lite (or heavy if you want) from the [Downloads page](https://www.raspberrypi.org/downloads/raspbian/) onto an SD card using `Win32DiskImager` if you are on Windows, or `dd` if you are on Linux/Mac. Boot the CLIENT Pi.

### Program USB Boot Mode
First, prepare the `/boot` directory with new `start.elf` and `bootcode.bin` files:
```
cd /boot
sudo rm start.elf bootcode.bin start_* fixup*
sudo wget https://github.com/raspberrypi/documentation/raw/master/hardware/raspberrypi/bootmodes/start.elf 
sudo wget https://github.com/raspberrypi/documentation/raw/master/hardware/raspberrypi/bootmodes/bootcode.bin
sudo sync
```

Then enable USB Boot Mode with:
```
echo program_usb_boot_mode=1 | sudo tee -a /boot/config.txt
```

which adds `program_usb_boot_mode=1` to the end of `/boot/config.txt`. Then reboot the Pi with `sudo reboot`. Once the client Pi has rebooted, check that the OTP is has been programmed with:

```
$ vcgencmd otp_dump | grep 17:
17:3020000a
```

Ensure the output `0x3020000a` is correct.

The client configuration is almost done. The final thing to do is to remove the `program_usb_boot_mode` line from config.txt. You can do this with `sudo nano /boot/config.txt` for example. Finally, shut down the client Pi with `sudo poweroff`

## Server configuration
Plug the SD card into the SERVER and boot the server. Before you do anything else, make sure you have ran `sudo raspi-config` and expanded the root filesystem to take up the entire SD card.

The client Pi will need a root filesystem to boot off, so before we do anything else on the server, we're going to make a full copy of its filesystem and put it in a directory called /nfs/client1.

```
sudo mkdir -p /nfs/client1
sudo apt-get install rsync
sudo rsync -xa --progress --exclude /nfs / /nfs/client1
```

Regenerate ssh host keys on client filesystem by chrooting into it
```
sudo chroot /nfs/client1
rm /etc/ssh/ssh_host_*
dpkg-reconfigure openssh-server
exit
```

You need to find the settings of your local network. You need to find the address of your router (or gateway), which you can find with:
```
ip route | grep default | awk '{print $3}'
```

Then run:

```
ip -4 addr show dev eth0 | grep inet
```

which should give an output like:

```
inet 10.42.0.211/24 brd 10.42.0.255 scope global eth0
```

The first address is the IP address of your server Pi on the network, and the part after the slash is the network size. It is highly likely that yours will be a `/24`, which means you can use any address between 10.42.0.1 and 10.42.0.254. When picking an address for your Pi, you need to exclude the gateway address. For example, if your gateway is 10.42.0.1 we suggest using 10.42.0.2 for your server Pi. Also note the `brd` (broadcast) address of the network. Note down the output of the previous two commands.

Finally note down the address of your DNS server, which is very likely the same address as your gateway. You can find this with:
```
cat /etc/resolv.conf
```

Configure a static network adddress on your server Pi by with `sudo nano /etc/network/interfaces` (where you replace nano with an editor of your choice). Change the line, `iface eth0 inet manual` to look something like, where the address is the first address from the command before last, the address as `255.255.255.0` and the gateway address as the number received from the last command. 

```
auto eth0
iface eth0 inet static 
        address 10.42.0.2
        netmask 255.255.255.0
        gateway 10.42.0.1
```

Then disable the DHCP client daemon and switch to standard Debian networking
```
sudo systemctl disable dhcpcd
sudo systemctl enable networking
```

Reboot for the changes to take effect
```
sudo reboot
```

At this point, you won't have working DNS, so you'll need to add the server you noted down before to `/etc/resolv.conf`:

```
echo "nameserver 10.42.0.1" | sudo tee /etc/resolv.conf
```

Then install software we need:
```
sudo apt-get update
sudo apt-get install dnsmasq tcpdump
```

Stop dnsmasq breaking DNS resolving:
```
sudo rm /etc/resolvconf/update.d/dnsmasq
sudo reboot
```

Then set your nameserver again (because dnsmasq broke it):
```
echo "nameserver 10.42.0.1" | sudo tee /etc/resolv.conf
```

Now start tcpdump so you can search for DHCP packets from the client Pi:

```
sudo tcpdump -i eth0 port bootpc
```

Connect the client Pi to your network and power it on. Check that the LEDs illuminate on the CLIENT after around 10 seconds, then should get a packet from the CLIENT "DHCP/BOOTP, Request from ..."

```
IP 0.0.0.0.bootpc > 255.255.255.255.bootps: BOOTP/DHCP, Request from b8:27:eb...
```

Now we need to modify the dnsmasq configuration to enable DHCP to reply to the device ...

```
sudo echo | sudo tee /etc/dnsmasq.conf
sudo nano /etc/dnsmasq.conf
```

Then replace the contents of dnsmasq.conf with:

```
port=0
dhcp-range=10.42.0.255,proxy
log-dhcp
enable-tftp
tftp-root=/tftpboot
pxe-service=0,"Raspberry Pi Boot"
```

Where the first address of the `dhcp-range` line is the broadcast address you noted down earlier.

Now create a /tftpboot directory

```
$ sudo mkdir /tftpboot
$ sudo chmod 777 /tftpboot
$ sudo systemctl enable dnsmasq.service
$ sudo systemctl restart dnsmasq.service
```

Now monitor the dnsmasq log

```
tail -f /var/log/daemon.log
```

You should see something like:
```
raspberrypi dnsmasq-tftp[1903]: file /tftpboot/bootcode.bin not found
```

Next, you will need to copy [bootcode.bin](bootcode.bin) and [start.elf](start.elf) into the /tftpboot directory, you should be able to do this by just copying the files from /boot since they are the right ones. Also we need a kernel so might as well copy the entire boot directory.

```
cp -r /boot/* /tftpboot
```

Restart dnsmasq for good measure
```
sudo systemctl restart dnsmasq
```

### Set up NFS root
This should now allow your Pi to boot through until it tried to load a root filesystem (which it doesn't have). All we have to do to get this working is to export the `/nfs/client1` filesystem we created earlier.

```
sudo apt-get install nfs-kernel-server
echo "/nfs/client1 *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports
sudo systemctl enable rpcbind
sudo systemctl restart rpcbind
sudo systemctl enable nfs-kernel-server
sudo systemctl restart nfs-kernel-server
```

Edit /tftpboot/cmdline.txt and from `root=` onwards, replace it with:

```
root=/dev/nfs nfsroot=10.42.0.2:/nfs/client1 rw ip=dhcp rootwait elevator=deadline
```

substituting your network settings as necessary.

Finally, edit /nfs/client1/etc/fstab and remove the /dev/mmcblkp1 and p2 lines (only proc should be left).

Good luck! If it doesn't boot first time keep trying. It can take a minute or so for the Pi to boot, so be patient.