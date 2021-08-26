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
NOTE: that password and username are first base64 encoded and then URL encoded so decodes URL to YWRtaW4= which decodes base64 to admin. Yes I tested this weith admin/amdin user password. Off course this needs to be changed as soon as possible.

### Finding what data to collect
After log in we should be able to get all needed info, there seems to be 1 endpoint for getting all the data we need to export metrics, and the good news is that the router responds in json format. The enpoint is `/goform/goform_get_cmd_process` and the it will return the data based on the paramenters passed to the http GET request.
```
curl -s -X GET http://192.168.150.1/goform/goform_get_cmd_process\?cmd\=realtime_tx_bytes%2Crealtime_rx_bytes%2Csimcard_roam%2Crealtime_tx_thrpt%2Crealtime_rx_thrpt%2Crealtime_time%2Cmonthly_tx_bytes%2Cmonthly_rx_bytes%2Cflux_month_total\&multi_data\=1\&_\=1629929177313 | jq
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



