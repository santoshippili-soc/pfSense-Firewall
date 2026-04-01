🔥 pfSense Firewall Home Lab 
A hands-on cybersecurity home lab simulating a Denial-of-Service (DoS) attack and its mitigation using pfSense, Kali Linux, and Ubuntu, all running in VirtualBox.

📌 Overview
This lab demonstrates:

Setting up a two-zone firewall environment with pfSense
Simulating a DoS/flood attack from an external attacker (Kali Linux)
Detecting the attack using Wireshark on the victim machine
Blocking the attack by creating a firewall rule in pfSense
Verifying the block through pfSense firewall logs


🗺️ Network Topology
Internet / Home LAN
        │
  ┌─────┴──────┐
  │  Kali Linux │  (Attacker)
  │ 192.168.1.5 │  [Bridged Adapter]
  └─────┬──────┘
        │
  ┌─────┴──────────────┐
  │   pfSense Firewall  │
  │  WAN: 192.168.1.6   │  [Bridged Adapter → vtnet0]
  │  LAN: 192.168.10.1  │  [Internal Network → vtnet1]
  └─────┬──────────────┘
        │  Internal Network: LabNet (192.168.10.0/24)
  ┌─────┴──────────┐
  │ Ubuntu Desktop  │  (Victim)
  │ 192.168.10.100  │  [Internal Network Adapter]
  └────────────────┘
DeviceAdapterInterfaceIP AddressRoleKali LinuxBridgedeth0192.168.1.5External AttackerpfSense WANBridgedvtnet0192.168.1.6Edge Firewall (WAN)pfSense LANInternal (LabNet)vtnet1192.168.10.1Default GW + DHCPUbuntu DesktopInternal (LabNet)eth0192.168.10.100Victim Host

🛠️ Prerequisites

VirtualBox ≥ 7.x
ISO images:

pfSense CE (pfSense-CE-2.7.x-amd64.iso)
Kali Linux (kali-linux-current-amd64.iso)
Ubuntu Desktop (ubuntu-22.04-desktop-amd64.iso)


Host minimum: 8 GB RAM, 40 GB free disk
Admin rights on the host machine


🖥️ VM Configuration
pfSense VM
SettingValueOS TypeFreeBSD 64-bitCPUs / RAM2 vCPU / 2 GBDisk20 GB VDI (dynamic)Adapter 1Bridged → Physical NIC (WAN)Adapter 2Internal Network → LabNet (LAN)
Kali Linux VM (Attacker)
SettingValueOS TypeDebian 64-bitCPUs / RAM2 vCPU / 2 GBDisk15 GB VDI (dynamic)Adapter 1Bridged (WAN side)
Ubuntu VM (Victim)
SettingValueOS TypeUbuntu 64-bitCPUs / RAM2 vCPU / 2 GBDisk15 GB VDI (dynamic)Adapter 1Internal Network → LabNet

⚙️ Lab Setup Steps
1. pfSense Installation & Interface Assignment

Boot pfSense VM from ISO and accept defaults during install
From the console menu, assign interfaces:

vtnet0 → WAN (Bridged)
vtnet1 → LAN (LabNet)


Leave WAN on DHCP — pfSense will receive 192.168.1.6 from the home router

2. Access pfSense WebGUI from the WAN
Access the pfSense shell and temporarily disable the firewall to allow initial GUI access:
bashpfctl -d

Browse to https://192.168.1.6 from Kali or host
Default credentials: admin / pfsense
Add a WAN firewall rule to allow GUI access:

Firewall → Rules → WAN → Add
Action: Pass | Protocol: TCP | Source: Home Network | Destination: pfSense WAN | Port: 443



3. Configure LAN DHCP

Navigate to Services → DHCP Server → LAN
Enable DHCP on LAN interface
Range: 192.168.10.100 – 192.168.10.199
DNS Server: 192.168.1.1 (home router)
Save and Apply

4. Allow Kali Access to Internal LAN (for Demo)

Firewall → Rules → WAN → Add
Action: Pass | Protocol: Any | Source: 192.168.1.5 (Kali) | Destination: 192.168.10.100 (Ubuntu)
Description: Allow Kali access to Ubuntu

5. Kali Linux — Add Static Route
Check IP:
bashifconfig
Add a route to reach the internal LAN via pfSense WAN:
bashsudo ip route add 192.168.10.0/24 via 192.168.1.6
6. Ubuntu (Victim) — Verify Connectivity
Ubuntu should automatically receive 192.168.10.100 via pfSense DHCP.
bashifconfig
ping -c 3 google.com
Test from Kali → Ubuntu:
bashping 192.168.10.100

💥 Launching the DoS Attack (hping3)
On Ubuntu — Start Wireshark Capture
bashsudo apt update && sudo apt install -y wireshark
sudo wireshark &   # Capture on eth0
On Kali — Launch the Flood
ICMP Flood:
bashsudo hping3 -1 --flood 192.168.10.100
SYN Flood (alternative):
bashsudo hping3 --flood -S -p 80 192.168.10.100
Observe the packet spike in Wireshark on Ubuntu — thousands of packets per second flooding in.

🛡️ Blocking the Attack in pfSense

Navigate to Firewall → Rules → WAN → Add (TOP)
Configure the block rule:

Action: Block
Source: 192.168.1.5 (Kali IP)
Destination: 192.168.10.100 (Ubuntu IP)
Log: ✅ Enabled
Description: Block Kali DoS


Click Apply Changes

✅ Traffic halts — Wireshark on Ubuntu should stop showing the flood traffic immediately.
Verify in pfSense Logs

Navigate to Status → System Logs → Firewall
Filter by the Block Kali DoS rule
You will see blocked entries confirming the flood packets are being dropped


🧰 Tools Used
ToolPurposepfSense CEFirewall / Router / DHCP ServerKali LinuxAttacker machinehping3DoS flood packet generatorUbuntu DesktopVictim machineWiresharkPacket capture and traffic visualizationVirtualBoxVirtualization platform

🔐 Key Concepts Demonstrated

Firewall rule ordering — Block rules placed at the TOP take priority over allow rules below
WAN vs LAN rule placement — Inbound attack traffic is filtered at the WAN interface
Firewall logging — Enabling logs on rules provides evidence of blocked malicious traffic
Static routing — Manual route injection on Kali to reach the isolated LAN segment
DHCP scoping — pfSense acting as DHCP server for the internal LabNet subnet
Network segmentation — WAN and LAN zones isolate the victim from the attacker


🔒 Security Best Practices

Keep the homelab isolated from production/home networks
Regularly update Ubuntu, Kali, and pfSense to patch known vulnerabilities
Always re-enable the pfSense firewall (pfctl -e) after initial setup
Use strong credentials — change the default admin/pfsense password immediately
Enable logging on critical firewall rules for audit trails


👤 Author
Santosh Ippili
B.Tech CSE (Cybersecurity) — VIT-AP University, 2025
EC-Council Certified SOC Analyst (CSA) | AWS Certified Cloud Practitioner

📜 License
This project is for educational purposes only. Do not use these techniques on systems you do not own or have explicit permission to test.
