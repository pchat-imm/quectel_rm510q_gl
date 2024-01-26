## Quectel_RM510Q_GL

## Table of contents
1. set data connection \
1.1. [setting up a data connection over QMI interface using libqmi](#setupqmi) \
1.2. [setting up a data connection use mmcli](#setupmmcli) \
2. AT command \
2.1. [start AT command with minicom](#atminicom_basic) \
2.2. [AT command for start using 5G](#atminicom_5G) \
2.3. [AT command for start using 4G only](#atminicom_4G) \
3. use case
3.1. [quectel for srsRAN4G](#quectel_srsRAN4G)

## successful test case
- quectel + sim 5G to internet
- quectel + sysmocom sim to srsRAN4G

## 1. setup data connection
### 1.1. setting up a data connection over QMI interface using libqmi <a name = "setupqmi"></a>
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
<details close>
<summary><b> other basic command to check the quectel board with qmicli command </b></summary>
	
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
my client sim card setup
```
sudo qmicli -p -d /dev/cdc-wdm0 --device-open-net='net-raw-ip|net-no-qos-header' --wds-start-network="apn='internet',username='true',password='true'" --client-no-release-cid
```
with sysmocom sim card
```
sudo qmicli -p -d /dev/cdc-wdm0 --device-open-net='net-raw-ip|net-no-qos-header' --wds-start-network="apn='srsapn'" --client-no-release-cid 
error: couldn't start network: QMI protocol error (14): 'CallFailed'
call end reason (3): generic-no-service
verbose call end reason (3,2001): [cm] no-service
[/dev/cdc-wdm0] Client ID not released:
	Service: 'wds'
	    CID: '16'
```
if the network timeout, it could be that you skip the state of `sudo ip link set wwan0 down` to `sudo ip link set wwan0 up`. Try again. But sometimes it is not this. 

- then configure the IP address and the default route with udhcpc
```
sudo udhcpc -q -f -i wwan0
	udhcpc: started, v1.30.1
	udhcpc: sending discover
	udhcpc: sending select for 10.38.223.119
	udhcpc: lease of 10.38.223.119 obtained, lease time 7200
```
in case you connect quectel with srsRAN_4G that you have already masq the interface to `wlp9s0' the selected interface will be changed
```
sudo udhcpc -q -f -i wlp9s0
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

<details close>
<summary><i> sysmocom sim </i></summary>
	
|IMSI	|ICCID	|ACC	|PIN1	|PUK1	|PIN2	|PUK2	|Ki	|OPC	|ADM1	|KIC1	|KID1	|KIK1	|KIC2	|KID2	|KIK2	|KIC3	|KID3	|KIK3
| ---  | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
|999700000062713	|8988211000000627136	|0008	|2481	|43215893	|5679	|21192366	|96E5235D7BD18E48BEF1B85521383C4E	|B1C0A05123C419D615B71EC0F8CE13AB	|73947583	|BAEFEE018E08B0DE276DCF03900BE2AF	|B2037C9475B7C9A2D8637F8B9651B835	|AED8AB5736726DB4BF6CF1FE44E61BF6	|EF00A3344612955BC3144E4DF8C719D4	|A42E9EBDFB3768C98AFEED6154E375F7	|240A034AE19677D51B1CB19DD5F63503	|6AC9B3640FD1FD90D50B43004C72C0A4	|EEA71035E53F67E7266E2C954212E6BC	|55CADF364D70E23D7ADFA510902ABFC2|
</details>

### 1.2. setting up a data connection use mmcli <a name = "setupmmcli"></a>
##### from https://github.com/srsran/srsRAN_Project/discussions/426#discussioncomment-8233829
list connected modem
```
>> sudo mmcli -L
    /org/freedesktop/ModemManager1/Modem/2 [Quectel] RM510Q-GL
```
The above has modem number = 2, then connect the network. The simple-connect command initiate a PDN connection for DNN "srsapn".
```
>> sudo mmcli -L
    /org/freedesktop/ModemManager1/Modem/2 [Quectel] RM510Q-GL
```
then set interface up, in my case is wlp9s0
```
>> sudo ip link set wlp9s0 up
```
then ping
```
>> ping 8.8.8.8 -I wlp9s0
```
## 2. AT command
### 2.1. start AT command with minicom <a name = "atminicom_basic"></a>
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

- using ctrl A + E = echo ON, to show what you type on the screen
- usign ctrl A + C = clear screen
- using ctrl A + X to quit the minicom
- if minicom freeze, open another window and try

<details close>
<summary><i> AT command basic </i></summary> 

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
</details>

### 2.2. AT command for start using 5G <a name = "atminicom_5G"></a
##### from https://hackmd.io/@yeneronur/SJDIPBWns#Instructions-for-Quectel 
- quectel information
```
ATI
Quectel                                                                         
RM510Q-GL                                                                       
Revision: RM510QGLAAR11A03M4G       
OK
```
- Firmware update
```
AT+QMBNCFG=”Select”,”Row_commercial”` returns `OK`
```
- reboot (always wait for the reboot to finish)
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
#### the main part of start 5G mode in quectel
unlock the quectel
```
at+qmbncfg="Select","Row_commercial"                                            
OK
```
display 5g nr frequencuy band
```
at+qnwprefcfg="nr5g_band"
+QNWPREFCFG: "nr5g_band",1:2:3:5:7:8:12:20:25:28:38:40:41:48:66:71:77:78:79
```
display then set operation mode to 5G NR, then enable 5G operation
```
at+qnwprefcfg="mode_pref",nr5g
+CCLK: "24/01/18,08:50:24+28"
OK
```
start using on 5G
```
at+qnwprefcfg="nr5g_disable_mode",0
+CCLK: "24/01/18,08:50:59+28"
OK
```

### 2.3. AT command for start using 4G only <a name = "atminicom_4G">
```
at+qnwprefcfg = "lte_band"
at+qnwprefcfg = "mode_pref"
at+qnwprefcfg = "mode_pref", LTE
```
## 3. use case
### 3.1. quectel as a UE for srsRAN4G <a name = "quectel_srsRAN4G">
1. masq the interface
```
cd ~/.config/srsran
sudo srsepc_if_masq.sh wlp9s0
ifconfig
```
2. run epc
```
cd ~/.config/srsran
sudo srsepc epc.conf
```
3. run enb
```
cd ~/.config/srsran
sudo srsenb enb.conf
```
4. start connect quectel to the masq
```
>> lsusb
>> sudo qmicli --device=/dev/cdc-wdm0 --dms-get-operating-mode 
>> sudo qmicli -p -d /dev/cdc-wdm0 --device-open-net='net-raw-ip|net-no-qos-header' --wds-start-network="apn='srsapn'" --client-no-release-cid 
>> sudo udhcpc -q -f -i wlp9s0
>> ifconfig wlp9s0
>> ping -I wwan0 -c 5 8.8.8.8
```
5. run AT command to ping to the gNB
```
>> sudo minicom -s

at
at+cops
at+qnwprefcfg = "lte_band"
at+qnwprefcfg = "mode_pref"
at+qnwprefcfg = "mode_pref", LTE
AT+QPING=1,"8.8.8.8"      
```

<details close>
<summary> log </summary>

log epc
```
chatchamon@chatchamon-ThinkPad-L14-Gen-2:~/.config/srsran$ sudo srsepc epc.conf 
[sudo] password for chatchamon: 

Built in Release mode using commit eea87b1d8 on branch master.


---  Software Radio Systems EPC  ---

Reading configuration file epc.conf...
HSS Initialized.
MME S11 Initialized
MME GTP-C Initialized
MME Initialized. MCC: 0xf999, MNC: 0xff70
SPGW GTP-U Initialized.
SPGW S11 Initialized.
SP-GW Initialized.
Received S1 Setup Request.
S1 Setup Request - eNB Name: srsenb01, eNB id: 0x19b
S1 Setup Request - MCC:999, MNC:70
S1 Setup Request - TAC 7, B-PLMN 0x99f907
S1 Setup Request - Paging DRX v128
Sending S1 Setup Response
Initial UE message: LIBLTE_MME_MSG_TYPE_ATTACH_REQUEST
Received Initial UE message -- Attach Request
Attach request -- IMSI: 999700000062713
Attach request -- eNB-UE S1AP Id: 1
Attach request -- Attach type: 2
Attach Request -- UE Network Capabilities EEA: 11110000
Attach Request -- UE Network Capabilities EIA: 01110000
Attach Request -- MS Network Capabilities Present: false
PDN Connectivity Request -- EPS Bearer Identity requested: 0
PDN Connectivity Request -- Procedure Transaction Id: 12
PDN Connectivity Request -- ESM Information Transfer requested: false
Downlink NAS: Sending Authentication Request
UL NAS: Received Authentication Response
Authentication Response -- IMSI 999700000062713
UE Authentication Accepted.
Generating KeNB with UL NAS COUNT: 0
Downlink NAS: Sending NAS Security Mode Command.
UL NAS: Received Security Mode Complete
Security Mode Command Complete -- IMSI: 999700000062713
Getting subscription information -- QCI 9
Sending Create Session Request.
Creating Session Response -- IMSI: 999700000062713
Creating Session Response -- MME control TEID: 1
Received GTP-C PDU. Message type: GTPC_MSG_TYPE_CREATE_SESSION_REQUEST
SPGW: Allocated Ctrl TEID 1
SPGW: Allocated User TEID 1
SPGW: Allocate UE IP 172.16.0.2
Received Create Session Response
Create Session Response -- SPGW control TEID 1
Create Session Response -- SPGW S1-U Address: 127.0.1.100
SPGW Allocated IP 172.16.0.2 to IMSI 999700000062713
Adding attach accept to Initial Context Setup Request
Sent Initial Context Setup Request. E-RAB id 5 
Received Initial Context Setup Response
E-RAB Context Setup. E-RAB id 5
E-RAB Context -- eNB TEID 0x1; eNB GTP-U Address 127.0.1.1
UL NAS: Received Attach Complete
Unpacked Attached Complete Message. IMSI 999700000062713
Unpacked Activate Default EPS Bearer message. EPS Bearer id 5
Received GTP-C PDU. Message type: GTPC_MSG_TYPE_MODIFY_BEARER_REQUEST
Sending EMM Information
Received UE Context Release Request. MME-UE S1AP Id 1
There are active E-RABs, send release access bearers request
Received GTP-C PDU. Message type: GTPC_MSG_TYPE_RELEASE_ACCESS_BEARERS_REQUEST
Received UE Context Release Complete. MME-UE S1AP Id 1
UE Context Release Completed.
Initial UE message: NAS Message Type Unknown
Received Initial UE message -- Service Request
Service request -- S-TMSI 0x3acabeef
Service request -- eNB UE S1AP Id 2
Service Request -- Short MAC valid
Service Request -- User is ECM DISCONNECTED
UE previously assigned IP: 172.16.0.2
Generating KeNB with UL NAS COUNT: 2
UE Ctr TEID 0
Sent Initial Context Setup Request. E-RAB id 5 
Received Initial Context Setup Response
E-RAB Context Setup. E-RAB id 5
E-RAB Context -- eNB TEID 0x2; eNB GTP-U Address 127.0.1.1
Initial Context Setup Response triggered from Service Request.
Sending Modify Bearer Request.
Received GTP-C PDU. Message type: GTPC_MSG_TYPE_MODIFY_BEARER_REQUEST
Received UE Context Release Request. MME-UE S1AP Id 2
There are active E-RABs, send release access bearers request
Received GTP-C PDU. Message type: GTPC_MSG_TYPE_RELEASE_ACCESS_BEARERS_REQUEST
Received UE Context Release Complete. MME-UE S1AP Id 2
UE Context Release Completed.
```

log enb
```
chatchamon@chatchamon-ThinkPad-L14-Gen-2:~/.config/srsran$ sudo srsenb enb.conf 
[sudo] password for chatchamon: 
Active RF plugins: libsrsran_rf_blade.so libsrsran_rf_zmq.so
Inactive RF plugins: 
---  Software Radio Systems LTE eNodeB  ---

Reading configuration file enb.conf...
WARNING: cpu0 scaling governor is not set to performance mode. Realtime processing could be compromised. Consider setting it to performance mode before running the application.

Built in Release mode using commit eea87b1d8 on branch master.

ing 2 channels in RF device=bladeRF with args=default
Supported RF device list: bladeRF zmq file
Opening bladeRF...
Set RX sampling rate 1.92 Mhz, filter BW: 1.92 Mhz

==== eNodeB started ===
Type <t> to view trace
Set RX sampling rate 3.84 Mhz, filter BW: 3.07 Mhz
Setting manual TX/RX offset to 27 samples
Setting frequency: DL=1842.5 Mhz, UL=1747.5 MHz for cc_idx=0 nof_prb=15
set TX frequency to 1842500000
set TX frequency to 1842500000
set RX frequency to 1747500000
set RX frequency to 1747500000
RACH:  tti=9661, cc=0, pci=1, preamble=47, offset=25, temp_crnti=0x46
RACH:  tti=9601, cc=0, pci=1, preamble=21, offset=25, temp_crnti=0x47
User 0x47 connected
Disconnecting rnti=0x46.
Disconnecting rnti=0x47.
t
Enter t to stop trace.
RF status: O=1, U=0, L=0
RACH:  tti=9071, cc=0, pci=1, preamble=39, offset=25, temp_crnti=0x48
User 0x48 connected

               -----------------DL----------------|-------------------------UL-------------------------
rat  pci rnti  cqi  ri  mcs  brate   ok  nok  (%) | pusch  pucch  phr  mcs  brate   ok  nok  (%)    bsr
lte    1   48   15   1    4   4.2k    7    0   0% |  14.0   11.5   40   10   146k   22    0   0%    0.0
lte    1   48   15   1    0      0    0    0   0% |   n/a   99.9    0    0      0    0    0   0%    0.0
lte    1   48   15   1    0      0    0    0   0% |   n/a   99.9    0    0      0    0    0   0%    0.0
lte    1   48   15   1    1     72    1    0   0% |  11.4   10.2   40   12   2.4k    1    0   0%    0.0
lte    1   48   15   1    0      0    0    0   0% |   n/a   99.9    0    0      0    0    0   0%    0.0
lte    1   48   15   1    0      0    0    0   0% |   n/a   99.9    0    0      0    0    0   0%    0.0
lte    1   48   15   1    0      0    0    0   0% |   n/a   99.9    0    0      0    0    0   0%    0.0
lte    1   48   15   1    1     72    1    0   0% |  12.9   10.5   40    9   1.9k    1    0   0%    0.0
lte    1   48   15   1    0      0    0    0   0% |   n/a   99.9    0    0      0    0    0   0%    0.0
lte    1   48   15   1    0      0    0    0   0% |   n/a   99.9    0    0      0    0    0   0%    0.0
lte    1   48   15   1    0      0    0    0   0% |   n/a   99.9    0    0      0    0    0   0%    0.0
```
</details>






