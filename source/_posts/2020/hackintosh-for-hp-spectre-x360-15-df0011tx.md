---
title: Hackintosh For HP Spectre x360 Late 2018
date: 2020-11-24 09:00:00
categories:
 - Tech
---
<div class="banner-img">
    <img src="/images/2020/hackintosh-banner.png">
</div>

<!--more-->

EFI download:
[https://github.com/Magentaize/HP-Spectre-X360-15-Late-2018-Hackintosh](https://github.com/Magentaize/HP-Spectre-X360-15-Late-2018-Hackintosh)

# Information
* macOS: 11.15.7 Catalina
* Model: HP Spectre x360 15-df0011tx
* CPU: Intel Core i7-8750H
* iGPU: Intel Graphics UHD 630
* RAM: 16GB
* Storage: TOSHIBA XG5 KXG50ZNV1T02 NVMe
* Audio: Realtek ALC285
* USB: USB3.1 Gen2 x 1, Thunderbolt 3 x 2
* WiFi: Intel Wireless AC 9560 160MHz
* Bluetooth: VID 8087 PID 0AAA USB
* Trackpad: SYNA329A
* Touchpad: ELAN2514

## What works
* Keyboard
* Trackpad(I2C)
* Touchpad(I2C)
* Built-in four speakers
* Headphone
* iGPU
* USB 2.0
* Battery
* Wi-Fi
* Bluetooth
* Lid
* Screen brightness adjustment
* App Store
* iCloud

## What doesn't work
* Built-in microphone
* Built-in camera
* Fingerprint

## What doesn't confirm
* Touch pen
* Thunderbolt 3
* USB3.1
* Type-C to HDMI
* microSD card reader
* macOS update

# Guide
## Installation image
Go to [https://blog.daliansky.net/WeChat-First-macOS-Catalina-10.15.7-19H15-official-version-Clover-5126-OC-WEPE-supports-both-INTEL-and-AMD-original-images.html](https://blog.daliansky.net/WeChat-First-macOS-Catalina-10.15.7-19H15-official-version-Clover-5126-OC-WEPE-supports-both-INTEL-and-AMD-original-images.html).
Boot to Clover and begin installation progress.

Notice:
The provided EFI files will lead to kernel panic when booting macOS Recovery Tool, you can use this EFI after installation.

## Stuck at booting Darwin
Try to boot OpenCore, press space and choose "reset nvram".

## Installation stuck at "About 2 minutes remaining"
If you have format volume to APFS before, just restart manually.

## Wireless and bluetooth
Add itlwm.kext, IntelBluetoothFirmware.kext, IntelBluetoothInjector.kext and install HeliPort.

## SMBIOS
Requirements:
* Clover Configurator

Choose MacBookPro15,1 and fill.

## iGPU
Requirements:
* Hackintool
* Clover Configurator

First open Hackintool, find `Platform Id` by `Device Id`.
<img src="/images/2020/hackintosh-igpu-0.jpg"
style="width: 50%;">

Configure Clover config.plist:
<img src="/images/2020/hackintosh-igpu-1.jpg"
style="width: 50%;">

<img src="/images/2020/hackintosh-igpu-2.jpg">

Generate Patch in Hackintool:
<img src="/images/2020/hackintosh-igpu-3.jpg"
style="width: 50%;">

<img src="/images/2020/hackintosh-igpu-4.jpg"
style="width: 50%;">

DO NOT USE File > Export > Bootload config.plist to apply patch, it will break config.plist!
You should apply patch manually by editing config.plist using TextEdit.
This is my patch:
> 			<key>PciRoot(0x0)/Pci(0x2,0x0)</key>
			<dict>
				<key>AAPL,ig-platform-id</key>
				<data>
				BgCbPg==
				</data>
				<key>AAPL,slot-name</key>
				<string>Internal@0,2,0</string>
				<key>device-id</key>
				<data>
				mz4AAA==
				</data>
				<key>device_type</key>
				<string>VGA compatible controller</string>
				<key>disable-external-gpu</key>
				<data>
				AQAAAA==
				</data>
				<key>enable-hdmi20</key>
				<data>
				AQAAAA==
				</data>
				<key>framebuffer-fbmem</key>
				<data>
				AACQAA==
				</data>
				<key>framebuffer-patch-enable</key>
				<data>
				AQAAAA==
				</data>
				<key>framebuffer-stolenmem</key>
				<data>
				AAAwAQ==
				</data>
				<key>framebuffer-unifiedmem</key>
				<data>
				AAAAgA==
				</data>
				<key>model</key>
				<string>Intel UHD Graphics 630 (Mobile)</string>
			</dict>

<img src="/images/2020/hackintosh-igpu-5.jpg"
style="width: 80%;">

## Audio
Requirements:
* Hackintool
* Loopback

ALC285 cannot be enabled by Clover audio injection, so you need to create a patch using Hackintool.
For AppleALC\@1.5.5 using layout-id=21 can only enable the front two speakers, so I find a modified AppleALC.kext which can enable all four speakers. You can find it here [https://github.com/jpuxdev/HP-Spectre-X360-13-Early-2019-Hackintosh](https://github.com/jpuxdev/HP-Spectre-X360-13-Early-2019-Hackintosh). Thanks to jpuxdev!

Generate Patch in Hackintool and apply it manually:
<img src="/images/2020/hackintosh-audio-0.jpg"
style="width: 50%;">

<img src="/images/2020/hackintosh-audio-1.jpg"
style="width: 50%;">

This is my patch:
> 			<key>PciRoot(0x0)/Pci(0x1F,0x3)</key>
			<dict>
				<key>AAPL,slot-name</key>
				<string>Internal@0,31,3</string>
				<key>device-id</key>
				<data>
				cKEAAA==
				</data>
				<key>device_type</key>
				<string>Multimedia audio controller</string>
				<key>layout-id</key>
				<data>
				RwAAAA==
				</data>
				<key>model</key>
				<string>Cannon Lake PCH cAVS</string>
			</dict>

Once 2(groups) speakers can be found in Sound, install Loopback to enable them together:
<img src="/images/2020/hackintosh-audio-2.jpg">

Due to that built-in microphone cannot work, I Choose my iPhone as external microphone.

## DSDT(battery, I2C, brightness)
Requirements:
* Clover
* MaciASL

Use Clover to dump your DSDT and open `DSDT.aml` in macOS.
Apply three patches:
* [bat]HP_Spectre_x360_apxxxx.txt
* [I2C]HP_Spectre_x360_apxxxx.txt
* [brightness_key]HP_Spectre_x360_apxxxx.txt
There will be a few compile errors, but you can fixes theme easilly.

## Trackpad and touchpad
Add VoodooI2C.kext, VoodooI2CHID.kext, VoodooI2CSynaptics.kext.

## Stuck at shotdown and restart
Add SSDT-PMC.aml.

## Stuck at login after typing password
Add NoTouchID.kext.

## Built-in keyboard
Only add or keep VoodooPS2Controller.kext.

## USB
All USB2.0 and USB3.0 devices works and my device doesn't contain any SS USB port.

## Power
Add SSDT-PLUG-DRTNIA.aml to enable options in Energy Saver.
Add boot argument "igfxrpsc=1" to improve iGPU performance.
DO NOT USE CPUFriend.kext!

## PrtSc disables trackpad
You can compile a SSDT to disable PrtSc key, more info here: [https://www.tonymacx86.com/threads/prt-sc-disabling-trackpad.235242/](https://www.tonymacx86.com/threads/prt-sc-disabling-trackpad.235242/).

## Boot from internal storage
According to [https://www.tonymacx86.com/threads/hp-spectre-x360-15-regular-clover-install-or-use-preloader-efi.280104/](https://www.tonymacx86.com/threads/hp-spectre-x360-15-regular-clover-install-or-use-preloader-efi.280104/), you need add preloader.efi and rename cloverx64.efi to loader.efi.

## Size of Apple logo changes when booting
Add the following ROOT node in config.plist:
> 	<key>BootGraphics</key>
  	  <dict>
  	      <key>#DefaultBackgroundColor</key>
  	      <string>0xF0F0F0</string>
  	      <key>UIScale</key>
  	      <integer>2</integer>
  	      <key>EFILoginHiDPI</key>
  	      <integer>1</integer>
  	  </dict>

## nvram
With my device, nvram is readable and wirtable but cannot clear. I don't know how to fix it.

## Cleanup
Requirements:
* Clover Configurator

Remove useless ACPI patches:
<img src="/images/2020/hackintosh-cleanup-0.jpg">