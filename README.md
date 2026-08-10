# RM520N AX3000 WiFi 6 Modem 5G Router

I bought this router from Amazon a year ago, but the selling company does not provide a support page. 
Communicating via the "Message the seller" is painful, so I am collecting here bits and pieces in the hope that they can be useful to others.

***A Strong Disclaimer***: Please use any information, code or setting from this site at your own risk. 
I am not in a position to check correctness or applicability to any device.

## Latest OS
The router runs a modified version of OpenWRT. The seller, via private a communication sent me [this update file](https://github.com/pitsianis/squarewiz-router/blob/main/WT7981P-7.8.1-251212-124428-sysupgrade.bin). 

After the upgrade, the system identifies as
```shell
Linux OpenWrt 5.4.270 #0 SMP Fri Dec 12 04:19:51 2025 aarch64 GNU/Linux
```
with release information
```shell
DISTRIB_ID='OpenWrt'
DISTRIB_RELEASE='21.02-SNAPSHOT'
DISTRIB_REVISION='7.8.1'
DISTRIB_TARGET='mediatek/mt7981'
DISTRIB_ARCH='aarch64_cortex-a53'
DISTRIB_DESCRIPTION='OpenWrt 21.02-SNAPSHOT 7.8.1'
DISTRIB_TAINTS='no-all busybox'
```
