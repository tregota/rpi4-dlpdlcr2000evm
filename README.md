# How I got the DLPDLCR2000EVM to work on a Raspberry Pi 4

A few years ago I saw [MickMake use the DLPDLCR2000EVM on a Raspberry Pi Zero](https://github.com/MickMake/Project-PiProjector) and wanted to test it on a Raspberry Pi 4. I wasn't aiming as high as he did with his custom PCB but I found someone who got it to work with a Raspberry Pi 3 on [this now defunct site](https://web.archive.org/web/20200708171522/http://frederickvandenbosch.be/?p=2948), so I followed his example and tweaked the config using suggestions [posted on the ti forums](https://e2e.ti.com/support/dlp-products-group/dlp/f/dlp-products-forum/850392/dlpdlcr2000evm-resolution-problem-settings-with-i2c-and-raspberry-pi). This worked pretty well.

I connected the pins as described in the WayBackMaching link above (I have the pin map images saved here in the repository in case the site disappears, credit to Frederick Vandenbosch) and I think the config required can be summed up in enabling I2C and adding:
```
# Add support for software i2c on gpio pins
dtoverlay=i2c-gpio,bus=3,i2c_gpio_sda=23,i2c_gpio_scl=24,i2c_gpio_delay_us=2

# DPI Video Setup
dtoverlay=dpi18
overscan_left=0
overscan_right=0
overscan_top=0
overscan_bottom=0
framebuffer_width=864
framebuffer_height=480

enable_dpi_lcd=1
display_default_lcd=1

dpi_group=2
dpi_mode=87
dpi_output_format=458773
dpi_timings=864 0 14 4 12 480 0 2 3 9 0 0 0 60 0 24883200 3
```

And then running these commands on boot
```
sudo i2cset -y 3 0x1b 0x0c 0x00 0x00 0x00 0x15 i
sudo i2cset -y 3 0x1b 0x0b 0x00 0x00 0x00 0x00 i
```  

## Now to get it to work for Android TV
```
If you plan to add Play store and other things using TWRP recovery. Do that via HDMI before adding all the following projector stuff.
```

I wanted to get it to work with Android TV running, so I installed [KonstaKangs build of LineageOS 20 Android TV](https://konstakang.com/devices/rpi4/LineageOS20-ATV/)  
First and foremost KonstaKang allows using GPIO for a power button, GPIO that we have connected to the projector. So we have to remove some config like:
```
# Ramdisk
gpio=21=ip,pu
[gpio21=1]
initramfs ramdisk.img followkernel
[gpio21=0]
initramfs ramdisk-recovery.img followkernel
[all]
```
must be changed to 
```
# Ramdisk
initramfs ramdisk.img followkernel
```

**Remove all gpio21 checks like that and leave whatever would have been configured if gpio21=1**

---

The second hurdle is that the old dpi config above only works with vc4-fkms-v3d while I now had to use the newer vc4-kms-v3d

[These new settings from the ti forums](https://e2e.ti.com/support/dlp-products-group/dlp/f/dlp-products-forum/1281455/dlpdlcr2000evm-raspberry-pi-configuration-using-dtoverlay-vc4-kms-dpi-generic-raspberry-pi-os-12-bookworm) worked though:
```
# Add support for software i2c on gpio pins
dtoverlay=i2c-gpio,bus=1,i2c_gpio_sda=23,i2c_gpio_scl=24,i2c_gpio_delay_us=2

# init vc3-kms-dpi-generic overlay with dlp2000 parameters
dtoverlay=vc4-kms-dpi-generic,clock-frequency=25000000
dtparam=hactive=640,hfp=14,hsync=4,hbp=12
dtparam=vactive=360,vfp=2,vsync=3,vbp=9
```

To run the i2cset commands on boot in Android TV I have added these rows to /etc/init/hw/init.rc at the end of "on early-init"
```
    exec - root root -- /vendor/bin/i2cset -y 1 0x1b 0x0c 0x00 0x00 0x00 0x1B i
    exec - root root -- /vendor/bin/i2cset -y 1 0x1b 0x0b 0x00 0x00 0x00 0x00 i
```