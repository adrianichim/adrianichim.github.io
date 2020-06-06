---
layout: post
date: 2020-02-21
title: Using Multipass and VirtualBox on Windows
subtitle: How to use all features of Multipass when running it on Windows with VirtualBox
tags: [Windows, Multipass, VirtualBox] 
---

## Introduction

[__Multipass__][multipasslink] is a tool created by Canonical to easily manage VM's in a development environment. 
It looks like a lightweight version of Hashicorp's [__Vagrant__][vagrantlink]. It has significant restrictions compared 
to Vagrant (uses only Ubuntu images, has a limited set of features) but it has a significant 
feature that convinced me to use it: it allows me to configure a VM using its [__cloud-init__][cloudinitlink] built-in 
provider. This is very useful if I want to test images for the cloud, because is significantluy faster - I'm 
developing and testing locally and don't wait for a cloud VM deployment. Also, if I use cloud-init I  
can avoid creating too many 
specialized VM images to deploy for a given application.

On my Windows development machine I am using [__VirtualBox__][virtualboxlink] as a virtualization engine, 
instead of the default [__Hyper-V__][hypervlink] available on Windows.  VirtualBox is 
multiplatform and has a richer set of features compared to Hyper-V. Using it on top of Hyper-V is possible, 
but it runs very slowy. When used with Multipass, VirtualBox has a major drawback: it does not offer a 
readily available way to access
the guest VM's from the host machine.  

## Objectives

I am describing below the steps required to access the VirtualBox 
guest machines from the Windows host:

There are three main methods to do this: 

1. [Bridged network](#bridged-new): I'm creating an additional interface on the guest VM and bridge it to 
an interface available on the host. The guest will be accesible from the host and from the external network.  
2. [Host-only network](#hostonly): I'm creating an additional interface  on the guest VM and connect it to 
 the internal "VirtualBox Host-Only Ethernet Adapter" switch. The guest will be accesible only from the host. 
3. [Port forwarding](#portforwarding): I will create a port forwarding rule between the application port on the guest and an available port 
on the host machine, using the host-only interface of the VirtualBox environment.  
I will use the command-line tool __vboxmanage__ to configure the guest VM's (see below).

I will create four VMs (__vm1__, __vm2__, __vm3__ and __vm4__), using different versions of  Ubuntu LTS:  
- vm1, vm2 and vm3 will use the newer versions of Ubuntu (__Bionic__ and __Focal__) 
and their internal network settings will be configured using the __netplan__ and __ip__ Linux tools. 
- vm1 will use method [(1)](#bridged-new) and vm2 will use method [(2)](#hostonly) above
- vm3 will use method [(3)](#portforwarding). To demonstrate the usage of the 
port forwarding method I will install the __Webmin__ software on vm3 and I will access it from a 
browser running on the host machine. 
- [vm4 will apply method (1) the older Ubuntu __Xenial__](#bridged-old) and I will use the __ifupdown__ tools to configure 
the internal network.

The multipass software runs in Windows under the SYSTEM account and I need the [__PsExec64.exe__][psexeclink] tool to interact with it, 
from an admin-mode PowerShell console.
The VirtualBox VM's will be configured using the [__vboxmanage__][vboxmanagelink] command-line utility, 
as installed by the VirtualBox package.

## Detailed prerequiste steps

__P00\.__ Read this document. Consult the [__references__](#references).  
__P01\.__ Download and install [__VirtualBox__][virtualboxlink].  
__P02\.__ Download and install [__Multipass__][multipasslink].  
__P03\.__ Download and install [__PsExec64.exe__][psexeclink] from Sysinternals tools package (Microsoft). I will assume 
PsExec64.exe is located in the `C:\AdmTools` folder and the folder is __added to the system path__ (update the system PATH environment variable)  
__P04\.__ Check where the __vboxmanage__ utility is located (in the Oracle VirtualBox installation folder). It is normally available as  
`C:\Program Files\Oracle\VirtualBox\vboxmanage.exe`  
__P05\.__ Locate and review the [vboxmanage documentation][vboxmanagelink]  
__P06\.__ Launch user-mode PowerShell (first console session - I will work with two PowerShell 
console sessions open concurrently). The user-mode console will be also used bu Multipass as a Linux console
interface :)  
__P07\.__ Launch admin-mode PowerShell (second console session)  
__P08\.__ Set VirtualBox as the hypervisor for Multipass (admin-mode PowerShell):  

```
PS C:\Windows\system32> multipass set local.driver=virtualbox
```

__P09\.__ List the machine images available on the Canonical repository (user-mode PowerShell):  

```
PS C:\Users\Adrian> multipass find
   Image                   Aliases           Version          Description
   16.04                   xenial            20200430         Ubuntu 16.04 LTS
   18.04                   bionic,lts        20200506         Ubuntu 18.04 LTS
   20.04                   focal             20200504         Ubuntu 20.04 LTS
PS C:\Users\Adrian>
```

__P10\.__ Create & launch 4 different VM's:

```
PS C:\Users\Adrian> multipass launch -n vm1 bionic
Launched: vm1
PS C:\Users\Adrian> multipass launch -n vm2 focal
Launched: vm2
PS C:\Users\Adrian> multipass launch -n vm3 -m 5G -c 2 bionic
Launched: vm3
PS C:\Users\Adrian> multipass launch -n vm4 xenial
Launched: vm4
PS C:\Users\Adrian>
```

__P11\.__ Stop the VM's:

```
PS C:\Users\Adrian> multipass stop --all
```

>
>
>
## Below I'm describing below the three access methods in detail.
>
>
---------------------------------------------

## <a name="bridged-new"></a> A. Configure access using bridged networking


__A01\.__ Start vm1:

```
PS C:\Users\Adrian> multipass shell vm1
```

__A02\.__ View configuration summary:

```
System load:  0.45              Processes:             86
Usage of /:   21.0% of 4.67GB   Users logged in:       0
Memory usage: 15%               IP address for enp0s3: 10.0.2.15
Swap usage:   0%
```

___Note___: only the __enp0s3__ interface is active, IP: 10.0.2.15

__A03\.__ Stop vm1:

```
PS C:\Users\Adrian> multipass stop vm1
```

__A04\.__ Get Windows network interfaces info, using the user-mode PowerShell:

```
PS C:\Users\Adrian> Get-NetAdapter -Physical | format-list -property "Name","DriverDescription"


Name              : Ethernet 3
DriverDescription : Intel(R) Ethernet Connection I219-LM

Name              : WiFi 2
DriverDescription : Intel(R) Dual Band Wireless-AC 8260
```

_Note_: the value of the"DriverDescription" property for the Windows interface I intend to use is

```
DriverDescription : "Intel(R) Dual Band Wireless-AC 8260"
```

__A05\.__ Get vm1 info from VirtualBox, using the admin-mode PowerShell:

```
PS C:\Windows\system32> PsExec64.exe -s "c:\program files\oracle\virtualbox\vboxmanage" showvminfo "vm1"
[showing here only the NIC related info]
NIC 1:                       MAC: 08002740E542, Attachment: NAT, Cable connected: on, Trace: off (file: none), Type:    82540EM, Reported speed: 0 Mbps, Boot priority: 0, Promisc Policy: deny, Bandwidth group: none
NIC 1 Settings:  MTU: 0, Socket (send: 64, receive: 64), TCP Window (send:64, receive: 64)
NIC 1 Rule(0):   name = ssh, protocol = tcp, host ip = , host port = 49975, guest ip = , guest port = 22
NIC 2:                       disabled
NIC 3:                       disabled
NIC 4:                       disabled
NIC 5:                       disabled
NIC 6:                       disabled
NIC 7:                       disabled
NIC 8:                       disabled
```

_Note_: vm1 has configured only NIC1, used to communicate through SSH with the host computer (Multipass session)

__A06\.__ Add a new NIC adapter (NIC2) to vm1:

```
PS C:\Windows\system32> PsExec64.exe -s "c:\program files\oracle\virtualbox\vboxmanage" modifyvm "vm1" --nic2 bridged --bridgeadapter2 "Intel(R) Dual Band Wireless-AC 8260" 
```

__A07\.__ Get vm1 info from VirtualBox, using the admin-mode PowerShell:

```
PS C:\Windows\system32> PsExec64.exe -s "c:\program files\oracle\virtualbox\vboxmanage" showvminfo "vm1"
[showing here only the NIC related info]
NIC 1:                       MAC: 08002740E542, Attachment: NAT, Cable connected: on, Trace: off (file: none), Type:    82540EM, Reported speed: 0 Mbps, Boot priority: 0, Promisc Policy: deny, Bandwidth group: none
NIC 1 Settings:  MTU: 0, Socket (send: 64, receive: 64), TCP Window (send:64, receive: 64)
NIC 1 Rule(0):   name = ssh, protocol = tcp, host ip = , host port = 49975, guest ip = , guest port = 22
NIC 2:                       MAC: 0800270F3144, Attachment: Bridged Interface 'Intel(R) Dual Band Wireless-AC 8260',    Cable connected: on, Trace: off (file: none), Type: 82540EM, Reported speed: 0 Mbps, Boot priority: 0, Promisc Policy:    deny, Bandwidth group: none
NIC 3:                       disabled
NIC 4:                       disabled
NIC 5:                       disabled
NIC 6:                       disabled
NIC 7:                       disabled
NIC 8:                       disabled
```

_Note_: new interface NIC2, bridged to the 'Intel(R) Dual Band Wireless-AC 8260'

__A08\.__ Start vm1:

```
PS C:\Users\Adrian> multipass shell vm1
[view the hardware info for vm1:]

System load:  0.89              Processes:             86
Usage of /:   21.0% of 4.67GB   Users logged in:       0
Memory usage: 15%               IP address for enp0s3: 10.0.2.15
Swap usage:   0%
```

__A09\.__ Get network info for vm1:

```
ubuntu@vm1:~$ ip address show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
        valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
        valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:40:e5:42 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s3
        valid_lft 86298sec preferred_lft 86298sec
    inet6 fe80::a00:27ff:fe40:e542/64 scope link
        valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 08:00:27:0f:31:44 brd ff:ff:ff:ff:ff:ff
```

_Note_: vm1 has an new unconfigured network interface __enp0s8__

__A10\.__ Check the content of __/etc/netplan__:

```
ubuntu@vm1:~$ sudo ls /etc/netplan
50-cloud-init.yaml
```

__A11\.__ Modify the **netplan** configuration:

```
ubuntu@vm1:~$ sudo nano /etc/netplan/60-extra-interfaces.yaml
```

__A12\.__ Add the lines below to the file and save:

```
network:
    ethernets:
    enp0s8:
        optional: yes
        dhcp4: yes
        dhcp4-overrides:
        route-metric: 10
        dhcp6: yes
        dhcp6-overrides:
        route-metric: 10
```

__A13\.__ Ask netplan to apply the new configuration:

```
ubuntu@vm1:~$ sudo netplan apply
```

__A14\.__ Check if the interface __enp0s8__ received a DHCP address:

```
ubuntu@vm1:~$ ip address show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
        valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
        valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:40:e5:42 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s3
        valid_lft 86180sec preferred_lft 86180sec
    inet6 fe80::a00:27ff:fe40:e542/64 scope link
        valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:0f:31:44 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.24/24 brd 192.168.0.255 scope global dynamic enp0s8
        valid_lft 86182sec preferred_lft 86182sec
    inet6 fe80::a00:27ff:fe0f:3144/64 scope link
        valid_lft forever preferred_lft forever
```

_Note_: __enp0s8__ is up and has an address allocated by DHCP 

__A15\.__ Check the routing table:

```
ubuntu@vm1:~$ ip route show
default via 192.168.0.1 dev enp0s8 proto dhcp src 192.168.0.24 metric 10
default via 10.0.2.2 dev enp0s3 proto dhcp src 10.0.2.15 metric 100
10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15
10.0.2.2 dev enp0s3 proto dhcp scope link src 10.0.2.15 metric 100
192.168.0.0/24 dev enp0s8 proto kernel scope link src 192.168.0.24
192.168.0.1 dev enp0s8 proto dhcp scope link src 192.168.0.24 metric 10
```

__A16\.__ Ping the interface from the user-mode PowerShell:

```
PS C:\Users\Adrian> ping 192.168.0.24

Pinging 192.168.0.24 with 32 bytes of data:
Reply from 192.168.0.24: bytes=32 time<1ms TTL=64
Reply from 192.168.0.24: bytes=32 time=1ms TTL=64
Reply from 192.168.0.24: bytes=32 time=1ms TTL=64
Reply from 192.168.0.24: bytes=32 time=1ms TTL=64

Ping statistics for 192.168.0.24:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 1ms, Average = 0ms
```

__A17\.__ Stop vm1:

```
PS C:\Users\Adrian> multipass stop vm1
```

__Important:__ Check the settings of the firewall installed on the host machine, as well as the firewall on the guest 
before using the guest for other purposes.

-----------------------------------------------

## <a name="hostonly"></a>B. Configure access using host-only networking


__B01\.__ Start vm2:

```
PS C:\Users\Adrian> multipass shell vm2
```

__B02\.__ View vm2 guest configuration summary:

```
System load:  0.35              Processes:               102
Usage of /:   25.7% of 4.67GB   Users logged in:         0
Memory usage: 16%               IPv4 address for enp0s3: 10.0.2.15
Swap usage:   0%
```

_Note_:  only __enp0s3__ interface is active

__B03\.__ Stop vm2:

```
PS C:\Users\Adrian> multipass stop vm2
```

__B04\.__ Get Windows network interfaces info, using the user-mode PowerShell:

```
PS C:\Users\Adrian> Get-NetAdapter -Physical | format-list -property "Name","DriverDescription"

Name              : Ethernet 3
DriverDescription : Intel(R) Ethernet Connection I219-LM

Name              : WiFi 2
DriverDescription : Intel(R) Dual Band Wireless-AC 8260
```

_Note_:  the value of the"DriverDescription" property for the Windows interface I will use:  
`DriverDescription : "Intel(R) Dual Band Wireless-AC 8260"`

__B05\.__ Get vm2 info from VirtualBox, using the admin-mode PowerShell:

```
PS C:\Windows\system32>  PsExec64.exe -s "c:\program files\oracle\virtualbox\vboxmanage" showvminfo "vm2"
[extracted only vm2 NIC related info]
   NIC 1:                       MAC: 080027A683AB, Attachment: NAT, Cable connected: on, Trace: off (file: none), Type:    82540EM, Reported speed: 0 Mbps, Boot priority: 0, Promisc Policy: deny, Bandwidth group: none
   NIC 1 Settings:  MTU: 0, Socket (send: 64, receive: 64), TCP Window (send:64, receive: 64)
   NIC 1 Rule(0):   name = ssh, protocol = tcp, host ip = , host port = 50137, guest ip = , guest port = 22
   NIC 2:                       disabled
   NIC 3:                       disabled
   NIC 4:                       disabled
   NIC 5:                       disabled
   NIC 6:                       disabled
   NIC 7:                       disabled
   NIC 8:                       disabled 
```

_Note_: vm2 has configured only NIC1, used to communicate through SSH with the host computer (Multipass session)

__B06\.__ Add a new NIC adapter (NIC2) to vm2:

```
PS C:\Windows\system32>  PsExec64.exe -s "c:\program files\oracle\virtualbox\vboxmanage" modifyvm "vm2" --nic2 hostonly    --hostonlyadapter2 "VirtualBox Host-Only Ethernet Adapter"
```

__B07\.__ Get vm2 info from VirtualBox, using the admin-mode PowerShell:

```
PS C:\Windows\system32> PsExec64.exe -s "c:\program files\oracle\virtualbox\vboxmanage" showvminfo "vm2"

[selected only NIC related info]
NIC 1:                       MAC: 080027A683AB, Attachment: NAT, Cable connected: on, Trace: off (file: none), 
Type: 82540EM, Reported speed: 0 Mbps, Boot priority: 0, Promisc Policy: deny, Bandwidth group: none
NIC 1 Settings:  MTU: 0, Socket (send: 64, receive: 64), TCP Window (send:64, receive: 64)
NIC 1 Rule(0):   name = ssh, protocol = tcp, host ip = , host port = 50137, guest ip = , guest port = 22
NIC 2:                       MAC: 0800278AB776, Attachment: Host-only Interface 'VirtualBox Host-Only Ethernet Adapter', 
Cable connected: on, Trace: off (file: none), Type: 82540EM, Reported speed: 0 Mbps, Boot priority: 0, 
Promisc Policy: deny, Bandwidth group: none
NIC 3:                       disabled
NIC 4:                       disabled
NIC 5:                       disabled
NIC 6:                       disabled
NIC 7:                       disabled
NIC 8:                       disabled
```

_Note_: new NIC2, connected to the host-only adapter

__B08\.__ Start vm2:

```
PS C:\Users\Adrian> multipass shell vm2
```

__B09\.__ Review the machine data:

```
System load:  0.97              Processes:               102
Usage of /:   25.8% of 4.67GB   Users logged in:         0
Memory usage: 17%               IPv4 address for enp0s3: 10.0.2.15
Swap usage:   0%
```

__B10\.__ Display the vm2 NIC configuration:

```   
ubuntu@vm2:~$ ip address show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
      inet 127.0.0.1/8 scope host lo
         valid_lft forever preferred_lft forever
      inet6 ::1/128 scope host
         valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
      link/ether 08:00:27:a6:83:ab brd ff:ff:ff:ff:ff:ff
      inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s3
         valid_lft 85417sec preferred_lft 85417sec
      inet6 fe80::a00:27ff:fea6:83ab/64 scope link
         valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
      link/ether 08:00:27:8a:b7:76 brd ff:ff:ff:ff:ff:ff
```

_Note_: adapter __enp0s8__ is not configured

__B11\.__ Check the content of __/etc/netplan__:

```
ubuntu@vm2:~$ sudo ls /etc/netplan
50-cloud-init.yaml
```

__B12\.__ Modify the __netplan__ configuration:


```
ubuntu@vm2:~$ sudo nano /etc/netplan/60-extra-interfaces.yaml
```

__B13\.__ Add the lines below to the file and save:

```
network:
   ethernets:
      enp0s8:
      optional: yes
      dhcp4: yes
      dhcp4-overrides:
         route-metric: 110
      dhcp6: yes
      dhcp6-overrides:
         route-metric: 110
```

__B14\.__ Ask netplan to apply the new configuration:

```
ubuntu@vm2:~$ sudo netplan apply
```

__B15\.__ Check if the interface __enp0s8__ received a DHCP address:

```
ubuntu@vm2:~$ ip address show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
      inet 127.0.0.1/8 scope host lo
         valid_lft forever preferred_lft forever
      inet6 ::1/128 scope host
         valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
      link/ether 08:00:27:a6:83:ab brd ff:ff:ff:ff:ff:ff
      inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s3
         valid_lft 86394sec preferred_lft 86394sec
      inet6 fe80::a00:27ff:fea6:83ab/64 scope link
         valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
      link/ether 08:00:27:8a:b7:76 brd ff:ff:ff:ff:ff:ff
      inet 192.168.56.101/24 brd 192.168.56.255 scope global dynamic enp0s8
         valid_lft 594sec preferred_lft 594sec
      inet6 fe80::a00:27ff:fe8a:b776/64 scope link
         valid_lft forever preferred_lft forever
```

_Note_: interface __enp0s8__ is up and has an address allocated by DHCP. The subnet is different from that used by the bridged networking.

__B16\.__ Check the routing table:

```
ubuntu@vm2:~$ ip route show
default via 10.0.2.2 dev enp0s3 proto dhcp src 10.0.2.15 metric 100
10.0.2.0/24 dev enp0s3 proto kernel scope link src 10.0.2.15
10.0.2.2 dev enp0s3 proto dhcp scope link src 10.0.2.15 metric 100
192.168.56.0/24 dev enp0s8 proto kernel scope link src 192.168.56.101
```

__B17\.__ Ping the interface from the user-mode PowerShell:

```
PS C:\Users\Adrian> ping 192.168.56.101

Pinging 192.168.56.101 with 32 bytes of data:
Reply from 192.168.56.101: bytes=32 time<1ms TTL=64
Reply from 192.168.56.101: bytes=32 time<1ms TTL=64
Reply from 192.168.56.101: bytes=32 time<1ms TTL=64
Reply from 192.168.56.101: bytes=32 time<1ms TTL=64

Ping statistics for 192.168.56.101:
      Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
      Minimum = 0ms, Maximum = 0ms, Average = 0ms
```

__B18\.__ Stop vm2:

```
PS C:\Users\Adrian> multipass stop vm2
```

__Important:__ Tune the settings of the firewall installed on the host machine, as well as the firewall on the guest 
before using the guest for other applications.

-----------------------------------------

## <a name="portforwarding"></a>C. Configure access using port forwarding

__C01\.__ Start vm3:

```
PS C:\Users\Adrian> multipass shell vm3
```

__C02\.__ View configuration:

```
System load:  0.61              Processes:             95
Usage of /:   21.0% of 4.67GB   Users logged in:       0
Memory usage: 2%                IP address for enp0s3: 10.0.2.15
Swap usage:   0%
```

_Note_: only the __enp0s3__ interface is active

__C03\.__ Stop vm3:

```
PS C:\Users\Adrian> multipass stop vm3
```

__C04\.__ Get Windows network interfaces info, using the user-mode PowerShell

```
PS C:\Users\Adrian> Get-NetAdapter -Physical | format-list -property "Name","DriverDescription"
```

Note the value of the"DriverDescription" property for the Windows interface I intend to use: \
`DriverDescription : "Intel(R) Dual Band Wireless-AC 8260"`  
   
__C05\.__ Get vm3 info from VirtualBox, using the admin-mode PowerShell:

```
PS C:\Windows\system32> PsExec64.exe -s "c:\program files\oracle\virtualbox\vboxmanage" showvminfo "vm3"

[reviewing only NIC1 related info - it has only one port forwarding rule called "ssh":]
NIC 1 Settings:  MTU: 0, Socket (send: 64, receive: 64), TCP Window (send:64, receive: 64)
NIC 1 Rule(0):   name = ssh, protocol = tcp, host ip = , host port = 54224, guest ip = , guest port = 22
```

__C06\.__ Add a second rule named "webmin", to forward the host port 10001 to the guest port 10000:

```
PS C:\Windows\system32> PsExec64.exe -s "c:\program files\oracle\virtualbox\vboxmanage" controlvm "vm3" natpf1    "webmin,tcp,127.0.0.1,10001,,10000"
```

__C07\.__ Check the forwarding rules of vm3:

```
PS C:\Windows\system32> PsExec64.exe -s "c:\program files\oracle\virtualbox\vboxmanage" showvminfo "vm3"
[review only NIC1 related info - see the new "webmin" rule(1):]
NIC 1: MAC: 080027583846, Attachment: NAT, Cable connected: on, Trace: off (file: none), Type: 82540EM, Reported speed: 0 Mbps, Boot     priority: 0, Promisc Policy: deny, Bandwidth group: none
NIC 1 Settings:  MTU: 0, Socket (send: 64, receive: 64), TCP Window (send:64, receive: 64)
NIC 1 Rule(0):   name = ssh, protocol = tcp, host ip = , host port = 54403, guest ip = , guest port = 22
NIC 1 Rule(1):   name = webmin, protocol = tcp, host ip = 127.0.0.1, host port = 10001, guest ip = , guest port = 10000
```

__C08\.__ Start [installing][webmininstalllink] [webmin][webmininstalllink] on vm3:

```
PS C:\Users\Adrian> multipass shell vm3
```

__C09\.__ Update packages, install the required dependencies:

```
ubuntu@vm3:~$ sudo apt-get update
ubuntu@vm3:~$ sudo apt-get install software-properties-common apt-transport-https wget
```

__C10\.__ Import the GPG key for webmin, using wget:

```
ubuntu@vm3:~$ wget -q http://www.webmin.com/jcameron-key.asc -O- | sudo apt-key add -
```

__C11\.__ Add the webmin repository and install webmin:

```
ubuntu@vm3:~$ sudo add-apt-repository "deb [arch=amd64] http://download.webmin.com/download/repository sarge contrib"
ubuntu@vm3:~$ sudo apt-get  install webmin
```

__C12\.__ Open port 10000 on vm3 using the __ufw__ firewall:

```
ubuntu@vm3:~$ sudo ufw allow 10000/tcp
```

__C13\.__ Add a new user with sudo rights - I will use it to access webmin from the host:

```
ubuntu@vm3:~$ sudo adduser adrian
ubuntu@vm3:~$ sudo usermod -aG sudo adrian
```

__C14\.__ Connect to vm3 webmin interface using a browser on the host machine to access <https://localhost:10001>, and the credentials of the user defined above. Ignore the warnings related to the unsafe site/invalid SSL certificate for localhost

___Note_:__ Tune the settings of the firewall installed on the host machine if you want to access the application 
running on the guest from a different machine than the host.
 
---------------------------------------------

## <a name="bridged-old"></a>D. Configure access using bridged networking - older version of Ubuntu

__D01\.__ Start vm4:

```
PS C:\Users\Adrian> multipass shell vm4
```

__D02\.__ View the network interfaces configuration summary:

```
ubuntu@vm4:~$ ifconfig -a
enp0s3    Link encap:Ethernet  HWaddr 08:00:27:58:b3:43
         inet addr:10.0.2.15  Bcast:10.0.2.255  Mask:255.255.255.0
         inet6 addr: fe80::a00:27ff:fe58:b343/64 Scope:Link
         UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
         RX packets:574 errors:0 dropped:0 overruns:0 frame:0
         TX packets:152 errors:0 dropped:0 overruns:0 carrier:0
         collisions:0 txqueuelen:1000
         RX bytes:630153 (630.1 KB)  TX bytes:16842 (16.8 KB)

lo        Link encap:Local Loopback
         inet addr:127.0.0.1  Mask:255.0.0.0
         inet6 addr: ::1/128 Scope:Host
         UP LOOPBACK RUNNING  MTU:65536  Metric:1
         RX packets:160 errors:0 dropped:0 overruns:0 frame:0
         TX packets:160 errors:0 dropped:0 overruns:0 carrier:0
         collisions:0 txqueuelen:1
         RX bytes:11840 (11.8 KB)  TX bytes:11840 (11.8 KB)
```

_Note_: only the __enp0s3__ interface is active, IP: 10.0.2.15

__D03\.__ Stop vm4:

```
PS C:\Users\Adrian> multipass stop vm4
```

__D04\.__ Get Windows network interfaces info, using the user-mode PowerShell:

```
Get-NetAdapter -Physical | format-list -property "Name","DriverDescription"
```

Note the value of the"DriverDescription" property for the Windows interface I will use: \
`DriverDescription : "Intel(R) Dual Band Wireless-AC 8260"`

__D05\.__ Get vm1 info from VirtualBox, using the admin-mode PowerShell:

```
PS C:\Windows\system32> PsExec64.exe -s "c:\program files\oracle\virtualbox\vboxmanage" showvminfo "vm4"
[review only NIC related info]
NIC 1:                       MAC: 08002758B343, Attachment: NAT, Cable connected: on, Trace: off (file: none), Type: 82540EM, Reported        speed: 0 Mbps, Boot priority: 0, Promisc Policy: deny, Bandwidth group: none
NIC 1 Settings:  MTU: 0, Socket (send: 64, receive: 64), TCP Window (send:64, receive: 64)
NIC 1 Rule(0):   name = ssh, protocol = tcp, host ip = , host port = 52935, guest ip = , guest port = 22
NIC 2:                       disabled
NIC 3:                       disabled
NIC 4:                       disabled
NIC 5:                       disabled
NIC 6:                       disabled
NIC 7:                       disabled
NIC 8:                       disabled
```

_Note_: vm4 uses only NIC1 to communicate through SSH with the host computer (Multipass session)

__D06\.__ Add a new NIC adapter (NIC2) to vm4:

```
PS C:\Users\Adrian> PsExec64.exe -s "c:\program files\oracle\virtualbox\vboxmanage" modifyvm "vm4" --nic2 bridged --bridgeadapter2 "Intel(R) Dual Band Wireless-AC 8260" 
```

__D07\.__ Get vm4 info from VirtualBox, using the admin-mode PowerShell:

```
PsExec64.exe -s "c:\program files\oracle\virtualbox\vboxmanage" showvminfo "vm4"
[review only NIC related info:]
NIC 1:                       MAC: 08002758B343, Attachment: NAT, Cable connected: on, Trace: off (file: none), Type: 82540EM, Reported    speed: 0 Mbps, Boot priority: 0, Promisc Policy: deny, Bandwidth group: none
NIC 1 Settings:  MTU: 0, Socket (send: 64, receive: 64), TCP Window (send:64, receive: 64)
NIC 1 Rule(0):   name = ssh, protocol = tcp, host ip = , host port = 52935, guest ip = , guest port = 22
NIC 2:                       MAC: 08002787F993, Attachment: Bridged Interface 'Intel(R) Dual Band Wireless-AC 8260', Cable connected: on,    Trace: off (file: none), Type: 82540EM, Reported speed: 0 Mbps, Boot priority: 0, Promisc Policy: deny, Bandwidth group: none
NIC 3:                       disabled
NIC 4:                       disabled
NIC 5:                       disabled
NIC 6:                       disabled
NIC 7:                       disabled
NIC 8:                       disabled
```

_Note_: new interface NIC2, bridged to the "Intel(R) Dual Band Wireless-AC 8260"

__D08\.__ Start vm4:

```
PS C:\Users\Adrian> multipass shell vm4
```

__D09\.__ View configuration summary:

```
ubuntu@vm4:~$ ifconfig -a
enp0s3    Link encap:Ethernet  HWaddr 08:00:27:58:b3:43
            inet addr:10.0.2.15  Bcast:10.0.2.255  Mask:255.255.255.0
            inet6 addr: fe80::a00:27ff:fe58:b343/64 Scope:Link
            UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
            RX packets:122 errors:0 dropped:0 overruns:0 frame:0
            TX packets:85 errors:0 dropped:0 overruns:0 carrier:0
            collisions:0 txqueuelen:1000
            RX bytes:19261 (19.2 KB)  TX bytes:12007 (12.0 KB)

enp0s8    Link encap:Ethernet  HWaddr 08:00:27:87:f9:93
            BROADCAST MULTICAST  MTU:1500  Metric:1
            RX packets:0 errors:0 dropped:0 overruns:0 frame:0
            TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
            collisions:0 txqueuelen:1000
            RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

lo        Link encap:Local Loopback
            inet addr:127.0.0.1  Mask:255.0.0.0
            inet6 addr: ::1/128 Scope:Host
            UP LOOPBACK RUNNING  MTU:65536  Metric:1
            RX packets:160 errors:0 dropped:0 overruns:0 frame:0
            TX packets:160 errors:0 dropped:0 overruns:0 carrier:0
            collisions:0 txqueuelen:1
            RX bytes:11840 (11.8 KB)  TX bytes:11840 (11.8 KB)
```

_Note_: vm4 has an new unconfigured network interface __enp0s8__

__D09\.__ Create a configuration file for the interface __enp0s8__ in the folder __/etc/network/interfaces.d__:

```
ubuntu@vm4:~$ sudo nano /etc/network/interfaces.d/80-enp0s8.cfg
```

__D10\.__ Insert the following data in the file and save:

```
allow-hotplug enp0s8
iface enp0s8 inet dhcp
  metric 10
```

__D11\.__ Activate the interface:

```
ubuntu@vm4:~$ sudo ifup enp0s8
Internet Systems Consortium DHCP Client 4.3.3
Copyright 2004-2015 Internet Systems Consortium.
All rights reserved.
For info, please visit https://www.isc.org/software/dhcp/

Listening on LPF/enp0s8/08:00:27:87:f9:93
Sending on   LPF/enp0s8/08:00:27:87:f9:93
Sending on   Socket/fallback
DHCPREQUEST of 192.168.0.25 on enp0s8 to 255.255.255.255 port 67 (xid=0x73062672)
DHCPACK of 192.168.0.25 from 192.168.0.1
bound to 192.168.0.25 -- renewal in 36389 seconds.
```

_Note_: __enp0s8__ is up and has an address allocated by DHCP

__D12\.__ Check the routing table:

```
ubuntu@vm4:~$ route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         10.0.2.2        0.0.0.0         UG    0      0        0 enp0s3
default         192.168.0.1     0.0.0.0         UG    10     0        0 enp0s8
10.0.2.0        *               255.255.255.0   U     0      0        0 enp0s3
192.168.0.0     *               255.255.255.0   U     0      0        0 enp0s8
```

__D13\.__ Ping the IP of the interface from the user-mode PowerShell:

```
PS C:\Users\Adrian> ping 192.168.0.25

Pinging 192.168.0.25 with 32 bytes of data:
Reply from 192.168.0.25: bytes=32 time<1ms TTL=64
Reply from 192.168.0.25: bytes=32 time<1ms TTL=64
Reply from 192.168.0.25: bytes=32 time<1ms TTL=64
Reply from 192.168.0.25: bytes=32 time<1ms TTL=64

Ping statistics for 192.168.0.25:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 0ms, Average = 0ms
```

__D14\.__ Stop vm4:

```
PS C:\Users\Adrian> multipass stop vm4
```

___Note_:__ Check the settings of the firewall installed on the host machine, as well as the firewall on the guest 
before using the guest for other tests.

-------------------------

## <a name="references"></a>E. References

__E01\.__ [An article by Jon Spriggs][spriggslink]. Extremely useful.  
__E02\.__ [Multipass on Windows][multipasswin], explained by the Multipass team (updated).  
__E03\.__ [Multipass website][multipasslink]  
__E04\.__ [VirtualBox website][virtualboxlink]  
__E05\.__ [VBoxManage manual][vboxmanagelink]  
__E06\.__ [PsExec.exe documentation][psexeclink]  
__E07\.__ [Webmin installation on Ubuntu 18.04 article][webmininstalllink]  

-------


[multipasslink]:https://multipass.run/  "Multipass website"
[multipasswin]:https://multipass.run/docs/using-virtualbox-in-multipass-windows
[vagrantlink]:https://www.vagrantup.com/  "Vagrant website"
[cloudinitlink]:https://cloudinit.readthedocs.io/en/latest/index.html#  "cloud-init documentation"
[virtualboxlink]:https://www.virtualbox.org/ "VirtualBox website"
[hypervlink]:https://docs.microsoft.com/en-us/virtualization/hyper-v-on-windows/about/ "Hyper-V documentation"
[webmininstalllink]:https://linuxize.com/post/how-to-install-webmin-on-ubuntu-18-04/ "Webmin installation on Ubuntu 18.04"
[psexeclink]:https://docs.microsoft.com/en-us/sysinternals/downloads/psexec "PsExec reference & download"
[vboxmanagelink]:https://www.virtualbox.org/manual/ch08.html  "VBoxManage online manual"
[spriggslink]:https://jon.sprig.gs/blog/post/1574 "Blog post"
