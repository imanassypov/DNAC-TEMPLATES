# PnP Preparation 
## Overview
This lab is the first one in a series of labs. You may use the steps in the [Cisco Enterprise Networks Hardware Sandbox](https://dcloud2-sjc.cisco.com/content/catalogue?search=Enterprise%20Networks%20Hardware%20Sandbox&screenCommand=openFilterScreen) environment, or equally, you might utilize them as part of a Proof of Concept setup at a customer's lab. These procedures may also help form part of a deployment or implementation. Use them to ensure that all the necessary steps are complete before onboarding any devices within DNA Center.

We will be utilizing the lab in this manner:
SJC Topology
![json](./images/DCLOUD_Topology_PnPLab2.png?raw=true "Import JSON")

## General Information
As you may recall, in the informational sections of this repository, we described various methods of discovery for a device and the preliminary things required for proper zero-touch provisioning. This lab will ensure a successful connection to DNA Center by helping to deploy the initial concepts.

### Lab Preparation
To set up the lab, please log into the console connection of the ***4331*** by clicking on dropdown button under the icon in DCLOUD, and selecting 'Console'. New tab will open, click 'Connect' - you should now be connected to the console of the router. To obtain enable password, navigate back to topology diagram in DCLOUD and click on any of Windows Clients or Devices and copy-paste the password into Hardware Console tab after issuing enable CLI.  Once you are in enable mode on 4331 hardware console, issue commands:

***Warning** use commands against the LAB Environment.*

SJC
```vtl
!
conf t
!
!disable port 0/0/1 for the templating lab
int gi 0/0/1
 shutdown
 end
!
wr
!
```


## Lab Section 1 - Device Connectivity
For PnP processes to work, we intend to have a management interface on the device. In this lab, we will set up a VLAN interface for both management and connectivity. You don't have to do it this way; we are just giving a relatively uncomplicated example, and you can alter this to suit your needs. As the device connects to the front-facing ports, we have to rely on the default configuration. 

As you may recall, a factory default configuration is using VLAN 1 as no other VLAN exists, and by default, it accepts DHCP addresses. We can use this method in the PnP process. However, the management VLAN may be different, and so may the native VLAN structure of our environment. To that end, we must use the *pnp startup-vlan* command, which allows the device to use varying VLANs in PnP and should be set up and configured on the upstream switch.

Of the discovery methods **DHCP** is the easiest to implement as no changes are required with the *Self Signed Certificate (SSC)* on **DNA Center** as it already includes the IP address by default. If you are deploying PnP using **DNS** discovery or you are building a cluster then you will need to go through the process of acquiring a certificate with Subject Alternative Names to include the **DNS** and **IP** entries to allow for the following:

1. All Node IP Addresses
2. All VIP Addresses for Cluster
3. All DNS Host entries for Nodes
4. VIP DNS Host entry for Cluster
5. pnpserver Host or CNAME entries

<details closed>
<summary>To build a certificate in dCLOUD follow these steps </summary>
To Build a certificate for use in DNA Center for PnP, please follow this outline of steps. Each step can take some time so plan accordingly.

1. On the Active Directory Server add the roles for the Certificate Authority to allow WEB enrollment
2. Add the required DNS entries for DNA Center as per the sections below
3. On DNA Center in CLI create a CSR using openssl 
4. Enroll DNA Center via the CSR on the Windows CA
5. Upload the Certificate to DNA Center

Follow this guide for more information on the finer details.

[DNA Center Security Best Practices Guide](./DNACenter_security_best_practices_guide.pdf)
</details>

### Step 1.1 - ***Upstream Neighbor Setup***
As depicted in the topology diagram above, Catalyst 9300-2 will serve as the upstream neighbor for this exercise and the environment's distribution switch. The Catalyst 9300-1 will act as the target switch, which we will deploy via PnP and Day 0 and N templates.

For the lab, we will utilize ***VLAN 5*** as the management VLAN. Connect to switch ***c9300-2*** and paste the following configuration:

```vtl
config t
!
vlan 5
name "mgntvlan"
!
int vlan 5 
 ip address 192.168.5.1 255.255.255.0
 ip ospf 1 area 0
 no shutdown
!
pnp startup-vlan 5
end
!
wr
!
```

The ***pnp startup-vlan 5*** command will program the target switches port connected with a trunk and automatically add the vlan and SVI to the target switch making that vlan ready to accept a DHCP address. The feature is available on switches running IOS-XE 16.6 code or greater as upstream neighbors. Older switches or upstream devices that cannot run the command should utilize VLAN 1 and then set up the correct management VLAN modified as part of the onboarding process.

We also need to ensure that our upstream ***c9300-2*** switch has IP reachability to DNA Center. You will notice that ***c9300-2*** switch interface Gi1/0/48 is connected via L2 segment passing through ***c9300-3*** to ***4331*** router Gi0/0/2 interface. The upstream switch ***c9300-2*** and ***4331*** should establish OSPF adjacency. 

Confirm that ***c9300-2*** device uplink interface Gi1/0/48 is configured as follows

```vtl
config t
!
interface GigabitEthernet1/0/48
 no switchport
 ip address 198.19.2.2 255.255.255.252
 ip ospf 1 area 0
end
!
wr
!
```

Confirm on ***c9300-2*** we have required OSPF received routes:

```vtl
Switch#sh ip route
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area 
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2, m - OMP
       n - NAT, Ni - NAT inside, No - NAT outside, Nd - NAT DIA
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       H - NHRP, G - NHRP registered, g - NHRP registration summary
       o - ODR, P - periodic downloaded static route, l - LISP
       a - application route
       + - replicated route, % - next hop override, p - overrides from PfR
       & - replicated local route overrides by connected

Gateway of last resort is 198.19.2.1 to network 0.0.0.0

O*E2  0.0.0.0/0 [110/1] via 198.19.2.1, 00:04:49, GigabitEthernet1/0/48
      192.168.5.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.5.0/24 is directly connected, Vlan5
L        192.168.5.1/32 is directly connected, Vlan5
O     198.18.128.0/18 [110/2] via 198.19.2.1, 00:04:49, GigabitEthernet1/0/48
      198.19.2.0/24 is variably subnetted, 2 subnets, 2 masks
C        198.19.2.0/30 is directly connected, GigabitEthernet1/0/48
```

Also confirm that the upstream ***c9300-2*** switch can reach DNA Center IP address

```vtl
Switch#ping 198.18.129.100 
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 198.18.129.100, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/1 ms
```
### Step 1.2 - ***DHCP Setup***
We need a DHCP scope to temporarily supply the address within the management network to complete the configuration and onboarding. Configure the scope to offer IP addresses from one part of the address's range, leaving the other part of the scope for static addresses. You could also make use of reservations as DHCP servers can reserve addresses for specific MAC addresses. One benefit of this is that DNS host entries are automatically updated depending on the DHCP Server.

The DHCP scope should therefore incorporate the following minimal configuration:

* network
* default gateway
* domain - ***required if option 2 is used below***
* name-server ip - ***required if option 2 or 3 is used below***
* DHCP relay or helper statement - ***to be added to the gateway interface pointing to the DHCP server***

There are many options for DHCP services. Although you have many options for DHCP, we will cover Windows and IOS configurations in this lab. Configure the DHCP scope to one of the following:

1. Switch or Router
2. Windows DHCP Server
3. InfoBlox or other 3rd party server

During this lab setup, please choose which option you wish to use for DHCP for PnP services and follow those subsections.

#### Step 1.2a - ***IOS DHCP Configuration***
Configured on an IOS device, the DHCP pool elements would be configured either on a router or switch in the network. 

If we want to use the IOS DHCP configuration method, connect to switch ***c9300-2*** and paste the following configuration:

```vtl
!
conf t
!
  ip dhcp excluded-address 192.168.5.1 192.168.5.1
  ip dhcp pool pnp_device_pool                         
     network 192.168.5.0 255.255.255.0                  
     default-router 192.168.5.1 
     end
!
wr
!
```

Next, we will configure the helper address statement on the management VLAN's SVI to point to the router or switch to the DHCP configuration. Connect to switch ***c9300-2*** and paste the following configuration:

```vtl
!
conf t
!
  interface Vlan 5                         
     ip helper-address 192.168.5.1                  
     end
!
wr
!
```

For a complete configuration example please see [Configuring the Cisco IOS DHCP Server](https://www.cisco.com/en/US/docs/ios/12_4t/ip_addr/configuration/guide/htdhcpsv.html#wp1046301)

#### Step 1.2b - ***Windows Server Configuration***
If we want to use the Windows DHCP service, connect to the windows ***AD1*** server. On the windows server, you have two options to deploy DHCP scopes the UI or PowerShell. We will deploy the scope via PowerShell. Paste the following into PowerShell to create the required DHCP scope:

```ps
Add-DhcpServerv4Scope -Name "DNAC-Templates-Lab" -StartRange 192.168.5.1 -EndRange 192.168.5.254 -SubnetMask 255.255.255.0 -LeaseDuration 6.00:00:00 -SuperScope "PnP Onboarding"
Set-DhcpServerv4OptionValue -ScopeId 192.168.5.0 -Router 192.168.5.1 
Add-Dhcpserverv4ExclusionRange -ScopeId 192.168.5.0 -StartRange 192.168.5.1 -EndRange 192.168.5.1
```

The DHCP scope will look like this in Windows DHCP Administrative tool:

![json](./images/WindowsDHCPscoperouteronly.png?raw=true "Import JSON")

Next, we will introduce the helper address statement on the management VLAN's SVI to point to the Windows DHCP server. Connect to switch ***c9300-2*** and paste the following configuration:

```vtl
!
conf t
!
  interface Vlan 5                         
     ip helper-address 198.18.133.1                  
     end
!
wr
!
```

## Lab Section 2 - DNA Center Discovery
As you may recall, for a device to discover DNA Center, the device uses a discovery method to help it find DNA Center. 

The PnP components are as follows:

![json](../../images/pnp-workflows.png?raw=true "Import JSON")

There are three automated methods to make that occur:

1. **DHCP with option 43** - ***requires the DHCP server to offer a PnP string via option 43***
2. **DNS lookup** - ***requires the DHCP server to offer a domain suffix and a name server to resolve the **pnpserver** address***
3. **Cloud re-direction via https://devicehelper.cisco.com/device-helper** - ***requires the DHCP server to offer a name server to make DNS resolutions***

### Step 2.1 - ***DNA Center Discovery***
Please choose one of the following subsections as the discovery method.

#### Step 2.1a - ***Option 43 with IOS DHCP Configuration***
If using the IOS DHCP Server and the desire is to use Option 43 discovery method, then paste the following configuration:

```vtl
!
conf t
!
  ip dhcp pool pnp_device_pool                    
     option 43 ascii "5A1N;B2;K4;I198.18.129.100;J80"
     end
!
wr
!
```

#### Step 2.1b - ***Option 43 with Windows DHCP Configuration***
If using the Windows DHCP Server and the desire is to use Option 43 discovery method, then paste the following configuration into PowerShell:

```ps
Set-DhcpServerv4OptionValue -ScopeId 192.168.5.0 -OptionId 43 -Value ([System.Text.Encoding]::ASCII.GetBytes("5A1N;B2;K4;I198.18.129.100;J80"))
```

The DHCP scope modification will resemble the following image of the Windows DHCP Administrative tool:

![json](./images/DNACDHCPoption43.png?raw=true "Import JSON")

#### Step 2.1c - ***DNS Lookup with IOS DHCP Configuration***
If using the IOS DHCP Server and the desire is to use the DNS Lookup discovery method, then paste the following configuration:

```vtl
!
conf t
!
  ip dhcp pool pnp_device_pool                          
     dns-server 198.18.133.1                           
     domain-name dcloud.cisco.com                       
     end
!
wr
!
```

Next, add the DNS entries to allow for the DNA Center to be discovered. This script will add an A host entry for the VIP address and a CNAME entry as an alias for the pnpserver record required for DNS discovery.

```ps
Add-DnsServerResourceRecordA -Name "dnac-vip" -ZoneName "dcloud.cisco.com" -AllowUpdateAny -IPv4Address "198.18.129.100" -TimeToLive 01:00:00
Add-DnsServerResourceRecordCName -Name "pnpserver" -HostNameAlias "dnac-vip.dcloud.cisco.com" -ZoneName "dcloud.cisco.com"
```

The DNS Zone will look like this in Windows DNS Administrative tool: 

![json](./images/DNACenterDNSentries.png?raw=true "Import JSON")

#### Step 2.1d - ***DNS Lookup with Windows DHCP Configuration***
If using the Windows DHCP Server and the desire is to use the DNS Lookup discovery method, then paste the following configuration into PowerShell:

```ps
Set-DhcpServerv4OptionValue -ScopeId 192.168.5.0 -DnsServer 198.18.133.1 -DnsDomain "dcloud.cisco.com"
```

The DHCP scope will resemble the following image of the Windows DHCP Administrative tool:

![json](./images/WindowsDHCPscope.png?raw=true "Import JSON")

Next, add the DNS entries to allow for the DNA Center to be discovered. This script will add an A host entry for the VIP address and a CNAME entry as an alias for the pnpserver record required for DNS discovery.

```ps
Add-DnsServerResourceRecordA -Name "dnac-vip" -ZoneName "dcloud.cisco.com" -AllowUpdateAny -IPv4Address "198.18.129.100" -TimeToLive 01:00:00
Add-DnsServerResourceRecordCName -Name "pnpserver" -HostNameAlias "dnac-vip.dcloud.cisco.com" -ZoneName "dcloud.cisco.com"
```

The DNS Zone will look like this in Windows DNS Administrative tool: 

![json](./images/DNACenterDNSentries.png?raw=true "Import JSON")

## Lab Section 3 - Target Connectivity
Typically, the Target switch is connected via a trunk to a single port or a bundle of ports as part of a port channel. 

If it is a single port connection to the target switch, then use a simplified configuration; however, we will not be utilizing this method in this lab. An example provided here. Theoretical example of a single physical interface configuration on a parent switch:

```vtl
!
conf t
!
  interface gi 1/0/10
     description PnP Test Environment to Cataylist 9300
     switchport mode trunk
     switchport trunk allowed vlan 5
     end
!
wr
!
```

In this exercise, the port where the Target switch connects is a layer two trunk as part of a Port Channel. While connected to ***c9300-2*** console:

```vtl
!
conf t
!
  interface range gi 1/0/10-11
     description PnP Test Environment to Catalyst 9300
     switchport mode trunk
     switchport trunk native vlan 5
     switchport trunk allowed vlan 5
     channel-protocol lacp
     channel-group 1 mode passive
!
  interface Port-channel1
     description PnP Test Environment to Catalyst 9300
     switchport trunk native vlan 5
     switchport trunk allowed vlan 5
     switchport mode trunk
     no port-channel standalone-disable
     end
!
wr
!
```

If we are using a port-channel initially, you want to ensure that the port-channel can operate as a single link within the bundle and, for that reason, use passive methods for building the port-channel bundles on both the Target and Upstream Neighbor for maximum flexibility. Additionally, add the **no port-channel standalone-disable** command to ensure the switch does not automatically disable the port-channel if it does not come up properly.

It is expected output to see native vlan mismatch message in the console since the downstream ***c9300-2*** is configured by default with Vlan 1

```
*Jul  4 15:17:45.817: %CDP-4-NATIVE_VLAN_MISMATCH: Native VLAN mismatch discovered on GigabitEthernet1/0/10 (5), with c9300-1.dcloud.cisco.com GigabitEthernet1/0/10 (1).
```

## Lab Section 4 - Testing
Please use the testing for the DNS Discovery method used above.

### Step 4.1a - ***DNS Resolution Tests***
To test the environment to ensure it's ready, we need to try a few things.

First, from a Windows host, use the *nslookup* command to resolve ***pnpserver.dcloud.cisco.com***. Connect to the Windows workstation, and within the search window, search for CMD. Open the application and type the following command:

```bash
nslookup pnpserver.dcloud.cisco.com
```

The following output or something similar shows the resolution of the alias to the A host record entry which identifies the VIP address for the DNA Center Cluster will display.

![json](./images/DNACenterDNStests.png?raw=true "Import JSON")

### Step 4.1b - ***DNS Resolution***
Second, we need to ensure the DNA Center responds on the VIP, so use the ping command within the CMD application window previously opened as follows:

```bash
ping pnpserver.dcloud.cisco.com
```

![json](./images/DNACenterDNStestPing.png?raw=true "Import JSON")

At this point, the environment should be set up to onboard devices within VLAN 5 using the network address ***192.168.5.0/24*** utilizing either ***option 43*** or ***DNS discovery ***.

### Step 4.2 - ***Reset EEM Script or PnP Service Reset***
When testing, you will frequently need to start again on the switch to test the whole flow. To accomplish this, paste this small script into the ***c9300-1*** target switch, which will create a file on flash which you may load into the running-configuration at any time to reset the device to factory settings:

There are now two methods for this The first and simplest method is to make use of the `pnp service reset` command as advised by Matthew Bishop. This command was introduced in a recent Train of IOS-XE code.

Failing that we have an EEM script which you may use iterated below.

```tcl
tclsh                            
puts [open "flash:prep4dnac" w+] {
!
! Remove any confirmation dialogs when accessing flash
file prompt quiet
!
no event manager applet prep4dnac
event manager applet prep4dnac
 event none sync yes
 action a1010 syslog msg "Starting: 'prep4dnac'  EEM applet."
 action a1020 puts "Preparing device to be discovered by device automation - This script will reboot the device."
 action b1010 cli command "enable"
 action b1020 puts "Saving config to update BOOT param."
 action b1030 cli command "write"
 action c1010 puts "Erasing startup-config."
 action c1020 cli command "wr er" pattern "confirm"
 action c1030 cli command "y"
 action d1010 puts "Clearing crypto keys."
 action d1020 cli command "config t"
 action d1030 cli command "crypto key zeroize" pattern "yes/no"
 action d1040 cli command "y"
 action e1010 puts "Clearing crypto PKI stuff."
 action e1020 cli command "no crypto pki cert pool" pattern "yes/no"
 action e1030 cli command "y"
 action e1040 cli command "exit"
 action f1010 puts "Deleting vlan.dat file."
 action f1020 cli command "delete /force vlan.dat"
 action g1010 puts "Deleting certificate files in NVRAM."
 action g1020 cli command "delete /force nvram:*.cer"
 action h0001 puts "Deleting PnP files"
 action h0010 cli command "delete /force flash:pnp*"
 action h0020 cli command "delete /force nvram:pnp*"
 action i0001 puts "Reseting Stack Priority"
 action i0010 cli command "switch 1 priority 1"
 action z1010 puts "Device is prepared for being discovered by device automation.  Rebooting."
 action z1020 syslog msg "Stopping: 'prep4dnac' EEM applet."
 action z1030 reload
exit
!
alias exec prep4dnac event manager run prep4dnac
!
end
}
tclquit
```
Additionally, for help with troubleshooting, install this helpful EEM script in the directory in the same manner as above. This will help to see which lines were sent to the switch and helps deduce where a template may be failing.
```tcl
tclsh
puts [open "flash:dnacts" w+] {
!
event manager applet CLI_COMMANDS-->
event cli pattern ".*" sync no skip no
action 1 syslog msg "$_cli_msg"
!
}
tclquit
```

### Step 4.3 - ***Reset Switch and Test Discovery***
Finally, we want to test the routing, connectivity, DHCP, DNS services, and discovery mechanism. Reset the ***c9300-1*** Target switch by pasting the following sequence into the console. We will watch the switch come up but not intercede or type anything into the console after the reboot has started.

```vtl
!
copy prep4dnac running-config
!
prep4dnac
!
```

The Switch should reboot and display this eventually in the console which acknowledges that the 9300 has discovered the DNA Center.
In DCLOUD tab, click on JumpHost icon, select 'Connect'. In the new tab that opens with RDP session to Jump Host, start Google Chrome and select DNAC

![json](./images/DNAC-IPV4-DISCOVERY.png?raw=true "Import JSON")

Additionally, within DNA Center on the Plug and Play window, the device should show as unclaimed.

![json](./images/DNAC-9300-Discovery.png?raw=true "Import JSON")

## Summary
The next step will be to build the PnP Onboarding settings and template on DNA Center, which we will cover in the next lab entitled [Onboarding Templates](../LAB-B-Onboarding-Template/README.md#Day0) - The next lab explains in-depth and how to deploy Day 0 templates.

## Feedback
If you found this set of Labs helpful, please fill in comments and [give feedback](https://app.smartsheet.com/b/form/f75ce15c2053435283a025b1872257fe) on how we can improve.
