# docs

Get ready for a lot of bad JavaScript, russian documentation and weird looking tools. 

![Modems](https://github.com/balong-toolkit/docs/raw/master/images/modems.jpg)

## Table of contents

<!-- vim-markdown-toc GFM -->

* [Getting access to modem](#getting-access-to-modem)
* [Enabling dev mode](#enabling-dev-mode)
* [Working with ROM](#working-with-rom)
	* [Existing tools](#existing-tools)
		* [Balong flash](#balong-flash)
	* [Flashing ROM](#flashing-rom)
	* [Getting partition info](#getting-partition-info)
	* [Extracting ROM](#extracting-rom)
* [Other resources](#other-resources)
* [Credits](#credits)

<!-- vim-markdown-toc -->

## Getting access to modem

I am going to assume you already have your modem unlocked. 

First thing you need to do is to enable [dev mode](#enabling-dev-mode).

After that, what we need to do is to set OEM password (unless you know what it is and do not want to change it)

Connect to `AT` command serial like this: `screen /dev/ttyUSB0` and then, send this command: 

```
at^sethwlock="OEM",00000000
```

Now enable UEAP prompt using this command: 

```
AT^NVWREX=33,0,4,2,0,0,0
```

Now, restart it using this command: 

```
at^reset
```

And after that put it back into dev mode. 

Now, look for serial device that is printing out messages like these: 

```
[000119311ms] U_ACM:(U_ERROR)acm_setup():acm ttyGS0 req21.22 v0003 i0004 l0
[000119318ms] U_ACM:(U_ERROR)acm_setup():acm ttyGS0 req21.22 v0003 i0004 l0
```

On that device, press enter, and you will be prompted for password. It is `00000000` if you set it in this step. 

Now, you will get `EUAP>` prompt and now you can either start `/bin/sh` or start telnet server `busybox telnetd -l /bin/sh`.

[Credits go to rust3028 from 4pda](https://4pda.ru/forum/index.php?s=&showtopic=582284&view=findpost&p=37475499)

## Enabling dev mode

First, create `sw_debug_mode.xml` with this content:

```xml
<?xml version="1.0" encoding="UTF-8" ?> 
<api version="1.0">
  <header>
    <function>switchMode</function>
  </header>
  <body>
    <request>
      <switchType>1</switchType> 
    </request>
  </body>
</api>
```

Afther that, issue this command: 

```bash
timeout 3 curl -X POST -d @sw_debug_mode.xml http://192.168.8.1/CGI
```

Now, it should go to dev mode. 

## Working with ROM

This section is about understanding ROM, updates, extracting, packing ROMs and flashing them. 

### Existing tools

* [Balong flash](https://github.com/forth32/balongflash)
* [Balong USBLoad](https://github.com/forth32/balong-usbdload)

#### Balong flash

Balong flash is toolkit for flashing ROMs to balong hardware. 

It can be used for parsing ROM info and getting more info on ROMs. 

Translated CLI help:

```
The utility is designed for flashing modems on the Balong V7 chipset.

balongflash [keys] <file name to load or the name of the file directory>

 The following keys are valid:

-p <tty> - serial port for communication with the bootloader (default / dev / ttyUSB0)
-n - multifile firmware mode from the specified directory
-g # - set the digital signature mode
  -gl - parameters description
  -gd - disable signature auto-detection
-m - output the firmware file map and exit
-e - parse the firmware file into sections without headers
-s - parse the firmware file into sections with headers
-k - do not restart the modem at the end of the firmware
-r - force reboot the modem without flashing partitions
-f - flash even if there are CRC errors in the source file
-d # - set the type of firmware (DLOAD_ID, 0..7), -dl - list of types
```

### Flashing ROM

`/dev/ttyUSB0` is serial device from balong device, in "flash" mode. 

`E3372h-153_UPDATE_22.315.01.00.00.BIN` is binary update file. 

```bash
sudo balongflash -p /dev/ttyUSB0 ./E3372h-153_UPDATE_22.315.01.00.00.BIN
```

### Getting partition info

`E3372h-153_UPDATE_22.315.01.00.00.BIN` is binary update file. 

```bash
balongflash -m E3372h-153_UPDATE_22.315.01.00.00.BIN
```

Output: 

```
 Программа для прошивки устройств на Balong-чипсете, V3.0.280, (c) forth32, 2015, GNU GPLv3
--------------------------------------------------------------------------------------------------

 Код файла прошивки: 9 (ONLY_FW)
                                 
 Цифровая подпись: 2958 байт
 Хеш открытого ключа: 778A8D175E602B7B779D9E05C330B5279B0661BF2EED99A20445B366D63DD697
 Версия прошивки: 22.315.01.00.00
 Платформа:       BV7R11HS
 Дата сборки:     2015.11.27 11:20:25
 Заголовок: версия 1, код соответствия: HWEW11.1
 Выделение разделов из файла прошивки:

 ## Смещение  Размер   Имя
-------------------------------------
 00 0000005c   224486  Fastboot
 01 00036e14     4530  M3Boot_R11
 02 0003802c     2048  M3Boot-ptable
 03 00038890  5681280  Kernel_R11
 04 005a444c  8645335  VxWorks_R11
 05 00de4004    45732  M3Image_R11
 06 00def324  2380084  DSP_R11
 07 01034948  1569746  Nvdload_R11
 08 011b407c  7420928  System
 09 018c8b08  2649600  APP
```

### Extracting ROM

First you need to [get partition details](#getting-partition-info) using balongflash. 

Let's say we want to extract APP partition with details like these: 

```
 09 018c8b08  2649600  APP
```

Second number is in hex format and needs to be converted to decimal (in this case, it is 25987848). It is offset of where ROM is in update file. 

Third number is size of that partition. 

First we need to separate that partition from the rest of the ROM. 

We can use dd for this. 

```bash
dd if=E3372h-153_UPDATE_22.315.01.00.00.BIN of=system skip=25987848 count=2649600 bs=1 status=progress
```

That leaves us with `cpio` filesystem. 

Easiest way you can extract it is using binwalk. 

```bash
binwalk -evP app
```

## Other resources

* [4pda topic for E3372h](https://4pda.ru/forum/index.php?showtopic=582284)

## Credits

* [Flocksocial team](https://flocksocial.io)

