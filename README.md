# Open5GS EPC & srsRAN 4G with ZeroMQ UE / RAN Sample Configuration - eUPF(eBPF/XDP UPF(PGW-U))
This describes a simple configuration for working Open5GS EPC and eUPF(eBPF/XDP UPF(PGW-U)).
In particular, see [here](https://github.com/s5uishida/install_eupf) for eUPF.

---

### [Sample Configurations and Miscellaneous for Mobile Network](https://github.com/s5uishida/sample_config_misc_for_mobile_network)

---

<a id="toc"></a>

## Table of Contents

- [Overview of Open5GS CUPS-enabled EPC Simulation Mobile Network](#overview)
- [Changes in configuration files of Open5GS EPC, eUPF and srsRAN 4G ZMQ UE / RAN](#changes)
  - [Changes in configuration files of Open5GS EPC C-Plane](#changes_cp)
  - [Changes in configuration files of Open5GS EPC U-Plane](#changes_up)
  - [Changes in configuration files of eUPF](#changes_eupf)
  - [Changes in configuration files of srsRAN 4G ZMQ UE / RAN](#changes_srs)
    - [Changes in configuration files of RAN](#changes_ran)
    - [Changes in configuration files of UE](#changes_ue)
- [Network settings of Open5GS EPC, eUPF and srsRAN 4G ZMQ UE / RAN](#network_settings)
  - [Network settings of eUPF and Data Network Gateway](#network_settings_up)
- [Build Open5GS, eUPF and srsRAN 4G ZMQ UE / RAN](#build)
- [Run Open5GS EPC, eUPF and srsRAN 4G ZMQ UE / RAN](#run)
  - [Run eUPF](#run_eupf)
  - [Run Open5GS EPC U-Plane](#run_up)
  - [Run Open5GS EPC C-Plane](#run_cp)
  - [Run srsRAN 4G ZMQ RAN](#run_ran)
  - [Run srsRAN 4G ZMQ UE](#run_ue)
- [Ping google.com](#ping)
  - [Case for going through PDN 10.45.0.0/16](#ping_1)
- [Changelog (summary)](#changelog)

<a id="overview"></a>

## Overview of Open5GS CUPS-enabled EPC Simulation Mobile Network

This describes a simple configuration of C-Plane, eUPF and Data Network Gateway for Open5GS EPC.
**Note that this configuration is implemented with Virtualbox VMs.**

The following minimum configuration was set as a condition.
- One SGW-U/UPF(PGW-U) and Data Network Gateway
- One UE and one APN

The built simulation environment is as follows.

<img src="./images/network-overview.png" title="./images/network-overview.png" width=1000px></img>

The EPC / eBPF/XDP PGW-U / UE / RAN used are as follows.
- EPC - Open5GS v2.6.6 (2023.11.04) - https://github.com/open5gs/open5gs
- eBPF/XDP PGW-U - eUPF `120-upf-ftup-fteid` branch (2023.11.22) - https://github.com/edgecomllc/eupf
- UE / RAN - srsRAN 4G (2023.06.19) - https://github.com/srsran/srsRAN_4G

Each VMs are as follows.  
| VM | SW & Role | IP address | OS | CPU<br>(Min) | Memory<br>(Min) | HDD<br>(Min) |
| --- | --- | --- | --- | --- | --- | --- |
| VM1 | Open5GS EPC C-Plane | 192.168.0.111/24 | Ubuntu 22.04 | 1 | 1GB | 20GB |
| VM2 | Open5GS EPC U-Plane(SGW-U) | 192.168.0.112/24 | Ubuntu 22.04 | 1 | 1GB | 20GB |
| VM-UP | eUPF U-Plane(PGW-U) | 192.168.0.151/24 | Ubuntu 22.04 | 1 | 2GB | 20GB |
| VM-DN | Data Network Gateway  | 192.168.0.152/24 | Ubuntu 22.04 | 1 | 1GB | 10GB |
| VM3 | srsRAN 4G ZMQ RAN (eNodeB) | 192.168.0.121/24 | Ubuntu 22.04 | 1 | 2GB | 10GB |
| VM4 | srsRAN 4G ZMQ UE | 192.168.0.122/24 | Ubuntu 22.04 | 1 | 2GB | 10GB |

The network interfaces of each VM are as follows.
| VM | Device | Network Adapter | IP address | Interface | XDP |
| --- | --- | --- | --- | --- | --- |
| VM1 | enp0s3 | NAT(default) | 10.0.2.15/24 | (VM default NW) | -- |
| | enp0s8 | Bridged Adapter | 192.168.0.111/24 | (Mgmt NW) | -- |
| | enp0s9 | NAT Network | 192.168.14.111/24 | Sxb (N4 for 5GC) | -- |
| VM2 | enp0s3 | NAT(default) | 10.0.2.15/24 | (VM default NW) | -- |
| | enp0s8 | Bridged Adapter | 192.168.0.112/24 | (Mgmt NW) | -- |
| | enp0s9 | NAT Network | 192.168.13.112/24 | S1-U,S5u (N3 for 5GC) | -- |
| VM-UP | ~~enp0s3~~ | ~~NAT(default)~~ | ~~10.0.2.15/24~~ | ~~(VM default NW)~~ ***down*** | -- |
| | enp0s8 | Bridged Adapter | 192.168.0.151/24 | (Mgmt NW) | -- |
| | enp0s9 | NAT Network | 192.168.13.151/24 | S5u (N3 for 5GC) | x |
| | enp0s10 | NAT Network | 192.168.14.151/24 | Sxb (N4 for 5GC) | -- |
| | enp0s16 | NAT Network | 192.168.16.151/24 | SGi (N6 for 5GC) | x |
| VM-DN | enp0s3 | NAT(default) | 10.0.2.15/24 | (VM default NW) | -- |
| | enp0s8 | Bridged Adapter | 192.168.0.152/24 | (Mgmt NW) | -- |
| | enp0s9 | NAT Network | 192.168.16.152/24 | SGi (N6 for 5GC),<br>***default GW for VM-UP*** | -- |
| VM3 | enp0s3 | NAT(default) | 10.0.2.15/24 | (VM default NW) | -- |
| | enp0s8 | Bridged Adapter | 192.168.0.121/24 | (Mgmt NW) | -- |
| | enp0s9 | NAT Network | 192.168.13.121/24 | S1-U (N3 for 5GC) | -- |
| VM4 | enp0s3 | NAT(default) | 10.0.2.15/24 | (VM default NW) | -- |
| | enp0s8 | Bridged Adapter | 192.168.0.122/24 | (Mgmt NW) | -- |

NAT networks of Virtualbox  are as follows.
| Network Name | Network CIDR | Note |
| --- | --- | --- |
| N3 | 192.168.13.0/24 | S1-U,S5u for EPC |
| N4 | 192.168.14.0/24 | Sxb for EPC |
| N6 | 192.168.16.0/24 | SGi for EPC |

Subscriber Information (other information is the same) is as follows.  
| UE | IMSI | APN | OP/OPc |
| --- | --- | --- | --- |
| UE | 001010000000100 | internet | OPc |

I registered these information with the Open5GS WebUI.
In addition, [3GPP TS 35.208](https://www.3gpp.org/DynaReport/35208.htm) "4.3 Test Sets" is published by 3GPP as test data for the 3GPP authentication and key generation functions (MILENAGE).

The PDN is as follows.
| PDN | APN | TUNnel interface of UE |
| --- | --- | --- |
| 10.45.0.0/16 | internet | tun_srsue |

The main information of eNodeB is as follows.
| MCC | MNC | TAC | eNodeB ID | Cell ID | E-UTRAN Cell ID |
| --- | --- | --- | --- | --- | --- |
| 001 | 01 | 1 | 0x19b | 0x01 | 0x19b01 |

<a id="changes"></a>

## Changes in configuration files of Open5GS EPC, eUPF and srsRAN 4G ZMQ UE / RAN

Please refer to the following for building Open5GS, eUPF and srsRAN 4G ZMQ respectively.
- Open5GS v2.6.6 (2023.11.04) - https://open5gs.org/open5gs/docs/guide/02-building-open5gs-from-sources/
- eUPF `120-upf-ftup-fteid` branch (2023.11.22) - https://github.com/s5uishida/install_eupf
- srsRAN 4G (2023.06.19) - https://docs.srsran.com/projects/4g/en/latest/

<a id="changes_cp"></a>

### Changes in configuration files of Open5GS EPC C-Plane

The following parameters including APN can be used in the logic that selects SGW-U as the connection destination by PFCP.

- APN
- TAC (Tracking Area Code)
- e_CellID

For the sake of simplicity, I used only APN this time. Please refer to [here](https://github.com/open5gs/open5gs/pull/560#issue-483001043) for the logic to select SGW-U.

- `open5gs/install/etc/open5gs/mme.yaml`
```diff
--- mme.yaml.orig       2023-11-10 20:39:04.000000000 +0900
+++ mme.yaml    2023-11-10 22:11:20.367130602 +0900
@@ -321,7 +321,7 @@
 mme:
     freeDiameter: /root/open5gs/install/etc/freeDiameter/mme.conf
     s1ap:
-      - addr: 127.0.0.2
+      - addr: 192.168.0.111
     gtpc:
       - addr: 127.0.0.2
     metrics:
@@ -329,14 +329,14 @@
         port: 9090
     gummei:
       plmn_id:
-        mcc: 999
-        mnc: 70
+        mcc: 001
+        mnc: 01
       mme_gid: 2
       mme_code: 1
     tai:
       plmn_id:
-        mcc: 999
-        mnc: 70
+        mcc: 001
+        mnc: 01
       tac: 1
     security:
         integrity_order : [ EIA2, EIA1, EIA0 ]
```
- `open5gs/install/etc/open5gs/sgwc.yaml`
```diff
--- sgwc.yaml.orig      2023-11-10 20:39:04.000000000 +0900
+++ sgwc.yaml   2023-11-10 22:11:37.213725908 +0900
@@ -81,7 +81,7 @@
     gtpc:
       - addr: 127.0.0.3
     pfcp:
-      - addr: 127.0.0.3
+      - addr: 192.168.0.111
 
 #
 #  <PFCP Client>>
@@ -130,7 +130,8 @@
 #
 sgwu:
     pfcp:
-      - addr: 127.0.0.6
+      - addr: 192.168.0.112
+        apn: internet
 
 #
 #  o Disable use of IPv4 addresses (only IPv6)
```
- `open5gs/install/etc/open5gs/smf.yaml`
```diff
--- smf.yaml.orig       2023-11-10 20:39:04.000000000 +0900
+++ smf.yaml    2023-11-10 22:12:02.452682602 +0900
@@ -598,29 +598,21 @@
 #      maximum_integrity_protected_data_rate_downlink: bitrate64kbs|maximum-UE-rate
 #
 smf:
-    sbi:
-      - addr: 127.0.0.4
-        port: 7777
     pfcp:
-      - addr: 127.0.0.4
-      - addr: ::1
+      - addr: 192.168.14.111
     gtpc:
       - addr: 127.0.0.4
-      - addr: ::1
     gtpu:
-      - addr: 127.0.0.4
-      - addr: ::1
+      - addr: 192.168.14.111
     metrics:
       - addr: 127.0.0.4
         port: 9090
     subnet:
       - addr: 10.45.0.1/16
-      - addr: 2001:db8:cafe::1/48
+        dnn: internet
     dns:
       - 8.8.8.8
       - 8.8.4.4
-      - 2001:4860:4860::8888
-      - 2001:4860:4860::8844
     mtu: 1400
     ctf:
       enabled: auto
@@ -690,10 +682,6 @@
 #          l_linger: 10
 #
 #
-scp:
-    sbi:
-      - addr: 127.0.1.10
-        port: 7777
 
 #
 #  <SBI Client>>
@@ -808,7 +796,8 @@
 #
 upf:
     pfcp:
-      - addr: 127.0.0.7
+      - addr: 192.168.14.151
+        dnn: internet
 
 #
 #  o Disable use of IPv4 addresses (only IPv6)
```

<a id="changes_up"></a>

### Changes in configuration files of Open5GS EPC U-Plane

- `open5gs/install/etc/open5gs/sgwu.yaml`
```diff
--- sgwu.yaml.orig      2023-11-10 20:39:04.000000000 +0900
+++ sgwu.yaml   2023-11-22 20:53:06.665341606 +0900
@@ -114,9 +114,9 @@
 #
 sgwu:
     pfcp:
-      - addr: 127.0.0.6
+      - addr: 192.168.0.112
     gtpu:
-      - addr: 127.0.0.6
+      - addr: 192.168.13.112
 
 #
 #  <PFCP Client>>
```

<a id="changes_eupf"></a>

### Changes in configuration files of eUPF

See [here](https://github.com/s5uishida/install_eupf#create-configuration-file) for the original files.

- `eupf/config.yml`  
There is no change.

<a id="changes_srs"></a>

### Changes in configuration files of srsRAN 4G ZMQ UE / RAN

<a id="changes_ran"></a>

#### Changes in configuration files of RAN

- `srsRAN_4G/build/srsenb/enb.conf`
```diff
--- enb.conf.example    2023-05-02 10:51:20.000000000 +0900
+++ enb.conf    2023-11-22 20:59:03.361310402 +0900
@@ -22,9 +22,9 @@
 enb_id = 0x19B
 mcc = 001
 mnc = 01
-mme_addr = 127.0.1.100
-gtp_bind_addr = 127.0.1.1
-s1c_bind_addr = 127.0.1.1
+mme_addr = 192.168.0.111
+gtp_bind_addr = 192.168.13.121
+s1c_bind_addr = 192.168.0.121
 s1c_bind_port = 0
 n_prb = 50
 #tm = 4
@@ -80,8 +80,8 @@
 #time_adv_nsamples = auto
 
 # Example for ZMQ-based operation with TCP transport for I/Q samples
-#device_name = zmq
-#device_args = fail_on_disconnect=true,tx_port=tcp://*:2000,rx_port=tcp://localhost:2001,id=enb,base_srate=23.04e6
+device_name = zmq
+device_args = fail_on_disconnect=true,tx_port=tcp://192.168.0.121:2000,rx_port=tcp://192.168.0.122:2001,id=enb,base_srate=23.04e6
 
 #####################################################################
 # Packet capture configuration
```
- `srsRAN_4G/build/srsenb/rr.conf`
```diff
--- rr.conf.example     2023-05-02 10:51:20.000000000 +0900
+++ rr.conf     2023-05-02 11:52:54.000000000 +0900
@@ -55,7 +55,7 @@
   {
     // rf_port = 0;
     cell_id = 0x01;
-    tac = 0x0007;
+    tac = 0x0001;
     pci = 1;
     // root_seq_idx = 204;
     dl_earfcn = 3350;
```

<a id="changes_ue"></a>

#### Changes in configuration files of UE

- `srsRAN_4G/build/srsue/ue.conf`
```diff
--- ue.conf.example     2023-05-02 10:51:20.000000000 +0900
+++ ue.conf     2023-05-02 12:01:28.000000000 +0900
@@ -42,8 +42,8 @@
 #continuous_tx     = auto
 
 # Example for ZMQ-based operation with TCP transport for I/Q samples
-#device_name = zmq
-#device_args = tx_port=tcp://*:2001,rx_port=tcp://localhost:2000,id=ue,base_srate=23.04e6
+device_name = zmq
+device_args = tx_port=tcp://192.168.0.122:2001,rx_port=tcp://192.168.0.121:2000,id=ue,base_srate=23.04e6
 
 #####################################################################
 # EUTRA RAT configuration
@@ -139,9 +139,9 @@
 [usim]
 mode = soft
 algo = milenage
-opc  = 63BFA50EE6523365FF14C1F45F88737D
-k    = 00112233445566778899aabbccddeeff
-imsi = 001010123456780
+opc  = E8ED289DEBA952E4283B54E88E6183CA
+k    = 465B5CE8B199B49FAA5F0A2EE238A6BC
+imsi = 001010000000100
 imei = 353490069873319
 #reader =
 #pin  = 1234
@@ -180,8 +180,8 @@
 #                      Supported: 0 - NULL, 1 - Snow3G, 2 - AES, 3 - ZUC
 #####################################################################
 [nas]
-#apn = internetinternet
-#apn_protocol = ipv4
+apn = internet
+apn_protocol = ipv4
 #user = srsuser
 #pass = srspass
 #force_imsi_attach = false
```

<a id="network_settings"></a>

## Network settings of Open5GS 5GC, eUPF and UERANSIM UE / RAN

<a id="network_settings_up"></a>

### Network settings of eUPF and Data Network Gateway

See [this1](https://github.com/s5uishida/install_eupf#setup-eupf-on-vm-up) and [this2](https://github.com/s5uishida/install_eupf#setup-data-network-gateway-on-vm-dn).

<a id="build"></a>

## Build Open5GS, eUPF and srsRAN 4G ZMQ UE / RAN

Please refer to the following for building Open5GS, eUPF and UERANSIM respectively.
- Open5GS v2.6.6 (2023.11.04) - https://open5gs.org/open5gs/docs/guide/02-building-open5gs-from-sources/
- eUPF `120-upf-ftup-fteid` branch (2023.11.22)- https://github.com/s5uishida/install_eupf
- srsRAN 4G (2023.06.19) - https://docs.srsran.com/projects/4g/en/latest/

Install MongoDB on Open5GS EPC C-Plane machine.
[MongoDB Compass](https://www.mongodb.com/products/compass) is a convenient tool to look at the MongoDB database.

<a id="run"></a>

## Run Open5GS EPC, eUPF and srsRAN 4G ZMQ UE / RAN

First run eUPF and EPC U-Plane(SGW-U), then EPC C-Plane, the RAN, and the UE.

<a id="run_eupf"></a>

### Run eUPF

See [this](https://github.com/s5uishida/install_eupf#run-eupf-on-vm-up).

<a id="run_up"></a>

### Run Open5GS EPC U-Plane

```
./install/bin/open5gs-sgwud &
```

<a id="run_cp"></a>

### Run Open5GS EPC C-Plane

```
./install/bin/open5gs-mmed &
./install/bin/open5gs-sgwcd &
./install/bin/open5gs-smfd &
./install/bin/open5gs-hssd &
./install/bin/open5gs-pcrfd &
```
The PFCP association log between eUPF and Open5GS SMF is as follows.
```
2023/11/22 21:22:13 INF Got Association Setup Request from: 192.168.14.111. 

2023/11/22 21:22:13 INF 
Association Setup Request:
  Node ID: 192.168.14.111
  Recovery Time: 2023-11-22 21:22:12 +0900 JST

2023/11/22 21:22:13 INF Saving new association: &{ID:192.168.14.111 Addr:192.168.14.111 NextSessionID:1 NextSequenceID:1 Sessions:map[] HeartbeatRetries:0 cancelRetries:<nil>}
```

<a id="run_ran"></a>

### Run srsRAN 4G ZMQ RAN

Run srsRAN 4G ZMQ RAN and connect to Open5GS EPC.
```
# cd srsRAN_4G/build/srsenb
# ./src/srsenb enb.conf
---  Software Radio Systems LTE eNodeB  ---

Reading configuration file enb.conf...

Built in Release mode using commit fa56836b1 on branch master.

Opening 1 channels in RF device=zmq with args=fail_on_disconnect=true,tx_port=tcp://192.168.0.121:2000,rx_port=tcp://192.168.0.122:2001,id=enb,base_srate=23.04e6
Supported RF device list: zmq file
CHx base_srate=23.04e6
CHx id=enb
Current sample rate is 1.92 MHz with a base rate of 23.04 MHz (x12 decimation)
CH0 rx_port=tcp://192.168.0.122:2001
CH0 tx_port=tcp://192.168.0.121:2000
CH0 fail_on_disconnect=true

==== eNodeB started ===
Type <t> to view trace
Current sample rate is 11.52 MHz with a base rate of 23.04 MHz (x2 decimation)
Current sample rate is 11.52 MHz with a base rate of 23.04 MHz (x2 decimation)
Setting frequency: DL=2680.0 Mhz, UL=2560.0 MHz for cc_idx=0 nof_prb=50
```
The Open5GS C-Plane log when executed is as follows.
```
11/22 21:22:51.381: [mme] INFO: eNB-S1 accepted[192.168.0.121]:55081 in s1_path module (../src/mme/s1ap-sctp.c:114)
11/22 21:22:51.381: [mme] INFO: eNB-S1 accepted[192.168.0.121] in master_sm module (../src/mme/mme-sm.c:108)
11/22 21:22:51.381: [mme] INFO: [Added] Number of eNBs is now 1 (../src/mme/mme-context.c:2558)
11/22 21:22:51.381: [mme] INFO: eNB-S1[192.168.0.121] max_num_of_ostreams : 30 (../src/mme/mme-sm.c:150)
```

<a id="run_ue"></a>

### Run srsRAN 4G ZMQ UE

Run srsRAN 4G ZMQ UE and connect to Open5GS EPC.
```
# cd srsRAN_4G/build/srsue
# ./src/srsue ue.conf
Reading configuration file ue.conf...

Built in Release mode using commit fa56836b1 on branch master.

Opening 1 channels in RF device=zmq with args=tx_port=tcp://192.168.0.122:2001,rx_port=tcp://192.168.0.121:2000,id=ue,base_srate=23.04e6
Supported RF device list: zmq file
CHx base_srate=23.04e6
CHx id=ue
Current sample rate is 1.92 MHz with a base rate of 23.04 MHz (x12 decimation)
CH0 rx_port=tcp://192.168.0.121:2000
CH0 tx_port=tcp://192.168.0.122:2001
Waiting PHY to initialize ... done!
Attaching UE...
Current sample rate is 1.92 MHz with a base rate of 23.04 MHz (x12 decimation)
Current sample rate is 1.92 MHz with a base rate of 23.04 MHz (x12 decimation)
.
Found Cell:  Mode=FDD, PCI=1, PRB=50, Ports=1, CP=Normal, CFO=-0.2 KHz
Current sample rate is 11.52 MHz with a base rate of 23.04 MHz (x2 decimation)
Current sample rate is 11.52 MHz with a base rate of 23.04 MHz (x2 decimation)
Found PLMN:  Id=00101, TAC=1
Random Access Transmission: seq=16, tti=341, ra-rnti=0x2
RRC Connected
Random Access Complete.     c-rnti=0x46, ta=0
Network attach successful. IP: 10.45.0.2
 nTp) 22/11/2023 12:24:0 TZ:99
```
The Open5GS C-Plane log when executed is as follows.
```
11/22 21:24:00.370: [mme] INFO: InitialUEMessage (../src/mme/s1ap-handler.c:402)
11/22 21:24:00.370: [mme] INFO: [Added] Number of eNB-UEs is now 1 (../src/mme/mme-context.c:4434)
11/22 21:24:00.370: [mme] INFO: Unknown UE by S_TMSI[G:2,C:1,M_TMSI:0xc0000626] (../src/mme/s1ap-handler.c:482)
11/22 21:24:00.370: [mme] INFO:     ENB_UE_S1AP_ID[1] MME_UE_S1AP_ID[1] TAC[1] CellID[0x19b01] (../src/mme/s1ap-handler.c:578)
11/22 21:24:00.370: [mme] INFO: Unknown UE by GUTI[G:2,C:1,M_TMSI:0xc0000626] (../src/mme/mme-context.c:3288)
11/22 21:24:00.370: [mme] INFO: [Added] Number of MME-UEs is now 1 (../src/mme/mme-context.c:3090)
11/22 21:24:00.370: [emm] INFO: [] Attach request (../src/mme/emm-sm.c:412)
11/22 21:24:00.370: [emm] INFO:     GUTI[G:2,C:1,M_TMSI:0xc0000626] IMSI[Unknown IMSI] (../src/mme/emm-handler.c:245)
11/22 21:24:00.442: [emm] INFO: Identity response (../src/mme/emm-sm.c:382)
11/22 21:24:00.442: [emm] INFO:     IMSI[001010000000100] (../src/mme/emm-handler.c:409)
11/22 21:24:00.563: [mme] INFO: [Added] Number of MME-Sessions is now 1 (../src/mme/mme-context.c:4448)
11/22 21:24:00.635: [sgwc] INFO: [Added] Number of SGWC-UEs is now 1 (../src/sgwc/context.c:237)
11/22 21:24:00.636: [sgwc] INFO: [Added] Number of SGWC-Sessions is now 1 (../src/sgwc/context.c:879)
11/22 21:24:00.636: [sgwc] INFO: UE IMSI[001010000000100] APN[internet] (../src/sgwc/s11-handler.c:237)
11/22 21:24:00.637: [gtp] INFO: gtp_connect() [127.0.0.4]:2123 (../lib/gtp/path.c:60)
11/22 21:24:00.638: [smf] INFO: [Added] Number of SMF-UEs is now 1 (../src/smf/context.c:1010)
11/22 21:24:00.638: [smf] INFO: [Added] Number of SMF-Sessions is now 1 (../src/smf/context.c:3057)
11/22 21:24:00.638: [smf] INFO: UE IMSI[001010000000100] APN[internet] IPv4[10.45.0.2] IPv6[] (../src/smf/s5c-handler.c:275)
11/22 21:24:00.643: [gtp] INFO: gtp_connect() [192.168.13.151]:2152 (../lib/gtp/path.c:60)
11/22 21:24:00.645: [gtp] INFO: gtp_connect() [127.0.0.4]:2123 (../lib/gtp/path.c:60)
11/22 21:24:00.975: [emm] INFO: [001010000000100] Attach complete (../src/mme/emm-sm.c:1298)
11/22 21:24:00.975: [emm] INFO:     IMSI[001010000000100] (../src/mme/emm-handler.c:283)
11/22 21:24:00.975: [emm] INFO:     UTC [2023-11-22T12:24:00] Timezone[0]/DST[0] (../src/mme/emm-handler.c:290)
11/22 21:24:00.976: [emm] INFO:     LOCAL [2023-11-22T21:24:00] Timezone[32400]/DST[0] (../src/mme/emm-handler.c:294)
```
The Open5GS U-Plane log when executed is as follows.
```
11/22 21:24:00.644: [sgwu] INFO: UE F-SEID[UP:0xc5f CP:0x4cf] (../src/sgwu/context.c:169)
11/22 21:24:00.644: [sgwu] INFO: [Added] Number of SGWU-Sessions is now 1 (../src/sgwu/context.c:174)
11/22 21:24:00.653: [gtp] INFO: gtp_connect() [192.168.13.151]:2152 (../lib/gtp/path.c:60)
11/22 21:24:00.984: [gtp] INFO: gtp_connect() [192.168.13.121]:2152 (../lib/gtp/path.c:60)
```
The PDU session establishment log of eUPF is as follows.
```
2023/11/22 21:24:01 INF Got Session Establishment Request from: 192.168.14.111.
2023/11/22 21:24:01 INF 
Session Establishment Request:
  CreatePDR ID: 1 
    FAR ID: 1 
    QER ID: 1 
    Source Interface: 1 
    UE IPv4 Address: 10.45.0.2 
  CreatePDR ID: 2 
    Outer Header Removal: 0 
    FAR ID: 2 
    QER ID: 1 
    Source Interface: 0 
    TEID: 0 
    Ipv4: <nil> 
    Ipv6: <nil> 
    UE IPv4 Address: 10.45.0.2 
  CreatePDR ID: 3 
    Outer Header Removal: 0 
    FAR ID: 1 
    Source Interface: 3 
    TEID: 0 
    Ipv4: <nil> 
    Ipv6: <nil> 
  CreatePDR ID: 4 
    Outer Header Removal: 0 
    FAR ID: 3 
    Source Interface: 0 
    TEID: 0 
    Ipv4: <nil> 
    Ipv6: <nil> 
    SDF Filter: permit out 58 from ff02::2/128 to assigned 
  CreateFAR ID: 1 
    Apply Action: [2 0] 
    Forwarding Parameters:
      Network Instance:internet 
      Outer Header Creation: &{OuterHeaderCreationDescription:256 TEID:5157 IPv4Address:192.168.13.112 IPv6Address:<nil> PortNumber:0 CTag:0 STag:0} 
  CreateFAR ID: 2 
    Apply Action: [2 0] 
    Forwarding Parameters:
      Network Instance:internet 
  CreateFAR ID: 3 
    Apply Action: [2 0] 
    Forwarding Parameters:
      Network Instance:internet 
      Outer Header Creation: &{OuterHeaderCreationDescription:256 TEID:1 IPv4Address:192.168.14.111 IPv6Address:<nil> PortNumber:0 CTag:0 STag:0} 
  CreateQER ID: 1 
    Gate Status DL: 0 
    Gate Status UL: 0 
    Max Bitrate DL: 1000000 
    Max Bitrate UL: 1000000 
  CreateBAR ID: 1

2023/11/22 21:24:01 INF Saving FAR info to session: 1, {Action:2 OuterHeaderCreation:1 Teid:5157 RemoteIP:1879943360 LocalIP:2534254784 TransportLevelMarking:0}
2023/11/22 21:24:01 INF WARN: No OuterHeaderCreation
2023/11/22 21:24:01 INF Saving FAR info to session: 2, {Action:2 OuterHeaderCreation:0 Teid:0 RemoteIP:0 LocalIP:2534254784 TransportLevelMarking:0}
2023/11/22 21:24:01 INF Saving FAR info to session: 3, {Action:2 OuterHeaderCreation:1 Teid:1 RemoteIP:1863231680 LocalIP:2534254784 TransportLevelMarking:0}
2023/11/22 21:24:01 INF Saving QER info to session: 1, {GateStatusUL:0 GateStatusDL:0 Qfi:0 MaxBitrateUL:1000000000 MaxBitrateDL:1000000000 StartUL:0 StartDL:0}
2023/11/22 21:24:01 Matched groups: [permit out 58 from ff02::2/128 to assigned 58 ff02::2/128 assigned]
2023/11/22 21:24:01 INF Session Establishment Request from 192.168.14.111 accepted.
```
The result of `ip addr show` on VM4 (UE) is as follows.
```
7: tun_srsue: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN group default qlen 500
    link/none 
    inet 10.45.0.2/24 scope global tun_srsue
       valid_lft forever preferred_lft forever
```

<a id="ping"></a>

## Ping google.com

Specify the UE's TUNnel interface and try ping.

<a id="ping_1"></a>

### Case for going through PDN 10.45.0.0/16

Run `tcpdump` on VM-DN and check that the packet goes through N6 (enp0s9).
- `ping google.com` on VM4 (UE)
```
# ping google.com -I tun_srsue -n
PING google.com (142.250.207.46) from 10.45.0.2 tun_srsue: 56(84) bytes of data.
64 bytes from 142.250.207.46: icmp_seq=1 ttl=61 time=80.1 ms
64 bytes from 142.250.207.46: icmp_seq=2 ttl=61 time=81.2 ms
64 bytes from 142.250.207.46: icmp_seq=3 ttl=61 time=55.4 ms
```
- Run `tcpdump` on VM-DN
```
# tcpdump -i enp0s9 -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on enp0s9, link-type EN10MB (Ethernet), snapshot length 262144 bytes
21:28:15.875215 IP 10.45.0.2 > 142.250.207.46: ICMP echo request, id 7, seq 1, length 64
21:28:15.900555 IP 142.250.207.46 > 10.45.0.2: ICMP echo reply, id 7, seq 1, length 64
21:28:16.886306 IP 10.45.0.2 > 142.250.207.46: ICMP echo request, id 7, seq 2, length 64
21:28:16.906262 IP 142.250.207.46 > 10.45.0.2: ICMP echo reply, id 7, seq 2, length 64
21:28:17.863169 IP 10.45.0.2 > 142.250.207.46: ICMP echo request, id 7, seq 3, length 64
21:28:17.880153 IP 142.250.207.46 > 10.45.0.2: ICMP echo reply, id 7, seq 3, length 64
```
- See `/sys/kernel/debug/tracing/trace_pipe` on VM-UP
```
# cat /sys/kernel/debug/tracing/trace_pipe
...
          <idle>-0       [000] d.s31  8465.913309: bpf_trace_printk: upf: gtp-u received
          <idle>-0       [000] dNs31  8465.913385: bpf_trace_printk: SDF: filter protocol: 4
          <idle>-0       [000] dNs31  8465.913388: bpf_trace_printk: SDF: filter source ip: 0.0.0.2, destination ip: 0.0.0.0
          <idle>-0       [000] dNs31  8465.913389: bpf_trace_printk: SDF: filter source ip mask: 255.255.255.255, destination ip mask: 0.0.0.0
          <idle>-0       [000] dNs31  8465.913390: bpf_trace_printk: SDF: filter source port lower bound: 1, source port upper bound: 65535
          <idle>-0       [000] dNs31  8465.913391: bpf_trace_printk: SDF: filter destination port lower bound: 1, destination port upper bound: 65535
          <idle>-0       [000] dNs31  8465.913392: bpf_trace_printk: SDF: packet protocol: 0
          <idle>-0       [000] dNs31  8465.913393: bpf_trace_printk: SDF: packet source ip: 10.45.0.2, destination ip: 142.250.207.46
          <idle>-0       [000] dNs31  8465.913393: bpf_trace_printk: SDF: packet source port: 0, destination port: 0
          <idle>-0       [000] dNs31  8465.913394: bpf_trace_printk: upf: sdf filter doesn't match teid:1
          <idle>-0       [000] dNs31  8465.913395: bpf_trace_printk: upf: far:1 action:2 outer_header_creation:0
          <idle>-0       [000] dNs31  8465.913396: bpf_trace_printk: upf: qer:0 gate_status:0 mbr:1000000000
          <idle>-0       [000] dNs31  8465.913398: bpf_trace_printk: upf: session for teid:1 far:1 outer_header_removal:0
          <idle>-0       [000] dNs31  8465.913405: bpf_trace_printk: upf: bpf_fib_lookup 10.45.0.2 -> 142.250.207.46: nexthop: 192.168.16.152
          <idle>-0       [000] d.s31  8465.938998: bpf_trace_printk: upf: downlink session for ip:10.45.0.2  far:0 action:2
          <idle>-0       [000] dNs31  8465.939016: bpf_trace_printk: upf: qer:0 gate_status:0 mbr:1000000000
          <idle>-0       [000] dNs31  8465.939017: bpf_trace_printk: upf: use mapping 10.45.0.2 -> TEID:5157
          <idle>-0       [000] dNs31  8465.939020: bpf_trace_printk: upf: send gtp pdu 192.168.13.151 -> 192.168.13.112
          <idle>-0       [000] dNs31  8465.939026: bpf_trace_printk: upf: bpf_fib_lookup 192.168.13.151 -> 192.168.13.112: nexthop: 192.168.13.112
          <idle>-0       [000] d.s31  8466.924460: bpf_trace_printk: upf: gtp-u received
          <idle>-0       [000] dNs31  8466.924478: bpf_trace_printk: SDF: filter protocol: 4
          <idle>-0       [000] dNs31  8466.924481: bpf_trace_printk: SDF: filter source ip: 0.0.0.2, destination ip: 0.0.0.0
          <idle>-0       [000] dNs31  8466.924482: bpf_trace_printk: SDF: filter source ip mask: 255.255.255.255, destination ip mask: 0.0.0.0
          <idle>-0       [000] dNs31  8466.924483: bpf_trace_printk: SDF: filter source port lower bound: 1, source port upper bound: 65535
          <idle>-0       [000] dNs31  8466.924484: bpf_trace_printk: SDF: filter destination port lower bound: 1, destination port upper bound: 65535
          <idle>-0       [000] dNs31  8466.924484: bpf_trace_printk: SDF: packet protocol: 0
          <idle>-0       [000] dNs31  8466.924485: bpf_trace_printk: SDF: packet source ip: 10.45.0.2, destination ip: 142.250.207.46
          <idle>-0       [000] dNs31  8466.924486: bpf_trace_printk: SDF: packet source port: 0, destination port: 0
          <idle>-0       [000] dNs31  8466.924487: bpf_trace_printk: upf: sdf filter doesn't match teid:1
          <idle>-0       [000] dNs31  8466.924488: bpf_trace_printk: upf: far:1 action:2 outer_header_creation:0
          <idle>-0       [000] dNs31  8466.924489: bpf_trace_printk: upf: qer:0 gate_status:0 mbr:1000000000
          <idle>-0       [000] dNs31  8466.924490: bpf_trace_printk: upf: session for teid:1 far:1 outer_header_removal:0
          <idle>-0       [000] dNs31  8466.924497: bpf_trace_printk: upf: bpf_fib_lookup 10.45.0.2 -> 142.250.207.46: nexthop: 192.168.16.152
          <idle>-0       [000] d.s31  8466.944686: bpf_trace_printk: upf: downlink session for ip:10.45.0.2  far:0 action:2
          <idle>-0       [000] dNs31  8466.944704: bpf_trace_printk: upf: qer:0 gate_status:0 mbr:1000000000
          <idle>-0       [000] dNs31  8466.944705: bpf_trace_printk: upf: use mapping 10.45.0.2 -> TEID:5157
          <idle>-0       [000] dNs31  8466.944707: bpf_trace_printk: upf: send gtp pdu 192.168.13.151 -> 192.168.13.112
          <idle>-0       [000] dNs31  8466.944714: bpf_trace_printk: upf: bpf_fib_lookup 192.168.13.151 -> 192.168.13.112: nexthop: 192.168.13.112
          <idle>-0       [000] d.s31  8467.900953: bpf_trace_printk: upf: gtp-u received
          <idle>-0       [000] dNs31  8467.900974: bpf_trace_printk: SDF: filter protocol: 4
          <idle>-0       [000] dNs31  8467.900977: bpf_trace_printk: SDF: filter source ip: 0.0.0.2, destination ip: 0.0.0.0
          <idle>-0       [000] dNs31  8467.900979: bpf_trace_printk: SDF: filter source ip mask: 255.255.255.255, destination ip mask: 0.0.0.0
          <idle>-0       [000] dNs31  8467.900980: bpf_trace_printk: SDF: filter source port lower bound: 1, source port upper bound: 65535
          <idle>-0       [000] dNs31  8467.900981: bpf_trace_printk: SDF: filter destination port lower bound: 1, destination port upper bound: 65535
          <idle>-0       [000] dNs31  8467.900982: bpf_trace_printk: SDF: packet protocol: 0
          <idle>-0       [000] dNs31  8467.900983: bpf_trace_printk: SDF: packet source ip: 10.45.0.2, destination ip: 142.250.207.46
          <idle>-0       [000] dNs31  8467.900984: bpf_trace_printk: SDF: packet source port: 0, destination port: 0
          <idle>-0       [000] dNs31  8467.900985: bpf_trace_printk: upf: sdf filter doesn't match teid:1
          <idle>-0       [000] dNs31  8467.900986: bpf_trace_printk: upf: far:1 action:2 outer_header_creation:0
          <idle>-0       [000] dNs31  8467.900987: bpf_trace_printk: upf: qer:0 gate_status:0 mbr:1000000000
          <idle>-0       [000] dNs31  8467.900988: bpf_trace_printk: upf: session for teid:1 far:1 outer_header_removal:0
          <idle>-0       [000] dNs31  8467.900996: bpf_trace_printk: upf: bpf_fib_lookup 10.45.0.2 -> 142.250.207.46: nexthop: 192.168.16.152
          <idle>-0       [000] d.s31  8467.918543: bpf_trace_printk: upf: downlink session for ip:10.45.0.2  far:0 action:2
          <idle>-0       [000] dNs31  8467.918560: bpf_trace_printk: upf: qer:0 gate_status:0 mbr:1000000000
          <idle>-0       [000] dNs31  8467.918562: bpf_trace_printk: upf: use mapping 10.45.0.2 -> TEID:5157
          <idle>-0       [000] dNs31  8467.918564: bpf_trace_printk: upf: send gtp pdu 192.168.13.151 -> 192.168.13.112
          <idle>-0       [000] dNs31  8467.918570: bpf_trace_printk: upf: bpf_fib_lookup 192.168.13.151 -> 192.168.13.112: nexthop: 192.168.13.112
...
```
In addition to `ping`, you may try to access the web by specifying the TUNnel interface with `curl` as follows.
- `curl google.com` on VM4 (UE)
```
# curl --interface tun_srsue google.com
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com/">here</A>.
</BODY></HTML>
```
- Run `tcpdump` on VM-DN
```
21:30:02.571352 IP 10.45.0.2.44914 > 142.250.207.46.80: Flags [S], seq 4207429670, win 64240, options [mss 1460,sackOK,TS val 2359817003 ecr 0,nop,wscale 7], length 0
21:30:02.621673 IP 142.250.207.46.80 > 10.45.0.2.44914: Flags [S.], seq 2304001, ack 4207429671, win 65535, options [mss 1460], length 0
21:30:02.759452 IP 10.45.0.2.44914 > 142.250.207.46.80: Flags [.], ack 1, win 64240, length 0
21:30:02.759452 IP 10.45.0.2.44914 > 142.250.207.46.80: Flags [P.], seq 1:75, ack 1, win 64240, length 74: HTTP: GET / HTTP/1.1
21:30:02.759632 IP 142.250.207.46.80 > 10.45.0.2.44914: Flags [.], ack 75, win 65535, length 0
21:30:02.855911 IP 142.250.207.46.80 > 10.45.0.2.44914: Flags [P.], seq 1:774, ack 75, win 65535, length 773: HTTP: HTTP/1.1 301 Moved Permanently
21:30:02.887509 IP 10.45.0.2.44914 > 142.250.207.46.80: Flags [.], ack 774, win 63467, length 0
21:30:02.887509 IP 10.45.0.2.44914 > 142.250.207.46.80: Flags [F.], seq 75, ack 774, win 63467, length 0
21:30:02.887704 IP 142.250.207.46.80 > 10.45.0.2.44914: Flags [.], ack 76, win 65535, length 0
21:30:02.949273 IP 142.250.207.46.80 > 10.45.0.2.44914: Flags [F.], seq 774, ack 76, win 65535, length 0
21:30:03.020910 IP 10.45.0.2.44914 > 142.250.207.46.80: Flags [.], ack 775, win 63467, length 0
```
You could now connect to the PDN and send any packets on the network using eUPF.

---

Now you could work Open5GS EPC with eUPF.
I would like to thank the excellent developers and all the contributors of Open5GS, eUPF and srsRAN 4G.

<a id="changelog"></a>

## Changelog (summary)

- [2023.11.22] Updated to eUPF `120-upf-ftup-fteid` branch that supports FTUP.
- [2023.10.29] Initial release.
