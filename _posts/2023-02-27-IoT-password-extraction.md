---
title: "Extracting the Wi-Fi password from a Tuya smart bulb"
author: "Martin Svalstuen Brunæs"
date: "2023-02-28"
layout: post
categories: post
---

## Extracting the Wi-Fi password from a Tuya smart bulb

### Introduction
About a year ago I purchased 10 LED smart bulbs that were generically branded for 10$ a piece. I wanted to check out the hype revolving smart home applications, but I was not interested in paying almost 4 times the amount for the popular Phillips Hue bulbs. However, I being a security enthusiast, I knew that a high focus on price and schedule tends to impact the quality attributes of the system, including security. I tried out the bulbs for several months and my impression was that they worked, but they did not provide a high quality feel in terms of app-responsiveness. Two of the bulbs also stopped working way before the claimed 30 000 hours lifetime. This gave some hope of finding low hanging security fruits.  


### User Installation Process

From a user perspective, you set up the smart bulbs following these steps:
1. Plug the bulb into a 230V E27 socket and turn on the power
2. Install the Smart Home App (In my case for IOS)
3. Follow the pairing wizard

### Technical View of Installation Process

1. The first time the pairing process is launched, the bulb boots and creates its own wifi network
2. The mobile phone running Smart Home connects to the bulb's network
3. The user inputs the home WiFi credentials and they are sent to the bulb for storage
4. The next time the bulb boots, it will join the home network using the saved credentials
5. From now on, the bulb is connected to the home Wi-Fi network
6. The Smart Home app communicates with the Tuya servers, which act as a transceiver between the smart bulb and the app.


### Security Concerns
Based on the above operation, there are multiple security concerns:
1. Since the system architecture requires internet access, you are essentially increasing the attack surface by exposing the device to the internet. There is a very high likelihood that the bulb contains vulnerabilities waiting to be exploited by adversaries. In short, system development often results in undesired emerging behavior, due to the interactions between system components that the developers did not anticipate.
2.  Since the bulb knows how to connect to the home Wi-fi it must have stored the SSID and password inside the device. For embedded devices, it is common to store such information in flash memory, either inside a SoC or in external flash memory. Embedded low-cost IoT devices may not have the proper security controls, such as the ability to encrypt the flash memory, which may leave sensitive information exposed to adversaries. 

In this blog post, we will explore the latter security concern by attempting to access the flash memory of the smart bulb and see if we can retrieve the password for the home Wi-Fi. This will require physical access to the device. However, if we consider the smart bulb's life cycle, we can easily imagine that one throws away a defective bulb in the trash which could end up in the hands of an adversary. Access to your home Wi-fi could be the foothold that an adversary needs to perform other attacks that may lead to important assets' confidentiality, integrity and availability being violated. 

### Step 1 - Exposing the hardware
I used a hacksaw to cut my way through the metal casing in a couple of minutes.
![image](https://user-images.githubusercontent.com/13424965/218526753-da06b0b5-29b0-4a52-b44c-d5326e2e7737.png)

The electrical system comprises two PCBs. One is driving the LED lights and the other contains the power step down circuit, connectivity and logic. The main PCB was easily separated from the LED PCB as it was only connected through header pins. 

![image](https://user-images.githubusercontent.com/13424965/218526914-ec42fa0b-c13f-4db2-85e9-b50885b6b8cc.png)

### Step 2 - Identifying the hardware components

Since we know that SoC's don't run at 230V directly, the board must contain circuitry to step down the voltage to the usual range of 2,7V - 6V. The through-hole components and transformers are related to this functionality. Obviously, the silver module board draws our attention and is the prime suspect of containing the flash memory that we are after. I wanted to check if I was still able to see the device in the app to check if it was still functioning, so I attached 3.3V to the input pins, and confirmed that it still worked. To be able to identify the module, I desoldered it from the PCB. When inspecting the backside of the module, it showed the silkprint.

After doing some OSINT, I suspected that this hardware module is probably the WB3S module developed by Tuya based on the formfactor and pinout [1]. The datasheet of the WB3S reads that it contains a 2MB flash onboard, and I did not find any other flash ICs on the PCB upon inspection.

![image](https://user-images.githubusercontent.com/13424965/218527002-c550c8f8-f4bd-4247-bacf-087ee8f981c2.png)

In order to verify that the circuit still worked, I powered it on. It still shows up in the app, which is a great relief. Upon closer inspection, by tearing back the silver metal cover, I observed the BK7231T SoC with some passive components and an oscillator. This physical layout also matches the Tuya product page, so we can be pretty sure we are dealing with a WB3S module. 

![image](https://user-images.githubusercontent.com/13424965/218554533-4a145527-0ca9-4996-9829-c4096a957495.png)

Now we have got direct access to the pins of the BK7231T, so even if the manufacturer tried to follow Tuya's advice of "Test pins are not recommended." we can still access what we want. 

### Step 3 - Probing the serial ports 

![image](https://user-images.githubusercontent.com/13424965/221281217-c9593ff6-a3f1-42aa-984f-e9f8d782c990.png)


Most embedded devices, if not all, contain serial interfaces that are used to debug and program the device before it leaves the factory. Sometimes, debug functionality is locked down before the device leaves the factory to reduce the attack surface, i.e. and increase security. However, this is not always the case, so it is good practice to check if the serial ports give away any information that you can use to identify the device or further attacks. I observed 2 pairs of "TX/RX" pins which I suspected were UART ports. Tuya's own product page states that there are indeed two UART ports. After inspecting both UART's TX ports on an oscilloscope, I observed that 2TX sends data when the device boots. By looking at the highest frequency part of the signal, it showed a baudrate of 115 200 Hz. After rebooting the device and connecting the TX port to a logic analyzer at 15200 Hz, I obtained the following information dump:

```


V:BK7231S_1.0.5

CPSR:000000D3

R0:BFF7DFFF

R1:FBDEFC77

R2:9A7C8FA6

R3:7BD5DFFE

R4:4BDBFAB4

R13:DEFF7D5C

R14(LR):FDEB7F7E

ST:3A589E3A

J 0x10000

prvHeapInit-start addr:0x4203d0, size:130096
[01-01 18:12:15 TUYA Info][mqc_app.c:175] mqc app init ...
[01-01 18:12:15 TUYA Info][sf_mqc_cb.c:42] register mqc app callback
[01-01 18:12:15 TUYA Debug][mqc_app.c:118] mq_pro:5 mqc_handler_cnt:1
[01-01 18:12:15 TUYA Debug][mqc_app.c:118] mq_pro:31 mqc_handler_cnt:2
[01-01 18:12:15 TUYA Debug][uni_thread.c:215] Thread:sys_timer Exec Start. Set to Running Status
[01-01 18:12:15 TUYA Debug][log_seq.c:732] read from uf. max:0 first:0 last:0
[01-01 18:12:15 TUYA Debug][svc_online_log.c:288] svc online log init success
[01-01 18:12:15 TUYA Err][tuya_ws_db.c:314] kvs_read fails gw_bi -1
[01-01 18:12:15 TUYA Err][ws_db_gw.c:111] gw base read fails -935
[01-01 18:12:15 TUYA Debug][tuya_bt_sdk.c:89] ty bt cmmod register finish 1
[01-01 18:12:15 TUYA Debug][tuya_ble_api.c:301] ble sdk inited
!!!!!!!!!!tuya_bt_port_init
[01-01 18:12:15 TUYA Debug][tuya_ble_api.c:337] ble sdk re_inited
[01-01 18:12:15 TUYA Notice][tuya_bt_sdk.c:130] ty bt sdk init success finish
[01-01 18:12:15 TUYA Notice][light_system.c:2196] go to pre device!
[01-01 18:12:15 TUYA Err][uf_flash_file_app.c:266] uf_open 8 err 8
[01-01 18:12:15 TUYA Err][soc_flash.c:77] uf file 8 can't open and read data!
[01-01 18:12:15 TUYA Err][light_memory.c:102] power_mem data read error,read cnt -1!
bk_rst:0 tuya_rst:0[01-01 18:12:15 TUYA Err][uf_flash_file_app.c:266] uf_open 8 err 8
[01-01 18:12:15 TUYA Err][soc_flash.c:77] uf file 8 can't open and read data!
[01-01 18:12:15 TUYA Err][light_memory.c:102] power_mem data read error,read cnt -1!
bk_rst:0 tuya_rst:0[01-01 18:12:15 TUYA Notice][light_system.c:2213] goto first bright up!
bk_rst:0 tuya_rst:0[01-01 18:12:15 TUYA Notice][simple_flash.c:432] key_addr: 0x1ee000   block_sz 4096
[01-01 18:12:15 TUYA Notice][simple_flash.c:500] get key:
0xcb 0x4e 0x3e 0xa4 0x0 0x30 0x9d 0xab 0x65 0x6d 0x8d 0xbf 0xe4 0xb9 0x3f 0x35 
[01-01 18:12:15 TUYA Notice][tuya_main.c:311] **********[oem_bk7231s_light_ty] [2.9.6] compiled at Oct 29 2020 14:38:00**********
[bk]tx_txdesc_flush
[rx_iq]rx_amp_err_rd: 0x000
[rx_iq]rx_phase_err_rd: 0xffffffec
[rx_iq]rx_ty2_rd: 0x000
*********** finally result **********
gtx_dcorMod            : 0x8
gtx_dcorPA             : 0xa
gtx_pre_gain           : 0x0
gtx_i_dc_comp          : 0x1ff
gtx_q_dc_comp          : 0x20b
gtx_i_gain_comp        : 0x3ff
gtx_q_gain_comp        : 0x3ee
gtx_ifilter_corner over: 0xb
gtx_qfilter_corner over: 0xb
gtx_phase_comp         : 0x1fa
gtx_phase_ty2          : 0x200
gbias_after_cal        : 0x13
gav_tssi               : 0x20
g_rx_dc_gain_tab 0 over: 0x86788478
g_rx_dc_gain_tab 1 over: 0x88768870
g_rx_dc_gain_tab 2 over: 0x90689068
g_rx_dc_gain_tab 3 over: 0xac40a450
g_rx_dc_gain_tab 4 over: 0xb03eae3e
g_rx_dc_gain_tab 5 over: 0xb040b040
g_rx_dc_gain_tab 6 over: 0xb042b041
g_rx_dc_gain_tab 7 over: 0xaf42b041
grx_amp_err_wr         : 0x200
grx_phase_err_wr       : 0x3f6
**************************************
ble use fit!
temp in flash is:263
lpf_i & q in flash is:11, 12
xtal in flash is:20
-----pwr_gain:12, g_idx:12, shift_b:0, shift_g:0
-----[pwr_gain]12
Initializing TCP/IP stack
[01-01 18:12:15 TUYA Notice][tuya_main.c:337] have actived over 15 min, not enter mf_init
[01-01 18:12:15 TUYA Notice][tuya_main.c:341] mf_init succ
[01-01 18:12:15 TUYA Notice][light_system.c:2262] < TUYA IOT SDK V:1.0.2 BS:40.00_PT:2.2_LAN:3.3_CAD:1.0.2_CD:1.0.0 >
< BUILD AT:2020_09_25_17_24_52 BY embed FOR ty_iot_wf_bt_sdk_bk AT bk7231t >
IOT DEFS < WIFI_GW:1 DEBUG:1 KV_FILE:0 SHUTDOWN_MODE:0 LI[01-01 18:12:15 TUYA Notice][light_system.c:2263] oem_bk7231s_light_ty:2.9.6
[01-01 18:12:15 TUYA Notice][device_config_load.c:432] device config data already load! Don't load again!!
[01-01 18:12:15 TUYA Notice][light_system.c:2345] connect mode is 5
[01-01 18:12:15 TUYA Err][uf_flash_file_app.c:266] uf_open 2 err 8
[01-01 18:12:15 TUYA Err][soc_flash.c:77] uf file 2 can't open and read data!
[01-01 18:12:15 TUYA Err][user_flash.c:135] Production data read error!
[01-01 18:12:15 TUYA Notice][tuya_main.c:109] have actived over 15 min, not enter mf_init
[01-01 18:12:15 TUYA Notice][light_system.c:2145] frame goto init!
[01-01 18:12:15 TUYA Notice][gw_intf.c:3671] serial_no:10d561268d2f
[01-01 18:12:15 TUYA Notice][gw_intf.c:3706] gw_cntl.gw_wsm.stat:2
[01-01 18:12:15 TUYA Notice][gw_intf.c:3709] gw_cntl.gw_wsm.nc_tp:9
[01-01 18:12:15 TUYA Notice][gw_intf.c:3710] gw_cntl.gw_wsm.md:0
[01-01 18:12:15 TUYA Notice][gw_intf.c:3754] gw_cntl.gw_if.abi:0 input:0
[01-01 18:12:15 TUYA Notice][gw_intf.c:3755] gw_cntl.gw_if.product_key:keytg5kq8gvkv9dh, input:keytg5kq8gvkv9dh
[01-01 18:12:15 TUYA Notice][gw_intf.c:3756] gw_cntl.gw_if.tp:0, input:0
[01-01 18:12:16 TUYA Notice][gw_intf.c:3758] gw_cntl.gw_if.firmware_key:keytg5kq8gvkv9dh, input:keytg5kq8gvkv9dh
[01-01 18:12:16 TUYA Notice][tuya_bt_sdk.c:148] ty bt update product:keytg5kq8gvkv9dh 1
[01-01 18:12:16 TUYA Notice][light_system.c:2154] frame init out!
[01-01 18:12:16 TUYA Notice][light_system.c:2160] frame init ok!
[01-01 18:12:16 TUYA Notice][light_system.c:1957] gw status heap stack 66248
[01-01 18:12:16 TUYA Notice][tuya_bt_sdk.c:157] ty bt update localkey
!!!!!!!!!!tuya_bt_reset_adv
[01-01 01:00:01 TUYA Notice][tuya_ble_api.c:398] ble adv && resp changed
gapm_cmp_evt_handler operation = 0x1, status = 0x0 
gapm_cmp_evt_handler operation = 0x3, status = 0x0 
STACK INIT OK
ble create new db
ble_env->start_hdl = 0x7gapm_cmp_evt_handler operation = 0x1b, status = 0x0 
CREATE DB SUCCESS
!!!!!!!!!!tuya_bt_reset_adv
[01-01 01:00:01 TUYA Notice][tuya_ble_api.c:398] ble adv && resp changed
!!!!!!!!!!tuya_before_netcfg_cb
appm start advertising
fast_connect
[bk]tx_txdesc_flush
bssid 60-03-a6-58-ee-02
security2cipher 2 2 16 16 security=5
cipher2security 2 2 16 16
enter low level!
mac 10:d5:61:26:8d:2f
leave low level!
ssid:HOMEWIFINAME, 1
ht in scan
scan_start_req_handler
found scan rst rssi -55 < -50
dis ht_support
me_set_ps_disable:840 0 0 0 464773 996040
sm_auth_send:1
sm_auth_handler
ht NOT in assoc req
sm_assoc_rsp_handler
rc_init: station_id=0 format_mod=0 pre_type=0 short_gi=473501 max_bw=996551
rc_init: nss_max=0 mcs_max=0 r_idx_min=255 r_idx_max=473543 no_samples=996618
__l2_packet_send: ret 0
__l2_packet_send: ret 0
sta_mgmt_add_key
ctrl_port_hdl:1
me_set_ps_disable:840 0 0 0 464773 996040

configuring interface mlan (with DHCP client)
new ie: 0 : 53 6f 6c 61 6e 
new ie: 1 : 82 84 8b 96 c 12 18 24 
new ie: 3 : d 
new ie: 30 : 1 0 0 f ac 4 1 0 0 f ac 4 1 0 0 f ac 2 c 0 
new ie: 2d : ef 19 3 ff ff ff 0 1 0 0 0 0 0 0 0 1 0 0 0 0 0 19 4 7 1 0 

me_send_ps_req 2 0 0
[01-01 01:00:01 TUYA Notice][mqtt_client.c:1317] mqtt get serve ip success
set_ps_mode_cfm:963 1 0 4 464361 995984
 [01-01 01:00:01 TUYA Notice][tuya_tls.c:554] ret = 0
[01-01 01:00:02 TUYA Notice][mqtt_client.c:1347] mqtt socket create success. begin to connect
do td cur_t:255--last:idx:13,t:263 -- new:idx:12,t:251 
--0xc:08, shift_b:-1, shift_g:-1, X:0
[01-01 01:00:02 TUYA Notice][mqtt_client.c:1373] mqtt socket connect success. begin to subscribe [smart/device/in/bf591cfbd21e710d3cmsbe]
    [01-01 01:00:03 TUYA Err][uf_flash_file_app.c:266] uf_open 3 err 8
[01-01 01:00:03 TUYA Err][soc_flash.c:77] uf file 3 can't open and read data!
[01-01 01:00:03 TUYA Err][light_rhythm.c:89] Rhythm data read error, read cnt: -1!
[01-01 01:00:03 TUYA Notice][light_rhythm.c:833] no rhythm data!
[01-01 01:00:03 TUYA Err][uf_flash_file_app.c:266] uf_open 4 err 8
[01-01 01:00:03 TUYA Err][soc_flash.c:77] uf file 4 can't open and read data!
[01-01 01:00:03 TUYA Err][light_sleep.c:93] sleep data read error -1!
[01-01 01:00:03 TUYA Err][uf_flash_file_app.c:266] uf_open 5 err 8
[01-01 01:00:03 TUYA Err][soc_flash.c:77] uf file 5 can't open and read data!
[01-01 01:00:03 TUYA Err][light_wake.c:93] wake data read error -1!
[01-01 01:00:03 TUYA Notice][light_sleep.c:681] sleep response
     [01-01 01:00:05 TUYA Err][uf_flash_file_app.c:266] uf_open netcfg_log err 8
ÿ  /  !!!!!!!!!!tuya_bt_close:3
gapm_cmp_evt_handler operation = 0xd, status = 0x44 
[01-01 01:00:11 TUYA Notice][uni_work_queue.c:146] work_queue add task:423980
[01-01 01:00:11 TUYA Notice][uni_work_queue.c:146] work_queue add task:423a50
bk_rst:0 tuya_rst:0[01-01 01:00:11 TUYA Notice][tuya_tls.c:554] ret = 0
[12-30 14:03:06 TUYA Notice][tuya_tls.c:554] ret = 0
[12-30 14:03:10 TUYA Notice][uni_work_queue.c:146] work_queue add task:423a50
[12-30 14:03:10 TUYA Notice][uni_work_queue.c:146] work_queue add task:423bc0
[12-30 14:03:10 TUYA Notice][tuya_tls.c:554] ret = 0
[12-30 14:03:10 TUYA Err][tuya_svc_upgrade.c:281] result is null!
[12-30 14:03:10 TUYA Notice][tuya_tls.c:554] ret = 0

```
The log shows the SDK version to be BK7231S_1.0.5 and it shows the SSID of the home network. There is also meta data about how way it works such as messaging protocols, boot order, and configuration info. 

When I researched how to read the memory of the BK7231T I found an open-source firmware flashing tool called OpenBeken [2]. OpenBeken handles the interfacing for the device and only a USB UART bridge (I used Bus Pirate 3.6) is needed. The memory is read of the WB3S 1TX and 1RX pins. 

![image](https://user-images.githubusercontent.com/13424965/221281801-594f4a45-1ff1-491e-8ebe-1e9beed0a800.png)

The memory dump results in a text file. By searching for SSID in a simple text editor, the search-result yields the SSDD and the password! See image below.

![image](https://user-images.githubusercontent.com/13424965/221283892-9a4be4ea-eb7e-4173-8c8e-10ad726d65da.png)

## Conclusion

Tuya smart bulbs are IoT devices that are functionally decent and cheap. However, like many IoT devices, security is not always a sufficient priority to mitigate all types of attacks (such as physical access attacks). The blog post looks into a physical access attack where the Wi-Fi password of the associated network is exposed due to being stored unencrypted in the flash memory. Adversaries may use this attack to gain a foothold in the target's Wi-Fi network that can be used for other attacks.

### References
[1] https://developer.tuya.com/en/docs/iot/wb3s-module-datasheet?id=K9dx20n6hz5n4 \
[2] https://github.com/openshwprojects/OpenBK7231T_App
