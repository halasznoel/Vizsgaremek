**Server** 

*2911 Router:*



ena

conf t

!

interface g0/1           ! WAN → WAN switch

&nbsp;ip address 192.168.50.3 255.255.255.0

&nbsp;no shutdown

!

interface g0/0           ! Servers LAN

&nbsp;ip address 192.168.30.1 255.255.255.0

&nbsp;no shutdown

!

router rip

&nbsp;version 2

&nbsp;no auto-summary

&nbsp;network 192.168.30.0

&nbsp;network 192.168.50.0

! passive-interface g0/0

end

wr



*5505 ASA:*



ena

conf t

!

interface e0/0

&nbsp;nameif inside

&nbsp;security-level 100

&nbsp;ip address 192.168.30.1 255.255.255.0

&nbsp;no shutdown

!

interface e0/1

&nbsp;nameif outside

&nbsp;security-level 0

&nbsp;ip address 192.168.31.2 255.255.255.0

&nbsp;no shutdown



*Magyarázat:*



nameif inside → belső hálózat neve

security-level 100 → legmagasabb biztonsági szint (LAN)

nameif outside → külső hálózat neve

security-level 0 → legkisebb biztonsági szint (WAN/Internet)



*2960 Switch*



! Hostname és alapok

conf t

hostname ACCESS-SW-Servers

no ip domain-lookup

service password-encryption



! Konzol és vty

line con 0

logging synchronous

exec-timeout 0 0

line vty 0 4

transport input telnet

password cisco

login



! NTP (Packet Tracerben opcionális), időzóna

clock timezone CET 1 0



! VLAN-ok (példa: Servers=VLAN 30, Mgmt=VLAN 99)

vlan 30

&nbsp;name SERVERS

vlan 99

&nbsp;name MGMT



! Menedzsment SVI (ha L2 switch és szükséges a táveléréshez)

interface vlan 99

&nbsp;ip address 192.168.99.2 255.255.255.0

&nbsp;no shutdown



! Default gateway a menedzsmenthez (L2 switch esetén)

ip default-gateway 192.168.99.1



! Edge (végpont) portok – PortFast + BPDU Guard a gyors és biztonságos felálláshoz

! Tegyük a szerver/PC portokat VLAN 30-ba és kapcsoljuk be PortFast/BPDU Guardot

interface range fa0/1 - 24

&nbsp;switchport mode access

&nbsp;switchport access vlan 30

&nbsp;spanning-tree portfast

&nbsp;spanning-tree bpduguard enable

! (Ha vannak gig portok a végpontokhoz)

interface range gi0/1 - 2

&nbsp;switchport mode access

&nbsp;switchport access vlan 30

&nbsp;spanning-tree portfast

&nbsp;spanning-tree bpduguard enable



! Uplink a router/ASA/l3 switch felé – TRUNK

! Példa: Fa0/24 legyen trunk az uplinkre

interface fa0/24

&nbsp;switchport trunk encapsulation dot1q

&nbsp;switchport mode trunk

&nbsp;switchport trunk allowed vlan 30,99

&nbsp;spanning-tree portfast trunk

! (ha gig uplink van)

! interface gi0/1

!  switchport trunk encapsulation dot1q

!  switchport mode trunk

!  switchport trunk allowed vlan 30,99

!  spanning-tree portfast trunk



! STP finomhangolás

spanning-tree mode rapid-pvst

spanning-tree extend system-id

spanning-tree portfast default



! DHCP snooping – védelem (csak access VLAN-on engedjük, uplink legyen trusted)

ip dhcp snooping

ip dhcp snooping vlan 30

! Uplink porton trusted (különben eldobja a DHCP-t)

interface fa0/24

&nbsp;ip dhcp snooping trust

! Edge portokon rate limit az alapszintű védelemhez

interface range fa0/1 - 23

&nbsp;ip dhcp snooping limit rate 30



! Alap QoS (opcionális)

mls qos

! (PT-ben kevés hatása van, de jó gyakorlat)



! Logging (opcionális)

logging buffered 100000

end





conf t

hostname WAN-CORE

no ip domain-lookup

service password-encryption



! Line beállítások

line con 0

&nbsp;logging synchronous

&nbsp;exec-timeout 0 0

line vty 0 4

&nbsp;transport input telnet

&nbsp;password cisco

&nbsp;login



! VLAN-ok – ha a WAN-on több logikai VLAN-t használsz (opcionális)

vlan 50

&nbsp;name WAN

vlan 99

&nbsp;name MGMT



! Menedzsment SVI

interface vlan 99

&nbsp;ip address 192.168.99.1 255.255.255.0

&nbsp;no shutdown



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

&nbsp;description UPLINK to Servers Router

&nbsp;switchport mode access

&nbsp;switchport access vlan 50

&nbsp;spanning-tree portfast disable



interface gi1/0/2

&nbsp;description UPLINK to HQ Router

&nbsp;switchport mode access

&nbsp;switchport access vlan 50

&nbsp;spanning-tree portfast disable



interface gi1/0/3

&nbsp;description UPLINK to HR Router

&nbsp;switchport mode access

&nbsp;switchport access vlan 50

&nbsp;spanning-tree portfast disable



! Ha a szerver is közvetlenül a WAN-ra csatlakozik:

interface fa0/5

&nbsp;description Server on WAN VLAN

&nbsp;switchport mode access

&nbsp;switchport access vlan 50

&nbsp;spanning-tree bpduguard enable

! (Server porton PortFast is mehet, ha L2 loop veszély nincs)

spanning-tree portfast edge default



! DHCP snooping – WAN VLAN védelem

ip dhcp snooping

ip dhcp snooping vlan 50

! Router uplinkek TRUSTED

interface gi1/0/1

&nbsp;ip dhcp snooping trust

interface gi1/0/2

&nbsp;ip dhcp snooping trust

interface gi1/0/3

&nbsp;ip dhcp snooping trust



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

interface g0/1            ! WAN switch felé

&nbsp;ip address 192.168.50.1 255.255.255.0

&nbsp;no shutdown

interface g0/0            ! HQ LAN/ASA felé

&nbsp;ip address 10.0.0.1 255.255.255.0

&nbsp;no shutdown

router rip

&nbsp;version 2

&nbsp;no auto-summary

&nbsp;network 10.0.0.0

&nbsp;network 192.168.50.0

&nbsp;passive-interface g0/0

end

wr





