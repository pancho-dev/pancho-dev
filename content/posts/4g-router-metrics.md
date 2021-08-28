---
title: "4G router metrics"
date: 2021-08-25T15:51:59-03:00
draft: true
tags: ["network","metrics", "prometheus"]
---

Since I have been working at home for over 5 years, I struggled a little with some ISPs and then though of getting redundant connections for those times my main ISP is having issues. Sometimes you only have 1 choice of ISP and you can get a second one in your area, and that becomes a problem. So I decided to test mobile operators afor my backup connection. For that endeavor first I tested the mobile operator I have for my phone and used it as tethering wiht my laptop.  
However, phone tethering had some disadvantages. I have a connected home so it means while main provider is out all my home gadgets stop working until main ISP is up and running again. I do have a linux router with multiple ports at home so I am capable of having multiple connections and having policy routing or even automatic failover of the conneciton when the primary connection goes down.  
Phone thethering also ate all my data on my phone a copuple of times and that is somthing I don't want. So I decided to get another prepaid sim card for this purpose, I would save money as I charge only when I need the sim card to have data. I tested a few vendors and chose the one that worked best. Then I found and old phone (Samsung Galaxy S5 mini) I had around and plug it to my linux router, it came up as an interface and voilá I had my second ISP for those times main main ISP is down.  
But, nothing is as easy as it looks. Turns out I was getitng 7Mb/s top download speed using the S5 mini, then I thought it was the prepaid sim card vendor, so I went and tested with my iPhone 12, and to my surprise I was getting 20-25 Mb/s download speeds which was pretty decent for my area and the signal strenght I usually get. But then, I would not leave a $700 phone to have a second ISP there, I could have scripted it to do all the magic when the iPhone got connected to the router, however I wanted something more permanent, where I could get some extra data.  

Then I  bought a cheap generic router with 4G/LTE capability that I would connect to my linux router. The only requirement I had was that the router provided a LAN port to conect it to my linux router. Not ideal as it would add an extra hop but it was just for backup purposes. Here are some pictures fo the router.
![](../images/4g-router-metrics/router1.png) ![](../images/4g-router-metrics/router2.png)

The router was cheap, ~$60 so it was in my price range and it is very generic, it doesn't have a lot of functions but serves my purpose to have a wireless provider as my backup ISP connected permanently to my linux router which does the more complex routing and functions for my home.  
At home, I have my netowrk monitored with prometheus, I have all ports, wifi and internet connection monitored so having a wireless router would be no stranger to that set up. Fortunately I have the 4G router connected to my linux router so monitoring bandwidth would just be as easy as monitor the port where the 4G router is connected to. But while configuring the 4G router I found a bunch of nice information I would like to have monitored in my prometheus, like signal strenght, SNR, and a bunch of cool stuff.  So I decided to investigate a bit more hot to get the data I can see in the UI and make a prometheus exporter that will be used to store this data in prometheus.


# Reverse engineering the router UI

In order to find out how to extract the router data I needed to inspect how the data was presented in the router's web interface. For that purpose I started using the web UI with developer tools from the brownser enabled to inspect the requests and responses and document them.  

### Finding how to log in
The first thing I needed to do was to find out how to log in to the 4G router, in the following picture is how I caught the login request

![](../images/4g-router-metrics/login1.png)

Then I could reproduce the log in with a `curl` request
```
curl -X POST  \
-H "Content-Type: application/x-www-form-urlencoded" \
-d "isTest=false&goformId=LOGIN&password=YWRtaW4%3D&username=YWRtaW4%3D" \
http://192.168.150.1/goform/goform_set_cmd_process | jq
```

Log in seems to work by logging the IP of the user (it doesn't set sessions or cockies) and there seems to be a restriction that 1 user can log in at a time. So if someone logs in to the router at the same time someone else logs in from a different IP, the original user will be kicked off. This is something to have in mind when building prometheus exporter that we might have unwanted behavior when pulling metrics from the device.  
Log in is a POST request and ofrm data parameters are
- goformId=LOGIN
- password=YWRtaW4%3D
- username=YWRtaW4%3D
NOTE: that password and username are first base64 encoded and then URL encoded so decodes URL to YWRtaW4= which decodes base64 to admin. Yes I tested this weith admin/admin user password. Off course this needs to be changed as soon as possible.

### Finding what data to collect
After log in we should be able to get all needed info, there seems to be 1 endpoint for getting all the data we need to export metrics, and the good news is that the router responds in json format. The enpoint is `/goform/goform_get_cmd_process` and the it will return the data based on the paramenters passed to the http GET request.
```
curl -s -X GET \
http://192.168.150.1/goform/goform_get_cmd_process\?cmd\=realtime_tx_bytes%2Crealtime_rx_bytes%2Csimcard_roam%2Crealtime_tx_thrpt%2Crealtime_rx_thrpt%2Crealtime_time%2Cmonthly_tx_bytes%2Cmonthly_rx_bytes%2Cflux_month_total\&multi_data\=1\&_\=1629929177313 | jq
{
  "realtime_tx_bytes": "422109",
  "realtime_rx_bytes": "6302856",
  "simcard_roam": "Home",
  "realtime_tx_thrpt": "0",
  "realtime_rx_thrpt": "0",
  "realtime_time": "7939",
  "monthly_tx_bytes": "6598555",
  "monthly_rx_bytes": "122883938",
  "flux_month_total": "129482493"
}
```
The previour curl request shows one example of the queries you can issue to. One parameter you can pass to the `/goform/goform_get_cmd_process` endpoint using GET method is named `cmd` and `multi_data=1`. To the cmd parameter we need to pass a comma separated list of fields we need.  After this step I needed to find first on the UI which data I want for metrics.  
Looking at the home 4G router's home page
![](../images/4g-router-metrics/home1.png)
There is a bunch of usefull information I would like to store in my monitoring system. I would to get wireless network information and signal information as well as traffic statistics from the home page. Also other information could be useful to get data for labelling metrics and such.  
There is also a page with more detailed wireless network information
![](../images/4g-router-metrics/network1.png)
From this page I want the get the most benefit. There is a bunch of good information to get out of the page. First the metrics I am interested in are RSRP/RSCP, SINR, RSSI and RSRQ which are metrics that indicate how the wireless signal is behaving, Also the other data is useful for loke operator, eNodebID, etc are candidates for labels in prometheus for the before mentioned metrics. So that way we can identify that we are jumping between cell towers or even operators. Also I would like to fin how the bar signal are calculated at the top right of the page, which is a more quick measure how the signal strenght is. All this values I want to keep them for long term to see how signal is is looking over time.

### How to find all the metrics
 Unfortunatelly I found there is no easy way to identify the fields in the json. A good thing here is that the json fields have exactly the same name as the parameters in the GET request, so we can mix adn match the request to get only the fields that we need.  

Here are some parameters I found that Would be helpful for me, I will be skipping anything related to wifi or LAN connection, I am only interested in the 4G/LTE metrics :

- flash_rest: Free router flash
- flash_total: total router flash
- nv_loadavg: router load average
- ram_rest: free RAM in MB
- ram_total: total RAM in MB
- ram_used: Percent usd RAM
- sms_unread_num: Number if unread SMS on the device (router). Metrics for router
- network_provider: is the id of the provider. Normally a number that identifies the provider of the sim card. This value can translate to label in the prometheus metrics related to celular network.
- signalbar: It's the parameter that indicates the signal bars at the top of the web interface. To my understanding this can take a value between 0 and 5. This value should be a metric for celular network
- network_type: Type of celular network connected; I am not sure what the values are, but I have seen 'LTE' only here. This value can translate to label in the prometheus metrics related to celular network.
- sub_network_type: Not what this is, but it related to network_type. I have seeen values here as 'FDD_LTE'. This can can also be a label for celular netowrk metrics.
- ppp_status: Status of the pppd connection. Value seen 'ppp_connected'. This could be translated to a metric 1 = ppp_connected , and 0 != ppp_connected. Also we can set as a label too as we don't know exactly what values this would take.
- simcard_roam: Not sure what this is the value I have seen is 'Home', But I would like to have as a value to compare when I move around and roam.
- monthly_rx_bytes: accumulated received bytes. This should be a metric counter for celular network
- monthly_tx_bytes: accumulated transfered bytes. This should be a metric counter for celular network
- realtime_tx_thrpt: instant received bytes per second. This should be a metric gauge for celular network
- realtime_rx_thrpt: instant received bytes per second. This should be a metric gauge for celular network
- realtime_time: This is the time the coonection has been set. THis is a counter that gets reset when session gets reset
- ipv6_wan_apn: wireless apn name for ipv6 connectivity
- wan_apn: wireless apn name for ipv6 connectivity
- nv_arfcn: EARFCN stands for E-UTRA Absolute Radio Frequency Channel Number.In LTE, the carrier frequency in the uplink and downlink is designated by EARFCN, which ranges between 0-65535. EARFCN uniquely identify the LTE band and carrier frequency. This should be used as label foe celular netowrk metrics
- nv_band: Celular network access band number. This should be used as label foe celular netowrk metrics
- nv_cellid: Celular network Cell ID. This should be used as label foe celular netowrk metrics
- nv_enodbid: Celular network eNodeB ID. This should be used as label foe celular netowrk metrics
- nv_globecellid: Celular network global ID. This should be used as label foe celular netowrk metrics
- nv_pci: A Physical Cell Id (PCI) is the identifier of a cell in the physical layer of the LTE net- work, which is used for separation of different transmitters. Due to the construction of PCIs, the number of PCIs are limited to 504. This should be used as label foe celular netowrk metrics
- wan_ipaddr: "100.85.118.91" This should be used as label foe celular netowrk metrics
- imei: Device IMEI number. This should be used as label foe celular netowrk metrics
- msisdn: Susbcription phone number. This should be used as label foe celular netowrk metrics
- sim_imsi: SIM IMSI number. This should be used as label foe celular netowrk metrics
- ziccid: SIm card ICCID number. This should be used as label foe celular netowrk metrics
- mcc: Mobile country code. Check mobile contry code mappings. This should be used as label foe celular netowrk metrics
- mnc: Mobile Network Code. This should be used as label foe celular netowrk metrics
- nv_sinr: Signal to Interference & Noise Ratio, ranges from 19.5dB (bad) to -3dB (good). This should be a metric for cellular network
- nv_rsrp: Reference Signal Receive Power. This should be a metric for cellular network
 RSRP mapping table
Code|	Range of RSRP in dBm|	Code |	Range of RSRP in dBm |	Code| 	Range of RSRP in dBm|
|---|---------------------|------|-----------------------|------|-----------------------|
0	|>-140	       | 33|	-108 to -107 |66|	-75 to -74|
1	|-140 to -139	 | 34|	-107 to -106   |67|	-74 to -73|
2	|-139 to -138	 | 35|	-106 to -105   |68|	-73 to -72|
3	|-138 to -137	 | 36|	-105 to -104   |69|	-72 to -71|
4	|-137 to -136	 | 37|	-104 to -103   |70|	-71 to -70|
5	|-136 to -135	 | 38|	-103 to -102   |71|	-70 to -69|
6	|-135 to -134	 | 39|	-102 to -101   |72|	-69 to -68|
7	|-134 to -133	 | 40|	-101 to -100   |73|	-68 to -67|
8	|-133 to -132	 | 41|	-100 to -99	   |74|	-67 to -66|
9	|-132 to -131	 | 42|	-99 to -98	   |75|	-66 to -65|
10|	-131 to -130|	43	|-98 to -97	   |76|	-65 to -64|
11|	-130 to -129|	44	|-97 to -96	   |77|	-64 to -63|
12|	-129 to -128|	45	|-96 to -95	   |78|	-63 to -62|
13|	-128 to -127|	46	|-95 to -94	   |79|	-62 to -61|
14|	-127 to -126|	47	|-94 to -93	   |80|	-61 to -60|
15|	-126 to -125|	48	|-93 to -92	   |81|	-60 to -59|
16|	-125 to -124|	49	|-92 to -91	   |82|	-59 to -58|
17|	-124 to -123|	50	|-91 to -90	   |83|	-58 to -57|
18|	-123 to -122|	51	|-90 to -89	   |84|	-57 to -56|
19|	-122 to -121|	52	|-89 to -88	   |85|	-56 to -55|
20|	-121 to -120|	53	|-88 to -87	   |86|	-55 to -54|
21|	-120 to -119|	54	|-87 to -86	   |87|	-54 to -53|
22|	-119 to -118|	55	|-86 to -85	   |88|	-53 to -52|
23|	-118 to -117|	56	|-85 to -84	   |89|	-52 to -51|
24|	-117 to -116|	57	|-84 to -83	   |90|	-51 to -50|
25|	-116 to -115|	58	|-83 to -82	   |91|	-50 to -49|
26|	-115 to -114|	59	|-82 to -81	   |92|	-49 to -48|
27|	-114 to -113|	60	|-81 to -80	   |93|	-48 to -47|
28|	-113 to -112|	61	|-80 to -79	   |94|	-47 to -46|
29|	-112 to -111|	62	|-79 to -78	   |95|	-46 to -45|
30|	-111 to -110|	63	|-78 to -77	   |96|	-45 to -44|
31|	-110 to -109|	64	|-77 to -76	   |97|	>-44|
32|	-109 to -108|	65	|-76 to -75|

- nv_rsrq: Reference Signal Receive Quality. This should be a metric for cellular network|

|RSRQ	  |From	 |To	  |Unit|
|-------|------|------|----|
|RSRQ_00|	-19.5|	    | dB |
|RSRQ_01|	-19.5| -19.0| dB |
|RSRQ_02|	-19.0| -18.5| dB |
|RSRQ_03|	-18.5| -18.0| dB |
|RSRQ_04|	-18.0| -17.5| dB |
|RSRQ_05|	-17.5| -17.0| dB |
|RSRQ_06|	-17.0| -16.5| dB |
|RSRQ_07|	-16.5| -16.0| dB |
|RSRQ_08|	-16.0| -15.5| dB |
|RSRQ_09|	-15.5| -15.0| dB |
|RSRQ_10|	-15.0| -14.5| dB |
|RSRQ_11|	-14.5| -14.0| dB |
|RSRQ_12|	-14.0| -13.5| dB |
|RSRQ_13|	-13.5| -13.0| dB |
|RSRQ_14|	-13.0| -12.5| dB |
|RSRQ_15|	-12.5| -12.0| dB |
|RSRQ_16|	-12.0| -11.5| dB |
|RSRQ_17|	-11.5| -11.0| dB |
|RSRQ_18|	-11.0| -10.5| dB |
|RSRQ_19|	-10.5| -10.0| dB |
|RSRQ_20|	-10.0| -9.5	| dB |
|RSRQ_21|	-9.5	|-9.0	| dB |
|RSRQ_22|	-9.0	|-8.5	| dB |
|RSRQ_23|	-8.5	|-8.0	| dB |
|RSRQ_24|	-8.0	|-7.5	| dB |
|RSRQ_25|	-7.5	|-7.0	| dB |
|RSRQ_26|	-7.0	|-6.5	| dB |
|RSRQ_27|	-6.5	|-6.0	| dB |
|RSRQ_28|	-6.0	|-5.5	| dB |
|RSRQ_29|	-5.5	|-5.0	| dB |
|RSRQ_30|	-5.0	|-4.5	| dB |
|RSRQ_31|	-4.5	|-4.0	| dB |
|RSRQ_32|	-4.0	|-3.5	| dB |
|RSRQ_33|	-3.5	|-3.0	| dB |
|RSRQ_34|	-3.0	|	    | dB |


With all the values gatthered here I am able to build now a request that will get me all of those in one shot. Here is an example request (sensitive values have been offuscated)
```
curl -s http://192.168.150.1/goform/goform_get_cmd_process\?multi_data\=1\&isTest\=false\&cmd\=flash_rest%2Cflash_total%2Cnv_loadavg%2Cram_rest%2Cram_total%2Cram_used%2Csms_unread_num%2Cnetwork_provider%2Csignalbar%2Cnetwork_type%2Csub_network_type%2Cppp_status%2Csimcard_roam%2Cmonthly_rx_bytes%2Cmonthly_tx_bytes%2Crealtime_tx_thrpt%2Crealtime_rx_thrpt%2Crealtime_time%2Cipv6_wan_apn%2Cwan_apn%2Cnv_arfcn%2Cnv_band%2Cnv_cellid%2Cnv_enodbid%2Cnv_globecellid%2Cnv_pci%2Cwan_ipaddr%2Cimei%2Cmsisdn%2Csim_imsi%2Cziccid%2Cmcc%2Cmnc%2Cnv_sinr%2Cnv_rsrp%2Cnv_rsrq | jq

{
  "flash_rest": "88.0M",
  "flash_total": "105.0M",
  "nv_loadavg": "0.47",
  "ram_rest": "22.3M",
  "ram_total": "54.7M",
  "ram_used": "59.2%",
  "sms_unread_num": "0",
  "network_provider": "XXXXX",
  "signalbar": "5",
  "network_type": "LTE",
  "sub_network_type": "FDD_LTE",
  "ppp_status": "ppp_connected",
  "simcard_roam": "Home",
  "monthly_rx_bytes": "263486974",
  "monthly_tx_bytes": "283894410",
  "realtime_tx_thrpt": "0",
  "realtime_rx_thrpt": "0",
  "realtime_time": "2510",
  "ipv6_wan_apn": "XXXX.XXXXX.XXX",
  "wan_apn": "XXXX.XXXXX.XXX",
  "nv_arfcn": "XXXX",
  "nv_band": "4",
  "nv_cellid": "2",
  "nv_enodbid": "XXXXXX",
  "nv_globecellid": "141391362",
  "nv_pci": "175",
  "wan_ipaddr": "100.87.82.217",
  "imei": "XXXXXXXXXXXXX",
  "msisdn": "+1111111111111",
  "sim_imsi": "123456789012345",
  "ziccid": "1234567890123456789",
  "mcc": "722",
  "mnc": "07",
  "nv_sinr": "7",
  "nv_rsrp": "48",
  "nv_rsrq": "23"
}
```


Now that we know how and which metrics to pull we can start building the prometheus exporter for this device.

# Building a prometheus exporter