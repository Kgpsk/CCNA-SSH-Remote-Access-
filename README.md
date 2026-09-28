# CCNA-SSH-Remote-Access-
No more console cables. Today I configured SSH so  can manage the router and switch remotely from a PC.

##ROUTER R1 — Base identity, security, SSH
enable                                          → enter privileged EXEC mode
configure terminal                              → enter global config mode
hostname R1                                     → required before RSA keys; replaces default "Router"
ip domain-name samlab.com                       → required before RSA key generation
username admin secret cisco123                  → creates local user for SSH login (secret = encrypted)
enable secret class                             → protects privileged mode with encrypted password
crypto key generate rsa                         → generates RSA keys; type 1024 at the prompt (not on same line)
ip ssh version 2                                → forces SSHv2 only (needs RSA keys first)
line vty 0 4                                    → enters the 5 VTY lines for remote access
 login local                                    → authenticate VTY using local username database
 transport input ssh                            → allow only SSH on VTY (blocks Telnet)
 exit                                           → leaves line config mode

 ##ROUTER R1 — Router-on-a-stick subinterface (VLAN 10)
 **interface g0/0                                  → enter physical WAN/LAN interface
 no shutdown                                    → bring physical link up
 exit
interface g0/0.10                               → create subinterface for VLAN 10
 encapsulation dot1Q 10                         → tag frames with VLAN 10 (802.1Q trunking)
 ip address 192.168.10.1 255.255.255.0          → gateway IP for PCs in VLAN 10
 no shutdown                                    → bring subinterface up
 exit
end                                             → return to privileged EXEC
write memory                                    → save running config to NVRAM

##SWITCH SW1 — Management IP, gateway, SSH
enable                                          → privileged EXEC mode
configure terminal                              → global config mode
hostname SW1                                    → required before RSA keys
interface vlan 10                               → enter the VLAN 10 SVI (management interface)
 ip address 192.168.10.2 255.255.255.0          → management IP for SSH access
 no shutdown                                    → bring SVI up
 exit
ip default-gateway 192.168.10.1                 → lets switch reply to off-subnet traffic (L2 device)
ip domain-name samlab.com                       → required before RSA key generation
username admin secret cisco123                  → local user for SSH login
crypto key generate rsa                         → generate RSA keys; type 1024 at prompt
ip ssh version 2                                → enforce SSHv2
line vty 0 4                                    → VTY lines for remote access
 login local                                    → authenticate via local user database
 transport input ssh                            → SSH only, no Telnet
 exit


 ##SWITCH SW1 — VLANs, access ports, trunk uplink
 vlan 10                                         → create VLAN 10
 name DATA                                      → optional descriptive name
 exit
interface range f0/1 - 3                        → select ports connected to PC0, PC1, PC2
 switchport mode access                         → set as access ports
 switchport access vlan 10                      → place PCs in VLAN 10
 no shutdown                                    → ensure ports are enabled
 exit
interface g0/1                                  → port connected to router R1
 switchport mode trunk                          → make it a trunk (carries VLAN tags)
 switchport trunk allowed vlan 10               → only allow VLAN 10 across the trunk
 no shutdown                                    → enable the port
 exit
end                                             → back to privileged EXEC
write memory                                    → save config

##PC0 — Endpoint configuration (GUI, not CLI)
Desktop → IP Configuration:
 IPv4 Address:   192.168.10.10                  → PC IP in VLAN 10 subnet
 Subnet Mask:    255.255.255.0                  → /24
 Default Gateway:192.168.10.1                   → router subinterface (G0/0.10)

 ##PC0 — Verification and SSH tests
 ipconfig                                        → confirm PC IP/mask/gateway
ping 192.168.10.1                               → test PC → router subinterface
ping 192.168.10.2                               → test PC → switch SVI
ssh -l admin 192.168.10.2                       → SSH into switch (password: cisco123)
ssh -l admin 192.168.10.1                       → SSH into router (password: cisco123, NOT class)

##VERIFICATION COMMANDS — R1
show ip interface brief                         → confirm G0/0 and G0/0.10 are up/up
show ip ssh                                     → confirm "SSH Enabled - version 2.0"
show crypto key mypubkey rsa                    → confirm RSA keys exist
show run | include username                     → confirm local user admin exists
show run | include enable                       → confirm enable secret is set
show interfaces g0/0.10                         → confirm dot1Q encapsulation VLAN 10

##VERIFICATION COMMANDS — SW1
show vlan brief                                 → confirm VLAN 10 contains Fa0/1-3
show ip interface brief                         → confirm Vlan10 = 192.168.10.2 up/up
show interfaces trunk                           → confirm Gi0/1 is trunking with VLAN 10 allowed
show interfaces g0/1 switchport                 → confirm Operational Mode: trunk
show ip ssh                                     → confirm SSHv2 enabled
show crypto key mypubkey rsa                    → confirm RSA keys exist

##KEY RULES / GOTCHAS (why certain commands are ordered as they are)
hostname FIRST, then ip domain-name, then crypto key generate rsa
   → IOS refuses RSA keys while hostname is "Router" or domain-name is unset

crypto key generate rsa  then answer 1024 at the prompt
   → "crypto key generate rsa 1024" is invalid syntax on this IOS

ip ssh version 2 must come AFTER keys are generated
   → otherwise you get "Please create RSA keys..." error

username admin secret (not password)
   → task requires encrypted secret; 'password' stores plaintext

At SSH login prompt → password is cisco123 (user secret)
At enable prompt    → password is class (enable secret)
   → mixing these causes "% Login invalid"

Switch needs ip default-gateway
   → it's an L2 device; without it, replies to off-subnet hosts fail

Router subinterface needs encapsulation dot1Q 10
   → otherwise frames arriving tagged for VLAN 10 are dropped

Switch port to router must be trunk, PC ports must be access in VLAN 10
   → mismatched VLANs cause ping timeouts (Vlan10 SVI shows up/down)


**
