### quectel_rm510q_gl
### setting up a data connection over QMI interface using libqmi

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
if not `online` set it
```
>> sudo qmicli --device=/dev/cdc-wdm0 --dms-set-operating-mode='online'
[/dev/cdc-wdm0] Operating mode set successfully
```

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

### start AT command with minicom
```
>> sudo dmesg | grep /dev/ttyUSB
```
it should show USB0,1,2,3. From https://bacnh.com/quectel-linux-usb-drivers-troubleshooting, it said 
> /dev/ttyUSB0 - DM \
> /dev/ttyUSB1 - For GPS NMEA message output \
> /dev/ttyUSB2 - For AT command communication \
> /dev/ttyUSB3 - For PPP connection or AT command communication \

Therefore, we are going to use /dev/ttyUSB2 for AT command

- setting serial port
```
>> sudo minicom -s
```
we are setting serial port 
<img src="https://github.com/pchat-imm/quectel_rm510q_gl/assets/40858099/eff7a2fd-395d-41b7-8725-8a9177d57f36" width="30%" height="30%"/> <br/>
make sure 
> serial device = /dev/ttyUSB2 \
> bps = 115200 (default) \
> Hardware flow control = No \

<img src="https://github.com/pchat-imm/quectel_rm510q_gl/assets/40858099/a289b780-4135-44cd-a02e-da1e2a03187a" width=50% height=50%/> <br/>
everytime finish `Save setup as dfl` before `exit`

## AT command
```
AT
OK
```
