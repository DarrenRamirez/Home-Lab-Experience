# Home-Lab-Experience

## Objective
This project demonstrates how to configure a Cisco router and switch using the Router-on-a-Stick method. The router previously had a configuration, so we begin by resetting it using ROMMON mode before moving on to VLAN, DHCP, and NAT configurations. The ultimate goal is to provide inter-VLAN routing and internet access to devices on multiple VLANs.

### Skills Learned
- Router & Switch Basics – Command line configuration, interface setup  
- VLAN Segmentation – Creating VLANs on the switch and subinterfaces on the router  
- DHCP & NAT – Enabling dynamic IP assignment and internet access for internal networks  
- Troubleshooting & Verification – Using show commands and pings to validate configurations  
- ROMMON Reset – Resetting a router to factory defaults to remove old configurations  

### Tools Used
- Cisco 2800 Series Router (or similar)  
- Cisco 3550 Series Switch (or similar)  
- Console Cable (RJ45 to USB) for CLI access  
- ISP Router (for internet connectivity)  
- Terminal Emulator (PuTTY, SecureCRT, Tera Term, etc.)  

---

## Steps

### 1. Resetting the Router Using ROMMON Mode
If the router has an existing configuration and you need to factory reset it:

<img src="https://imgur.com/335EqTh.png" alt="Imgur Image" />

*Ref 1a: Old password I dont know*


1. Enter ROMMON Mode  
   - Power cycle the router and press `Ctrl + Break` as it boots.  
   - Once at the `rommon>` prompt, set the config register to ignore startup configs:
     ```bash
     rommon 1> confreg 0x2142
     rommon 2> reset
     ```
   - This tells the router to skip the old configuration at next boot.


2. Erase Old Configuration  
   - After the router reboots, enter privileged mode:
     ```bash
     enable
     erase startup-config
     reload
     ```
   - The router will then restart without any previous settings.

<img src="https://imgur.com/zSIspGS.png" alt="Imgur Image" />

*Ref 1b: Resetting the router in ROMMON mode*

---

### 2. Network Diagram
Before configuring, plan how the devices will connect. The diagram typically shows:
- ISP Router → Cisco Router (WAN on FastEthernet 0/0)  
- Cisco Router (LAN on FastEthernet 0/1) → Switch (trunk port)  
- End devices on VLAN 10, 20, 30 access ports on the switch  

<img src="https://imgur.com/a1VrJPH.png" alt="Network Diagram" />

*Ref 2: Router Network Diagram*

---

### 3. Physical Setup

#### 3.1 Connecting the Router to ISP
- FastEthernet 0/0 on the Cisco router → ISP Router LAN port  
- This port will obtain an IP address via DHCP from the ISP router  

<img src="https://imgur.com/DCmKsH5.png" alt="ISP Connection" />

*Ref 3: Physical cable from ISP router to Cisco router*

#### 3.2 Connecting the Router to the Switch
- GigabitEthernet 0/1 on the Cisco router → GigabitEthernet 0/1 on the switch (trunk port)  
- This trunk will carry multiple VLANs between the router and switch  

<img src="https://imgur.com/7lMQsEu.png" alt="Router to Switch" />

*Ref 4: Router’s LAN interface to Switch’s trunk port*

#### 3.3 Console Connection
- Connect a console cable (RJ45 → USB) from the router’s console port to your PC for CLI access  

<img src="https://imgur.com/DWGNL5c.png" alt="Console Cable Setup" />

*Ref 5: Console cable for CLI access*

---

### 4. Router Configuration

#### 4.1 Verify Interfaces
```bash
enable
show ip interface brief
```

Shows the current status and IP addresses of interfaces

<img src="https://imgur.com/GhXpidJ.png" alt="Show IP Interface Brief" />

*Ref 6: Confirming interface statuses*

#### 4.2 Configure WAN (gigabitEthernet 0/0)

```bash
configure terminal
interface gigabitEthernet 0/0
ip address dhcp
no shutdown
exit
```

WAN interface obtains an IP from the ISP router via DHCP

<img src="https://imgur.com/cPAzfFR.png" alt="WAN Configuration" />

*Ref 7: WAN interface set to DHCP*

#### 4.3 Default Route & Ping

```bash
configure terminal
ip route 0.0.0.0 0.0.0.0 10.0.0.1
exit
ping 10.0.0.1
```
Set the default route toward your ISP gateway, then test connectivity

<img src="https://imgur.com/wWl0orN.png" alt="Ping Test to ISP Router" />

*Ref 8: Verifying internet connectivity*

---

### 5. VLAN and Subinterface Configuration
#### 5.1 Create VLAN Subinterfaces
```bash

interface fastethernet 0/1.10
encapsulation dot1q 10
ip address 10.10.10.1 255.255.255.0
no shutdown
exit

interface fastethernet 0/1.20
encapsulation dot1q 20
ip address 10.10.20.1 255.255.255.0
no shutdown
exit

interface fastethernet 0/1.30
encapsulation dot1q 30
ip address 10.10.30.1 255.255.255.0
no shutdown
exit
```

These subinterfaces allow the router to route between VLANs 10, 20, and 30 on a single physical interface

<img src="https://imgur.com/dWrZcpA.png" alt="Subinterface Configuration" />

*Ref 9: Router-on-a-Stick subinterfaces*

---

### 6. DHCP Configuration
To assign IP addresses automatically:

```bash
configure terminal

ip dhcp pool VLAN10
network 10.10.10.0 255.255.255.0
default-router 10.10.10.1
dns-server 8.8.8.8
exit

ip dhcp pool VLAN20
network 10.10.20.0 255.255.255.0
default-router 10.10.20.1
dns-server 8.8.8.8
exit

ip dhcp pool VLAN30
network 10.10.30.0 255.255.255.0
default-router 10.10.30.1
dns-server 8.8.8.8
exit
```

<img src="https://imgur.com/FYtqqRB.png" alt="DHCP Configuration" />

*Ref 10: Router acting as DHCP server*

### 7. NAT Configuration (For Internet Access)

To allow VLANs to access the internet:

1. Mark WAN as outside, subinterfaces as inside:

```bash
interface fastethernet 0/0
ip nat outside
exit

interface fastethernet 0/1.10
ip nat inside
exit

interface fastethernet 0/1.20
ip nat inside
exit

interface fastethernet 0/1.30
ip nat inside
exit
```
2. Create an ACL for inside networks and apply NAT overload:

```bash
ip access-list standard LOCAL
permit 10.10.10.0 0.0.0.255
permit 10.10.20.0 0.0.0.255
permit 10.10.30.0 0.0.0.255
exit

ip nat inside source list LOCAL interface fastethernet 0/0 overload
```

<img src="https://imgur.com/5wiroxg.png" alt="NAT Configuration" />

*Ref 11: NAT Overload for internet access*

---

### 8. Switch Configuration
#### 8.1 Configure Trunk Port
```bash
interface gigabitEthernet 0/48
switchport mode trunk
no shutdown
exit
```
- This port carries VLAN 10, 20, and 30 traffic to the router
  
<img src="https://imgur.com/YourSwitchTrunk.png" alt="Switch Trunk Port" />

*Ref 12: Trunk port configuration*

#### 8.2 Create and Assign VLANs
```bash
Copy
Edit
vlan 10
exit
vlan 20
exit
vlan 30
exit

interface range gigabitEthernet 0/1-16
switchport mode access
switchport access vlan 10
exit

interface range gigabitEthernet 0/17-32
switchport mode access
switchport access vlan 20
exit

interface range gigabitEthernet 0/33-44
switchport mode access
switchport access vlan 30
exit
```

<img src="https://imgur.com/1yuEHNL.png" alt="VLAN Assignment" />

*Ref 13: Access port assignment to respective VLANs*


---

### 9. Testing & Verification

#### 9.1 Check VLAN Connectivity
- From a VLAN 10 device, ping 10.10.10.1
- From a VLAN 20 device, ping 10.10.20.1
- From a VLAN 30 device, ping 10.10.30.1

<img src="https://imgur.com/YourPingTest.png" alt="Ping Gateways" />

*Ref 14: Ensuring gateway reachability in each VLAN*

#### 9.2 Verify DHCP
- End devices should receive IP addresses in their respective VLAN subnets (e.g., 10.10.10.x)
  
#### 9.3 Test Internet Access
- Attempt to browse or ping an external IP/URL (e.g., ping 8.8.8.8)

<img src="https://imgur.com/YourInternetTest.png" alt="Internet Test" />

*Ref 15: Confirming successful NAT and internet access*

---

### Conclusion
By resetting the router in ROMMON mode, creating subinterfaces for each VLAN, configuring DHCP and NAT, and assigning VLANs on the switch, we achieved inter-VLAN routing and internet access. This Router-on-a-Stick architecture provides efficient segmentation, easy expandability, and centralized control of multiple subnets—all on a single physical router interface.


