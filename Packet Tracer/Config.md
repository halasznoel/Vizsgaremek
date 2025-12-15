Server

Router:

ena

conf t

int g0/1

ip add 192.168.50.3 255.255.255.0

no shu

int g0/0

ip add 192.168.30.1 255.255.255.0

no shu

exit

router rip

version 2

no auto-summary

network 192.168.30.0

network 192.168.50.0

passive-interface g0/0

end

wr



5505 ASA:



ena

conf t

interface e0/0

nameif inside

security-level 100

ip address 192.168.30.1 255.255.255.0

no shutdown

interface e0/1

nameif outside

security-level 0

ip address 192.168.31.2 255.255.255.0

no shutdown

route outside 0.0.0.0 0.0.0.0 192.168.31.1

access-list OUTSIDE-IN extended permit icmp any any

access-group OUTSIDE-IN in interface outside

nat (inside,outside) dynamic interface



*Switch:*

ena

conf t

hostname ACCESS-SW-Servers

no ip domain-lookup

service password-encryption



line con 0

logging synchronous

exec-timeout 0 0

line vty 0 4

transport input telnet

password cisco

login



exit

clock timezone CET 1 0





vlan 30

 name SERVERS

vlan 99

 name MGMT



exit

interface vlan 99

 ip address 192.168.99.2 255.255.255.0

 no shutdown



exit

ip default-gateway 192.168.99.1



interface range fa0/1 - 24

 switchport mode access

 switchport access vlan 30

 spanning-tree portfast

 spanning-tree bpduguard enable



interface range gi0/1 - 2

 switchport mode access

 switchport access vlan 30

 spanning-tree portfast

 spanning-tree bpduguard enable



interface fa0/24

 switchport trunk encapsulation dot1q

 switchport mode trunk

 switchport trunk allowed vlan 30,99

 spanning-tree portfast trunk



interface gi0/1

&nbsp; switchport trunk encapsulation dot1q

&nbsp; switchport mode trunk

&nbsp; switchport trunk allowed vlan 30,99

&nbsp; spanning-tree portfast trunk



exit

spanning-tree mode rapid-pvst

spanning-tree extend system-id

spanning-tree portfast default



exit

ip dhcp snooping

ip dhcp snooping vlan 30



interface fa0/24

 ip dhcp snooping trust



interface range fa0/1 - 23

 ip dhcp snooping limit rate 30



exit

mls qos





conf t

hostname WAN-CORE

no ip domain-lookup

service password-encryption



! Line beállítások

line con 0

 logging synchronous

 exec-timeout 0 0

line vty 0 4

 transport input telnet

 password cisco

 login



! VLAN-ok – ha a WAN-on több logikai VLAN-t használsz (opcionális)

vlan 50

 name WAN

vlan 99

 name MGMT



! Menedzsment SVI

interface vlan 99

 ip address 192.168.99.1 255.255.255.0

 no shutdown



! Default gateway (L2 core esetén)

ip default-gateway 192.168.99.254



! STP – a WAN-CORE legyen a root-bridge (rapid-PVST)

spanning-tree mode rapid-pvst

spanning-tree extend system-id

spanning-tree vlan 1,50,99 root primary

spanning-tree portfast default



! Router uplink portok – csatlakoznak a 3 routerhez (Servers/HQ/HR)

! Példa: Gi1/0/1 → Servers Router (192.168.50.3)

!         Gi1/0/2 → HQ Router     (192.168.50.1)

!         Gi1/0/3 → HR Router     (192.168.50.2)

interface gi1/0/1

 description UPLINK to Servers Router

 switchport mode access

 switchport access vlan 50

 spanning-tree portfast disable



interface gi1/0/2

 description UPLINK to HQ Router

 switchport mode access

 switchport access vlan 50

 spanning-tree portfast disable



interface gi1/0/3

 description UPLINK to HR Router

 switchport mode access

 switchport access vlan 50

 spanning-tree portfast disable



! Ha a szerver is közvetlenül a WAN-ra csatlakozik:

interface fa0/5

 description Server on WAN VLAN

 switchport mode access

 switchport access vlan 50

 spanning-tree bpduguard enable

! (Server porton PortFast is mehet, ha L2 loop veszély nincs)

spanning-tree portfast edge default



! DHCP snooping – WAN VLAN védelem

ip dhcp snooping

ip dhcp snooping vlan 50

! Router uplinkek TRUSTED

interface gi1/0/1

 ip dhcp snooping trust

interface gi1/0/2

 ip dhcp snooping trust

interface gi1/0/3

 ip dhcp snooping trust



! Opcionális EtherChannel a redundáns linkekhez (ha két link megy egy routerhez)

! interface range gi1/0/1, gi1/0/4

!  channel-group 1 mode active

! interface port-channel1

!  switchport mode access

!  switchport access vlan 50



end

wr







**HQ**

*router:*



conf t

interface g0/1

ip address 192.168.50.1 255.255.255.0

no shutdown

interface g0/0

ip address 10.0.0.1 255.255.255.0

no shutdown

router rip

version 2

no auto-summary

network 10.0.0.0

network 192.168.50.0

passive-interface g0/0

end

wr

