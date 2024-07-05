## Quectel_RM510Q_GL

## Table of contents

<!-- toc -->

- [successful test case](#successful-test-case)
- [0. setup Antenna](#0-setup-antenna)
- [1. setup data connection](#1-setup-data-connection)
  * [1.1. setting up a data connection over QMI interface using libqmi](#11-setting-up-a-data-connection-over-qmi-interface-using-libqmi)
  * [1.2. setting up a data connection use mmcli](#12-setting-up-a-data-connection-use-mmcli)
- [2. AT command](#2-at-command)
  * [2.1. start minicom](#21-start-minicom-)
  * [2.2 AT command basic](#22-at-command-basic)
    + [- don't forget to connect with broadband](#--dont-forget-to-connect-with-broadband)
    + [- check connection, enable module, and reboot](#--check-connection-enable-module-and-reboot)
    + [- set module as 5G](#--set-module-as-5g)
    + [- check registration status](#--check-registration-status)
    + [- check PDP context](#--check-pdp-context)
    + [- check connection](#--check-connection)
    + [- check network strength](#--check-network-strength)
    + [8. (optional) shutdown](#8-optional-shutdown)
- [3. use case](#3-use-case)
  * [3.1. quectel as a UE for srsRAN4G](#31-quectel-as-a-ue-for-srsran4g-)
  * [3.2. quectel to enable computator to connect to 5G and iperf to UNAI](#32-quectel-to-enable-computator-to-connect-to-5g-and-iperf-to-unai)
  * [3.3. quectel to enable computator to connect to 4G and iperf to UNAI](#33-quectel-to-enable-computator-to-connect-to-4g-and-iperf-to-unai)

<!-- tocstop -->

### successful test case
- quectel + sim true 5G to internet
	- to ping
	- to iperf (on 5G to UNAI)
- quectel + sim true 4G (selected) to internet
    - to ping
    - to iperf (on 4G to UNAI)
- quectel on rpi w/ sysmocom sim connect to srsRAN4G
    - ping
    - connect to web server, run speedtest

### 0. setup Antenna
- connect to ANT0 and ANT1 (see more on hardware design)
- in case want to enhance recieve signal quality, connect ANT2 and ANT3

### 1. setup data connection
#### 1.1. setting up a data connection over QMI interface using libqmi 
source:
- mainly from: https://docs.sixfab.com/page/setting-up-a-data-connection-over-qmi-interface-using-libqmi
- check basic qmi command: https://techship.com/faq/how-to-step-by-step-set-up-a-data-connection-over-qmi-interface-using-qmicli-and-in-kernel-driver-qmi-wwan-in-linux/
- other: https://solidrun.atlassian.net/wiki/spaces/developer/pages/326631427/Setting+up+a+data+connection+over+QMI+interface+using+libqmi

##### steps:
- check the compatability of the module  
```
lsusb
Bus 003 Device 012: ID 2c7c:0800 Quectel Wireless Solutions Co., Ltd. RM510Q-GL

lsusb -t
 |__ Port 1: Dev 12, If 4, Class=Vendor Specific Class, Driver=qmi_wwan, 480
```

- install required packages
```
sudo apt update
sudo apt update && sudo apt install libqmi-utils udhcpc
```

- get operating mode
```
sudo qmicli -d /dev/cdc-wdm0 --dms-get-operating-mode 

[/dev/cdc-wdm0] Operating mode retrieved:
	Mode: 'online'
	HW restricted: 'no'
```
If not `online` set it as online
```
sudo qmicli --device=/dev/cdc-wdm0 --dms-set-operating-mode='online'

[/dev/cdc-wdm0] Operating mode set successfully
```
<details close>
<summary><b> other basic command to check the quectel board with qmicli command </b></summary>
	
```
## get revision of quectel board
sudo qmicli --device=/dev/cdc-wdm0 --device-open-proxy --dms-get-revision

[/dev/cdc-wdm0] Device revision retrieved:
    Revision: 'RM510QGLAAR11A03M4G'


## get id
sudo qmicli --device=/dev/cdc-wdm0 --device-open-proxy --dms-get-ids

[/dev/cdc-wdm0] Device IDs retrieved:
    ESN: '0'
    IMEI: '867034040025018'
    MEID: 'unknown'
    IMEI SV: '27'
```
</details>


- configure the network interface 
```
## down the interface
sudo ip link set wwan0 down

## set the interface into raw mode (IP packets not encapsulated in Ethernet frames)
echo 'Y' | sudo tee /sys/class/net/wwan0/qmi/raw_ip

## restart the interface
sudo ip link set wwan0 up
```
- when the wwan0 is up, connect to the network by changing `apn, username, password`
```
## connect to the network
sudo qmicli -p -d /dev/cdc-wdm0 --device-open-net='net-raw-ip|net-no-qos-header' --wds-start-network="apn='Your_APN',username='Your_Username',password='Your_Password'" --client-no-release-cid
```
example of configuration
```
## for client sim card
sudo qmicli -p -d /dev/cdc-wdm0 --device-open-net='net-raw-ip|net-no-qos-header' --wds-start-network="apn='internet',username='true',password='true'" --client-no-release-cid

[/dev/cdc-wdm0] Network started
    Packet data handle: '3059871808'
[/dev/cdc-wdm0] Client ID not released:
    Service: 'wds'
        CID: '23'


## for sysmocom sim card
sudo qmicli -p -d /dev/cdc-wdm0 --device-open-net='net-raw-ip|net-no-qos-header' --wds-start-network="apn='srsapn'" --client-no-release-cid 

error: couldn't start network: QMI protocol error (14): 'CallFailed'
call end reason (3): generic-no-service
verbose call end reason (3,2001): [cm] no-service
[/dev/cdc-wdm0] Client ID not released:
    Service: 'wds'
        CID: '16'
```
if the network timeout, it could be that you skip the state of `sudo ip link set wwan0 down` to `sudo ip link set wwan0 up`. Try again. But sometimes it is not this. 

- use `udhcpc` to configure the IP address and the default route
```
sudo udhcpc -q -f -i wwan0
	udhcpc: started, v1.30.1
	udhcpc: sending discover
	udhcpc: sending select for 10.38.223.119
	udhcpc: lease of 10.38.223.119 obtained, lease time 7200
```
- use `ifconfig` to check the assigned IP address
```
>> ifconfig wwan0
wwan0: flags=4305<UP,POINTOPOINT,RUNNING,NOARP,MULTICAST>  mtu 1500
        inet 10.38.223.119  netmask 255.255.255.240  destination 10.38.223.119
        inet6 fe80::d972:314d:2f0:fc43  prefixlen 64  scopeid 0x20<link>
        unspec 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00  txqueuelen 1000  (UNSPEC)
        RX packets 2  bytes 612 (612.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 903  bytes 70464 (70.4 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```
- use `ping` to check the network connection
```
>> ping -I wwan0 -c 5 8.8.8.8
PING 8.8.8.8 (8.8.8.8) from 10.38.223.119 wwan0: 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=253 time=50.0 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=253 time=28.6 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=253 time=32.2 ms
64 bytes from 8.8.8.8: icmp_seq=4 ttl=253 time=40.7 ms
64 bytes from 8.8.8.8: icmp_seq=5 ttl=253 time=33.7 ms

--- 8.8.8.8 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4006ms
rtt min/avg/max/mdev = 28.580/37.034/50.015/7.594 ms
```

#### 1.2. setting up a data connection use mmcli 
source:
- from https://github.com/srsran/srsRAN_Project/discussions/426#discussioncomment-8233829 <br>
**note: for using the UE first time, it is not recommend to connect using `mmcli`, `qmicli` will be better** <br />

##### steps:
- connect to the modem
```
## list connected modem
sudo mmcli -L
    /org/freedesktop/ModemManager1/Modem/2 [Quectel] RM510Q-GL

## enable then connect the network
sudo mmcli -m 22 -e
    successfully enabled the modem

## connect to the network using apn
sudo mmcli -m 22 --simple-connect="apn=srsapn"
    successfully connected the modem
```
- get ip address and set link up
```
sudo ip link set wwan0 up
sudo udhcpc -q -f -i wwan0
ifconfig wwan0
```
in case the `udhcpc` not working, just add ip directly
```
sudo ip a add 10.45.0.2/24 dev wwan0
ip a
```
- then ping to see if it can access internet
```
ping -I wwan0 8.8.8.8
ping -I wwan0 10.45.0.1   # core network
```
<details close>
<summary> sudo mmcli -m 2 connect to lte </summary>

```
sudo mmcli -m 22
  -----------------------------------
  General  |                    path: /org/freedesktop/ModemManager1/Modem/22
           |               device id: 6535a45484985ff207e2b40c0e2487ae3914bb2b
  -----------------------------------
  Hardware |            manufacturer: Quectel
           |                   model: RM510Q-GL
           |       firmware revision: RM510QGLAAR11A03M4G
           |          carrier config: ROW_Commercial
           | carrier config revision: 0A010809
           |            h/w revision: 20000
           |               supported: gsm-umts, lte, 5gnr
           |                 current: gsm-umts, lte, 5gnr
           |            equipment id: 867034040025018
  -----------------------------------
  System   |                  device: /sys/devices/pci0000:00/0000:00:14.0/usb3/3-1
           |                 drivers: qmi_wwan, option
           |                  plugin: quectel
           |            primary port: cdc-wdm0
           |                   ports: cdc-wdm0 (qmi), ttyUSB0 (qcdm), ttyUSB1 (gps), 
           |                          ttyUSB2 (at), ttyUSB3 (at), wwan0 (net)
  -----------------------------------
  Status   |                    lock: sim-pin2
           |          unlock retries: sim-pin (3), sim-puk (10), sim-pin2 (3), sim-puk2 (10)
           |                   state: registered
           |             power state: on
           |             access tech: lte
           |          signal quality: 86% (recent)
  -----------------------------------
  Modes    |               supported: allowed: 3g; preferred: none
           |                          allowed: 4g; preferred: none
           |                          allowed: 3g, 4g; preferred: 4g
           |                          allowed: 3g, 4g; preferred: 3g
           |                          allowed: 5g; preferred: none
           |                          allowed: 4g, 5g; preferred: 5g
           |                          allowed: 4g, 5g; preferred: 4g
           |                          allowed: 3g, 5g; preferred: 5g
           |                          allowed: 3g, 5g; preferred: 3g
           |                          allowed: 3g, 4g, 5g; preferred: 5g
           |                          allowed: 3g, 4g, 5g; preferred: 4g
           |                          allowed: 3g, 4g, 5g; preferred: 3g
           |                 current: allowed: 4g; preferred: none
  -----------------------------------
  Bands    |               supported: utran-1, utran-3, utran-4, utran-6, utran-5, utran-8, 
           |                          utran-9, utran-2, eutran-1, eutran-2, eutran-3, eutran-4, eutran-5, 
           |                          eutran-7, eutran-8, eutran-12, eutran-13, eutran-14, eutran-17, 
           |                          eutran-18, eutran-19, eutran-20, eutran-25, eutran-26, eutran-28, 
           |                          eutran-29, eutran-30, eutran-32, eutran-34, eutran-38, eutran-39, 
           |                          eutran-40, eutran-41, eutran-42, eutran-43, eutran-46, eutran-48, 
           |                          eutran-66, eutran-71, utran-19, ngran-1, ngran-2, ngran-3, ngran-5, 
           |                          ngran-7, ngran-8, ngran-12, ngran-14, ngran-20, ngran-25, ngran-28, 
           |                          ngran-38, ngran-40, ngran-41, ngran-48, ngran-66, ngran-71, ngran-77, 
           |                          ngran-78, ngran-79, ngran-257, ngran-258, ngran-260, ngran-261
           |                 current: utran-1, utran-3, utran-4, utran-6, utran-5, utran-8, 
           |                          utran-2, eutran-1, eutran-2, eutran-3, eutran-4, eutran-5, eutran-7, 
           |                          eutran-8, eutran-12, eutran-13, eutran-14, eutran-18, eutran-19, 
           |                          eutran-20, eutran-25, eutran-26, eutran-28, eutran-29, eutran-30, 
           |                          eutran-32, eutran-34, eutran-38, eutran-39, eutran-40, eutran-41, 
           |                          eutran-42, eutran-43, eutran-46, eutran-48, eutran-66, eutran-71, 
           |                          utran-19, ngran-1, ngran-2, ngran-3, ngran-5, ngran-7, ngran-8, 
           |                          ngran-12, ngran-20, ngran-25, ngran-28, ngran-38, ngran-40, ngran-41, 
           |                          ngran-48, ngran-66, ngran-71, ngran-77, ngran-78, ngran-79
  -----------------------------------
  IP       |               supported: ipv4, ipv6, ipv4v6
  -----------------------------------
  3GPP     |                    imei: 867034040025018
           |           enabled locks: fixed-dialing
           |             operator id: 99970
           |           operator name: 999 70
           |            registration: home
           |    packet service state: attached
  -----------------------------------
  3GPP EPS |    ue mode of operation: csps-2
           |     initial bearer path: /org/freedesktop/ModemManager1/Bearer/15
           |  initial bearer ip type: ipv4v6
  -----------------------------------
  SIM      |        primary sim path: /org/freedesktop/ModemManager1/SIM/13
           |          sim slot paths: slot 1: /org/freedesktop/ModemManager1/SIM/13 (active)
           |                          slot 2: none

```
</details>

### 2. AT command
#### 2.1. start minicom <a name = "atminicom_basic"></a>

##### steps:
- check USB interface
```
sudo dmesg | grep /dev/ttyUSB
sudo dmesg | grep ttyUSB
```
From https://bacnh.com/quectel-linux-usb-drivers-troubleshooting, indicate we will use `ttyUSB2` for `AT command`
> /dev/ttyUSB0 - DM \
> /dev/ttyUSB1 - For GPS NMEA message output <br> 
> /dev/ttyUSB2 - For AT command communication \
> /dev/ttyUSB3 - For PPP connection or AT command communication \

- enter `minicom`
```
sudo minicom -s
```
- setting serial port
<img src="https://github.com/pchat-imm/quectel_rm510q_gl/assets/40858099/eff7a2fd-395d-41b7-8725-8a9177d57f36" width="40%" height="30%" align="left">
<img src="https://github.com/pchat-imm/quectel_rm510q_gl/assets/40858099/a289b780-4135-44cd-a02e-da1e2a03187a" width=50% height=50%/> <br/>

make sure </br>
- serial device = /dev/ttyUSB2 
- bps = 115200 (default) 
- Hardware flow control = No
  
everytime finish `Save setup as dfl` before `exit`

#### 2.2 AT command basic 
source: 
- [Ettus/OAI Reference Architecture for 5G and 6G Research with USRP](https://kb.ettus.com/OAI_Reference_Architecture_for_5G_and_6G_Research_with_USRP)
- [OAI/NR_SA_Tutorial_COTS_UE.md](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/doc/NR_SA_Tutorial_COTS_UE.md)


##### steps:
- don't forget to connect with broadband <br>
<img src="https://github.com/pchat-imm/quectel_rm510q_gl/assets/40858099/a288c95d-3c49-4146-b53c-3e8528cbc1b3" width="70%" height="70%"/> <br/>

basic `minicom` menu
> - using ctrl A + E = echo ON, to show what you type on the screen
> - usign ctrl A + C = clear screen
> - using ctrl A + X to quit the minicom
> - if minicom freeze, open another window and try
<br>

- check connection, enable module, and reboot
  
```
## check if AT command on
AT
OK

## check usbnet
AT+QCFG="data_interface"        ## 0 = usb, 1 = pcle

## check if device recognise sim
AT+CPIN?
+CPIN: READY

## enable module
AT+QMBNCFG="select","Row_commercial"
OK

## reboot (or unplug -> repulg) 
AT+CFUN=1,1                     ## wait 30 sec!!!
```
<details close>
    <summary> result AT+CFUN=1,1 </summary>

```
at+cfun=1,1
OK
RDY
+CPIN: READY
+QUSIM: 1
+CFUN: 1
+QIND: SMS DONE
+QIND: PB DONE
TATE0
OK
OK
OK
+CRSM: 148,8,""
OK
+CEMODE: 2
OK
+QGPS: (1-4),(1-255),(1-3),(100-65535)
OK
+CPMS: "ME",18,127,"ME",18,127,"ME",18,127
OK
+CTZU: (0,1)
OK
+CCLK: "24/01/18,08:58:19+28"
OK
RM510QGLAAR11A03M4G                                                             
OK                                                                              
RM510QGLAAR11A03M4G_01.001.01.001                                               
OK 
```
</details>

- set module as 5G
```
AT+QNWPREFCFG="nr5g_band"
AT+QNWPREFCFG="mode_pref",NR5G
AT+QNWPREFCFG="nr5g_disable_mode",0   ## enable 5G operation both SA and NSA - (0 is no mode is disable)
AT+QNWPREFCFG="roam_pref",1-255       ## 255 = Roaming
```
- check registration status
```
AT+CREG=?        
+CREG:1,0       ## 1 = registered home network
                ## 5 = registered, roaming
AT+C5GREG?      ## 0 = disable / 1 = enable network
                ## 1 = registered home network
```
- check PDP context
```
## check PDP context
AT+CGDCONT?
+CGDCONT: 1,"IP","internet"     ## CID = 1

## disable other unused PDP
AT+CGDCONT=2
AT+CGDCONT=3

## activate PDP context
AT+CGACT=1,1    ## second number is CID
                ## return 1 = connect
```
- check connection
```
AT+COPS?
+COPS: 0,0,"Open5GS Magic",11
+COPS: 0,0,"TRUE-H TRUE-H",7 

AT+QMAP="wwan0"     
+QMAP="wwan0": 1,1,"IPV4","10.45.0.2"

AT+CGPADDR
+CGPADDR: 1,"10.45.0.2"
```
for `AT+COPS` if it returns no network name, change setting from `5G only` to `3G, 4G, 5G` 
- check network strength
```
AT+QCSQ
+QCSQ: <RAT>,<RSRP>,<SINR>,<RSRQ>        
+QCSQ: "NR5G",-68,38,-11

AT+QNEG="servingcell"
+QENG="servingcell": <CON>,<RAT>,<TDD/FDD>,<MCC>,<MNC>,xxxxxx,xx,xx,<ARFCN>,<band>,xx,<RSRP>,<RSRQ>,<SINR>,xx
+QENG="servingcell": "NOCONN","NR5G-SA","TDD",901,70,66C000,1,7,626976,78,3,-72,-11,37,1


## add on

AT+QNWCFG="nr5g_csi"
+QNWCFG="nr5g_csi": <MCS>, <RI>, <CQI>, <PMI>

AT+QNWINFO?
+QNWINFO: "TDD NR5G","90170","NR5G_BAND 78",626976
```
(worst to best)
- AT+QCSQ : RSRP [-140,-40], SINR [-20,40], RSRQ [-20,-3] 
- AT+QNWCFG : MCS[0-31], CQI[0-15]
<br>

- (optional) shutdown
```
AT+QPOWD
```

- (optional) ping using AT command - not recommend, mostly returns `+QPING:569`
```
at+qping=1,"google.com"
OK
+QPING: 0,"216.58.200.14",32,73,255
+QPING: 0,"216.58.200.14",32,56,255
+QPING: 0,"216.58.200.14",32,65,255
+QPING: 0,"216.58.200.14",32,72,255
+QPING: 0,4,4,0,56,73,66
+QIURC: "pdpdeact",1

at+qping=1,"www.chula.ac.th"
OK
+QPING: 0,"45.60.126.77",32,70,255
+QPING: 0,"45.60.126.77",32,47,255
+QPING: 0,"45.60.126.77",32,25,255
+QPING: 0,"45.60.126.77",32,41,255
+QPING: 0,4,4,0,25,70,45
```
table 1: explanation from any line of `AT+QPING`
| line | status | dest IP              | length (bytes)      | RTT (ms)  | TTL          |       
|------|--------|----------------------|---------------------|-----------|--------------|        
| any  | 0      | "45.60.126.77"       | 32                  | 70        | 255          | 

table 2: explanation from the last line of `AT+QPING`
| line | status | total requested ping | total received ping | lost ping | min RTT (ms) | max RTT (ms) | avg RTT (ms) |
|------|--------|----------------------|---------------------|-----------|--------------|--------------|--------------|
| last | 0      | 4                    | 4                   | 0         | 25           | 70           | 45           |

note:
- status 0 = successful, 569 = timeout
- the `569` could happend to any network that not working fast enough (both 4G and 5G) 

the command `AT+QPING` in wireshark (ICMPV6) could show the list below, whihc you need `Echo ping request` and `Echo ping reply` to ensure that the ping is sent successfully. Other commands include:
- Neighbour solicitaiton
- Neighbour Advertisement
- Echo (ping) request
- Echo (ping) reply

<img src="https://github.com/pchat-imm/quectel_rm510q_gl/assets/40858099/719ca952-2a9a-46be-b9cd-44d0d6376b13" width="80%" height="auto">

### 3. use case
#### 3.1. quectel as a UE for srsRAN4G <a name = "quectel_srsRAN4G">
0. config open5gs and firewall
```
### Enable IPv4/IPv6 Forwarding
>> sudo sysctl -w net.ipv4.ip_forward=1
>> sudo sysctl -w net.ipv6.conf.all.forwarding=1

### Add NAT Rule
>> sudo iptables -t nat -A POSTROUTING -s 10.45.0.0/16 ! -o ogstun -j MASQUERADE

>> sudo ufw status
Status: inactive
```

1. masq the interface
```
>> cd ~/.config/srsran
>> ifconfig
ogstun: flags=4305<UP,POINTOPOINT,RUNNING,NOARP,MULTICAST>  mtu 1400
        inet 10.45.0.1  netmask 255.255.0.0  destination 10.45.0.1
        inet6 2001:db8:cafe::1  prefixlen 48  scopeid 0x0<global>
        inet6 fe80::5402:c1e3:c1fd:909e  prefixlen 64  scopeid 0x20<link>
        unspec 00-00-00-00-00-00-00-00-00-00-00-00-00-00-00-00  txqueuelen 500  (UNSPEC)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 96  bytes 26868 (26.8 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

wlp9s0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.221.40.241  netmask 255.255.248.0  broadcast 10.221.47.255
        inet6 fe80::211c:6916:6b4:453  prefixlen 64  scopeid 0x20<link>
        ether c0:3c:59:6e:c0:b7  txqueuelen 1000  (Ethernet)
        RX packets 836003  bytes 1062674583 (1.0 GB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 128719  bytes 17799853 (17.7 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

>> route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         _gateway        0.0.0.0         UG    600    0        0 wlp9s0
10.45.0.0       0.0.0.0         255.255.0.0     U     0      0        0 ogstun
10.53.1.0       0.0.0.0         255.255.255.0   U     0      0        0 br-2d2baf4f3a8c
10.221.40.0     0.0.0.0         255.255.248.0   U     600    0        0 wlp9s0
link-local      0.0.0.0         255.255.0.0     U     1000   0        0 wlp9s0


>> sudo srsepc_if_masq.sh wlp9s0
>> ifconfig
```
2. run epc
```
>> cd ~/.config/srsran
>> sudo srsepc epc.conf
```
in case route is not presented
```
### Enable IPv4/IPv6 Forwarding
$ sudo sysctl -w net.ipv4.ip_forward=1
$ sudo sysctl -w net.ipv6.conf.all.forwarding=1

### Add NAT Rule
$ sudo iptables -t nat -A POSTROUTING -s 10.45.0.0/16 ! -o ogstun -j MASQUERADE
```
3. run enb
```
cd ~/.config/srsran
sudo srsenb enb.conf
```
4. start connect quectel to the masq <br>
4.1. method on discussion forum
```
sudo mmcli -L
sudo mmcli -m 2
sudo mmcli -m 2 -e
sudo mmcli -m 2 --simple-connect="apn=srsapn"
```
4.2 method from this document
```
lsusb
sudo qmicli --device=/dev/cdc-wdm0 --dms-get-operating-mode 
sudo ip link set wwan0 down
echo 'Y' | sudo tee /sys/class/net/wwan0/qmi/raw_ip
sudo ip link set wwan0 up

sudo qmicli -p -d /dev/cdc-wdm0 --device-open-net='net-raw-ip|net-no-qos-header' --wds-start-network="apn='srsapn'" --client-no-release-cid 

sudo udhcpc -q -f -i wwan0
ifconfig 
ping -I wwan0 -c 5 8.8.8.8
```
5. click connect on the network setting <br>
5.1. on laptop in Ubuntu OS 
<img src="https://github.com/pchat-imm/o-ran-e2-kpm/assets/40858099/3fe17e17-cf33-4044-8f7e-ee40bbda1ff3" width="30%" height="auto">

6. run AT command to ping to the gNB
```
>> sudo minicom -s

at
at+cops?
+COPS: 0,0,"Software Radio Systems R",7

at+qnwprefcfg = "lte_band"
at+qnwprefcfg = "mode_pref", LTE
AT+QPING=1,"8.8.8.8"      
```

#### 3.2. quectel to enable computator to connect to 5G and iperf to UNAI
```
>> iperf3 -c <IP> -p 9051 -b 10M
Connecting to host <IP>, port 9051
[  5] local 10.33.134.148 port 48042 connected to 203.185.137.212 port 9051
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec   941 KBytes  7.71 Mbits/sec    4   69.1 KBytes       
[  5]   1.00-2.00   sec   255 KBytes  2.09 Mbits/sec    1   1.36 KBytes       
[  5]   2.00-3.00   sec   748 KBytes  6.13 Mbits/sec    3   63.7 KBytes       
[  5]   3.00-4.00   sec   998 KBytes  8.17 Mbits/sec    4   35.2 KBytes       
[  5]   4.00-5.00   sec   624 KBytes  5.11 Mbits/sec    2   23.0 KBytes       
[  5]   5.00-6.00   sec   249 KBytes  2.04 Mbits/sec    0   28.5 KBytes       
[  5]   6.00-7.00   sec   249 KBytes  2.04 Mbits/sec    0   33.9 KBytes       
[  5]   7.00-8.00   sec   249 KBytes  2.04 Mbits/sec    3   29.8 KBytes       
[  5]   8.00-9.00   sec   249 KBytes  2.04 Mbits/sec    1   36.6 KBytes       
[  5]   9.00-10.00  sec  1.10 MBytes  9.19 Mbits/sec    0   93.5 KBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  5.55 MBytes  4.66 Mbits/sec   18             sender
[  5]   0.00-10.07  sec  5.43 MBytes  4.53 Mbits/sec                  receiver
```

#### 3.3. quectel to enable computator to connect to 4G and iperf to UNAI
```
chatchamon@worker01:~$ iperf3 -c 203.185.137.212 -p 9051 -b 10M
Connecting to host 203.185.137.212, port 9051
[  5] local 10.97.9.86 port 60844 connected to 203.185.137.212 port 9051
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec  1.32 MBytes  11.0 Mbits/sec    0    133 KBytes       
[  5]   1.00-2.00   sec  1.08 MBytes  9.05 Mbits/sec   32   20.3 KBytes       
[  5]   2.00-3.00   sec   374 KBytes  3.06 Mbits/sec    1   27.1 KBytes       
[  5]   3.00-4.00   sec   873 KBytes  7.15 Mbits/sec    0   44.7 KBytes       
[  5]   4.00-5.00   sec   624 KBytes  5.11 Mbits/sec   12   14.9 KBytes       
[  5]   5.00-6.00   sec   499 KBytes  4.09 Mbits/sec    0   31.2 KBytes       
[  5]   6.00-7.00   sec   998 KBytes  8.17 Mbits/sec    0   48.8 KBytes       
[  5]   7.00-8.00   sec   748 KBytes  6.13 Mbits/sec    3   23.0 KBytes       
[  5]   8.00-9.00   sec   748 KBytes  6.13 Mbits/sec    0   39.3 KBytes       
[  5]   9.00-10.00  sec   748 KBytes  6.13 Mbits/sec    4   27.1 KBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.00  sec  7.87 MBytes  6.61 Mbits/sec   52             sender
[  5]   0.00-10.04  sec  7.68 MBytes  6.42 Mbits/sec                  receiver
```
