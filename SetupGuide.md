
# Bootloader

Chips come blank and need to be programmed via ISP pins until a bootloader is put on.
This can be done with various hardware probes (J-Link, ATMEL-ICE), or a raspberry pi.

## Raspberry pi and OpenOCD

OpenOCD can use a pi's GPIO pins but needs to be specifically compiled with the "--enable-sysfsgpio --enable-bcm2835gpio" flags (the package from apt may or may not have this flag but seems to stay quite far behind anyway)

to compile openOCD install the following dependencies

```bash
sudo apt-get update
sudo apt-get install git autoconf libtool make pkg-config libusb-1.0-0 libusb-1.0-0-dev
```

then download the source code, configure, make and install.  

```bash
git clone http://openocd.zylin.com/openocd
cd openocd
./bootstrap
./configure --enable-sysfsgpio --enable-bcm2835gpio
make
sudo make install
```

check `/usr/local/share/openocd/scripts/interface/raspberrypi2-native.cfg` for the pin outs, it lists the native GPIO numbers not the renumbered pi pin numbers. and <https://www.raspberrypi-spy.co.uk/2012/06/simple-guide-to-the-rpi-gpio-header-and-pins/> for physical reference

## Find bootloader

Download your bootloader file for your chip. For our generic use of the SAMD21J21A chip i found this repository to have the most reliable one, specifically build for the chip.
<https://github.com/mattairtech/ArduinoCore-samd/tree/master/bootloaders/zero/binaries>

```bash
cd ~
mkdir bootloader
cd bootloader
#wget ${Bootloader.bin}
wget https://github.com/mattairtech/ArduinoCore-samd/blob/master/bootloaders/zero/binaries/sam_ba_Generic_x21J_SAMD21J17A.bin 
# or download and move the bootloader file here manually
```

## Create OpenOCD config

```bash
cd ~/bootloader
nano openocd.cfg
```

```bash
source [find interface/raspberrypi2-native.cfg]
transport select swd
 
#set CHIPNAME ${yourChipName} 
set CHIPNAME at91samd21j17
source [find target/at91samdXX.cfg]
 
#reset_config srst_only
#reset_config  srst_nogate #Sometimes you can uncomment this but SAMD21J17 works fine without
 
#adapter srst delay 100 # Docs say to do this but it usually breaks things for me
#adapter srst pulse_width 100
 
init
targets
reset halt

#at91samd bootloader 0 # this is apparently buggy and can set the fuses wrong which causes some weirdness (rebooting)
#program ${Bootloader.bin} verify 
program sam_ba_Generic_x21J_SAMD21J17A.bin verify
#at91samd bootloader 8192 # not strictly necessary just protects the bootloader from being overwritten by openOCD without removing the protection first 
reset
shutdown

```

running `sudo openocd` in the same directory will run the config file as a script and attempt to flash the bootloader. If it succeeds you will see `**Verified OK**`

Note it generally leaves the chip in a halted state, requiring a manual reset/power cycle.

I've also seen it fail a few times, if there isn't an obvious reason just retry a few times, and maybe hit the reset button. Latest flashing required running just `at91samd bootloader 0`, power cycle then `program sam_ba_Generic_x21J_SAMD21J17A.bin verify` and `at91samd bootloader 8192`, then power cycle again before trying to load a program. Running the following "Fixing Fuses" to be safe.

## Fixing Fuses

If you ever run the `at91samd bootloader 0` command there is a chance the fuses could have been set incorrectly. In order to fix this there is a script created by the [uf2 bootloader team](https://github.com/adafruit/uf2-samdx1#samd21:~:text=Fuses-,SAMD21,-The%20SAMD21%20supports), but in order to use it it requires some modifications. Most significantly it looks for openOCD installed by Arduino, which if you have it will not have been compiled with the flags to allow native pi usage (also the script usually references the wrong file path anyway as Arduino changes). So clone the repo and we will make some modifications. It also requires nodejs installed in order to run so install that to.

```bash
sudo apt-get install nodejs
git clone https://github.com/adafruit/uf2-samdx1.git

cd uf2-samdx1/scripts
nano dbgtool.js
```

Find and delete, or comment out, this whole section:

```javascript
  let dirs = [
        process.env["HOME"] + "/Library/Arduino15",
        process.env["USERPROFILE"] + "/AppData/Local/Arduino15",
        process.env["HOME"] + "/.arduino15",
    ]

    let pkgDir = ""

    for (let d of dirs) {
        pkgDir = d + "/packages/arduino/"
        if (fs.existsSync(pkgDir)) break
        pkgDir = ""
    }

    if (!pkgDir) fatal("cannot find Arduino packages directory")
```

Next find ``` let openocdPath = ``` just under that section and change it to your compiled file path, likely `"/usr/local/"`

finally find the args assignment and replace it with:

```javascript
  let args = ["-d2",
        "-s", openocdPath + "/share/openocd/scripts/",
        "-f", "interface/raspberrypi2-native.cfg",
        "-f", "target/at91samdXX.cfg",
        "-c", cmd]
```

unfortunately we also need to modify the `target/at91samdXX.cfg` file, so `sudo nano /usr/local/share/openocd/scripts/target/at91samdXX.cfg` then at the very top of the file add `transport select swd`, and just after `source [find target/swj-dp.tcl]` add `set CHIPNAME at91samd21j17` (use your chipname if using something else)

And now alterations are done, the script should be able to run by invoking `./scripts/dbgtool.js fuses` from the uf2-samdx1 repo directory

# Programming the chip

Now that the bootloader and fuses are configured the chip should be ready to be programmed. With the SAM-BA bootloader we used a **double tap** of the **reset button** is required to enter into Bootloader mode, creating the com port that the program can be sent to. Pi's also frequently have undervoltage issues, the sudden powering of boards can trigger sudden drops that can damage the SD card. To avoid this **power the board before connecting the programming pins**.

## Arduino custom board

As this chip is a variant of the chip used for the `feather m0` we can use the Arduino library's and IDE with a couple file modifications to adjust the board definitions.

Find your Arduino hardware folder usually located in some variant of `C:\Users\User\AppData\Local\Arduino15\packages\adafruit\hardware\`

Its possible we also need to add the `-D__SAMD21J17A__` build flag to `adafruit_feather_m0.build.extra_flags=` in the `.\samd\1.7.10\boards.txt` file.

We definitely need to modify the memory layout the linker uses so edit `.\samd\1.7.10\variants\feather_m0\linker_scripts\gcc\flash_with_bootloader.ld` and change

```LinkerScript
MEMORY
{
  FLASH (rx) : ORIGIN = 0x00000000+0x2000, LENGTH = 0x00040000-0x2000 /* First 8KB used by bootloader */
  RAM (rwx) : ORIGIN = 0x20000000, LENGTH = 0x00008000
}
```

to

```LinkerScript
MEMORY
{
  FLASH (rx) : ORIGIN = 0x00000000+0x2000, LENGTH = 0x00020000-0x2000 /* First 8KB used by bootloader */
  RAM (rwx) : ORIGIN = 0x20000000, LENGTH = 0x00004000
}
```

This is because the SAMD21J17 has half the memory and ram of the SAMD21G18 used for the feather m0.

The last hidden file we need to change is `C:\Users\User\AppData\Local\Arduino15\packages\adafruit\hardware\samd\1.7.10\cores\arduino\startup.c` where we need to add a `#define CRYSTALLESS true` to the top, as the board we are using doesn't have an external crystal clock.

Now of course the way i'm changing files here will alter all boards that reference these config files, its not a problem for me as we aren't going to be using any of the Adafruit boards, but if you are you should set it up as a proper new board rather than piggy backing on the feather m0 files, I don't know how to do that at the moment so you'll have to look elsewhere.

## PlatformIO custom board

PlatformIO is my favorite way of programming for micro-controllers as its quite flexible, however it also requires some similar setup for the custom board. Same change is required for the linker file, for platformIO it is located `C:\Users\User\.platformio\packages\framework-arduino-samd-adafruit\variants\feather_m0\linker_scripts\gcc\flash_with_bootloader.ld`.

As for the `CRYSTALLESS` define we could edit the equivalent platformIO file, however instead we can just add a build flag to the projects `platformio.ini` file. Under you board specific environment simply add:

```ini
build_flags =
    -D CRYSTALLESS
```

For completeness and best compatibility you should also change `C:\Users\User\.platformio\platforms\atmelsam\boards\adafruit_feather_m0.json` to reference `SAMD21J17A` everywhere it mentions the `SAMD21G18A`, and the upload checks to:

```
    "maximum_ram_size": 16384,
    "maximum_size": 131072,
```

You could also/alternatively put the `-D_CRYSTALLESS` build flag in this file under `extra flags:`

If you are using other build flags be sure to add them on separate lines, and be aware board specific environment variables overwrite default ones if you have them.

## Uploading Program

I've also found PlatformIO's upload button doesn't like to work, Ardunio's did so i copied its command and use it manually

```bash
C:\Users\User\AppData\Local\Arduino15\packages\adafruit\tools\bossac\1.8.0-48-gb176eee/bossac -i -d --port=COM3 -U -i --offset=0x2000 -w -v C:\Users\User\Documents\SpectrumControllerPIO\.pio\build\adafruit_feather_m0\firmware.bin -R
```
