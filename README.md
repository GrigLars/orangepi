# Orange Pi
Notes and files related to Orange Pi

So, I got an Orange Pi 4b the other day, and all their documentation was outadted and partially wrong.  I stumbled my way through a lot of basic setup to get Ubuntu on there, and some of these notes might help someone else.

They still don't have a case for it.  I might have to make one in Thingverse or something.

The Orange Pi 4b is a step up from the Orange Pi 4.  Like the rest of the line from Shenzhen Xunlong Software CO., Limited, it's an open-source single-board computer (SBC) that can run Android 8.1, Ubuntu 16.04, Ubuntu 18.04, and, Debian 9e. It uses the Rockchip RK3399 (6-core ARM 64-bit processor, main frequency speeds up to 2.0GHz), and equipped with Dual 4GB LPDDR4, 16GB eMMC flash, and it's got a Mali-T864 GPU.  You can attach peripherals, like SDHC, 2x USB, 2x cameras, and other stuff.  All the specs [are here](http://www.orangepi.org/Orange%20Pi%204B/) if you want the nitty gritty.  This also has a SPR2801S, adopt MPE and APiM unique AI architecture for an NPU, which is why I got it.

But just getting Ubuntu 18.04 to work on it was a mess.  First, [I got the image here](http://www.orangepi.org/downloadresources/OrangePi4B/2020-01-03/orangepi3_f9433ee3d9493fd6fc711c065d.html), which is on a Google Drive and took forever to download.  I might Bitorrent this later.  Three hours!  We can do better.

You unzip, then unzip again, and burn the IMG file to an SDHC using something like [Etcher](https://www.balena.io/etcher/) or [Rufus](https://rufus.ie/).  I had a 32gb card, but the image takes up about 5gb, so you have to expand it.

I loaded it onto the card, made a crude case out of some scraps of retail packaging I had lying around (really), hooked it up, and spun her up.  Thankfully, openssh-server was already installed.  I got in via the LAN port, and wireless was operational, but not set up properly.  A lot of their documentation went off the rails right after this, partly because it was based on Ubuntu 14.04, but also because of poor spelling, a mix of pathnames, and just general vague comments.  But, I have a lot of experience with SBCs, mainly RaspberryPi, and some other boards like Sokris, Le Potato, and Fitlet. Here are some steps to just get *basic Ubuntu 18.04 LTS* working on an Orange Pi 4b, but who knows, might help someone with another Orange Pi board:

Change OrangePi user to not ask for a sudo password.  This drove me nuts.  I also needed it for an ansible push later on for setup.  You'll have to log back in for changes to take effect.

```sudo echo "orangepi ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/orangepi```

For some reason, all of /etc is owned by orangepi, which causes weird errors when the environment expects it to be owned by root.

 ```chown root /etc```

Next, you'll barely have a few MB of space left to do anything, including download programs.  
```
orangepi@OrangePi:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/root       5.0G  4.7G   39M 100% /
devtmpfs        1.9G     0  1.9G   0% /dev
tmpfs           1.9G     0  1.9G   0% /dev/shm
tmpfs           1.9G   17M  1.9G   1% /run
tmpfs           5.0M  4.0K  5.0M   1% /run/lock
tmpfs           1.9G     0  1.9G   0% /sys/fs/cgroup
tmpfs           385M  4.0K  385M   1% /run/user/107
tmpfs           385M     0  385M   0% /run/user/1000
```
The way they set it up is that you have several block devices:
```
NAME         MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
mmcblk1      179:0    0 14.6G  0 disk 
mmcblk1boot0 179:32   0    4M  1 disk 
mmcblk1boot1 179:64   0    4M  1 disk 
mmcblk1rpmb  179:96   0    4M  0 disk 
mmcblk0      179:128  0 29.8G  0 disk 
├─mmcblk0p1  179:129  0    4M  0 part 
├─mmcblk0p2  179:130  0    4M  0 part 
├─mmcblk0p3  179:131  0   32M  0 part 
└─mmcblk0p4  179:132  0  5.2G  0 part /
```
Documentation states, "if it's not automatically done, please run ```resize-helper```."  That file does not exist.  [I found it here](https://github.com/fboudra/96boards-tools/blob/master/resize-helper), but it didn't work.  Long story short, the image itself was not written correctly, so even resize2fs didn't work properly at first.  I had to do combinations of:

* touch /forcefsck [and reboot]
* Use cfdisk to fix the partition, [needs to be written]
* sudo resize2fs -f /dev/mmcblk0p4

Before it "took."
```
Filesystem      Size  Used Avail Use% Mounted on
/dev/root        30G  4.2G   24G  15% /
```
Now I could fix other things.

DNS doesn't work.  This might be because of /etc above, not sure. But I had to put their own mirror server in the /etc/hosts file just so that it could get packages:
```
127.0.0.1 localhost
127.0.1.1 orangepi
101.6.8.193 mirrors.tuna.tsinghua.edu.cn
# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```
Next, run "unminimize" to fix the "minimized packaging setup", and this will take a while:
```
sudo unminimize
```
While the package **iw** is installed, **wireless-tools** and **wpasupplicant** is not, and you can't fix wireless until you do:
```
sudo apt install wireless-tools wpasupplicant
```
Now you have to disable all the bratty NetworkManager stuff:
```
sudo systemctl disable NetworkManager-wait-online NetworkManager-dispatcher NetworkManager
```
Search for your wireless networks (and make sure it works):
```
orangepi@OrangePi:~$ sudo iwlist wlan0  scan | grep ESSID
                    ESSID:"Abraham Linksys"
                    ESSID:"Omega-K2SO"
                    ESSID:"DIRECT-roku-411"
                    ESSID:"Wi-Fight the Inevitable"
                    ESSID:"It Burns When IP"
                    ESSID:"NETGEAR64"
                    ESSID:"FBI Surveillance Van 12"
                    ESSID:"Wireless-AC"
                    ESSID:"Puggalicious-5G"
```
Create the wpa_supplicant config file; note the space *before* the command, or the password will be stored in your bash history file.
```
 wpa_passphrase your-ESSID your-wifi-passphrase | sudo tee /etc/wpa_supplicant.conf 
```
If your wireless router doesn’t broadcast ESSID, then you need to add the following line in /etc/wpa_supplicant.conf file below "psk=": ```scan_ssid=1```

Stop Network Manager:
```
sudo systemctl stop NetworkManager
```
Test, should have "wlan0: CTRL-EVENT-CONNECTED" listed.  Control+C to stop.
```
sudo wpa_supplicant -c /etc/wpa_supplicant.conf -i wlan0
Successfully initialized wpa_supplicant
wlan0: Trying to associate with 00:02:6f:8e:38:51 (SSID='KatanaTikiKoi' freq=2437 MHz)
wlan0: Associated with 00:02:6f:8e:38:51
wlan0: CTRL-EVENT-SUBNET-STATUS-UPDATE status=0
wlan0: WPA: Key negotiation completed with 00:02:6f:8e:38:51 [PTK=CCMP GTK=CCMP]
wlan0: CTRL-EVENT-CONNECTED - Connection to 00:02:6f:8e:38:51 completed [id=0 id_str=]
```
You can now run it it background in another window
```
sudo wpa_supplicant -B -c /etc/wpa_supplicant.conf -i wlan0
```
Get an IP address for your wireless connection
```
sudo dhclient wlan0
```
To automatically connect to wireless network at boot time, we need to edit the wpa_supplicant.service file.
```
sudo cp /lib/systemd/system/wpa_supplicant.service /etc/systemd/system/wpa_supplicant.service
sudo vim /etc/systemd/system/wpa_supplicant.service
```
Change the ExecStart line to this and comment out the Alias:
```
ExecStart=/sbin/wpa_supplicant -u -s -c /etc/wpa_supplicant.conf -i wlan0
[...]
# Alias=dbus-fi.w1.wpa_supplicant1.service
```
Enable the service at boot
```
sudo systemctl enable wpa_supplicant.service
```
Create a new service to get an IP address:
```
sudo vim /etc/systemd/system/dhclient.service
```
Put in the following:
```
        [Unit]
        Description= DHCP Client
        Before=network.target
        After=wpa_supplicant.service

        [Service]
        Type=simple
        ExecStart=/sbin/dhclient wlan0 -v
        ExecStop=/sbin/dhclient wlan0 -r
        
        [Install]
        WantedBy=multi-user.target
```
Enable THAT service:
```
sudo systemctl enable dhclient.service
```
