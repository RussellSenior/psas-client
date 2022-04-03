# PSAS Client

These files combined with OpenWrt were used to build a TP-Link TL-WDR3600 v1 device that will allow ethernet only devices to connect to the internet via the "PSU Guest" wifi.

## Motivation

Some devices under development need access to the internet. These devices sometimes do not have a wifi interface, and even if they do, they don't have browser to negotiate 
the captive portal on the guest network. Portland State University does not provide usable ethernet jacks. Yes, it is the 21st Century.

## Solution

Use a wireless router with custom software to connect "backwards". That is, with one of the wifi interfaces in station-mode instead of ap-mode as the WAN interface
and the ethernet ports (including WAN port) as part of the LAN network, utilizing the normal NAT'ing function, allows all of the connected devices to appear to the
upstream network as one device.

It is still necessary for someone to "authorize" the wireless router. Any browser-equipped computer, connecting through the device, will tell the upstream network that, yes,
you agree to the terms and conditions. Thereafter, any attached device will be able to reach the internet. The negotiation of the captive portal will be needed as often as 
the upstream network decides, usually this is once per connection or once per day. Your mileage may vary.

## Rebuilding

It might be necessary to reproduce the firmware, to support a different router, or to add support a device or some other feature. These files are provided to help you do that, 
if needed. These are the files that were used to build the original version.

To rebuild, you will need a computer capable of building OpenWrt. See [here](https://openwrt.org/docs/guide-developer/toolchain/install-buildsystem).

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

 - scp the firmware to the /tmp/ directory of the device, then run `sysupgrade -v -n /tmp/openwrt-ath79-generic-tplink_tl-wdr3600-v1-squashfs-sysupgrade.bin`
 - use the luci web admin interface
 - learn how to reflash using TFTP, see details [here](https://openwrt.org/toh/tp-link/tl-wdr3600_v1).

## Gotchas

 - Only one wifi radio is enabled in this firmware, the 5GHz radio
 - The root (and admin) password is empty (just press enter), you will be encouraged to set one.
 - You will only be able to ssh or reach the Luci interface from the LAN network.
 - If you lock yourself out, or otherwise screw something up, you can reset to the starting configuration, by letting the device boot fully, and then pressing and holding the "reset" button for >5 seconds. 
   It will "Factory reset" to the default configuration when you release the button.

## Nitty-Gritty details

 - the "files" tree that you copy into the build system is used to overlay the default root filesystem when it is being created.
 - the most important part of the files tree is in files/etc/uci-defaults/, these scripts run on each boot, but if they exit with status 0, they are deleted and therefore don't run again
 - if your uci-defaults scripts exit with an error, they'll run again on every boot. try not to have errors in your uci-defaults scripts!
 - OpenWrt uses an overlay filesystem, a readonly squashfs and a read/write filesystem (jffs2 in this case) on top
 - the jffs2 filesystem is formatted on first boot and then mounted "over" the squashfs.
 - the original squashfs filesystem remains accessible in the /rom tree.
 - most transitory files are held in tmpfs filesystems, persistent files are stored in the jffs2 overlay.
 - a "factory reset" amounts to erasing the jffs2 and returning to a first boot state.
