# EAIDK-310 SBC board
## Product Introduction
  * EAIDK 310: Arm cpu and ARM MaLI GPU ,Main chip uses the RK3228H of mainstream performance Arm Soc.
  * CPU: ARM 4 core Cortex-A53 ,64 bit processor
  * RAM:LPDDR3 1GB
  * Wifi :2.4G/5GHz,Bluetooth 5.0
  * Power:Micro USB 5v/2A HDMI:2.0, 1*Type -A
  * Operation system :Linux and Android 8.1,
  * Video code API :Hard decoding and hardcode

What works:
  - Mainline u-boot. (have detect sdio problem).
  - Mainline stable kernel, latest stable version on build date.
  - Mainline ATF provided as Trusted Execution Environment.
  - All 4 cores are working.
  - Ethernet
  - Wifi+BT (with old u-boot)
  - fbdev with ILI9341 SPI
  - sdmmc_ext SDIO with [esp-hosted](https://github.com/espressif/esp-hosted), tested with `ESP32`.

Releases:
  * Kernel 6.8.4, The latest stable version of the day.
  * Debian 12(bookworm)
  * default user: eaidk , password: 1234

## Test `ILI9341`

* lcd with eaidk connect table.

|  RK3328 CON1  | 2.8 inch SPI |
| :-----------: | :----------: |
| GPIO3_A0 (23) |     SCK      |
| GPIO3_A1 (19) |  SDI(MOSI)   |
| GPIO3_A2 (21) |  SDO(MISO)   |
| GPIO2_C4 (36) |      DC      |
| GPIO3_B0 (24) |      CS      |
| GPIO2_B7 (7)  |    RESET     |
|  VCC 3v3 (1)  |      BK      |
|  VCC 3v3 (17) |     VCC      |
|      GND (6)  |     GND      |


* device tree overlay plugin patch

```sh
/dts-v1/;
/plugin/;

#include <dt-bindings/gpio/gpio.h>
#include <dt-bindings/pinctrl/rockchip.h>
#include <dt-bindings/interrupt-controller/irq.h>

/{
    fragment@0 {
        target= <&spi0>;
        __overlay__ {
                /delete-property/ flash@0;
                status = "okay";
                #address-cells = <1>;
			          #size-cells = <0>;
                ili9341@0 {
                    compatible = "ilitek,ili9341", "spidev";
                    reg = <0>;
                    spi-max-frequency = <50000000>;
                    rotate = <0>;
                    bgr;
                    fps = <30>;
                    buswidth = <8>;
                    reset-gpios = <&gpio2 RK_PB7 GPIO_ACTIVE_LOW>;
                    dc-gpios = <&gpio2 RK_PC4 GPIO_ACTIVE_HIGH>;
                    debug = <0>;
            };
        };
    };
};
```
* if load above success , will show below in dmesg

```sh
[   66.204021] fbtft: module is from the staging directory, the quality is unknown, you have been warned.
[   66.208716] fb_ili9341: module is from the staging directory, the quality is unknown, you have been warned.
[   66.210464] fb_ili9341 spi0.0: fbtft_property_value: buswidth = 8
[   66.211049] fb_ili9341 spi0.0: fbtft_property_value: debug = 0
[   66.211576] fb_ili9341 spi0.0: fbtft_property_value: rotate = 270
[   66.212128] fb_ili9341 spi0.0: fbtft_property_value: fps = 30
[   66.507261] Console: switching to colour frame buffer device 40x30
[   66.508754] graphics fb0: fb_ili9341 frame buffer, 320x240, 150 KiB video memory, 16 KiB buffer memory, fps=31, spi0.0 at 50 MHz

```
* Now you can output `rgb565` bmp files  to LCD for display.

```sh
~ # tail --bytes 153600 /home/eaidk/ffmpeg_rgb565.bmp > /dev/fb0
```

* images

![board](board.png)
![show_image](show_image.png)
![show_console](show_console.png)
![output](output.gif)

### Xorg with IceWM and x11vnc

```sh
~$ sudo apt-get install icewm xinit xserver-xorg-video-fbdev x11vnc
```

* create a `/etc/X11/xorg.conf`.

```sh
~# cat /etc/X11/xorg.conf
Section "Monitor"
    Identifier          "Monitor0"
EndSection
Section "Device"
         Option     "ShadowFB"              "false"
         Option     "Rotate"                "CW"
         Option     "fbdev"                 "/dev/fb0"
         Option     "debug"                 "true"
         Identifier  "Card0"
         Driver      "fbdev"
EndSection
Section "Screen"
    Identifier          "Screen0"
    Device              "Card0"
    Monitor             "Monitor0"
    DefaultDepth        16
EndSection

```
* run `startx`

```sh
~$ startx /usr/bin/icewm-session
```
* run `x11vnc`

```sh
~$ VNC_PASSWORD=123456 x11vnc -many  -display :2
```
![rk3328-icewm.png](rk3328-icewm.png)
![icewm](icewm.gif)

## ESP-Hosted test

* [ESP-Hosted](https://github.com/espressif/esp-hosted) is an open source solution that provides a way to use Espressif SoCs and modules as a communication co-processor. This solution provides wireless connectivity (Wi-Fi and BT/BLE) to the host microprocessor or microcontroller, allowing it to communicate with other devices.

* `ESP32-WROOM-32U` with `EAIDK-310` connect table.

|  eaidk-310  |     ESP32      | Function    |
| :---------: | :------------: |:-----------:|
| GPIO3_A7 22 | IO13 (pull_up) | D3          |
| GPIO3_A6 10 | IO12 (pull_up) | D2          |
| GPIO3_A5 16 | IO4  (pull_up) | D1          |
| GPIO3_A4 8  | IO2  (pull_up) | D0          |
| GPIO3_A2 21 |      IO14      | CLK         |
| GPIO3_A0 23 | IO15 (pull_up) | CMD         |
| GPIO2_C4 36 |       EN       | ESP32 Reset |
|  GND    6   |      GND       | ESP32       |
|  5V     4   |       5V       | ESP32       |


* ![eaidk-310-with-esp32.png](eaidk-310-with-esp32.png)

* insmod first

```sh
root@eaidk-310:~# insmod ./esp32_sdio.ko resetpin=84
root@eaidk-310:~# dmesg | grep "esp32"
[  275.846457] esp32_sdio: loading out-of-tree module taints kernel.
[  275.849309] esp32_sdio: esp_reset: Triggering ESP reset.
[  276.057114] esp32_sdio: probe of mmc3:0001:1 failed with error -110
[ 1155.841868] esp32_sdio: esp_reset: Triggering ESP reset.
[ 1156.049720] esp32_sdio: probe of mmc3:0001:1 failed with error -110

```

* manual trigger mmc3

```sh
root@eaidk-310:~# echo ff5f0000.mmc > /sys/bus/platform/drivers/dwmmc_rockchip/unbind
root@eaidk-310:~# echo ff5f0000.mmc > /sys/bus/platform/drivers/dwmmc_rockchip/bind
```

* detected esp32 network adapter device by SDIO.

```sh
[ 1242.548630] mmc3: card 0001 removed
[ 1246.419890] dwmmc_rockchip ff5f0000.mmc: IDMAC supports 32-bit address mode.
[ 1246.420761] dwmmc_rockchip ff5f0000.mmc: Using internal DMA controller.
[ 1246.421396] dwmmc_rockchip ff5f0000.mmc: Version ID is 270a
[ 1246.421994] dwmmc_rockchip ff5f0000.mmc: DW MMC controller at irq 47,32 bit host data width,256 deep fifo
[ 1246.423516] mmc_host mmc3: card is non-removable.
[ 1246.437457] mmc_host mmc3: Bus speed (slot 0) = 400000Hz (slot req 400000Hz, actual 400000HZ div = 0)
[ 1246.477417] mmc3: queuing unknown CIS tuple 0x01 [d9 01 ff] (3 bytes)
[ 1246.486079] mmc3: queuing unknown CIS tuple 0x1a [01 01 00 02 07] (5 bytes)
[ 1246.490666] mmc3: queuing unknown CIS tuple 0x1b [c1 41 30 30 ff ff ff ff] (8 bytes)
[ 1246.491829] mmc_host mmc3: Bus speed (slot 0) = 25000000Hz (slot req 25000000Hz, actual 25000000HZ div = 0)
[ 1246.497985] mmc3: new SDIO card at address 0001
[ 1246.499304] esp32_sdio: esp_probe: ESP network device detected
[ 1246.500177] esp32_sdio: get_firmware_data: Rx Pre ====== 0
[ 1246.500851] esp32_sdio: get_firmware_data: Rx Pos ======  0
[ 1246.501484] esp32_sdio: get_firmware_data: Tx Pre ======  0
[ 1246.502018] esp32_sdio: get_firmware_data: Tx Pos ======  10
[ 1246.502978] esp32_sdio: esp_probe: ESP SDIO probe completed
[ 1246.556452] esp32_sdio: process_esp_bootup_event: Received ESP bootup event
[ 1246.557134] esp32_sdio: process_event_esp_bootup: Bootup Event tag: 3
[ 1246.557727] esp32_sdio: esp_validate_chipset: Chipset=ESP32 ID=00 detected over SDIO
[ 1246.558421] esp32_sdio: process_event_esp_bootup: Bootup Event tag: 0
[ 1246.558999] esp32_sdio: process_event_esp_bootup: Bootup Event tag: 1
[ 1246.559576] esp32_sdio: process_fw_data: ESP chipset\'s last reset cause:
[ 1246.560176] esp32_sdio: print_reset_reason: POWERON_RESET
[ 1246.560744] esp32_sdio: check_esp_version: ESP Firmware version: 1.0.3
[ 1246.562185] esp32_sdio: esp_reg_notifier: Driver init is ongoing
[ 1246.626914] esp32_sdio: esp_cfg80211_get_tx_power:
[ 1246.832511] esp32_sdio: init_bt: ESP Bluetooth init
[ 1246.834178] esp32_sdio: print_capabilities: Capabilities: 0x1d. Features supported are:
[ 1246.835002] esp32_sdio: print_capabilities: 	 * WLAN on SDIO
[ 1246.835557] esp32_sdio: print_capabilities: 	 * BT/BLE
[ 1246.836055] esp32_sdio: print_capabilities: 	   - HCI over SDIO
[ 1246.836837] esp32_sdio: print_capabilities: 	   - BT/BLE dual mode
[ 1247.316894] Bluetooth: MGMT ver 1.22
[ 1247.337968] NET: Registered PF_ALG protocol family

root@eaidk-310:~# ifconfig espsta0
espsta0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet6 fe80::c5a8:a34c:9347:296c  prefixlen 64  scopeid 0x20<link>
        ether 78:21:84:9c:16:a0  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 11 overruns 0  carrier 0  collisions 0

```

* Use `wpa_supplicant` to test

```sh
~$ wpa_supplicant -B -D wext -i espsta0 -c <(wpa_passphrase YOUR_SSID YOUR_WIFI_PWD)
Successfully initialized wpa_supplicant
ioctl[SIOCSIWENCODEEXT]: Invalid argument
ioctl[SIOCSIWENCODEEXT]: Invalid argument
```

## Benchmarks

* CPU info

```sh
~$ lscpu
Architecture:             aarch64
  CPU op-mode(s):         32-bit, 64-bit
  Byte Order:             Little Endian
CPU(s):                   4
  On-line CPU(s) list:    0-3
Vendor ID:                ARM
  Model name:             Cortex-A53
    Model:                4
    Thread(s) per core:   1
    Core(s) per cluster:  4
    Socket(s):            -
    Cluster(s):           1
    Stepping:             r0p4
    CPU(s) scaling MHz:   78%
    CPU max MHz:          1296.0000
    CPU min MHz:          408.0000
    BogoMIPS:             48.00
    Flags:                fp asimd evtstrm aes pmull sha1 sha2 crc32 cpuid
NUMA:
  NUMA node(s):           1
  NUMA node0 CPU(s):      0-3
Vulnerabilities:
  Gather data sampling:   Not affected
  Itlb multihit:          Not affected
  L1tf:                   Not affected
  Mds:                    Not affected
  Meltdown:               Not affected
  Mmio stale data:        Not affected
  Reg file data sampling: Not affected
  Retbleed:               Not affected
  Spec rstack overflow:   Not affected
  Spec store bypass:      Not affected
  Spectre v1:             Mitigation; __user pointer sanitization
  Spectre v2:             Not affected
  Srbds:                  Not affected
  Tsx async abort:        Not affected

~$  cat /proc/cpuinfo
processor	: 0
BogoMIPS	: 48.00
Features	: fp asimd evtstrm aes pmull sha1 sha2 crc32 cpuid
CPU implementer	: 0x41
CPU architecture: 8
CPU variant	: 0x0
CPU part	: 0xd03
CPU revision	: 4

processor	: 1
BogoMIPS	: 48.00
Features	: fp asimd evtstrm aes pmull sha1 sha2 crc32 cpuid
CPU implementer	: 0x41
CPU architecture: 8
CPU variant	: 0x0
CPU part	: 0xd03
CPU revision	: 4

processor	: 2
BogoMIPS	: 48.00
Features	: fp asimd evtstrm aes pmull sha1 sha2 crc32 cpuid
CPU implementer	: 0x41
CPU architecture: 8
CPU variant	: 0x0
CPU part	: 0xd03
CPU revision	: 4

processor	: 3
BogoMIPS	: 48.00
Features	: fp asimd evtstrm aes pmull sha1 sha2 crc32 cpuid
CPU implementer	: 0x41
CPU architecture: 8
CPU variant	: 0x0
CPU part	: 0xd03
CPU revision	: 4
```

* openssl (crypto)

```sh
eaidk@eaidk-310:~$ openssl speed -elapsed -evp aes-128-gcm aes-128-cbc sha256
You have chosen to measure elapsed time instead of user CPU time.
Doing sha256 for 3s on 16 size blocks: 1729302 sha256's in 3.00s
Doing sha256 for 3s on 64 size blocks: 1628260 sha256's in 3.00s
Doing sha256 for 3s on 256 size blocks: 1381553 sha256's in 3.00s
Doing sha256 for 3s on 1024 size blocks: 852353 sha256's in 3.00s
Doing sha256 for 3s on 8192 size blocks: 187225 sha256's in 3.00s
Doing sha256 for 3s on 16384 size blocks: 98572 sha256's in 3.00s
Doing aes-128-cbc for 3s on 16 size blocks: 18429939 aes-128-cbc's in 3.00s
Doing aes-128-cbc for 3s on 64 size blocks: 14175832 aes-128-cbc's in 3.00s
Doing aes-128-cbc for 3s on 256 size blocks: 7219643 aes-128-cbc's in 3.00s
Doing aes-128-cbc for 3s on 1024 size blocks: 2508817 aes-128-cbc's in 3.00s
Doing aes-128-cbc for 3s on 8192 size blocks: 353689 aes-128-cbc's in 3.00s
Doing aes-128-cbc for 3s on 16384 size blocks: 177769 aes-128-cbc's in 3.00s
Doing AES-128-GCM for 3s on 16 size blocks: 10169150 AES-128-GCM's in 3.00s
Doing AES-128-GCM for 3s on 64 size blocks: 7809045 AES-128-GCM's in 3.00s
Doing AES-128-GCM for 3s on 256 size blocks: 4100438 AES-128-GCM's in 3.00s
Doing AES-128-GCM for 3s on 1024 size blocks: 1524611 AES-128-GCM's in 3.00s
Doing AES-128-GCM for 3s on 8192 size blocks: 225044 AES-128-GCM's in 3.00s
Doing AES-128-GCM for 3s on 16384 size blocks: 113699 AES-128-GCM's in 3.00s
version: 3.0.11
built on: Mon Oct 23 17:52:22 2023 UTC
options: bn(64,64)
compiler: gcc -fPIC -pthread -Wa,--noexecstack -Wall -fzero-call-used-regs=used-gpr -DOPENSSL_TLS_SECURITY_LEVEL=2 -Wa,--noexecstack -g -O2 -ffile-prefix-map=/build/reproducible-path/openssl-3.0.11=. -fstack-protector-strong -Wformat -Werror=format-security -DOPENSSL_USE_NODELETE -DOPENSSL_PIC -DOPENSSL_BUILDING_OPENSSL -DNDEBUG -Wdate-time -D_FORTIFY_SOURCE=2
CPUINFO: OPENSSL_armcap=0xbd
The 'numbers' are in 1000s of bytes per second processed.
type             16 bytes     64 bytes    256 bytes   1024 bytes   8192 bytes  16384 bytes
sha256            9222.94k    34736.21k   117892.52k   290936.49k   511249.07k   538334.55k
aes-128-cbc      98293.01k   302417.75k   616076.20k   856342.87k   965806.76k   970855.77k
AES-128-GCM      54235.47k   166592.96k   349904.04k   520400.55k   614520.15k   620948.14k


```
## tailscale

```sh
eaidk@eaidk-310:~$ ifconfig tailscale0
tailscale0: flags=4305<UP,POINTOPOINT,RUNNING,NOARP,MULTICAST>  mtu 1280
        inet6 fe80::a62c:89cc:8c9f:8f1f  prefixlen 64  scopeid 0x20<link>
        unspec 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00  txqueuelen 500  (UNSPEC)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 3  bytes 144 (144.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

```

## Boot log (booting from SD Card)

* custom motd

```sh
$ ssh eaidk
 _____    _    ___ ____  _  __    _____ _  ___
| ____|  / \  |_ _|  _ \| |/ /   |___ // |/ _ \
|  _|   / _ \  | || | | | ' /_____ |_ \| | | | |
| |___ / ___ \ | || |_| | . \_____|__) | | |_| |
|_____/_/   \_\___|____/|_|\_\   |____/|_|\___/

Welcome to Debian GNU/Linux 12 (bookworm) with Linux 6.8.4-rk3328

System load:   38%           	Up time:       1 min
Memory usage:  10% of 975M   	IP:	       192.168.1.248
CPU temp:      43°C
RX today:      319.1 KiB

Last login: Sat Jan 27 05:49:09 2024

```

* dmesg
```sh
eaidk@eaidk-310:~$ dmesg
[    0.000000] Booting Linux on physical CPU 0x0000000000 [0x410fd034]
[    0.000000] Linux version 6.8.4-rk3328 (yjdwbj@gmail.com) (aarch64-linux-gnu-gcc (Debian 12.2.0-14) 12.2.0, GNU ld (GNU Binutils for Debian) 2.40) #1 SMP PREEMPT Mon Apr  8 18:19:54 CST 2024
[    0.000000] Machine model: EAIDK-310 build by lcy v2
[    0.000000] earlycon: uart0 at MMIO32 0x00000000ff130000 (options '1500000n8')
[    0.000000] printk: legacy bootconsole [uart0] enabled
[    0.000000] efi: UEFI not found.
[    0.000000] NUMA: No NUMA configuration found
[    0.000000] NUMA: Faking a node at [mem 0x0000000000200000-0x000000003fffffff]
[    0.000000] NUMA: NODE_DATA [mem 0x3fdb99c0-0x3fdbbfff]
[    0.000000] Zone ranges:
[    0.000000]   DMA      [mem 0x0000000000200000-0x000000003fffffff]
[    0.000000]   DMA32    empty
[    0.000000]   Normal   empty
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000000200000-0x000000003fffffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000000200000-0x000000003fffffff]
[    0.000000] On node 0, zone DMA: 512 pages in unavailable ranges
[    0.000000] cma: Reserved 128 MiB at 0x0000000035400000 on node -1
[    0.000000] psci: probing for conduit method from DT.
[    0.000000] psci: PSCIv1.0 detected in firmware.
[    0.000000] psci: Using standard PSCI v0.2 function IDs
[    0.000000] psci: MIGRATE_INFO_TYPE not supported.
[    0.000000] psci: SMC Calling Convention v1.0
[    0.000000] percpu: Embedded 30 pages/cpu s82280 r8192 d32408 u122880
[    0.000000] pcpu-alloc: s82280 r8192 d32408 u122880 alloc=30*4096
[    0.000000] pcpu-alloc: [0] 0 [0] 1 [0] 2 [0] 3
[    0.000000] Detected VIPT I-cache on CPU0
[    0.000000] CPU features: detected: ARM erratum 845719
[    0.000000] alternatives: applying boot alternatives
[    0.000000] Kernel command line: root=UUID=bd2a6dbf-c55f-4d5e-8738-797e581ee7c9 rootwait rootfstype=ext4 net.ifnames=0  earlycon console=ttyS2,1500000n8 console=tty1 consoleblank=0 loglevel=7
[    0.000000] Dentry cache hash table entries: 131072 (order: 8, 1048576 bytes, linear)
[    0.000000] Inode-cache hash table entries: 65536 (order: 7, 524288 bytes, linear)
[    0.000000] Fallback order for Node 0: 0
[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 257544
[    0.000000] Policy zone: DMA
[    0.000000] mem auto-init: stack:off, heap alloc:on, heap free:off
[    0.000000] software IO TLB: SWIOTLB bounce buffer size adjusted to 0MB
[    0.000000] software IO TLB: area num 4.
[    0.000000] software IO TLB: mapped [mem 0x000000003ea80000-0x000000003eb80000] (1MB)
[    0.000000] Memory: 852572K/1046528K available (16384K kernel code, 2342K rwdata, 6056K rodata, 4480K init, 570K bss, 62884K reserved, 131072K cma-reserved)
[    0.000000] SLUB: HWalign=64, Order=0-3, MinObjects=0, CPUs=4, Nodes=1
[    0.000000] trace event string verifier disabled
[    0.000000] rcu: Preemptible hierarchical RCU implementation.
[    0.000000] rcu: 	RCU event tracing is enabled.
[    0.000000] rcu: 	RCU restricting CPUs from NR_CPUS=256 to nr_cpu_ids=4.
[    0.000000] 	Trampoline variant of Tasks RCU enabled.
[    0.000000] 	Tracing variant of Tasks RCU enabled.
[    0.000000] rcu: RCU calculated value of scheduler-enlistment delay is 25 jiffies.
[    0.000000] rcu: Adjusting geometry for rcu_fanout_leaf=16, nr_cpu_ids=4
[    0.000000] NR_IRQS: 64, nr_irqs: 64, preallocated irqs: 0
[    0.000000] Root IRQ handler: gic_handle_irq
[    0.000000] GIC: Using split EOI/Deactivate mode
[    0.000000] rcu: srcu_init: Setting srcu_struct sizes based on contention.
[    0.000000] arch_timer: cp15 timer(s) running at 24.00MHz (phys).
[    0.000000] clocksource: arch_sys_counter: mask: 0xffffffffffffff max_cycles: 0x588fe9dc0, max_idle_ns: 440795202592 ns
[    0.000001] sched_clock: 56 bits at 24MHz, resolution 41ns, wraps every 4398046511097ns
[    0.001700] Console: colour dummy device 80x25
[    0.002211] printk: legacy console [tty1] enabled
[    0.002673] printk: legacy bootconsole [uart0] disabled
[    0.003337] Calibrating delay loop (skipped), value calculated using timer frequency.. 48.00 BogoMIPS (lpj=96000)
[    0.003386] pid_max: default: 32768 minimum: 301
[    0.003560] LSM: initializing lsm=capability,yama,apparmor,integrity
[    0.003640] Yama: becoming mindful.
[    0.003814] AppArmor: AppArmor initialized
[    0.003900] stackdepot: allocating hash table of 65536 entries via kvcalloc
[    0.004938] Mount-cache hash table entries: 2048 (order: 2, 16384 bytes, linear)
[    0.004987] Mountpoint-cache hash table entries: 2048 (order: 2, 16384 bytes, linear)
[    0.009258] RCU Tasks: Setting shift to 2 and lim to 1 rcu_task_cb_adjust=1.
[    0.009557] RCU Tasks Trace: Setting shift to 2 and lim to 1 rcu_task_cb_adjust=1.
[    0.010123] rcu: Hierarchical SRCU implementation.
[    0.010152] rcu: 	Max phase no-delay instances is 1000.
[    0.012198] EFI services will not be available.
[    0.013054] smp: Bringing up secondary CPUs ...
[    0.014370] Detected VIPT I-cache on CPU1
[    0.014573] CPU1: Booted secondary processor 0x0000000001 [0x410fd034]
[    0.016167] Detected VIPT I-cache on CPU2
[    0.016363] CPU2: Booted secondary processor 0x0000000002 [0x410fd034]
[    0.017888] Detected VIPT I-cache on CPU3
[    0.018083] CPU3: Booted secondary processor 0x0000000003 [0x410fd034]
[    0.018345] smp: Brought up 1 node, 4 CPUs
[    0.018551] SMP: Total of 4 processors activated.
[    0.018577] CPU: All CPU(s) started at EL2
[    0.018601] CPU features: detected: 32-bit EL0 Support
[    0.018629] CPU features: detected: CRC32 instructions
[    0.018747] alternatives: applying system-wide alternatives
[    0.021653] devtmpfs: initialized
[    0.041404] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 7645041785100000 ns
[    0.041527] futex hash table entries: 1024 (order: 4, 65536 bytes, linear)
[    0.053094] pinctrl core: initialized pinctrl subsystem
[    0.054273] DMI not present or invalid.
[    0.056554] NET: Registered PF_NETLINK/PF_ROUTE protocol family
[    0.060256] DMA: preallocated 128 KiB GFP_KERNEL pool for atomic allocations
[    0.061120] DMA: preallocated 128 KiB GFP_KERNEL|GFP_DMA pool for atomic allocations
[    0.062555] DMA: preallocated 128 KiB GFP_KERNEL|GFP_DMA32 pool for atomic allocations
[    0.063397] audit: initializing netlink subsys (disabled)
[    0.064011] audit: type=2000 audit(0.064:1): state=initialized audit_enabled=0 res=1
[    0.065719] thermal_sys: Registered thermal governor 'fair_share'
[    0.065736] thermal_sys: Registered thermal governor 'bang_bang'
[    0.065770] thermal_sys: Registered thermal governor 'step_wise'
[    0.065796] thermal_sys: Registered thermal governor 'user_space'
[    0.065951] cpuidle: using governor menu
[    0.066453] hw-breakpoint: found 6 breakpoint and 4 watchpoint registers.
[    0.066727] ASID allocator initialised with 65536 entries
[    0.067281] Serial: AMBA PL011 UART driver
[    0.085542] platform ff370000.vop: Fixed dependency cycle(s) with /hdmi@ff3c0000
[    0.085742] platform ff3c0000.hdmi: Fixed dependency cycle(s) with /vop@ff370000
[    0.097000] platform pinctrl: Fixed dependency cycle(s) with /pinctrl/clk_32k/clk-32k-out
[    0.108862] gpio gpiochip0: Static allocation of GPIO base is deprecated, use dynamic allocation.
[    0.109666] rockchip-gpio ff210000.gpio: probed /pinctrl/gpio@ff210000
[    0.110799] gpio gpiochip1: Static allocation of GPIO base is deprecated, use dynamic allocation.
[    0.111472] rockchip-gpio ff220000.gpio: probed /pinctrl/gpio@ff220000
[    0.112581] gpio gpiochip2: Static allocation of GPIO base is deprecated, use dynamic allocation.
[    0.113244] rockchip-gpio ff230000.gpio: probed /pinctrl/gpio@ff230000
[    0.114102] gpio gpiochip3: Static allocation of GPIO base is deprecated, use dynamic allocation.
[    0.114739] rockchip-gpio ff240000.gpio: probed /pinctrl/gpio@ff240000
[    0.122075] Modules: 25264 pages in range for non-PLT usage
[    0.122102] Modules: 516784 pages in range for PLT usage
[    0.124308] HugeTLB: registered 1.00 GiB page size, pre-allocated 0 pages
[    0.124391] HugeTLB: 0 KiB vmemmap can be freed for a 1.00 GiB page
[    0.124420] HugeTLB: registered 32.0 MiB page size, pre-allocated 0 pages
[    0.124448] HugeTLB: 0 KiB vmemmap can be freed for a 32.0 MiB page
[    0.124476] HugeTLB: registered 2.00 MiB page size, pre-allocated 0 pages
[    0.124503] HugeTLB: 0 KiB vmemmap can be freed for a 2.00 MiB page
[    0.124532] HugeTLB: registered 64.0 KiB page size, pre-allocated 0 pages
[    0.124559] HugeTLB: 0 KiB vmemmap can be freed for a 64.0 KiB page
[    0.126360] cryptd: max_cpu_qlen set to 1000
[    0.196119] raid6: neonx8   gen()  1100 MB/s
[    0.264269] raid6: neonx4   gen()  1081 MB/s
[    0.332423] raid6: neonx2   gen()  1026 MB/s
[    0.400582] raid6: neonx1   gen()   877 MB/s
[    0.468750] raid6: int64x8  gen()   707 MB/s
[    0.536909] raid6: int64x4  gen()   780 MB/s
[    0.605062] raid6: int64x2  gen()   697 MB/s
[    0.673217] raid6: int64x1  gen()   516 MB/s
[    0.673247] raid6: using algorithm neonx8 gen() 1100 MB/s
[    0.741369] raid6: .... xor() 809 MB/s, rmw enabled
[    0.741398] raid6: using neon recovery algorithm
[    0.743176] iommu: Default domain type: Translated
[    0.743241] iommu: DMA domain TLB invalidation policy: strict mode
[    0.744911] SCSI subsystem initialized
[    0.745403] libata version 3.00 loaded.
[    0.745978] usbcore: registered new interface driver usbfs
[    0.746110] usbcore: registered new interface driver hub
[    0.746231] usbcore: registered new device driver usb
[    0.747256] pps_core: LinuxPPS API ver. 1 registered
[    0.747288] pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>
[    0.747348] PTP clock support registered
[    0.747452] EDAC MC: Ver: 3.0.0
[    0.748895] scmi_core: SCMI protocol bus registered
[    0.751351] NetLabel: Initializing
[    0.751406] NetLabel:  domain hash size = 128
[    0.751429] NetLabel:  protocols = UNLABELED CIPSOv4 CALIPSO
[    0.751591] NetLabel:  unlabeled traffic allowed by default
[    0.752401] vgaarb: loaded
[    0.753314] clocksource: Switched to clocksource arch_sys_counter
[    0.758016] VFS: Disk quotas dquot_6.6.0
[    0.758160] VFS: Dquot-cache hash table entries: 512 (order 0, 4096 bytes)
[    0.759990] AppArmor: AppArmor Filesystem Enabled
[    0.777881] NET: Registered PF_INET protocol family
[    0.778307] IP idents hash table entries: 16384 (order: 5, 131072 bytes, linear)
[    0.780978] tcp_listen_portaddr_hash hash table entries: 512 (order: 1, 8192 bytes, linear)
[    0.781190] Table-perturb hash table entries: 65536 (order: 6, 262144 bytes, linear)
[    0.781280] TCP established hash table entries: 8192 (order: 4, 65536 bytes, linear)
[    0.781636] TCP bind hash table entries: 8192 (order: 6, 262144 bytes, linear)
[    0.782043] TCP: Hash tables configured (established 8192 bind 8192)
[    0.782561] MPTCP token hash table entries: 1024 (order: 2, 24576 bytes, linear)
[    0.782751] UDP hash table entries: 512 (order: 2, 16384 bytes, linear)
[    0.782838] UDP-Lite hash table entries: 512 (order: 2, 16384 bytes, linear)
[    0.783258] NET: Registered PF_UNIX/PF_LOCAL protocol family
[    0.785939] NET: Registered PF_XDP protocol family
[    0.786019] PCI: CLS 0 bytes, default 64
[    0.786544] Trying to unpack rootfs image as initramfs...
[    0.793151] Initialise system trusted keyrings
[    0.793433] Key type blacklist registered
[    0.794037] workingset: timestamp_bits=44 max_order=18 bucket_order=0
[    0.794239] zbud: loaded
[    0.795426] squashfs: version 4.0 (2009/01/31) Phillip Lougher
[    0.796461] fuse: init (API version 7.39)
[    0.800024] integrity: Platform Keyring initialized
[    0.909692] xor: measuring software checksum speed
[    0.917667]    8regs           :  1257 MB/sec
[    0.925605]    32regs          :  1261 MB/sec
[    0.934852]    arm64_neon      :  1081 MB/sec
[    0.934921] xor: using function: 32regs (1261 MB/sec)
[    0.934972] Key type asymmetric registered
[    0.935004] Asymmetric key parser 'x509' registered
[    0.935294] Block layer SCSI generic (bsg) driver version 0.4 loaded (major 247)
[    0.935888] io scheduler mq-deadline registered
[    0.935947] io scheduler kyber registered
[    0.936074] io scheduler bfq registered
[    0.953112] dma-pl330 ff1f0000.dma-controller: Loaded driver for PL330 DMAC-241330
[    0.953195] dma-pl330 ff1f0000.dma-controller: 	DBUFF-128x8bytes Num_Chans-8 Num_Peri-20 Num_Events-16
[    0.958204] Serial: 8250/16550 driver, 8 ports, IRQ sharing disabled
[    0.969278] ff110000.serial: ttyS0 at MMIO 0xff110000 (irq = 21, base_baud = 1500000) is a 16550A
[    0.970180] serial serial0: tty port ttyS0 registered
[    0.972805] ff130000.serial: ttyS2 at MMIO 0xff130000 (irq = 22, base_baud = 1500000) is a 16550A
[    0.972957] printk: legacy console [ttyS2] enabled
[    1.075842] Serial: AMBA driver
[    1.085501] rockchip-vop ff370000.vop: Adding to iommu group 0
[    1.114661] loop: module loaded
[    1.126257] tun: Universal TUN/TAP device driver, 1.6
[    1.127348] e1000e: Intel(R) PRO/1000 Network Driver
[    1.127844] e1000e: Copyright(c) 1999 - 2015 Intel Corporation.
[    1.128499] igb: Intel(R) Gigabit Ethernet Network Driver
[    1.129003] igb: Copyright (c) 2007-2014 Intel Corporation.
[    1.134406] dwc2 ff580000.usb: supply vusb_d not found, using dummy regulator
[    1.135477] dwc2 ff580000.usb: supply vusb_a not found, using dummy regulator
[    1.149708] dwc2 ff580000.usb: DWC OTG Controller
[    1.150254] dwc2 ff580000.usb: new USB bus registered, assigned bus number 1
[    1.150985] dwc2 ff580000.usb: irq 30, io mem 0xff580000
[    1.152284] usb usb1: New USB device found, idVendor=1d6b, idProduct=0002, bcdDevice= 6.08
[    1.153092] usb usb1: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    1.153812] usb usb1: Product: DWC OTG Controller
[    1.154266] usb usb1: Manufacturer: Linux 6.8.4-rk3328 dwc2_hsotg
[    1.154829] usb usb1: SerialNumber: ff580000.usb
[    1.156830] hub 1-0:1.0: USB hub found
[    1.157354] hub 1-0:1.0: 1 port detected
[    1.162201] xhci-hcd xhci-hcd.0.auto: xHCI Host Controller
[    1.162809] xhci-hcd xhci-hcd.0.auto: new USB bus registered, assigned bus number 2
[    1.163825] xhci-hcd xhci-hcd.0.auto: hcc params 0x0220fe64 hci version 0x110 quirks 0x0000008002000010
[    1.163850] ehci-platform ff5c0000.usb: EHCI Host Controller
[    1.163898] ohci-platform ff5d0000.usb: Generic Platform OHCI controller
[    1.163917] ehci-platform ff5c0000.usb: new USB bus registered, assigned bus number 3
[    1.164154] ehci-platform ff5c0000.usb: irq 31, io mem 0xff5c0000
[    1.164795] xhci-hcd xhci-hcd.0.auto: irq 29, io mem 0xff600000
[    1.165496] ohci-platform ff5d0000.usb: new USB bus registered, assigned bus number 4
[    1.166866] xhci-hcd xhci-hcd.0.auto: xHCI Host Controller
[    1.167547] ohci-platform ff5d0000.usb: irq 32, io mem 0xff5d0000
[    1.167847] xhci-hcd xhci-hcd.0.auto: new USB bus registered, assigned bus number 5
[    1.170388] xhci-hcd xhci-hcd.0.auto: Host supports USB 3.0 SuperSpeed
[    1.171702] usb usb2: New USB device found, idVendor=1d6b, idProduct=0002, bcdDevice= 6.08
[    1.172526] usb usb2: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    1.173204] usb usb2: Product: xHCI Host Controller
[    1.173701] usb usb2: Manufacturer: Linux 6.8.4-rk3328 xhci-hcd
[    1.174262] usb usb2: SerialNumber: xhci-hcd.0.auto
[    1.176448] hub 2-0:1.0: USB hub found
[    1.176971] hub 2-0:1.0: 1 port detected
[    1.177446] ehci-platform ff5c0000.usb: USB 2.0 started, EHCI 1.00
[    1.179497] usb usb5: We don't know the algorithms for LPM for this host, disabling LPM.
[    1.180758] usb usb5: New USB device found, idVendor=1d6b, idProduct=0003, bcdDevice= 6.08
[    1.181583] usb usb5: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    1.182260] usb usb5: Product: xHCI Host Controller
[    1.182717] usb usb5: Manufacturer: Linux 6.8.4-rk3328 xhci-hcd
[    1.183266] usb usb5: SerialNumber: xhci-hcd.0.auto
[    1.185182] hub 5-0:1.0: USB hub found
[    1.185732] hub 5-0:1.0: 1 port detected
[    1.187572] usbcore: registered new interface driver usb-storage
[    1.189861] usb usb3: New USB device found, idVendor=1d6b, idProduct=0002, bcdDevice= 6.08
[    1.190684] usb usb3: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    1.191370] usb usb3: Product: EHCI Host Controller
[    1.191834] usb usb3: Manufacturer: Linux 6.8.4-rk3328 ehci_hcd
[    1.192381] usb usb3: SerialNumber: ff5c0000.usb
[    1.196857] hub 3-0:1.0: USB hub found
[    1.196939] mousedev: PS/2 mouse device common for all mice
[    1.197417] hub 3-0:1.0: 1 port detected
[    1.199264] i2c_dev: i2c /dev entries driver
[    1.202464] i2c 1-0018: Fixed dependency cycle(s) with /i2c@ff160000/pmic@18/regulators/DCDC_REG4
[    1.213543] rk808-regulator rk808-regulator.2.auto: there is no dvs0 gpio
[    1.214259] rk808-regulator rk808-regulator.2.auto: there is no dvs1 gpio
[    1.229898] usb usb4: New USB device found, idVendor=1d6b, idProduct=0001, bcdDevice= 6.08
[    1.230743] usb usb4: New USB device strings: Mfr=3, Product=2, SerialNumber=1
[    1.231447] usb usb4: Product: Generic Platform OHCI controller
[    1.232050] usb usb4: Manufacturer: Linux 6.8.4-rk3328 ohci_hcd
[    1.232618] usb usb4: SerialNumber: ff5d0000.usb
[    1.234645] hub 4-0:1.0: USB hub found
[    1.235198] hub 4-0:1.0: 1 port detected
[    1.250955] rk808-rtc rk808-rtc.4.auto: registered as rtc0
[    1.253485] rk808-rtc rk808-rtc.4.auto: setting system clock to 2024-04-08T10:57:07 UTC (1712573827)
[    1.257188] input: rk805 pwrkey as /devices/platform/ff160000.i2c/i2c-1/1-0018/rk805-pwrkey.6.auto/input/input0
[    1.266735] dw_wdt ff1a0000.watchdog: No valid TOPs array specified
[    1.274458] sdhci: Secure Digital Host Controller Interface driver
[    1.275076] sdhci: Copyright(c) Pierre Ossman
[    1.275570] Synopsys Designware Multimedia Card Interface Driver
[    1.278534] sdhci-pltfm: SDHCI platform and OF driver helper
[    1.279913] dwmmc_rockchip ff510000.mmc: IDMAC supports 32-bit address mode.
[    1.280687] dwmmc_rockchip ff510000.mmc: Using internal DMA controller.
[    1.281407] dwmmc_rockchip ff510000.mmc: Version ID is 270a
[    1.281718] dwmmc_rockchip ff520000.mmc: IDMAC supports 32-bit address mode.
[    1.282087] dwmmc_rockchip ff510000.mmc: DW MMC controller at irq 46,32 bit host data width,256 deep fifo
[    1.282683] dwmmc_rockchip ff520000.mmc: Using internal DMA controller.
[    1.282704] dwmmc_rockchip ff520000.mmc: Version ID is 270a
[    1.283916] dwmmc_rockchip ff510000.mmc: allocated mmc-pwrseq
[    1.284272] dwmmc_rockchip ff520000.mmc: DW MMC controller at irq 45,32 bit host data width,256 deep fifo
[    1.284789] mmc_host mmc1: card is non-removable.
[    1.285398] dwmmc_rockchip ff5f0000.mmc: IDMAC supports 32-bit address mode.
[    1.287262] dwmmc_rockchip ff5f0000.mmc: Using internal DMA controller.
[    1.287901] dwmmc_rockchip ff5f0000.mmc: Version ID is 270a
[    1.288523] dwmmc_rockchip ff5f0000.mmc: DW MMC controller at irq 47,32 bit host data width,256 deep fifo
[    1.290610] ledtrig-cpu: registered to indicate activity on CPUs
[    1.291520] mmc_host mmc3: card is non-removable.
[    1.292311] mmc_host mmc2: card is non-removable.
[    1.292922] hid: raw HID events driver (C) Jiri Kosina
[    1.294081] usbcore: registered new interface driver usbhid
[    1.294637] usbhid: USB HID core driver
[    1.297711] mmc_host mmc1: Bus speed (slot 0) = 400000Hz (slot req 400000Hz, actual 400000HZ div = 0)
[    1.299615] hw perfevents: enabled with armv8_cortex_a53 PMU driver, 7 counters available
[    1.304617] drop_monitor: Initializing network drop monitor service
[    1.305051] mmc_host mmc3: Bus speed (slot 0) = 400000Hz (slot req 400000Hz, actual 400000HZ div = 0)
[    1.305791] mmc_host mmc2: Bus speed (slot 0) = 400000Hz (slot req 400000Hz, actual 400000HZ div = 0)
[    1.305914] NET: Registered PF_INET6 protocol family
[    1.342819] mmc3: queuing unknown CIS tuple 0x01 [d9 01 ff] (3 bytes)
[    1.351713] mmc3: queuing unknown CIS tuple 0x1a [01 01 00 02 07] (5 bytes)
[    1.355853] mmc3: queuing unknown CIS tuple 0x1b [c1 41 30 30 ff ff ff ff] (8 bytes)
[    1.357108] mmc_host mmc3: Bus speed (slot 0) = 25000000Hz (slot req 25000000Hz, actual 25000000HZ div = 0)
[    1.363882] mmc3: new SDIO card at address 0001
[    1.424008] mmc_host mmc1: Bus speed (slot 0) = 200000000Hz (slot req 208000000Hz, actual 200000000HZ div = 0)
[    1.432733] mmc_host mmc2: Bus speed (slot 0) = 50000000Hz (slot req 52000000Hz, actual 50000000HZ div = 0)
[    1.433860] mmc_host mmc2: Bus speed (slot 0) = 150000000Hz (slot req 150000000Hz, actual 150000000HZ div = 0)
[    1.469450] usb 3-1: new high-speed USB device number 2 using ehci-platform
[    1.632128] usb 3-1: New USB device found, idVendor=05e3, idProduct=0608, bcdDevice=85.37
[    1.632932] usb 3-1: New USB device strings: Mfr=0, Product=1, SerialNumber=0
[    1.633638] usb 3-1: Product: USB2.0 Hub
[    1.636088] hub 3-1:1.0: USB hub found
[    1.637014] hub 3-1:1.0: 4 ports detected
[    1.679220] dwmmc_rockchip ff520000.mmc: Successfully tuned phase to 198
[    1.680009] mmc2: new HS200 MMC card at address 0001
[    1.683144] mmcblk2: mmc2:0001 HBD08G 7.28 GiB
[    1.694286]  mmcblk2: p1 p2 p3 p4 p5 p6 p7 p8 p9
[    1.700229] mmcblk2boot0: mmc2:0001 HBD08G 4.00 MiB
[    1.706630] mmcblk2boot1: mmc2:0001 HBD08G 4.00 MiB
[    1.711978] mmcblk2rpmb: mmc2:0001 HBD08G 4.00 MiB, chardev (244:0)
[    1.975790] Freeing initrd memory: 11052K
[    2.062893] Segment Routing with IPv6
[    2.063415] In-situ OAM (IOAM) with IPv6
[    2.064003] NET: Registered PF_PACKET protocol family
[    2.065073] 8021q: 802.1Q VLAN Support v1.8
[    2.066194] Key type dns_resolver registered
[    2.088577] registered taskstats version 1
[    2.089281] Loading compiled-in X.509 certificates
[    2.148664] zswap: loaded using pool zstd/z3fold
[    2.151069] Key type .fscrypt registered
[    2.151484] Key type fscrypt-provisioning registered
[    2.157665] Btrfs loaded, zoned=yes, fsverity=yes
[    2.158474] Key type encrypted registered
[    2.158871] AppArmor: AppArmor sha256 policy hashing enabled
[    2.223897] rockchip-drm display-subsystem: bound ff370000.vop (ops vop_component_ops)
[    2.224828] dwhdmi-rockchip ff3c0000.hdmi: supply avdd-0v9 not found, using dummy regulator
[    2.226076] dwhdmi-rockchip ff3c0000.hdmi: supply avdd-1v8 not found, using dummy regulator
[    2.227420] dwhdmi-rockchip ff3c0000.hdmi: Detected HDMI TX controller v2.11a with HDCP (inno_dw_hdmi_phy2)
[    2.230144] dwhdmi-rockchip ff3c0000.hdmi: registered DesignWare HDMI I2C bus driver
[    2.231725] rockchip-drm display-subsystem: bound ff3c0000.hdmi (ops dw_hdmi_rockchip_ops)
[    2.234358] [drm] Initialized rockchip 1.0.0 20140818 for display-subsystem on minor 0
[    2.235302] rockchip-drm display-subsystem: [drm] Cannot find any crtc or sizes
[    2.236146] rockchip-drm display-subsystem: [drm] Cannot find any crtc or sizes
[    2.239015] of_cfs_init
[    2.239362] of_cfs_init: OK
[    2.239983] dwmmc_rockchip ff500000.mmc: IDMAC supports 32-bit address mode.
[    2.240732] clk: Disabling unused clocks
[    2.240731] dwmmc_rockchip ff500000.mmc: Using internal DMA controller.
[    2.241795] dwmmc_rockchip ff500000.mmc: Version ID is 270a
[    2.242474] dwmmc_rockchip ff500000.mmc: DW MMC controller at irq 54,32 bit host data width,256 deep fifo
[    2.257080] mmc_host mmc0: Bus speed (slot 0) = 400000Hz (slot req 400000Hz, actual 400000HZ div = 0)
[    2.274572] Freeing unused kernel memory: 4480K
[    2.275209] Run /init as init process
[    2.275562]   with arguments:
[    2.275577]     /init
[    2.275590]   with environment:
[    2.275600]     HOME=/
[    2.275612]     TERM=linux
[    2.473479] dwmmc_rockchip ff510000.mmc: Successfully tuned phase to 278
[    2.476822] mmc1: new ultra high speed SDR104 SDIO card at address 0001
[    3.015347] dwmmc_rockchip ff500000.mmc: Busy; trying anyway
[    3.359000] gpio-syscon ff100000.syscon:gpio: can't read the data register offset!
[    3.626958] rk_gmac-dwmac ff550000.ethernet: IRQ eth_wake_irq not found
[    3.627618] rk_gmac-dwmac ff550000.ethernet: IRQ eth_lpi not found
[    3.628543] rk_gmac-dwmac ff550000.ethernet: PTP uses main clock
[    3.629760] rk_gmac-dwmac ff550000.ethernet: clock input or output? (output).
[    3.630472] rk_gmac-dwmac ff550000.ethernet: Can not read property: tx_delay.
[    3.631143] rk_gmac-dwmac ff550000.ethernet: set tx_delay to 0x30
[    3.631720] rk_gmac-dwmac ff550000.ethernet: Can not read property: rx_delay.
[    3.632384] rk_gmac-dwmac ff550000.ethernet: set rx_delay to 0x10
[    3.633002] rk_gmac-dwmac ff550000.ethernet: integrated PHY? (yes).
[    3.647171] rk_gmac-dwmac ff550000.ethernet: init for RMII
[    3.705470] rk_gmac-dwmac ff550000.ethernet: User ID: 0x10, Synopsys ID: 0x35
[    3.706201] rk_gmac-dwmac ff550000.ethernet: 	DWMAC1000
[    3.706700] rk_gmac-dwmac ff550000.ethernet: DMA HW capability register supported
[    3.707393] rk_gmac-dwmac ff550000.ethernet: RX Checksum Offload Engine supported
[    3.708084] rk_gmac-dwmac ff550000.ethernet: COE Type 2
[    3.708576] rk_gmac-dwmac ff550000.ethernet: TX Checksum insertion supported
[    3.709227] rk_gmac-dwmac ff550000.ethernet: Wake-Up On Lan supported
[    3.714165] rk_gmac-dwmac ff550000.ethernet: Normal descriptors
[    3.714902] rk_gmac-dwmac ff550000.ethernet: Ring mode enabled
[    3.715667] rk_gmac-dwmac ff550000.ethernet: Enable RX Mitigation via HW Watchdog Timer
[    3.716711] rk_gmac-dwmac ff550000.ethernet: device MAC address 82:22:d5:6e:39:1e
[    3.752215] mmc_host mmc0: Timeout sending command (cmd 0x202000 arg 0x0 status 0x80202000)
[    4.482660] dwmmc_rockchip ff500000.mmc: Busy; trying anyway
[    5.214523] mmc_host mmc0: Timeout sending command (cmd 0x202000 arg 0x0 status 0x80202000)
[    5.950454] dwmmc_rockchip ff500000.mmc: Busy; trying anyway
[    6.681387] mmc_host mmc0: Timeout sending command (cmd 0x202000 arg 0x0 status 0x80202000)
[    7.443072] dwmmc_rockchip ff500000.mmc: Busy; trying anyway
[    8.175035] mmc_host mmc0: Timeout sending command (cmd 0x202000 arg 0x0 status 0x80202000)
[    8.285386] random: crng init done
[    8.886724] dwmmc_rockchip ff500000.mmc: Busy; trying anyway
[    9.618519] mmc_host mmc0: Timeout sending command (cmd 0x202000 arg 0x0 status 0x80202000)
[   10.353056] dwmmc_rockchip ff500000.mmc: Busy; trying anyway
[   11.085446] mmc_host mmc0: Timeout sending command (cmd 0x202000 arg 0x0 status 0x80202000)
[   11.846897] dwmmc_rockchip ff500000.mmc: Busy; trying anyway
[   12.578626] mmc_host mmc0: Timeout sending command (cmd 0x202000 arg 0x0 status 0x80202000)
[   13.300540] dwmmc_rockchip ff500000.mmc: Busy; trying anyway
[   14.031765] mmc_host mmc0: Timeout sending command (cmd 0x202000 arg 0x0 status 0x80202000)
[   14.775965] dwmmc_rockchip ff500000.mmc: Busy; trying anyway
[   15.507434] mmc_host mmc0: Timeout sending command (cmd 0x202000 arg 0x0 status 0x80202000)
[   15.643106] rockchip-pm-domain ff100000.syscon:power-controller: failed to get ack on domain 'hevc', val=0x88220
[   16.270810] dwmmc_rockchip ff500000.mmc: Busy; trying anyway
[   17.003999] mmc_host mmc0: Timeout sending command (cmd 0x202000 arg 0x0 status 0x80202000)
[   17.032564] mmc0: Skipping voltage switch
[   17.192695] mmc_host mmc0: Bus speed (slot 0) = 50000000Hz (slot req 50000000Hz, actual 50000000HZ div = 0)
[   17.193863] mmc0: new high speed SDXC card at address 13ab
[   17.196876] mmcblk0: mmc0:13ab SE128 115 GiB
[   17.214032]  mmcblk0: p1 p2
[   17.930066] EXT4-fs (mmcblk0p2): mounted filesystem bd2a6dbf-c55f-4d5e-8738-797e581ee7c9 ro with ordered data mode. Quota mode: none.
[   18.939913] systemd[1]: Inserted module 'autofs4'
[   19.074754] systemd[1]: systemd 252.22-1~deb12u1 running in system mode (+PAM +AUDIT +SELINUX +APPARMOR +IMA +SMACK +SECCOMP +GCRYPT -GNUTLS +OPENSSL +ACL +BLKID +CURL +ELFUTILS +FIDO2 +IDN2 -IDN +IPTC +KMOD +LIBCRYPTSETUP +LIBFDISK +PCRE2 -PWQUALITY +P11KIT +QRENCODE +TPM2 +BZIP2 +LZ4 +XZ +ZLIB +ZSTD -BPF_FRAMEWORK -XKBCOMMON +UTMP +SYSVINIT default-hierarchy=unified)
[   19.077893] systemd[1]: Detected architecture arm64.
[   19.088348] systemd[1]: Hostname set to <eaidk-310>.
[   19.333471] dw-apb-uart ff130000.serial: forbid DMA for kernel console
[   19.997438] systemd[1]: /etc/systemd/system/rc-local.service:12: Support for option SysVStartPriority= has been removed and it is ignored
[   20.384683] systemd[1]: Queued start job for default target graphical.target.
[   20.402306] systemd[1]: Created slice system-getty.slice - Slice /system/getty.
[   20.408122] systemd[1]: Created slice system-modprobe.slice - Slice /system/modprobe.
[   20.414288] systemd[1]: Created slice system-serial\x2dgetty.slice - Slice /system/serial-getty.
[   20.419837] systemd[1]: Created slice system-systemd\x2dfsck.slice - Slice /system/systemd-fsck.
[   20.424404] systemd[1]: Created slice user.slice - User and Session Slice.
[   20.426670] systemd[1]: Started systemd-ask-password-console.path - Dispatch Password Requests to Console Directory Watch.
[   20.429003] systemd[1]: Started systemd-ask-password-wall.path - Forward Password Requests to Wall Directory Watch.
[   20.432588] systemd[1]: Set up automount proc-sys-fs-binfmt_misc.automount - Arbitrary Executable File Formats File System Automount Point.
[   20.434730] systemd[1]: Expecting device dev-disk-by\x2duuid-6d0dcc44\x2db483\x2d4c9d\x2d910a\x2d1b060d59ab3f.device - /dev/disk/by-uuid/6d0dcc44-b483-4c9d-910a-1b060d59ab3f...
[   20.436791] systemd[1]: Expecting device dev-ttyS2.device - /dev/ttyS2...
[   20.438214] systemd[1]: Reached target cryptsetup.target - Local Encrypted Volumes.
[   20.439707] systemd[1]: Reached target integritysetup.target - Local Integrity Protected Volumes.
[   20.441436] systemd[1]: Reached target network-pre.target - Preparation for Network.
[   20.442974] systemd[1]: Reached target paths.target - Path Units.
[   20.444290] systemd[1]: Reached target remote-fs.target - Remote File Systems.
[   20.445736] systemd[1]: Reached target slices.target - Slice Units.
[   20.447153] systemd[1]: Reached target swap.target - Swaps.
[   20.448474] systemd[1]: Reached target veritysetup.target - Local Verity Protected Volumes.
[   20.451306] systemd[1]: Listening on systemd-fsckd.socket - fsck to fsckd communication Socket.
[   20.453700] systemd[1]: Listening on systemd-initctl.socket - initctl Compatibility Named Pipe.
[   20.457877] systemd[1]: Listening on systemd-journald-audit.socket - Journal Audit Socket.
[   20.460832] systemd[1]: Listening on systemd-journald-dev-log.socket - Journal Socket (/dev/log).
[   20.464059] systemd[1]: Listening on systemd-journald.socket - Journal Socket.
[   20.467266] systemd[1]: Listening on systemd-udevd-control.socket - udev Control Socket.
[   20.469982] systemd[1]: Listening on systemd-udevd-kernel.socket - udev Kernel Socket.
[   20.481089] systemd[1]: Mounting dev-hugepages.mount - Huge Pages File System...
[   20.492908] systemd[1]: Mounting dev-mqueue.mount - POSIX Message Queue File System...
[   20.505854] systemd[1]: Mounting sys-kernel-debug.mount - Kernel Debug File System...
[   20.518245] systemd[1]: Mounting sys-kernel-tracing.mount - Kernel Trace File System...
[   20.533573] systemd[1]: Starting keyboard-setup.service - Set the console keyboard layout...
[   20.547699] systemd[1]: Starting kmod-static-nodes.service - Create List of Static Device Nodes...
[   20.562224] systemd[1]: Starting modprobe@configfs.service - Load Kernel Module configfs...
[   20.578099] systemd[1]: Starting modprobe@dm_mod.service - Load Kernel Module dm_mod...
[   20.592609] systemd[1]: Starting modprobe@drm.service - Load Kernel Module drm...
[   20.607364] systemd[1]: Starting modprobe@efi_pstore.service - Load Kernel Module efi_pstore...
[   20.622423] systemd[1]: Starting modprobe@fuse.service - Load Kernel Module fuse...
[   20.638611] systemd[1]: Starting modprobe@loop.service - Load Kernel Module loop...
[   20.649725] device-mapper: uevent: version 1.0.3
[   20.652290] device-mapper: ioctl: 4.48.0-ioctl (2023-03-01) initialised: dm-devel@redhat.com
[   20.655408] systemd[1]: Starting systemd-fsck-root.service - File System Check on Root Device...
[   20.680399] systemd[1]: Starting systemd-journald.service - Journal Service...
[   20.703154] systemd[1]: Starting systemd-modules-load.service - Load Kernel Modules...
[   20.718964] systemd[1]: Starting systemd-udev-trigger.service - Coldplug All udev Devices...
[   20.755414] systemd[1]: Mounted dev-hugepages.mount - Huge Pages File System.
[   20.758792] systemd[1]: Mounted dev-mqueue.mount - POSIX Message Queue File System.
[   20.762526] systemd[1]: Mounted sys-kernel-debug.mount - Kernel Debug File System.
[   20.765896] systemd[1]: Mounted sys-kernel-tracing.mount - Kernel Trace File System.
[   20.774319] systemd[1]: Finished kmod-static-nodes.service - Create List of Static Device Nodes.
[   20.791774] systemd[1]: modprobe@configfs.service: Deactivated successfully.
[   20.794866] systemd[1]: Finished modprobe@configfs.service - Load Kernel Module configfs.
[   20.802835] systemd[1]: modprobe@dm_mod.service: Deactivated successfully.
[   20.809105] systemd[1]: Finished modprobe@dm_mod.service - Load Kernel Module dm_mod.
[   20.816228] systemd[1]: modprobe@drm.service: Deactivated successfully.
[   20.820209] systemd[1]: Finished modprobe@drm.service - Load Kernel Module drm.
[   20.826684] systemd[1]: modprobe@efi_pstore.service: Deactivated successfully.
[   20.831362] systemd[1]: Finished modprobe@efi_pstore.service - Load Kernel Module efi_pstore.
[   20.837812] systemd[1]: modprobe@fuse.service: Deactivated successfully.
[   20.840819] systemd[1]: Finished modprobe@fuse.service - Load Kernel Module fuse.
[   20.846691] systemd[1]: modprobe@loop.service: Deactivated successfully.
[   20.850032] systemd[1]: Finished modprobe@loop.service - Load Kernel Module loop.
[   20.854997] systemd[1]: Finished systemd-modules-load.service - Load Kernel Modules.
[   20.873439] systemd[1]: Mounting sys-fs-fuse-connections.mount - FUSE Control File System...
[   20.890371] systemd[1]: Mounting sys-kernel-config.mount - Kernel Configuration File System...
[   20.907293] systemd[1]: Started systemd-fsckd.service - File System Check Daemon to report status.
[   20.911822] systemd[1]: systemd-repart.service - Repartition Root Disk was skipped because no trigger condition checks were met.
[   20.931068] systemd[1]: Starting systemd-sysctl.service - Apply Kernel Variables...
[   20.955851] systemd[1]: Mounted sys-fs-fuse-connections.mount - FUSE Control File System.
[   20.959100] systemd[1]: Mounted sys-kernel-config.mount - Kernel Configuration File System.
[   21.093880] systemd[1]: Finished systemd-fsck-root.service - File System Check on Root Device.
[   21.121974] systemd[1]: Starting systemd-remount-fs.service - Remount Root and Kernel File Systems...
[   21.130110] systemd[1]: Finished systemd-sysctl.service - Apply Kernel Variables.
[   21.277973] EXT4-fs (mmcblk0p2): re-mounted bd2a6dbf-c55f-4d5e-8738-797e581ee7c9 r/w. Quota mode: none.
[   21.297551] systemd[1]: Finished systemd-remount-fs.service - Remount Root and Kernel File Systems.
[   21.300374] systemd[1]: systemd-firstboot.service - First Boot Wizard was skipped because of an unmet condition check (ConditionFirstBoot=yes).
[   21.302807] systemd[1]: systemd-pstore.service - Platform Persistent Storage Archival was skipped because of an unmet condition check (ConditionDirectoryNotEmpty=/sys/fs/pstore).
[   21.316965] systemd[1]: Starting systemd-random-seed.service - Load/Save Random Seed...
[   21.334067] systemd[1]: Starting systemd-sysusers.service - Create System Users...
[   21.443342] systemd[1]: Started systemd-journald.service - Journal Service.
[   21.554802] systemd-journald[380]: Received client request to flush runtime journal.
[   24.518776] rk3288-crypto ff060000.crypto: will run requests pump with realtime priority
[   24.519852] rk3288-crypto ff060000.crypto: Register ecb(aes) as ecb-aes-rk
[   24.520695] rk3288-crypto ff060000.crypto: Register cbc(aes) as cbc-aes-rk
[   24.524359] rk3288-crypto ff060000.crypto: Register ecb(des) as ecb-des-rk
[   24.525135] rk3288-crypto ff060000.crypto: Register cbc(des) as cbc-des-rk
[   24.526031] rk3288-crypto ff060000.crypto: Register ecb(des3_ede) as ecb-des3-ede-rk
[   24.526843] rk3288-crypto ff060000.crypto: Register cbc(des3_ede) as cbc-des3-ede-rk
[   24.527635] rk3288-crypto ff060000.crypto: Register sha1 as rk-sha1
[   24.528284] rk3288-crypto ff060000.crypto: Register sha256 as rk-sha256
[   24.528963] rk3288-crypto ff060000.crypto: Register md5 as rk-md5
[   24.776937] Bluetooth: Core ver 2.22
[   24.777581] NET: Registered PF_BLUETOOTH protocol family
[   24.778101] Bluetooth: HCI device and connection manager initialized
[   24.778708] Bluetooth: HCI socket layer initialized
[   24.779266] Bluetooth: L2CAP socket layer initialized
[   24.779881] Bluetooth: SCO socket layer initialized
[   24.808432] lima ff300000.gpu: gp - mali450 version major 0 minor 0
[   24.809652] lima ff300000.gpu: pp0 - mali450 version major 0 minor 0
[   24.810620] lima ff300000.gpu: pp1 - mali450 version major 0 minor 0
[   24.811396] lima ff300000.gpu: l2 cache 8K, 4-way, 64byte cache line, 128bit external bus
[   24.812185] lima ff300000.gpu: l2 cache 64K, 4-way, 64byte cache line, 128bit external bus
[   24.814392] lima ff300000.gpu: bus rate = 491520000
[   24.814895] lima ff300000.gpu: mod rate = 491520000
[   24.829510] [drm] Initialized lima 1.1.0 20191231 for ff300000.gpu on minor 1
[   24.868228] mc: Linux media interface: v0.10
[   24.884574] Bluetooth: HCI UART driver ver 2.3
[   24.885048] Bluetooth: HCI UART protocol H4 registered
[   24.885638] Bluetooth: HCI UART protocol BCSP registered
[   24.886359] Bluetooth: HCI UART protocol LL registered
[   24.886877] Bluetooth: HCI UART protocol ATH3K registered
[   24.887606] Bluetooth: HCI UART protocol Three-wire (H5) registered
[   24.888607] Bluetooth: HCI UART protocol Intel registered
[   24.900332] rk3328-codec ff410000.codec: spk_depop_time use default value.
[   24.922089] Bluetooth: HCI UART protocol Broadcom registered
[   24.922945] Bluetooth: HCI UART protocol QCA registered
[   24.923470] Bluetooth: HCI UART protocol AG6XX registered
[   24.924194] Bluetooth: HCI UART protocol Marvell registered
[   24.948666] videodev: Linux video capture interface: v2.00
[   25.087742] rockchip-rga ff390000.rga: HW Version: 0x04.00
[   25.093015] rockchip-rga ff390000.rga: Registered rockchip-rga as /dev/video0
[   25.134013] rockchip_vdec: module is from the staging directory, the quality is unknown, you have been warned.
[   25.136395] hantro-vpu ff350000.video-codec: Adding to iommu group 1
[   25.138338] rkvdec ff360000.video-codec: Adding to iommu group 2
[   25.139867] hantro-vpu ff350000.video-codec: registered rockchip,rk3328-vpu-dec as /dev/video1
[   25.223170] brcmfmac: F1 signature read @0x18000000=0x15264345
[   25.233092] brcmfmac: brcmf_fw_alloc_request: using brcm/brcmfmac43455-sdio for chip BCM4345/6
[   25.244719] brcmfmac mmc1:0001:1: Direct firmware load for brcm/brcmfmac43455-sdio.pine64,rock64.bin failed with error -2
[   25.474756] brcmfmac: brcmf_c_process_txcap_blob: no txcap_blob available (err=-2)
[   25.476495] brcmfmac: brcmf_c_preinit_dcmds: Firmware: BCM4345/6 wl0: Aug 25 2015 18:58:57 version 7.45.69 (r581703) FWID 01-24037f6e
[   28.813084] EXT4-fs (mmcblk0p1): mounted filesystem 6d0dcc44-b483-4c9d-910a-1b060d59ab3f r/w with ordered data mode. Quota mode: none.
[   29.386086] rk_gmac-dwmac ff550000.ethernet eth0: Register MEM_TYPE_PAGE_POOL RxQ-0
[   29.431830] rk_gmac-dwmac ff550000.ethernet eth0: PHY [stmmac-0:00] driver [Rockchip integrated EPHY] (irq=POLL)
[   29.441513] rk_gmac-dwmac ff550000.ethernet eth0: No Safety Features support found
[   29.442572] rk_gmac-dwmac ff550000.ethernet eth0: PTP not supported by HW
[   29.447087] rk_gmac-dwmac ff550000.ethernet eth0: configuring for phy/rmii link mode
[   30.954745] Bluetooth: BNEP (Ethernet Emulation) ver 1.3
[   30.955331] Bluetooth: BNEP filters: protocol multicast
[   30.955896] Bluetooth: BNEP socket layer initialized
[   31.494180] rk_gmac-dwmac ff550000.ethernet eth0: Link is Up - 100Mbps/Full - flow control rx/tx

```

## Issues

* The on board module `K019-CW43-DW(brcm 43455) wifi+bt` does not working on the mainline uboot.

## Credits

I would like to thank the armbian project for inspiring me.
