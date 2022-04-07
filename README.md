# PSAS Client

These files combined with OpenWrt were used to build a TP-Link TL-WDR3600 v1 device that will allow ethernet-only devices to connect
to the internet via the "PSU Guest" wifi.

## Motivation

Some devices under development need access to the internet. These devices sometimes do not have a wifi interface, and even if they do,
they don't have a browser to negotiate the captive portal on the guest network. Portland State University does not provide usable ethernet jacks. Yes (checks watch), it is the 21st Century.

## Solution

Use a wireless router with custom software to connect "backwards". That is, with one of the wifi interfaces in station-mode instead of
ap-mode as the WAN interface and the ethernet ports (including WAN port) as part of the LAN network, utilizing the normal NAT'ing
function, allows all of the connected devices to appear to the upstream network as one device.

The WDR3600 also has two USB host ports. Many common USB-ethernet dongles can be plugged into these USB ports for up to two additional
ethernet ports. Jumpering to an ethernet switch could also be used to expand the number of ethernet ports available. Currently, the LAN
is limited to a class C (/24) network with ~150 DHCP leases available, but can be reconfigured.

It is still necessary for someone to "authorize" the wireless router with the "PSU Guest" captive portal. Any browser-equipped computer,
connecting through the device, will tell the upstream network that, yes, you agree to the terms and conditions. Thereafter, any attached
device will be able to reach the internet. The negotiation of the captive portal will be needed as often as the upstream network decides,
usually this is once per connection or once per day. Your mileage may vary.

## How To Use:

 - Plug in the WDR3600 to a 12V 1A power supply;
 - If no LEDs light, try the power button near the power supply connector;
 - Connect a laptop or other browser-equipped computer to an ethernet port on the WDR3600;
 - Disconnect your computer from any wifi network to force your connection through the WDR3600;
 - Your computer should get a DHCP lease;
 - If your computer has a captive-portal detection system, you might be automatically redirected to the PSU Guest captive portal.
   If not, go to any non-https website, [such as http://www.gstatic.com/generate_204](http://www.gstatic.com/generate_204), and 
   you should be directed to the PSU Guest captive portal. Agree, on behalf of yourself and anyone else who will connect to the
   WDR3600, to the terms and conditions;
 - Connect your device(s) needing internet access to an ethernet port on the WDR3600;
 - Check to see if it can reach the internet: `ping 4.2.2.2` (connectivity) and `ping google.com` (DNS)
 - Yay, your device is connected to the internet!

## Rebuilding

It might be necessary to reproduce the firmware, to support a different router, or to add support for a device or some other feature. 
The files in this repository are provided to help you do that, if needed. These are the files that were used to build the original version.

To rebuild, you will need a computer capable of building OpenWrt. 
See [here](https://openwrt.org/docs/guide-developer/toolchain/install-buildsystem).

Generally, follow the instructions [here](https://openwrt.org/docs/guide-developer/toolchain/use-buildsystem).

The original version of this system was built using r19336 (git hash 483fe539c4), and a feeds.conf equivalent to:

```
src-git packages https://git.openwrt.org/feed/packages.git^22d202e3a
src-git luci https://git.openwrt.org/project/luci.git^5ef73b625
src-git routing https://git.openwrt.org/feed/routing.git^5702d2e
src-git telephony https://git.openwrt.org/feed/telephony.git^24acd46
```

After cloning openwrt.git, with $TOPDIR representing the top of the openwrt checkout:

 - change directories to the psas-client checkout
 - copy config.diff to .config in the openwrt git tree, e.g.: `cp config.diff $TOPDIR/.config`
 - copy the full files tree to the openwrt git tree, e.g.: `cp -a files $TOPDIR/files`
 - change directories to the openwrt git tree, e.g.: `cd $TOPDIR`
 - generate a full .config from the stub provided: `make defconfig`
 - if you want to modify the build, use: `make menuconfig`
 - build the firmware: `time make -j$(nproc) BUILD_LOG=1 IGNORE_ERRORS=m V=s`
 - wait, as the build system does its thing
 - find the resulting firmware in: `$TOPDIR/bin/targets/ath79/generic/openwrt-ath79-generic-tplink_tl-wdr3600-v1-squashfs-sysupgrade.bin`

Use one of the several methods of copying the firmware to the device and use it to upgrade:

 - scp the firmware to the /tmp/ directory of the device, then from an ssh or serial session on the device, 
   run: `sysupgrade -v -n /tmp/openwrt-ath79-generic-tplink_tl-wdr3600-v1-squashfs-sysupgrade.bin`
 - use the luci web admin interface
 - learn how to reflash using TFTP, see details [here](https://openwrt.org/toh/tp-link/tl-wdr3600_v1).

## Gotchas

 - Only one wifi radio (radio1) is enabled in this firmware, the 5GHz radio, and it is in station-mode. If you want to switch to a 
   2.4GHz network, set radio0 to channel "auto" and change the wan wifi-iface to use radio0.
 - The root password is empty (just press enter when prompted), you will be encouraged to set one. If you do, write it on a stickie note
   with the date and attach it to the router. If you try and the password doesn't work, try the reset instructions, below, to go back to
   an empty password.
 - You will only be able to ssh or reach the Luci interface from the LAN network (there is a commented out section in /etc/config/firewall
   to allow ssh from WAN, but probaby not useful)
 - If you lock yourself out, or otherwise screw something up, you can reset to the starting configuration by letting the device boot fully,
   and then pressing and holding the "reset" button for >5 seconds. It will "Factory reset" to the default configuration when you release
   the button.
 - If it is super not working, and "factory reset" procedure above doesn't seem to help, don't chuck it in the bin, contact 
   Russell (on Slack or github) and he'll maybe help you fix it.

## Nitty-Gritty details

 - the "files" tree that you copy into the build system is used to overlay the default root filesystem when it is being created.
 - the most important part of the files tree is in files/etc/uci-defaults/, these scripts run on each boot, but if they exit with 
   status 0, they are deleted and therefore don't run again.
 - files/etc/uci-defaults/psas.00.defaults set up the magical configuration.
 - files/etc/uci-defaults/psas.zzz.defaults records the initial configuration to persistent storage
 - files/usr/bin/since-last-flash.sh provides a script that will report changes to persistent storage and configuration since the 
   time of flashing (particularly useful if there are manual changes you would like to bake in to a new build).
 - if your uci-defaults scripts exit with an error, they'll run again on every boot. try not to have errors in your uci-defaults scripts!
 - OpenWrt uses an overlay filesystem, a readonly squashfs and a read/write filesystem (jffs2 in this case) on top.
 - the jffs2 filesystem is formatted on first boot and then mounted "over" the squashfs.
 - the original squashfs filesystem remains accessible in the /rom tree.
 - most transitory files are held in tmpfs filesystems, persistent files are stored in the jffs2 overlay.
 - a "factory reset" amounts to erasing the jffs2 and returning to a first boot state.
