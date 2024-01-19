## quectel_rm510q_gl
### current task
22/01/2024
- [ ] quectel with new sysmocom SIM with srsRAN 4G \
for gNB, change epc.conf (MCC), user_db.csv, enb.conf (MCC) \
for core, add UE on open5GS (localhost:9999)
- [ ] use this sim with mobile
- [ ] use this sim with quectel

## Table of contents
- [setting up a data connection over QMI interface using libqmi](#setupqmi)
- [start AT command with minicom](#atminicom)

## setting up a data connection over QMI interface using libqmi <a name = "setupqmi"></a>
- mainly from: https://docs.sixfab.com/page/setting-up-a-data-connection-over-qmi-interface-using-libqmi
- check basic qmi command: https://techship.com/faq/how-to-step-by-step-set-up-a-data-connection-over-qmi-interface-using-qmicli-and-in-kernel-driver-qmi-wwan-in-linux/
- other: https://solidrun.atlassian.net/wiki/spaces/developer/pages/326631427/Setting+up+a+data+connection+over+QMI+interface+using+libqmi

- check the compatability of the module with 
```
>> lsusb

Bus 003 Device 012: ID 2c7c:0800 Quectel Wireless Solutions Co., Ltd. RM510Q-GL

>> lsusb -t
 |__ Port 1: Dev 12, If 4, Class=Vendor Specific Class, Driver=qmi_wwan, 480
```

- install required packages
```
>> sudo apt update
>> sudo apt update && sudo apt install libqmi-utils udhcpc
```

- check operating mode
```
>> sudo qmicli --device=/dev/cdc-wdm0 --
>> sudo qmicli --device=/dev/cdc-wdm0 --dms-get-operating-mode 
[/dev/cdc-wdm0] Operating mode retrieved:
	Mode: 'online'
	HW restricted: 'no'
```
if not `online` set try
```
>> sudo qmicli -p --device=/dev/cdc-wdm0 --dms-get-operating-mode 
```
or set it as online
```
>> sudo qmicli --device=/dev/cdc-wdm0 --dms-set-operating-mode='online'
[/dev/cdc-wdm0] Operating mode set successfully
```
<details open>
<summary>other basic command to check the quectel board with qmicli command</summary>
	
```
>> sudo qmicli --device=/dev/cdc-wdm0 --device-open-proxy --uim-get-card-status
[/dev/cdc-wdm0] Successfully got card status
Provisioning applications:
	Primary GW:   slot '1', application '1'
	Primary 1X:   session doesn't exist
	Secondary GW: session doesn't exist
	Secondary 1X: session doesn't exist
Slot [1]:
	Card state: 'present'
	UPIN state: 'not-initialized'
		UPIN retries: '0'
		UPUK retries: '0'
	Application [1]:
		Application type:  'usim (2)'
		Application state: 'ready'
		Application ID:
			A0:00:00:00:87:10:02:FF:66:FF:FF:89:FF:FF:FF
		Personalization state: 'ready'
		UPIN replaces PIN1: 'no'
		PIN1 state: 'disabled'
			PIN1 retries: '3'
			PUK1 retries: '10'
		PIN2 state: 'enabled-not-verified'
			PIN2 retries: '3'
			PUK2 retries: '10'
```
```
>> sudo qmicli --device=/dev/cdc-wdm0 --device-open-proxy --dms-get-ids
[/dev/cdc-wdm0] Device IDs retrieved:
	ESN: '0'
	IMEI: '867034040025018'
	MEID: 'unknown'
	IMEI SV: '27'
```
```
>> sudo qmicli --device=/dev/cdc-wdm0 --device-open-proxy --dms-get-revision
[/dev/cdc-wdm0] Device revision retrieved:
	Revision: 'RM510QGLAAR11A03M4G'
```
</details>


- configure the network interface
```
>> sudo ip link set wwan0 down
```
set the interface into raw mode
```
>> echo 'Y' | sudo tee /sys/class/net/wwan0/qmi/raw_ip
```
then restart the interface
```
>> sudo ip link set wwan0 up
```

- when the wwan0 is up, connect the network by changing apn, username, password
```
>> sudo qmicli -p -d /dev/cdc-wdm0 --device-open-net='net-raw-ip|net-no-qos-header' --wds-start-network="apn='Your_APN',username='Your_Username',password='Your_Password'" --client-no-release-cid

[/dev/cdc-wdm0] Network started
	Packet data handle: '3059871808'
[/dev/cdc-wdm0] Client ID not released:
	Service: 'wds'
	    CID: '23'
```
my client setup
```
sudo qmicli -p -d /dev/cdc-wdm0 --device-open-net='net-raw-ip|net-no-qos-header' --wds-start-network="apn='internet',username='true',password='true'" --client-no-release-cid
```


if the network timeout, it could be that you skip the state of `sudo ip link set wwan0 down` to `sudo ip link set wwan0 up`. Try again. But sometimes it is not this. 

- then configure the IP address and the default route with udhcpc
```
>> sudo udhcpc -q -f -i wwan0
udhcpc: started, v1.30.1
udhcpc: sending discover
udhcpc: sending select for 10.38.223.119
udhcpc: lease of 10.38.223.119 obtained, lease time 7200
```

- check the assigned IP address
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

- check the network connection
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

## start AT command with minicom <a name = "atminicom"></a>
```
>> sudo dmesg | grep /dev/ttyUSB
```
it should show USB0,1,2,3. From https://bacnh.com/quectel-linux-usb-drivers-troubleshooting, it said 
> - /dev/ttyUSB0 - DM \
> - /dev/ttyUSB1 - For GPS NMEA message output \
> - /dev/ttyUSB2 - For AT command communication \
> - /dev/ttyUSB3 - For PPP connection or AT command communication \

Therefore, we are going to use /dev/ttyUSB2 for AT command

- setting serial port
```
>> sudo minicom -s
```
we are setting serial port </br>
<img src="https://github.com/pchat-imm/quectel_rm510q_gl/assets/40858099/eff7a2fd-395d-41b7-8725-8a9177d57f36" width="30%" height="30%"/> <br/>
make sure </br>
> - serial device = /dev/ttyUSB2 \
> - bps = 115200 (default) \
> - Hardware flow control = No \

<img src="https://github.com/pchat-imm/quectel_rm510q_gl/assets/40858099/a289b780-4135-44cd-a02e-da1e2a03187a" width=50% height=50%/> <br/>
everytime finish `Save setup as dfl` before `exit`

to exit minicom type `ctrl + A` then `x`
and to enter special menu `ctrl + A` then `z`

## AT command
from
- [Ettus/OAI Reference Architecture for 5G and 6G Research with USRP](https://kb.ettus.com/OAI_Reference_Architecture_for_5G_and_6G_Research_with_USRP)
- [OAI/NR_SA_Tutorial_COTS_UE.md](https://gitlab.eurecom.fr/oai/openairinterface5g/-/blob/develop/doc/NR_SA_Tutorial_COTS_UE.md)

- using ctrl A + E = echo ON, to show what you type on the screen
- usign ctrl A + C = clear screen
- using ctrl A + X to quit the minicom
- if minicom freeze, open another window and try
```
>> ps aux | grep minicom
>> sudo kill -SIGTERM <PID>
```

```
AT
OK

AT+COPS?                                                                           
+COPS: 0,0,"TRUE-H TRUE-H",13                                                
                                                                  
AT+CGFCONT?                                                                     
+CGDCONT: 1,"IPV4V6","","0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0",0,0,0,0,,,,,,,,,"",,,0
+CGDCONT: 2,"IPV4V6","ims","0.0.0.0.0.0.0.                                   

AT+CPIN?                                                                            
+CPIN: READY

AT+QNWCFG=?
+QNWCFG: "lte_cdrx",(0,1),(0,1)
+QNWCFG: "nr5g_cdrx",(0,1)
+QNWCFG: "csi_ctrl",(0,1),(0,1)
+QNWCFG: "lte_csi",(0-31),<ri>,

AT+QCAINFO
+QCAINFO: "PCC",250,50,"LTE BAND 1",1,167,-116,-13,-86,7

AT+QENG=?
+QENG: "servingcell","NOCONN"                                         
+QENG: "LTE","FDD",520,04,4D8002,167,250,1,3,3,7E7,-128,-17,-94,7,10,23-
+QENG: "NR5G-NSA",                                                    
OK

AT+QNWPREFCFG=?
+QNWPREFCFG: "gw_band",B1:...:BN
+QNWPREFCFG: "lte_band",B1:...:BN
+QNWPREFCFG: "nsa_nr5g_band",B1:...:BN
+QNWPREFCFG: "nr5g_band",B1:...:BN
+QNWPREFCFG: "mode_pref",RAT1:...:RATN
+QNWPREFCFG: "srv_domain",(0-2)
+QNWPREFCFG: "voice_domain",(0-3)
+QNWPREFCFG: "roam_pref",(1,3,255)
+QNWPREFCFG: "ue_usage_setting",(0,1)
+QNWPREFCFG: "policy_band"
+QNWPREFCFG: "ue_capabi

AT+QNWPREFCFG="nr5g_band"
+QNWPREFCFG: "nr5g_band",1:2:3:5:7:8:12:20:25:28:38:40:41:48:66:719
```

### more AT command operation from https://hackmd.io/@yeneronur/SJDIPBWns#Instructions-for-Quectel 
- quectel information
```
ATI
Quectel                                                                         
RM510Q-GL                                                                       
Revision: RM510QGLAAR11A03M4G       
OK
```
- Firmware update `AT+QMBNCFG=”Select”,”Row_commercial”` returns `OK`
- reboot (always wait for the reboot to finish)
```
AT+CFUN=1,1                                                                     
OK                                                                              
RDY                                                                             
+CPIN: READY                                                                    
+QUSIM: 1                                                                       
+CFUN: 1                                                                        
+QIND: SMS DONE                                                                 
+QIND: PB DONE                                                                  
ATE0                                                                            
OK
+CRSM: 148,8,""                                                                 
OK                                                                              
+CEMODE: 2
OK                                                                               
+CCLK: "24/01/16,08:46:44+28"                                                   
OK  
```
```
at+cfun=1,1
OK
RDY
+CPIN: READY
+QUSIM: 1
+CFUN: 1
+QIND: SMS DONE
+QIND: PB DONE
ATE0
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
```

- activate PDP context (if it show +CME ERROR: 30 means you didn't wait for the reboot to finish,reboot again and wait for it to finish)
```
AT+CGACT=1,1
+CCLK: "24/01/18,08:39:27+28"                                                     OK
```
- show PDP address (it will return address in "". If there is no address here, reboot again and wait for it to finish)
```
AT+CGPADDR=1                                                                    
+CGPADDR: 1,"10.101.133.178" 
OK  
```
- verify network setting `AT+CGDCONT?`
- ping website
```
AT+QPING=1,"8.8.8.8"                                                            
OK                                                                              
+QPING: 561


AT+QPING=1,"openairinterface.org"                                               
OK                                                                              
+QPING: 561

at+qping=1,"8.8.8.8"
+QPING: 569
```
```
at+qping=1,"openairinterface.org"
OK
+QPING: 0,"137.74.50.85",32,224,255
+QPING: 0,"137.74.50.85",32,225,255
+QPING: 0,"137.74.50.85",32,225,255
+QPING: 0,"137.74.50.85",32,221,255
+QPING: 0,4,4,0,221,225,223

at+qping=1,"google.com"
OK
+QPING: 0,"216.58.200.14",32,49,255
+QPING: 0,"216.58.200.14",32,42,255
+QPING: 0,"216.58.200.14",32,58,255
+QPING: 0,"216.58.200.14",32,44,255
+QPING: 0,4,4,0,42,58,47
```

show current firmware version
```
at+gmr                                  
RM510QGLAAR11A03M4G   
```
show IMSI and IMEI of (U)SIM
```
at+cimi                                 
+CCLK: "24/01/18,08:32:36+28"           
OK

at+gsn
867034040025018
OK
```

display 5g nr frequencuy band
```
at+qnwprefcfg="nr5g_band"
+QNWPREFCFG: "nr5g_band",1:2:3:5:7:8:12:20:25:28:38:40:41:48:66:71:77:78:79
```

display then set operation mode to 5G NR, then enable 5G operation
```
at+qnwprefcfg="mode_pref"
+QNWPREFCFG: "mode_pref",AUTO
OK

at+qnwprefcfg="mode_pref",nr5g
+CCLK: "24/01/18,08:50:24+28"
OK

at+qnwprefcfg="nr5g_disable_mode",0
+CCLK: "24/01/18,08:50:59+28"
OK

```
