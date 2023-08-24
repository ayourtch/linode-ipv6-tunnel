# linode-ipv6-tunnel
IPv6 test lab using Linode VMs and OpenWRT

The IPv6 test lab allows to work around the limitation whereby the IPv6 may not be available in a given
location for testing, as well as attempting to establish a reproducible environment for such testing.

This is achieved by running two components:
- a "Cloud Router" - an OpenWRT instance, running in Linode, with a /56 of IPv6 delegated to it,
  serving as an endpoint to accept the connection from the test CPE.
- a "Test CPE" - an OpenWRT instance, running locally on a Raspberry Pi, which connects to the local
  ISP and uses its IPv4 connection as a carrier to establish the tunnel to the Cloud Router.

In order to maintain maximum resemblance to the real-world IPv6 setups, the tunnel is engineered
to maintain the end-to-end MTU of 1500 bytes. This is done by having two tunnels on top of each other:

- an outer L2TP tunnel with an MTU of 774 (750+20+4) bytes, which is used only as the transport for the packets
  of the inner tunnel after its encapsulation and possibly fragmentation
- an inner GRE tunnel with an MTU of 1500 bytes, with tunnel path MTU discovery disabled.

Such an arrangement results in outer packets always having the full IPv4 and UDP header - thus,
they are treated the same by all the firewalls/routers (contrary to IP fragments, which are
very often handled in the exception paths).

Also, due to the outer tunnel MTU being approximately the half of
the ethernet MTU, the 1500-bytes packets after encapsulation get fragmented into two
roughly equally sized chunks, again, with the aim of having the traffic look as
uniform as possible.

The tests show that such an approach allows to easily achieve performances in excess of 50Mbps
for the access speeds - as opposed to 1-2Mbps in the case of a simple L2TP tunnel.

Notice, that there is no encryption - the tunnel is treated the same way as any other "internet" traffic.

Installation:

1) Obtain an account on Linode (linode.com) and login

2) Select "Create" button and in the dropdown select "Linode"

3) Use Alpine OS (or any other, really - this will be just a temporary, but Alpine is probably smaller),
  select the Region that is closest to you, amd select the smallest "Shared CPU" Linode Plan (Nanode 1GB)
  Edit the Linode label to say "openwrt" or something along these lines. Select a temporary strong password.
  hit "Create Linode" button. You will see it appear yellow "Provisioning" status

4) Still in the page for this linode, select "Network" menu item. You will see the list of IP Addresses
at the bottom - one public IPv4, one link-local IPv6, and a SLAAC IPv6 address. Click "Add IP Address"
and under "IPv6", select "/56", and then hit "Allocate". You will see an "IPv6 - Range" appearing in the list
of the addresses for the linode. The Address column will read something like "2600:3c07:e002:f500::/56" -
copy it down, you will need it later.

5)  in the left-hand menu click "Images" and hit "Create Image" button on the right-top corner, in the
next page that pops up select "Upload Image" tab.

6) Type in a label for the image, e.g. "OpenWRT0001-ds", select the same Region that you used in (3),
and then browse to the file openwrt-x86-64-generic-ext4-combined.img.gz and upload it. After upload
it will change state to "Pending" - it will take a couple of minutes before it finishes processing.
After a while it will change to "0MB" - then you can refresh the page and you will see it is 121MB.

7) Click on the "..." on the right side of the image, and select "Rebuild existing linode". From
the drop-down select your Linode that you have just created in step 3 - probably called "openwrt",
and click "Restore image".

8) In the next screen, again select the image in the "Images" dropdown, select some strong password
(it is not used by openwrt image, but is required to enter), and type in the name of the linode
again at the bottom input, and hit "Rebuild Linode". The rebuild will take a few minutes and you will
see the percentage progress indicator.

9) When the reimage finishes, click the "Launch LISH Console". You should have a new window popping up,
with a somewhat scary output, as below. Do not worry - it is expected, as we need to change the boot
type for this image and reboot it.

```
[    2.222695]  fuseblk
[    2.222985]  udf
[    2.223298]  jfs
[    2.223580]  xfs
[    2.223859]  gfs2
[    2.224134]  gfs2meta
[    2.224416]  btrfs
[    2.224719]
[    2.225250] Kernel panic - not syncing: VFS: Unable to mount root fs on unkn)
[    2.226023] CPU: 0 PID: 1 Comm: swapper/0 Not tainted 6.2.9-x86_64-linode160b
[    2.227008] Hardware name: Linode Compute Instance, BIOS Not Specified
[    2.227847] Call Trace:
[    2.228182]  <TASK>
[    2.228500]  dump_stack_lvl+0x47/0x60
[    2.229015]  panic+0x316/0x330
[    2.229411]  mount_block_root+0x278/0x280
[    2.229874]  ? __pfx_ignore_unknown_bootoption+0x10/0x10
[    2.230442]  prepare_namespace+0x156/0x180
[    2.230917]  kernel_init_freeable+0x2d6/0x320
[    2.231416]  ? __pfx_kernel_init+0x10/0x10
[    2.231891]  kernel_init+0x1a/0x140
[    2.232320]  ret_from_fork+0x2c/0x50
[    2.232755]  </TASK>
[    2.233890] Kernel Offset: 0x31a00000 from 0xffffffff81000000 (relocation ra)
[    2.234924] ---[ end Kernel panic - not syncing: VFS: Unable to mount root f-
```

10) Go to "Configurations" tab for the linode, and you will see at the bottom a table "Config - Disks - Network Interfaces".
Hit "Edit" next to the line that you see there with the configuration.

11) Scroll to "Select a Kernel", and in dropdown select "Direct Disk". Do not change anything else, and hit "Save Changes".

12) Now hit "Reboot Linode" and wait for the linode to come back up. It will take a few minutes.

13) While waiting, you can go and download the Raspberry Pi imager from https://www.raspberrypi.com/software/ - we will need it soon.

14) On the Linode, click "Launch LISH Console" button again, and this time you will see something as follows
after hitting ENTER:

```
[    7.606695] i2c_dev: i2c /dev entries driver
[    7.614527] e1000e: Intel(R) PRO/1000 Network Driver
[    7.615999] e1000e: Copyright(c) 1999 - 2015 Intel Corporation.
[    7.619090] Intel(R) 2.5G Ethernet Linux Driver
[    7.620152] Copyright(c) 2018 Intel Corporation.
[    7.627419] xt_time: kernel timezone is -0000
[    7.637603] PPP generic driver version 2.4.2
[    7.638903] NET: Registered PF_PPPOX protocol family
[    7.640321] wireguard: WireGuard 1.0.0 loaded. See www.wireguard.com for information.
[    7.642006] wireguard: Copyright (C) 2015-2019 Jason A. Donenfeld <Jason@zx2c4.com>. All Rights Reserved.
[    7.644293] l2tp_ppp: PPPoL2TP kernel driver, V2.0
[    7.648994] kmodloader: done loading kernel modules from /etc/modules.d/*
[    9.196838] 8021q: adding VLAN 0 to HW filter on device eth0



BusyBox v1.36.1 (2023-06-26 11:20:39 UTC) built-in shell (ash)

  _______                     ________        __
 |       |.-----.-----.-----.|  |  |  |.----.|  |_
 |   -   ||  _  |  -__|     ||  |  |  ||   _||   _|
 |_______||   __|_____|__|__||________||__|  |____|
          |__| W I R E L E S S   F R E E D O M
 -----------------------------------------------------
 OpenWrt 23.05.0-rc2, r23228-cd17d8df2a
 -----------------------------------------------------
root@OpenWrtHub:/# 
```


15) (optional): set the SECURE root password. If you want to be able to login to the cloud hub using ssh or https,
you will need to change the root password using "passwd". TAKE CARE TO CHOOSE A VERY SECURE PASSWORD. This is
an ssh on the open internet, and fail2ban is not fully configured in this image.

16) Complete the setup and prepare an image for your Raspberry Pi by typing /root/finalize-setup and following the prompts.

After the interactive portion the script will take a few seconds to prepare the image for the on-premise CPE,
and will output something along the lines of:

```
************************************************************
Please download the file from https://172.232.54.224/cpe.img.gz
File sha256: b48a56cb0c0920797d4b014d1bb28118300ac9134dd909ef4dcb1e4815bbc5f1  /www/cpe.img.gz
************************************************************
```

Save this image and use the Raspberry Pi imager to write it onto the flash for Raspberry (select the "Use custom" item
in the "Operating System" menu. You may need to ungzip image using "gzip -d cpe.img.gz")

17) Insert flash into the Raspberry Pi, and connect the uplink (either the iPhone or Ethernet), and boot it.

18) Connect to the WiFi that you have configured on the raspberry pi, and point the browser to ttps://192.168.1.1/

19) Fire up the browser and try going to https://testipv6.com - it should show you that you have both IPv4 and IPv6 with Akamai/Linode.




