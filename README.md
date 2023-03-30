# mcp251x.c
can: mcp251x: mcp251x_setup: Message Acceptance Filters and Masks for 11/29 bit CAN ID

Note that the data match / filter function for standard CAN ID is not implemented.

Only CAN ID mask / filter is applied.


mcp251x CAN driver with hardware filtering for the Raspberry Pi

Based on patch from:
https://support.criticallink.com/redmine/attachments/download/14913/0001-can-mcp251x-Add-ability-to-use-CAN-ID-hw-filter.patch

# Compiling the driver
First download the kernel headers:
```
$ sudo apt-get install raspberrypi-kernel-headers
```
Now download this repo
```
$ git clone https://github.com/craigpeacock/mcp251x.git 
```
Make the kernel module:
```
$ cd mcp251x
$ make
```
If successful it should display something similar to below and create you a mcp251x.ko file. 
```
make -C /lib/modules/4.19.75+/build M=/home/pi/mcp251x modules
make[1]: Entering directory '/usr/src/linux-headers-4.19.75+'
  CC [M]  /home/pi/mcp251x/mcp251x.o
  Building modules, stage 2.
  MODPOST 1 modules
  CC      /home/pi/mcp251x/mcp251x.mod.o
  LD [M]  /home/pi/mcp251x/mcp251x.ko
make[1]: Leaving directory '/usr/src/linux-headers-4.19.75+'
```

# Testing
To test the driver, remove the old one (if loaded) and insert your new module into the kernel using:
```
$ sudo rmmod mcp251x
$ sudo insmod mcp251x.ko
```
and check your kernel messages:
```
$ dmesg | grep -i mcp251x
[ 1396.047462] mcp251x: loading out-of-tree module taints kernel.
[ 1582.644169] mcp251x spi0.0 can0: MCP2515 successfully initialized.
```
Parameters to specify CAN filters and masks can be sent to the kernel using insmod:
```
$ sudo insmod mcp251x.ko rxbn_op_mode=1,1 rxbn_filters=0x7E8,0x762 rxbn_mask=0xFFF,0xFFF
```
Modinfo can be used to determine parameters:
```
$ modinfo mcp251x.ko
filename:       //mcp251x.ko
license:        GPL v2
description:    Microchip 251x/25625 CAN driver
author:         Chris Elston <celston@katalix.com>, Christian Pellegrin <chripell@evolware.org>
alias:          of:N*T*Cmicrochip,mcp25625C*
alias:          of:N*T*Cmicrochip,mcp25625
alias:          of:N*T*Cmicrochip,mcp2515C*
alias:          of:N*T*Cmicrochip,mcp2515
alias:          of:N*T*Cmicrochip,mcp2510C*
alias:          of:N*T*Cmicrochip,mcp2510
alias:          spi:mcp25625
alias:          spi:mcp2515
alias:          spi:mcp2510
depends:
intree:         Y
vermagic:       4.9.272 SMP preempt mod_unload ARMv7 p2v8
parm:           mcp251x_enable_dma:Enable SPI DMA. Default: 0 (Off) (int)
parm:           rxbn_op_mode:0 = (default) MCP2515 hardware filtering will be disabled for receive buffer n (0 or 1). rxb0 controls filters 0 and 1, rxb1 controls filters 2-5 Note there is kernel level filtering, but for high traffic scenarios kernel may not be able to keep up. 1 = DO NOT USE, Setting the RXM[1:0] bits to ‘01’ or ‘10’ is not recommended. 2 = DO NOT USE, Setting the RXM[1:0] bits to ‘01’ or ‘10’ is not recommended 3 = use rxbn_mask and rxbn filters, and accept ext or std CAN ids. (array of int)
parm:           rxbn_mask:Mask used for receive buffer n if rxbn_op_mode  is 3. Bits 28-18 for std ids. Bits 17-0 for ext ids. (array of int)
parm:           rxbn_filters:Filter used for receive buffer n if rxbn_op_mode is 1, 2 or 3. Bits 10-0 for std ids. Bits 29-11 for ext ids (also need to set bit 30 for ext id filtering). Note that filters 0 and 1 correspond to rxbn_op_mode[0] and rxbn_mask[0], while filters 2-5 corresponds to rxbn_op_mode[1] and rxbn_mask[1] (array of int)
parm:           rxbn_exide:Extended Identifier Enable bit used for receive buffer n (0 or 1).The RXM[1:0] bits (RXBnCTRL[6:5]) set special Receive modes. These bits are cleared to ‘00’ to enable acceptance filters. In this case, the determination of whether or not to receive standard or extended messages is determined by the EXIDE bit (RFXnSIDL[3]).0 = disable EXIDE (default), 1 = enable EXIDE to filter extended IDs (array of bool)
```
When the driver is loaded, you may check the current parameters via the sysfs. e.g.
```
$ cat /sys/module/mcp251x/parameters/rxbn_op_mode
1,1
$ cat /sys/module/mcp251x/parameters/rxbn_filters
2024,1890,0,0,0,0
$ cat /sys/module/mcp251x/parameters/rxbn_mask
4095,4095
```
# Installation
To install the driver:
```
sudo cp mcp251x.ko /lib/modules/$(uname -r)/kernel/drivers/net/can/spi/
```
Parameters can be passed to the driver using the modprobe daemon. Create a new file /etc/modprobe.d/mcp251x.conf and add the following:

11-bit CAN frame
```
options mcp251x rxbn_op_mode=3,3 rxbn_filters=0x7E8,0x762 rxbn_mask=0xFFF,0xFFF
```
29-bit CAN frame
```
options mcp251x rxbn_op_mode=3,3 rxbn_exide=0,1 rxbn_mask=0x1FFFFFFF,0x1FFFFFFF rxbn_filters=0x1851E0D0,0x1852E0D0
```
