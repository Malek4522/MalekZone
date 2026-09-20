+++
title = "How to Simulate PROFINET Traffic Using p-net and CODESYS"
description = "A walkthrough for generating PROFINET traffic over the wire without any physical hardware. Everything is simulated, and the captured traffic can be used as a baseline for parsers, loggers, and other projects."
date = 2026-09-19
[taxonomies]
tags = ["OT security", "OT network", "profinet", "simulation", "pnet", "codesys", "softplc"]
+++

# Pre-requisites

To get started we need two computers — one running Linux and one running Windows — connected by an Ethernet cable. On the Linux machine (PC1), you need p-net compiled and ready. You can grab either the release binary or the source code from the RT-Labs GitHub and compile it yourself. On the Windows machine (PC2), you need CODESYS Control Win installed, which is available from the official CODESYS website.

Some people skip the Linux machine entirely and use a Raspberry Pi instead, which works fine. In my setup I temporarily disabled the firewall on both machines so it would not interfere with connections and ports.

## Starting

The first thing to sort out is connectivity. After linking the two machines with the Ethernet cable, I assigned the Linux machine (PC1) the address `192.168.1.100`:

```
(malek㉿malek)-[~/Desktop/profinet]
└─$ ip addr show eth0
2: eth0: <BROADCAST,MULTICAST,ALLMULTI,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:2b:67:b7:34:ec brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.100/24 brd 192.168.1.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::6188:e4f:68c2:942a/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
```

And the Windows machine (PC2) gets `192.168.1.200`:

```
PS C:\Users\AKOPOPS> Get-NetIPConfiguration -InterfaceAlias "Ethernet"

InterfaceAlias       : Ethernet
InterfaceIndex       : 8
InterfaceDescription : Realtek PCIe GbE Family Controller
NetProfile.Name      : Network 2
IPv4Address          : 192.168.1.200
IPv6DefaultGateway   :
IPv4DefaultGateway   : 192.168.1.100
DNSServer            : 1.1.1.1
```

Now let's test connectivity:

Linux side:
```
(malek㉿malek)-[~/Desktop/profinet]
└─$ ping 192.168.1.200
PING 192.168.1.200 (192.168.1.200) 56(84) bytes of data.
64 bytes from 192.168.1.200: icmp_seq=1 ttl=128 time=0.221 ms
64 bytes from 192.168.1.200: icmp_seq=2 ttl=128 time=0.218 ms
64 bytes from 192.168.1.200: icmp_seq=3 ttl=128 time=0.431 ms
^C
--- 192.168.1.200 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2051ms
rtt min/avg/max/mdev = 0.218/0.290/0.431/0.099 ms
```

Windows side:
```
PS C:\Users\AKOPOPS> ping 192.168.1.100

Pinging 192.168.1.100 with 32 bytes of data:
Reply from 192.168.1.100: bytes=32 time<1ms TTL=64
Reply from 192.168.1.100: bytes=32 time<1ms TTL=64
Reply from 192.168.1.100: bytes=32 time<1ms TTL=64
Reply from 192.168.1.100: bytes=32 time<1ms TTL=64

Ping statistics for 192.168.1.100:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round time milli-seconds:
    Minimum = 0ms, Maximum = 0ms, Average = 0ms
```

Great. Next, hop on the Windows machine and make sure that both the Gateway and the SoftPLC are started. If they are stopped, their tray icons appear grey.

<img src="../../profinet/codesys-icon.png" alt="CODESYS tray icons in stopped state" style="max-width: 100%; height: auto; border-radius: 6px;" />

You can start them by right-clicking on each icon and selecting **Start**.

<img src="../../profinet/codesys-icon-start.png" alt="Starting CODESYS services from the tray" style="max-width: 100%; height: auto; border-radius: 6px;" />

Next, open **CODESYS V3.5 SP22 Patch 3** (our IDE) and create a new project. Keep the default settings — Standard Project — and click OK.

<img src="../../profinet/codesys-new-project.png" alt="Creating a new CODESYS project" style="max-width: 100%; height: auto; border-radius: 6px;" />

Keep the default project settings as well:

<img src="../../profinet/codesys-project-settings.png" alt="Default CODESYS project settings" style="max-width: 100%; height: auto; border-radius: 6px;" />

Then go to **Tools → Device Repository**:

<img src="../../profinet/device-repo.png" alt="Device Repository menu" style="max-width: 100%; height: auto; border-radius: 6px;" />

Click **Install** and point it to your GSDML file. Below is the version I used — as long as it matches the p-net version you downloaded, it will work:

[Download GSDML file](/profinet/GSDML-V2.43-RT-Labs-P-Net-Sample-App-20240530.xml)

The p-net device should now appear in the list. In my screenshot three variants show up because I tested multiple GSDML versions — you only need one. Click **Renew Repository** and confirm with **Yes**.

<img src="../../profinet/renew-repo.png" alt="Renewing the device repository" style="max-width: 100%; height: auto; border-radius: 6px;" />

Next, go to the project tree on the left, double-click on **Device**, click **Scan Network**, select your desktop, and click OK.

<img src="../../profinet/device-scan.png" alt="Scanning the network for devices" style="max-width: 100%; height: auto; border-radius: 6px;" />

If things are going well you will see green dots next to the devices:

<img src="../../profinet/device-scan-good.png" alt="Green dots confirming device connectivity" style="max-width: 100%; height: auto; border-radius: 6px;" />

Now right-click on **Device** in the project tree, select **Add**, then navigate to **PROFINET → Ethernet Adapter → Ethernet**:

<img src="../../profinet/add-ethernet.png" alt="Adding an Ethernet adapter" style="max-width: 100%; height: auto; border-radius: 6px;" />

Do the same thing but this time right-click on the newly added Ethernet adapter and add a **PN Controller**:

<img src="../../profinet/add-pn-controller.png" alt="Adding a PN Controller" style="max-width: 100%; height: auto; border-radius: 6px;" />

Repeat once more: right-click on the PN Controller and add **p-net**:

<img src="../../profinet/pnet-add.png" alt="Adding p-net device" style="max-width: 100%; height: auto; border-radius: 6px;" />

If everything went well, the project tree should now look like this:

<img src="../../profinet/project-tree.png" alt="Final project tree structure" style="max-width: 100%; height: auto; border-radius: 6px;" />

Double-click on **Ethernet** and click **Browse Network Interface**:

<img src="../../profinet/net-interface.png" alt="Browsing network interfaces" style="max-width: 100%; height: auto; border-radius: 6px;" />

A login or register screen may appear — just fill in your credentials:

<img src="../../profinet/login.png" alt="Login screen" style="max-width: 100%; height: auto; border-radius: 6px;" />

Then select the network adapter that matches the IP address we assigned to the Windows machine (`192.168.1.200`):

<img src="../../profinet/network-adapter-choose.png" alt="Selecting the correct network adapter" style="max-width: 100%; height: auto; border-radius: 6px;" />

Next, double-click on **PN Controller** and set the IP configuration to match our subnet. In our case that is `192.168.1.X` with a mask of `255.255.255.0`:

<img src="../../profinet/pn-controller-ip.png" alt="PN Controller IP settings" style="max-width: 100%; height: auto; border-radius: 6px;" />

<img src="../../profinet/pn-controller-ip-valide.png" alt="PN Controller IP validated" style="max-width: 100%; height: auto; border-radius: 6px;" />

Then select the p-net IP address that matches the Linux machine:

<img src="../../profinet/pnet-ip.png" alt="p-net IP settings" style="max-width: 100%; height: auto; border-radius: 6px;" />

In our case that is `192.168.1.100`:

<img src="../../profinet/pnet-ip-valide.png" alt="p-net IP validated" style="max-width: 100%; height: auto; border-radius: 6px;" />

Now click the **Login** button in the top bar and confirm with **Yes**. A login screen may appear again.

<img src="../../profinet/login-icon.png" alt="Login icon in the top bar" style="max-width: 100%; height: auto; border-radius: 6px;" />

You will land on the **PLC STOP** screen:

<img src="../../profinet/stop-screen.png" alt="PLC STOP screen" style="max-width: 100%; height: auto; border-radius: 6px;" />

Click the **Run** icon to the right of the Login button, and confirm the green run screen:

<img src="../../profinet/run-screen.png" alt="PLC Run screen" style="max-width: 100%; height: auto; border-radius: 6px;" />

In Wireshark you can already see the SoftPLC actively running and looking for a station named `rt-labs-dev` using the **PN-DCP** protocol:

<img src="../../profinet/wireshark-pndcp.png" alt="Wireshark showing PN-DCP discovery traffic" style="max-width: 100%; height: auto; border-radius: 6px;" />

## Running p-net on Linux

Now hop back to PC1 (Linux). A quick tip: connect via SSH to the Linux machine so you are not constantly switching between computers — it saves a lot of time.

I wrote a small script to run p-net cleanly each time:

```bash
┌──(malek㉿malek)-[~/Desktop/profinet]
└─$ cat run.sh
#!/bin/bash

sudo ./pn_dev -r
sudo ./pn_dev -f
rm -f pnet*
sudo ./pn_dev -i eth0 -s rt-labs-dev -v -v -v -v
```

The first two lines (`-r` and `-f`) remove the old configuration and saved state from a previous run, since p-net persists data between runs. The third line cleans up any generated files. Together they guarantee a reproducible fresh start every time.

The last line runs p-net on the selected network interface, gives it the station name `rt-labs-dev` to match what we configured in the IDE, and adds four `-v` flags for verbose debug output (you can skip those if you want less noise).

Let's run it:

```
┌──(malek㉿malek)-[~/Desktop/profinet]
└─$ ./run.sh
[sudo] password for malek:
** Starting P-Net sample application 1.0.2+v1.0.2 **

Removing stored files
Exit application

** Starting P-Net sample application 1.0.2+v1.0.2 **
Network script for eth0:  Set IP 192.168.1.100   Netmask 255.255.255.0   Gateway 192.168.1.1 ...
Error: ipv4: Address already assigned.
Failed to set IP address and netmask

Performing factory reset
Exit application
...

** Starting P-Net sample application 1.0.2+v1.0.2 **
Number of slots:      5 (incl slot for DAP module)
P-net log level:      4 (DEBUG=0, FATAL=4)
App log level:        0 (DEBUG=0, FATAL=4)
Max number of ports:  1
Network interfaces:   eth0
Default station name: rt-labs-dev
Management port:      eth0 00:2B:67:B7:34:EC
Physical port [1]:    eth0 00:2B:67:B7:34:EC
Hostname:             malek
IP address:           0.0.0.0
...

Init P-Net stack and sample application
Start sample application main loop

Plug DAP module and its submodules
...
Done plugging DAP

Waiting for PLC connect request

PLC connect indication. AREP: 1
Event indication PNET_EVENT_STARTUP   AREP: 1
...
Event indication PNET_EVENT_DATA   AREP: 1
Cyclic data transmission started
```

The "Address already assigned" error on the first run is normal — p-net resets itself and starts cleanly on the second attempt. What matters is the end: it connected to the SoftPLC and started exchanging cyclic traffic.

## Verifying in Wireshark

Open Wireshark and check what is happening on the wire:

<img src="../../profinet/wireshark-pniops.png" alt="Wireshark showing PNIO_PS cyclic traffic" style="max-width: 100%; height: auto; border-radius: 6px;" />

The dominant sub-protocol is **PNIO_PS**, which is the cyclic IO data stream. It generates frames at a very high rate. One tip: to get Wireshark to decode the PNIO cyclic data payload, right-click on a frame, go to **Protocol Preferences**, and point it to the folder containing the GSDML file.

Scrolling to the top of the capture, you can also see the **CM** (Connection Management) exchange that happened when the session was established:

<img src="../../profinet/wireshark-cm.png" alt="Wireshark showing CM protocol exchange" style="max-width: 100%; height: auto; border-radius: 6px;" />

## Bonus: Blinking the LED

One more thing worth trying. Go back to the IDE, right-click on **PN Controller** in the project tree, and select **Device Scan → Scan**:

<img src="../../profinet/pn-controller-scan.png" alt="PN Controller device scan" style="max-width: 100%; height: auto; border-radius: 6px;" />

Then click the **Blink LED** button:

<img src="../../profinet/blink-led.png" alt="Blink LED button" style="max-width: 100%; height: auto; border-radius: 6px;" />

Switch back to the p-net terminal and you can see it responding in real time:

```
Cyclic data transmission started

Profinet signal LED indication. New state: 1
Profinet signal LED indication. New state: 0
Profinet signal LED indication. New state: 1
Profinet signal LED indication. New state: 0
Profinet signal LED indication. New state: 1
Profinet signal LED indication. New state: 0
```

That confirms two-way communication is working — the PLC sent a command and p-net acted on it.

## Downloads

I am sharing the captured network traffic from this session in case you want to inspect it or use it as a baseline for your own projects:

[Download profinet.pcapng](/profinet/profinet.pcapng)

You can also import the full CODESYS project to replay the lab:

[Download profinet2.project](/profinet/profinet2.project)

## What Did Not Work

Before landing on this setup, I tried several other combinations. I am including these here so you do not waste time going down the same dead ends.

**p-net with profinet-py** — failed with a lot of errors. Since both can run on the same Linux machine I tried that, but I could not even get them to establish a connection. The results were bad enough that I abandoned it early.

**Factory IO with PLCSIM Advanced** — failed because PLCSIM Advanced cannot generate real PROFINET traffic over the wire. It did connect to Factory IO, but only using S7 traffic, which is a different protocol.

**CODESYS WinControl with p-net through a home router** — failed because most home routers cannot switch PROFINET traffic, especially the real-time cyclic frames. A direct Ethernet link is necessary.

**CODESYS WinControl with p-net over a VM** (Windows in VM, Linux as host) — failed for reasons I could not fully pin down. The real-time traffic could not cross from the VM to the host even when connectivity was fine and firewalls were disabled. This was actually the closest setup to the working one, so it might be worth investigating further if you need a single-machine solution, but I could not get it to behave.
