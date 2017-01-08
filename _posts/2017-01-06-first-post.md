---
layout: post
title: pfSense & Splunk
description: "logs are fun"
tags: [security, splunk, pfsense, logs]
---

I have a virtual instance of [pfSense](https://pfsense.org/) running on my ESXi box that serves as a firewall/gateway for my VMs.  One of these VMs happens to be running [Splunk](https://www.splunk.com/).  I'm going to fumble my way through getting some of the great logs out of pfSense and into Splunk to see just what is actually going on within my network.
 

Please note I'm not an expert in either pfSense or Splunk and would gladly welcome any feedback!  Following this guide is no guarantee of setting things up correctly.  Please reach out if you find errors or know of better ways of accomplishing things.  


## System Info

I'm currently running the following sytems.  If there is interest I can do a quick write-up on deploying them.

* pfSense - 2.3.2-RELEASE-p1
* Splunk 6.5.1, installed on Ubuntu 16.04.1 LTS


# Splunk

### Add Data

Fresh off of a new Splunk install, log into Splunk `http://[Splunk]:8000` and navigate to __Settings__ and click the big __Add Data__ button.
  
![Splunk Add Data]({{ site.url }}/images/pfSplunk/splunkAddData.png)

We're going to set ultimately set up a _UDP Forwarded input_.  Start by clicking the big eye icon titled __monitor__ and then select `TCP / UDP` as your source.  I want to set this listener up as a generic syslog listener for my whole network so I'm going to configure the following settings:


### Select Source

* `TCP / UDP` - Select __UDP__
* `Port` - Enter __514__
* `Source name override` - _blank_
* `Only accept connection from` - _blank_ so all of my hosts can forward their logs

Click the __Next >__ button at the top of the screen.


### Input Settings

* `Source type` - Select `Select` and then find `syslog` in the drop down
* `APp context` - Keep the default `Search & Reporting`
* `Host` - I'm using `DNS` as my home network has decent dynamic records setup for each hostname
* `Index` - Leaving this as `Default`

Click the __Review >__ button at the top of the screen.


### Review

Splunk will show you all the settings you just set.  Here's a redundant list of my settings:

* `Input Type` - UDP Port
* `Port Number` - 514
* `Source name override` - N/A
* `Restrict to Host` - N/A
* `Source Type` - syslog
* `App` Context - search
* `Host` - (DNS entry of the remote server)
* `Index` - default

Click the __Submit >__ button at the top of the screen.  Splunk is now listening on that port for logs.  You can verify this by SSH'ing to your Splunk host running netstat.  The output below is truncated to only show port 514 that I opened just now.

```bash
$ sudo netstat -lnp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
udp        0      0 0.0.0.0:514             0.0.0.0:*                           15629/splunkd
```
_Note: sudo allows you to see the PID name, running without would just show the port_


# pfSense Settings

I'm a fan of logging __everything__ so we start this adventure by logging into pfSense and going to `Status / System Logs / Settings`.
 
 

### General Logging Options
 
My recommendation is to check all of the __Log firewall default blocks__ options, but you may want to consider impacts to performance and disk space (more checks = more logs).  

![pfSense Logs]({{ site.url }}/images/pfSplunk/01-logs.png)

 

### Remote Logging Options

This is where we tell pfSense to remotely log to our Splunk instance.  Checking `Enable Remote Logging` will expand the options screen to show all of the fun settings.  My recommendations/interpretations:

* `Source Address` - I prefer to __not__ select `Default (any)` and instead select the interface that is either directly connected to the Splunk VM's network OR select an internal/LAN interface.  In my case I'm selecting the `SERVERS` interface becuase that's where Splunk lives.  Assuming you don't go moving your Splunk server around all that often this won't make a difference, but I'd rather be specific for this types of settings.
* `IP Protocol` - Select IPv4 unless you're specifically doing IPv6
* `Remote log servers` - This is the IP and port for your Splunk server.
* `Remote Syslog Contents` - Did I mention that I like logging everything?  

![pfSense Remote Logging Options]({{ site.url }}/images/pfSplunk/pfRemoteLogOpts.png)

Click __Save__ to enabling logging!


# Verify Logging

Go back to your Splunk instance and select the `Search & Reporting` app on the left sidebar.  Under `What to Search` you should see events being indexed.  Clicking `Data Summary` and going through the three tabs will also show you a, _drumroll please_, summary of the data.  I just enabled logging a few minutes ago:

![splunk What to Search]({{ site.url }}/images/pfSplunk/splunkWhatToSearch.png)
<figure class="third">
	<a href="/images/pfSplunk/splunkDataSummary.png"><img src="/images/pfSplunk/splunkDataSummary.png" alt=""></a>
	<a href="/images/pfSplunk/splunkSources.png"><img src="/images/pfSplunk/splunkSources.png" alt=""></a>
	<a href="/images/pfSplunk/splunkSourceTypes.png"><img src="/images/pfSplunk/splunkSourceTypes.png" alt=""></a>
	<figcaption>Splunk Data Summary, click to englarge</figcaption>
</figure>

Viola, we have logs.  Next up, parsing!

