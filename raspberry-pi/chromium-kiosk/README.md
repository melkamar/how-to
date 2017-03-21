# Chromium browser in kiosk mode on start

## Installing Raspbian
- Download Raspbian with PIXEL from https://www.raspberrypi.org/downloads/raspbian/
- Unzip, extract the image on your SD card
  - For Windows I used e.g. https://sourceforge.net/projects/win32diskimager/
  - See this answer for more details: http://raspberrypi.stackexchange.com/questions/931/how-do-i-install-an-os-image-onto-an-sd-card
- Boot RPI

## Initial setup
- Upgrade Raspbian:
```
sudo su
apt-get update
apt-get upgrade
reboot
```
- Customize locale, make sure the whole SD card is used and RPI boots to GUI without login prompt:
```
sudo raspi-config  # Choose needed items in the menu
reboot
```

- Cleanup packages you don't need that use up space:
```
sudo su
apt-get remove --purge wolfram-engine triggerhappy
apt-get autoremove --purge
```

- Install packages you will need:
```
sudo apt-get install vim noclutter
```
  - `noclutter` will hide mouse cursor to get it out of the way

## Making filesystem read-only
- Edit `/etc/fstab` accordingly:
```
proc            /proc             proc    defaults             0       0
/dev/mmcblk0p1  /boot             vfat    defaults,ro          0       2
/dev/mmcblk0p2  /                 ext4    defaults,noatime,ro  0       1
tmpfs	          /var/log	        tmpfs   nodev,nosuid	       0       0
tmpfs	          /var/tmp	        tmpfs	  nodev,nosuid	       0       0
tmpfs           /tmp              tmpfs   nodev,nosuid         0       0
tmpfs           /var/lib/lightdm  tmpfs   nodev,nosuid         0       0
tmpfs           /home/pi          tmpfs   nodev,nosuid         0       0
```
- Explanation:
   - Anything physically mounted from the SD card is read-only (`ro` flag).
   - Temporary and log folders are writable, but mounted as tmpfs (i.e. in memory)
   - Trial and error showed that `lightdm` insists on writing to its `/var/lib/lightdm` folder, so mount that as tmpfs. 
   - `Xserver` needs to write to `~/.Xauthority`, Chromium might want to write something to home folder as well, so mount it as tmpfs.
  
- Using DHCP requires write access to `/etc/resolv.conf`, so make that a symlink from a writeble directory:
```
touch /tmp/dhcpcd.resolv.conf
rm /etc/resolv.conf
ln -s /tmp/dhcpcd.resolv.conf /etc/resolv.conf
```

- Also disable swap and start the FS as read-only by adding `noswap ro` at the and of the line in `/boot/cmdline.txt`.

## Automatically start Chromium in kiosk mode
- Edit the `/etc/xdg/lxsession/LXDE-pi/autostart` file to have the following:
```
@lxpanel --profile LXDE-pi
@pcmanfm --desktop --profile LXDE-pi
#@xscreensaver -no-splash  # Turn of screensaver
@point-rpi
@/usr/bin/chromium-browser --kiosk --disable-restore-session-state --no-first-run http://your-page.example
@unclutter -idle 0.1 -root
```

## Optional - set up WPA Supplicant for 802.1X network authentication
- Edit `/etc/wpa_supplicant/wpa_supplicant.conf`. This will depend on your particular network, refer to your administrators for more details.
```
ctrl_interface=/var/run/wpa_supplicant
ctrl_interface_group=0
eapol_version=1
ap_scan=0
fast_reauth=1

network={
    eapol_flags=0
    key_mgmt=IEEE8021X
    eap=PEAP
    identity="username"
    password="password"
    ca_cert="/path/to/certificate.crt"
    phase1="auth=PEAP"
    phase2="auth=MSCHAPV2"
}
```
- Set up your network interfaces to automatically use this config file: `/etc/network/interfaces`. If using wired connection, make sure to include the `wpa-driver` line!
```
# ...
iface eth0 inet manual
    wpa-driver wired
    wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf

allow-hotplug wlan0
iface wlan0 inet manual
    wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf

# ...
```

## Ease of use
- Set up handy aliases to switch between read-only and writable fs, including a prompt help (thanks [[1]](#references)):
```
# set variable identifying the filesystem you work in (used in the prompt below)
fs_mode=$(mount | sed -n -e "s/^.* on \/ .*(\(r[w|o]\).*/\1/p")
# alias ro/rw 
alias ro='mount -o remount,ro / ; fs_mode=$(mount | sed -n -e "s/^.* on \/ .*(\(r[w|o]\).*/\1/p")'
alias rw='mount -o remount,rw / ; fs_mode=$(mount | sed -n -e "s/^.* on \/ .*(\(r[w|o]\).*/\1/p")'
# setup fancy prompt
export PS1='\[\033[01;32m\]\u@\h${fs_mode:+($fs_mode)}\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ '
# aliases for mounting boot volume
alias roboot='mount -o remount,ro /boot'
alias rwboot='mount -o remount,rw /boot'
```
# References 
[1] Steps mostly adapted from [this blogpost](http://petr.io/en/blog/2015/11/09/read-only-raspberry-pi-with-jessie/).

[2] http://alexba.in/blog/2013/01/07/use-your-raspberrypi-to-power-a-company-dashboard/
