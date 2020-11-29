---
layout: post
title: Downloading Apple Silicon Mac IPSWs
date: 2020-11-28 18:07 -0600
---

Admins who have worked with non-Mac Apple devices for a long time are already familiar with IPSW ([iPod software](https://en.wikipedia.org/wiki/IPSW)) files. IPSWs are the OS installers for iOS, iPadOS, tvOS, and other variations in the iDevice family. You may have used an IPSW in the past without knowing it. When restoring a device through iTunes/Finder or Apple Configurator 2 (AC 2), an IPSW is downloaded to reinstall the OS as part of that process. 

![IPSW download](/images/ac2_ipsw_download.png)

The same is true for Apple Silicon Macs. They can be booted to [DFU and the OS restored through AC 2.](https://support.apple.com/guide/apple-configurator-2/revive-or-restore-a-mac-with-apple-silicon-apdd5f3c75ad/mac) The process can be especially fidgety, but the steps on [T2](https://mrmacintosh.com/how-to-restore-bridgeos-on-a-t2-mac-how-to-put-a-mac-into-dfu-mode/) and [Apple Silicon Macs](https://mrmacintosh.com/restore-macos-firmware-on-an-apple-silicon-mac-boot-to-dfu-mode/) are well documented by Mr. Macintosh. I recommend reading those articles on how to boot to DFU since the ports, cables, and precise timing are all important pieces to making that particular magic work. This article is primarily about how to download and distribute those Apple Silicon IPSWs to more easily restore a Mac through AC 2. As of writing this there are two ways to kick off the restore/revive process.

1. Select restore/revive in AC 2 once a Mac is booted into DFU. The IPSW file will be downloaded directly from Apple's CDN.
2. Take an already downloaded IPSW and drag/drop it onto a DFU booted Mac within AC 2. 

The main reason someone might want to use option 2 is the latest Big Sur 11.0.1 IPSW comes in at a whopping 13.6 GB. To have to download 13.6 GB on a regular basis across multiple technician Macs or other scenarios can be a problem depending on networking constraints. It also doesn't appear IPSW files are part of [the content types supported by content caching](https://support.apple.com/en-us/HT204675), but if someone has information to the contrary please let me know. And lastly, although this is becoming less common, older IPSWs can be used to restore a different OS version. Say 11.2 is released with a breaking change. If a Mac needs to be restored through AC 2 it will by default get 11.2 installed as that's the latest supported release. With a 11.1 IPSW on hand 11.2 can be avoided until the issue is fixed. The IPSW _must_ be signed by Apple. It used to be they'd keep multiple IPSWs signed - 14.1 and 14.2 on iOS for example - for a few weeks, but that's rarely the case anymore.

# Apple Configurator 2 Cache
AC 2 keeps a IPSW cache at `~/Library/Group Containers/K36BKF7T3D.group.com.apple.configurator/Library/Caches/Firmware`. Everytime a new iDevice is restored, AC 2 check its cache to see if the latest, signed IPSW is already available at that location. If it is, no need to download a new IPSW, use the existing one. That IPSW can be then be packaged or archived to be shared with others. Boot the problem Mac to DFU, drag/drop the IPSW onto the device in AC 

![IPSW download](/images/ac2_firmware_cache.png)

As part of the restore process the IPSW is unpacked to a temporary directory like `/System/Volumes/Data/private/var/folders/yf/5l1s2jzx1qlb1_bpszkjjk400000gp/T/com.apple.configurator.xpc.DeviceService/TemporaryItems/82BB4EE1-4BA6-4509-89E2-9DAFE1512A76/Firmware`. If you do a restore/revive, delete IPSWs in `~/Library/Group Containers/K36BKF7T3D.group.com.apple.configurator/Library/Caches/Firmware` and then restore again, IPSWs most likely will not be downloaded again to the cache directory because the required files already exist in `/private/var`. To get usable IPSWs in the cache again you may need to go through the process on a different Mac.

# Apple's Software Update Feed
AC 2 references [https://mesu.apple.com/assets/macos/com_apple_macOSIPSW/com_apple_macOSIPSW.xml](https://mesu.apple.com/assets/macos/com_apple_macOSIPSW/com_apple_macOSIPSW.xml) directly for IPSW download URL data. This XML is public information. You're free to copy the `FirmwareURL` string for your particular model and download the IPSW just like AC 2 does. The good news here so far is IPSWs for Apple Silicon Macs are universal. The same IPSW is used across all models, available here - [http://updates-http.cdn-apple.com/2020FallFCS/fullrestores/001-79706/93A4C43C-2B45-47AF-84EA-794F8886B85C/UniversalMac_11.0.1_20B29_Restore.ipsw](http://updates-http.cdn-apple.com/2020FallFCS/fullrestores/001-79706/93A4C43C-2B45-47AF-84EA-794F8886B85C/UniversalMac_11.0.1_20B29_Restore.ipsw).

With more Mac hardware on the way who knows how long IPSWs will stay unified. For now it's nice to only have to track one. The downside to using the XML feed is on each macOS release it would have to be checked again manually, or some code written to check for new data on a regular basis.

# ipsw.me
If you're not interested in fiddling about with an "enterprise" tool like AC 2 or parsing XML, let [ipsw.me](https://ipsw.me) do the work for you. Their service [tracks new IPSWs on each release](https://twitter.com/iOSReleases) and [https://twitter.com/tssstatus](the signing status of existing versions). Remember if a IPSW is no longer being signed it's pretty much useless. It can't be used to restore/revive a Mac with the drag/drop method.

This is the method I recommend for most people. Go to [https://ipsw.me/product/Mac](https://ipsw.me/product/Mac), select your Mac model (all M1 IPSWs are unified right now), and download the IPSW. Downloads from ipsw.me are using the same Apple URLs parsed from Apple's software update feed. They aren't hosted or redistributed so you can be assured you're getting real, genuine Apple generated IPSWs.

![IPSW download](/images/ipsw-me.png)
