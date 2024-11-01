# Build Raspberry Pi 5 Linux Kernel with W5500 Support

## Overview

This guide walks you through building and customizing the Linux kernel for the Raspberry Pi 5, including support for the WIZnet W5500 Ethernet module via the WIZ850io. This setup uses Windows with WSL 2 as the development environment and covers kernel modifications, hardware setup, and essential debugging steps.


## Development Environment
- **Target Device**: [Raspberry Pi 5](https://www.raspberrypi.com/products/raspberry-pi-5/)
- **Ethernet Module**: [WIZnet WIZ850io](https://docs.wiznet.io/Product/ioModule/wiz850io)
- **Host Machine**: Windows with WSL 2
- **Kernel Source**: [Raspberry Pi Linux Kernel Source](https://github.com/raspberrypi/linux)

## Hardware Setup

Connect the Raspberry Pi 5 to the WIZ850io module as follows:

| **Raspberry Pi 5**  | **WIZ850io**     |
|---------------------|------------------|
| 6 (GND)             | GND              |
| 1 (3.3V)            | 3.3V             |
| 23 (SPI0_SCLK)      | SCLK             |
| 19 (SPI0_MOSI)      | MOSI             |
| 21 (SPI0_MISO)      | MISO             |
| 18 (BCM24)          | RSTn             |
| 22 (BCM25)          | INTn             |
| 24 (SPI0_CE0)       | SCn              |  

![image](https://github.com/user-attachments/assets/01287496-79fd-4b97-9cc6-c7d3bdbb7187)  

Refer to the image below for an example of the wiring:
![Hardware Connection Example](https://github.com/user-attachments/assets/ced33d4e-e3f8-481a-b611-2122c941c120)

The kernel overlay configuration for the W5500 Ethernet module is as follows. This allows setting the INT pin, SPI speed, and chip select (CS) pin parameters as needed. 

Refer to the official [Kernel Overlay README](https://github.com/raspberrypi/firmware/blob/master/boot/overlays/README) for additional details.

```text
Name:   w5500
Info:   Overlay for the Wiznet W5500 Ethernet Controller on SPI0
Load:   dtoverlay=w5500,<param>=<val>
Params: int_pin    GPIO used for INT (default 25)
        speed      SPI bus speed (default 30000000)
        cs         SPI bus Chip Select (default 0)
```


## Kernel and Driver Source Modifications

To monitor the operation of key functions within the Linux kernel and the W5500 driver, debug messages were added, and some sections were modified.  

An alignment error in the receive buffer was encountered, which required the following modification. The root cause of this issue is still under investigation:  
![{E619D12E-7BDE-45CD-820C-AAEC33EBCD86}](https://github.com/user-attachments/assets/84d8f324-7106-489b-9454-61c93993616c)

```c
static int dw_axi_dma_set_hw_desc(struct axi_dma_chan *chan,
				  struct axi_dma_hw_desc *hw_desc,
				  dma_addr_t mem_addr, size_t len)
{
    unsigned int data_width = BIT(chan->chip->dw->hdata->m_data_width);
    unsigned int reg_width;
    unsigned int mem_width;
    dma_addr_t device_addr;
    size_t axi_block_ts;
    size_t block_ts;
    u32 ctllo, ctlhi;
    u32 burst_len = 0, mem_burst_msize, reg_burst_msize;

    axi_block_ts = chan->chip->dw->hdata->block_size[chan->id];

    mem_width = __ffs(data_width | mem_addr | len);

/*
    if (!IS_ALIGNED(mem_addr, 4)) {
        dev_err(chan->chip->dev, "invalid buffer alignment\n");
        return -EINVAL;
    }
*/
    ..................................
```

A dedicated thread was implemented to monitor the Ethernet link status of the W5500 in real-time. This thread checks the link state periodically and logs status changes, such as when the link goes up or down.
```c
// sekim 20241015 add thread for Link
static int w5100_monitor_thread(void *data)
{
    struct w5100_priv *priv = (struct w5100_priv *)data;
    ................

    while (!kthread_should_stop()) 
    {  
        ..............
        if ( old_check_link!=now_check_link )
        {
            if ( now_check_link==1)
            {
                 printk(KERN_INFO "W5K : W5x00 Link Up\n");
                 ...................
            }
            else
            {
                 printk(KERN_INFO "W5K : W5x00 Link Down\n");
                 ...................
            }
        .................
```

After applying this monitoring thread, if you disconnect and reconnect the Ethernet port, you should see debug messages similar to the following:  
![image](https://github.com/user-attachments/assets/26111875-84e5-49f6-834f-b93392ebc7fb)

To assign a specific MAC address to the W5500, modify the W5500 overlay file as shown below. This customization allows control over the MAC address, which can be useful for network identification and management.  
![image](https://github.com/user-attachments/assets/fcd9aaa2-1abd-41f8-b2ec-7cd43b35c95f)

## Preparing a USB Drive

To enable easy swapping, a USB stick is used instead of an SD card. The following steps outline how to set up the Raspberry Pi OS on a USB drive.  

1. Download and install [Raspberry Pi Imager](https://www.raspberrypi.com/software/).
2. Insert the USB drive into your computer.
3. Open Raspberry Pi Imager, select the latest version of Raspberry Pi OS, and choose the USB drive as the destination.
4. Start the installation process and wait until it completes.

This setup allows the Raspberry Pi to boot from the USB drive, making it easier to access and modify files without the need for an SD card.  
![image](https://github.com/user-attachments/assets/d5c7d7fd-deb5-4332-a374-9c890a449f10)

To access the USB drive within WSL (Windows Subsystem for Linux), follow these steps to mount it correctly:

```cmd
(DOS) usbipd list
(DOS) usbipd attach --wsl --busid 11-4
```
```bash
lsblk
sudo mount /dev/sdd1 mnt/boot; sudo mount /dev/sdd2 mnt/root
```
```bash
sudo umount mnt/boot; sudo umount mnt/root
```

To enable USB boot on the Raspberry Pi 5, add usb_max_current_enable=1 and the w5500 overlay to the config.txt file.
(This task requires mounting the USB drive first.)
```bash
sudo nano mnt/boot/config.txt
```

To remove unnecessary splash screens and improve the natural behavior of the mouse, I added `"usbhid.mousepoll=0"` and removed `"splash"`.
```bash
sudo nano mnt/boot/cmdline.txt
```

## Build Kernel 
Refer to the following link for Raspberry Pi kernel compilation: https://www.raspberrypi.com/documentation/computers/linux_kernel.html

Set the default configuration when building the kernel.

```bash
KERNEL=kernel_2712
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- bcm2712_defconfig
```

Now, build the kernel and back up and copy files like the kernel and device tree files to the mounted USB drive.  
Before proceeding with the steps below, you need to mount the USB drive in WSL.
```bash
sudo mount /dev/sdd1 mnt/boot; sudo mount /dev/sdd2 mnt/root
```
```bash
sudo make -j$(nproc) ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- Image modules dtbs
```
```bash
sudo make -j$(nproc) ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- INSTALL_MOD_PATH=mnt/root modules_install
sudo cp mnt/boot/$KERNEL.img mnt/boot/$KERNEL-backup.img
sudo cp arch/arm64/boot/Image mnt/boot/$KERNEL.img
sudo cp arch/arm64/boot/dts/broadcom/*.dtb mnt/boot/
sudo cp arch/arm64/boot/dts/overlays/*.dtb* mnt/boot/overlays/
sudo cp arch/arm64/boot/dts/overlays/README mnt/boot/overlays/
sudo umount mnt/boot; sudo umount mnt/root
```
![image](https://github.com/user-attachments/assets/850a40b5-d04c-49b2-84ab-4f82e5babd3d)

## Checking Boot Messages
After completing all the steps, insert the USB drive into the Raspberry Pi and boot it up.
In the log below, you can check that the W5500 is recognized and initialized correctly during the boot process.
``` bash
ssekim@raspberrypi:~ $ dmesg | grep W5K
[    0.000000] W5K : start_kernel()
[    9.261926] W5K : w5100_spi_probe
[    9.261936] W5K : w5100_probe (irq:171, link_gpio:-22)
[    9.265813] W5K : w5500_spi_init()
[    9.265820] W5K : w5100_hw_reset()
[    9.271024] W5K : w5500_memory_configure()
[    9.272864] W5K : W5x00 Link Up
[    9.322150] W5K : w5100_get_drvinfo()
[   13.795703] W5K : w5100_get_drvinfo()
[   13.795810] W5K : w5100_get_drvinfo()
[   14.410316] W5K : w5100_get_drvinfo()
[   14.414438] W5K : w5100_open()
[   14.414446] W5K : w5100_hw_start()
[   14.415222] W5K : w5100_get_link()
[   14.463926] W5K : w5100_get_link()
``` 
![image](https://github.com/user-attachments/assets/ce79eaa4-e26b-455f-b6c3-39f4bb04747a)

## IP Configuration and Testing
Use the following command to set a static IP on the eth1 interface.

```bash
sudo ip addr add 192.168.11.17/24 dev eth1
```
![image](https://github.com/user-attachments/assets/e2275bee-4a47-4de8-b33c-f429b14df375)

If you want the settings to persist after a reboot, use the following command.
```bash
nmcli connection show
sudo nmcli connection modify be76cf51-6064-3d37-b026-9250d47ea478 ipv4.addresses "192.168.11.17/24" ipv4.gateway "192.168.11.1" ipv4.dns "8.8.8.8" ipv4.method "manual"
```
![image](https://github.com/user-attachments/assets/42bdf6af-9a90-43c5-b14c-17498155f089)

## ※ Appendix A: iPerf Test
Below is the result of the test conducted with iPerf.
![image](https://github.com/user-attachments/assets/48d69203-5156-4464-9295-6d4461d2d17d)

## ※ Appendix B: SPI Low-Level Test  
Below is the content of a low-level SPI communication test with the W5500.  
It includes an example of a command to read the MAC Address of the W5500 and an SPI signal screen.
C source file([w5x00_spi_test.c](w5x00work/w5x00_spi_test.c])) is compiled on R-Pi 5.  
![image](https://github.com/user-attachments/assets/d39a048a-2171-4371-8de3-7917b463bc9b)
![image](https://github.com/user-attachments/assets/91ad4045-3691-4fbf-93f7-0e6150c1cd59)

## ※ Appendix C: Reload SPI module
Using the following process, you can reset the W5100 driver and SPI module, and restart W5500-related operations.

Check the current W5500 driver status in the kernel.  
```bash
lsmod | grep w5
```
Remove the W5100-related SPI module from the kernel using the `modprobe` command.  
```bash
sudo modprobe -r w5100-spi; sudo modprobe -r w5100
```
Reload the SPI module to add it back to the kernel.
```bash
sudo modprobe w5100; sudo modprobe w5100-spi
```

## ※ Appendix D: Additional useful commands for testing

USB 메모리를 WSL에 Attach 한다.
현재 커널 버전의 W5x00 드라이버를 확인한다.
Overlay를 이용해서 

```cmd
usbipd list
usbipd attach --wsl --busid 11-4
usbipd detach --busid 11-4
```

```bash
ls -al /lib/modules/$(uname -r)/kernel/drivers/net/ethernet/wiznet/
```

```bash
lsmod | grep w5
sudo modprobe -r w5100-spi; sudo modprobe -r w5100
sudo modprobe w5100; sudo modprobe w5100-spi
lsmod | grep w5
```

```bash
ls /boot/overlays/ | grep w5
sudo dtoverlay -h w5500
sudo dtoverlay w5500
sudo dtoverlay w5500 int_pin=25 speed=500000
sudo dtoverlay -l
sudo dtoverlay -r 0
```
```bash
sudo raspi-config
```








