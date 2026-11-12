LEAF31
==========================================================
'''
root@Leaf31# run show configuration | display set                           
set version 26.2R1.7-EVOI20260619095247-evo-builder-1
set system host-name Leaf31
set system root-authentication encrypted-password "$6$pcHSsc3j$OFU3juedo70CfPT4chtvJr.Wmst0lI7AgEhWg6mLzSDp
0AtX4uNdUFl7K4WioeZXjc8gZopxAE6fOAeZVvg280"
set system login user admin uid 2000
set system login user admin class super-user
set system login user admin authentication encrypted-password "$6$o2bP3yUN$Fy67t9GeguTAzfibAX0tGs9omd3/SxCy
cbKWX0EZZJ/AqSfLa8Q1v9V3JJz.Ser2bVlVGkxX9B5umFu7aSz6t0"
set system syslog file interactive-commands interactive-commands any
set system syslog file messages any notice
set system syslog file messages authorization info
set system services ssh
set system packet-forwarding-options forwarding-profile host-profile
set security pki ca-profile ISRG_Root_X1 ca-identity ISRG_Root_X1
set security pki ca-profile ISRG_Root_X1 pre-load
set security pki ca-profile Lets_Encrypt ca-identity Lets_Encrypt
set security pki ca-profile Lets_Encrypt enrollment url https://acme-v02.api.letsencrypt.org/directory
set interfaces et-0/0/1 description To-Spine4
set interfaces et-0/0/1 mtu 9230
set interfaces et-0/0/1 unit 0 family inet address 172.16.221.0/31
set interfaces et-0/0/3 unit 0 family ethernet-switching interface-mode access
set interfaces et-0/0/3 unit 0 family ethernet-switching vlan members VLAN10
set interfaces et-0/0/4 description To-Servers-VLAN10
set interfaces et-0/0/4 unit 0 family ethernet-switching interface-mode access
set interfaces et-0/0/4 unit 0 family ethernet-switching vlan members VLAN10
set interfaces et-0/0/5 description To-Servers-VLAN20
set interfaces et-0/0/5 unit 0 family ethernet-switching interface-mode access
set interfaces et-0/0/5 unit 0 family ethernet-switching vlan members VLAN20
set interfaces irb unit 10 virtual-gateway-accept-data
set interfaces irb unit 10 family inet address 10.0.0.254/24
set interfaces irb unit 10 family inet address 10.0.0.253/24 preferred
set interfaces irb unit 10 family inet address 10.0.0.253/24 virtual-gateway-address 10.0.0.254
set interfaces irb unit 10 virtual-gateway-v4-mac 00:01:00:01:00:01
set interfaces irb unit 20 virtual-gateway-accept-data
set interfaces irb unit 20 family inet address 20.0.0.254/24
set interfaces irb unit 20 family inet address 20.0.0.253/24 preferred
set interfaces irb unit 20 family inet address 20.0.0.253/24 virtual-gateway-address 20.0.0.254
set interfaces irb unit 20 virtual-gateway-v4-mac 00:01:00:01:00:01
set interfaces irb unit 77 family inet
set interfaces lo0 unit 0 family inet address 172.0.0.31/32 primary
set interfaces re0:mgmt-0 unit 0 family inet address 192.168.1.31/24
set forwarding-options tunnel-termination
set policy-options policy-statement ECMP then load-balance per-flow
set policy-options policy-statement v10_accept term 1 from community v10
set policy-options policy-statement v10_accept term 1 then accept
set policy-options policy-statement v10_accept term 2 then reject
set policy-options policy-statement v20_accept term 1 from community v20
set policy-options policy-statement v20_accept term 1 then accept
set policy-options policy-statement v20_accept term 2 then reject
set policy-options community v10 members target:9999:10000
set policy-options community v20 members target:9999:20000
set routing-instances VL10-EVPN-VXLAN instance-type mac-vrf
set routing-instances VL10-EVPN-VXLAN protocols evpn encapsulation vxlan
set routing-instances VL10-EVPN-VXLAN protocols evpn default-gateway do-not-advertise
set routing-instances VL10-EVPN-VXLAN protocols evpn extended-vni-list all
set routing-instances VL10-EVPN-VXLAN protocols evpn multicast-mode ingress-replication
set routing-instances VL10-EVPN-VXLAN protocols evpn vni-options vni 10000 vrf-target target:9999:10000
set routing-instances VL10-EVPN-VXLAN protocols evpn vni-options vni 770000 vrf-target target:7777:770000
set routing-instances VL10-EVPN-VXLAN vtep-source-interface lo0.0
set routing-instances VL10-EVPN-VXLAN service-type vlan-based
set routing-instances VL10-EVPN-VXLAN interface et-0/0/3.0
set routing-instances VL10-EVPN-VXLAN interface et-0/0/4.0
set routing-instances VL10-EVPN-VXLAN route-distinguisher 65003:10000
set routing-instances VL10-EVPN-VXLAN vrf-import v20_accept
set routing-instances VL10-EVPN-VXLAN vrf-import v10_accept
set routing-instances VL10-EVPN-VXLAN vrf-target target:9999:10000
set routing-instances VL10-EVPN-VXLAN vlans VLAN10 vlan-id 10
set routing-instances VL10-EVPN-VXLAN vlans VLAN10 l3-interface irb.10
set routing-instances VL10-EVPN-VXLAN vlans VLAN10 vxlan vni 10000
set routing-instances VL20-EVPN-VXLAN instance-type mac-vrf
set routing-instances VL20-EVPN-VXLAN protocols evpn encapsulation vxlan
set routing-instances VL20-EVPN-VXLAN protocols evpn default-gateway do-not-advertise
set routing-instances VL20-EVPN-VXLAN protocols evpn extended-vni-list all
set routing-instances VL20-EVPN-VXLAN protocols evpn multicast-mode ingress-replication
set routing-instances VL20-EVPN-VXLAN protocols evpn vni-options vni 20000 vrf-target target:9999:20000
set routing-instances VL20-EVPN-VXLAN protocols evpn vni-options vni 770000 vrf-target target:7777:770000
set routing-instances VL20-EVPN-VXLAN vtep-source-interface lo0.0
set routing-instances VL20-EVPN-VXLAN service-type vlan-based
set routing-instances VL20-EVPN-VXLAN interface et-0/0/5.0
set routing-instances VL20-EVPN-VXLAN route-distinguisher 65003:20000
set routing-instances VL20-EVPN-VXLAN vrf-import v10_accept
set routing-instances VL20-EVPN-VXLAN vrf-import v20_accept
set routing-instances VL20-EVPN-VXLAN vrf-target target:9999:20000
set routing-instances VL20-EVPN-VXLAN vlans VLAN20 vlan-id 20
set routing-instances VL20-EVPN-VXLAN vlans VLAN20 l3-interface irb.20
set routing-instances VL20-EVPN-VXLAN vlans VLAN20 vxlan vni 20000
set routing-instances VRF-A instance-type vrf
set routing-instances VRF-A protocols evpn irb-symmetric-routing vni 770000
set routing-instances VRF-A protocols evpn ip-prefix-routes advertise direct-nexthop
set routing-instances VRF-A protocols evpn ip-prefix-routes encapsulation vxlan
set routing-instances VRF-A protocols evpn ip-prefix-routes vni 770000
set routing-instances VRF-A vtep-source-interface lo0.0
set routing-instances VRF-A interface irb.10
set routing-instances VRF-A interface irb.20
set routing-instances VRF-A interface irb.77
set routing-instances VRF-A route-distinguisher 65003:7777
set routing-instances VRF-A vrf-target target:7777:770000
set routing-instances VRF-A vrf-table-label
set routing-options router-id 172.0.0.31
set routing-options autonomous-system 65003
set protocols bgp group FABRIC-SPINE type internal
set protocols bgp group FABRIC-SPINE local-address 172.0.0.31
set protocols bgp group FABRIC-SPINE family evpn signaling
set protocols bgp group FABRIC-SPINE multipath
set protocols bgp group FABRIC-SPINE neighbor 172.0.0.114 peer-as 65003
set protocols ospf area 0.0.0.0 interface lo0.0
set protocols ospf area 0.0.0.0 interface et-0/0/1.0 interface-type p2p
set protocols lldp port-id-subtype interface-name
set protocols lldp interface all
set switch-options vtep-source-interface lo0.0
'''

SPINE4
=======================================================
'''
root@Spine4# run show configuration | display set 
set version 26.2R1.7-EVOI20260619095247-evo-builder-1
set system host-name Spine4
set system root-authentication encrypted-password "$6$DoOL3CAi$CcDpNeOf5Ma4Wxbn7s9ucbTHhV3ciGevZDg5YSs44NsD7j51tOx5mjP12ihksY72.pzFE7r3Ff0M.k29IiuCY0"
set system syslog file interactive-commands interactive-commands any
set system syslog file messages any notice
set system syslog file messages authorization info
set security pki ca-profile ISRG_Root_X1 ca-identity ISRG_Root_X1
set security pki ca-profile ISRG_Root_X1 pre-load
set security pki ca-profile Lets_Encrypt ca-identity Lets_Encrypt
set security pki ca-profile Lets_Encrypt enrollment url https://acme-v02.api.letsencrypt.org/directory
set interfaces et-0/0/1 description To-Leaf31
set interfaces et-0/0/1 mtu 9230
set interfaces et-0/0/1 unit 0 family inet address 172.16.221.1/31
set interfaces et-0/0/4 description To-SuperSpine
set interfaces et-0/0/4 mtu 9230
set interfaces et-0/0/4 unit 0 family inet address 172.16.114.1/31
set interfaces irb unit 77 family inet
set interfaces lo0 unit 0 family inet address 172.0.0.114/32 primary
set forwarding-options tunnel-termination
set policy-options policy-statement EVPN-NH term 1 from family evpn
set policy-options policy-statement EVPN-NH term 1 from next-hop 100.0.0.1
set policy-options policy-statement EVPN-NH term 1 then next-hop 172.0.0.114
set policy-options policy-statement EVPN-NH term 1 then accept
set policy-options policy-statement EVPN-NH term 2 from family evpn
set policy-options policy-statement EVPN-NH term 2 from next-hop 100.0.0.2
set policy-options policy-statement EVPN-NH term 2 then next-hop 172.0.0.114
set policy-options policy-statement EVPN-NH term 2 then accept
set policy-options policy-statement EVPN-NH term 3 then accept
set policy-options policy-statement HRI term 1 from protocol evpn
set policy-options policy-statement HRI term 1 from route-filter 0.0.0.0/0 prefix-length-range /32-/32
set policy-options policy-statement HRI term 1 then accept
set policy-options policy-statement HRI term 2 then reject
set policy-options policy-statement VRF-EXPORT-DCI-FILTER term 1 from family evpn
set policy-options policy-statement VRF-EXPORT-DCI-FILTER term 1 from next-hop 100.0.0.1
set policy-options policy-statement VRF-EXPORT-DCI-FILTER term 1 from next-hop 100.0.0.2
set policy-options policy-statement VRF-EXPORT-DCI-FILTER term 1 from nlri-route-type 2
set policy-options policy-statement VRF-EXPORT-DCI-FILTER term 1 then reject
set policy-options policy-statement VRF-EXPORT-DCI-FILTER term 2 then accept
set policy-options community DCI-ORIG-111 members origin:172.0.0.111:512
set policy-options community DCI-ORIG-112 members origin:172.0.0.112:512
set policy-options community DCI-ORIG-113 members origin:172.0.0.113:512
set policy-options community L2VNI-10 members target:9999:10000
set policy-options community L2VNI-20 members target:9999:20000
set routing-instances VL10-EVPN-VXLAN instance-type mac-vrf
set routing-instances VL10-EVPN-VXLAN protocols evpn encapsulation vxlan
set routing-instances VL10-EVPN-VXLAN protocols evpn default-gateway no-gateway-community
set routing-instances VL10-EVPN-VXLAN protocols evpn extended-vni-list all
set routing-instances VL10-EVPN-VXLAN protocols evpn interconnect vrf-target target:9999:10000
set routing-instances VL10-EVPN-VXLAN protocols evpn interconnect route-distinguisher 172.0.0.14:10000
set routing-instances VL10-EVPN-VXLAN protocols evpn interconnect domain-path-id 65003:2:evpn
set routing-instances VL10-EVPN-VXLAN protocols evpn interconnect esi 00:00:44:44:44:44:44:44:44:44
set routing-instances VL10-EVPN-VXLAN protocols evpn interconnect esi all-active
set routing-instances VL10-EVPN-VXLAN protocols evpn interconnect interconnected-vni-list 10000
set routing-instances VL10-EVPN-VXLAN protocols evpn vni-options vni 10000 vrf-target target:9999:10000
set routing-instances VL10-EVPN-VXLAN vtep-source-interface lo0.0
set routing-instances VL10-EVPN-VXLAN service-type vlan-based
set routing-instances VL10-EVPN-VXLAN route-distinguisher 172.0.0.114:10
set routing-instances VL10-EVPN-VXLAN vrf-export VRF-EXPORT-DCI-FILTER
set routing-instances VL10-EVPN-VXLAN vrf-target target:9999:10000
set routing-instances VL10-EVPN-VXLAN vlans VLAN10 vlan-id 10
set routing-instances VL10-EVPN-VXLAN vlans VLAN10 vxlan vni 10000
set routing-instances VL10-EVPN-VXLAN vlans VLAN10 vxlan ingress-node-replication
set routing-instances VL20-EVPN-VXLAN instance-type mac-vrf
set routing-instances VL20-EVPN-VXLAN protocols evpn encapsulation vxlan
set routing-instances VL20-EVPN-VXLAN protocols evpn default-gateway no-gateway-community
set routing-instances VL20-EVPN-VXLAN protocols evpn extended-vni-list all
set routing-instances VL20-EVPN-VXLAN protocols evpn interconnect vrf-target target:9999:20000
set routing-instances VL20-EVPN-VXLAN protocols evpn interconnect route-distinguisher 172.0.0.114:20000
set routing-instances VL20-EVPN-VXLAN protocols evpn interconnect domain-path-id 65003:2:evpn
set routing-instances VL20-EVPN-VXLAN protocols evpn interconnect esi 00:00:44:44:44:44:44:44:44:44
set routing-instances VL20-EVPN-VXLAN protocols evpn interconnect esi all-active
set routing-instances VL20-EVPN-VXLAN protocols evpn interconnect interconnected-vni-list 20000
set routing-instances VL20-EVPN-VXLAN protocols evpn vni-options vni 20000 vrf-target target:9999:20000
set routing-instances VL20-EVPN-VXLAN vtep-source-interface lo0.0
set routing-instances VL20-EVPN-VXLAN service-type vlan-based
set routing-instances VL20-EVPN-VXLAN route-distinguisher 172.0.0.114:20
set routing-instances VL20-EVPN-VXLAN vrf-export VRF-EXPORT-DCI-FILTER
set routing-instances VL20-EVPN-VXLAN vrf-target target:9999:20000
set routing-instances VL20-EVPN-VXLAN vlans VLAN20 vlan-id 20
set routing-instances VL20-EVPN-VXLAN vlans VLAN20 vxlan vni 20000
set routing-instances VL20-EVPN-VXLAN vlans VLAN20 vxlan ingress-node-replication
set routing-instances VRF-A instance-type vrf
set routing-instances VRF-A protocols evpn irb-symmetric-routing vni 770000
set routing-instances VRF-A protocols evpn interconnect vrf-target target:7777:770000
set routing-instances VRF-A protocols evpn interconnect route-distinguisher 172.0.0.114:100
set routing-instances VRF-A protocols evpn interconnect domain-path-id 65003:2:evpn
set routing-instances VRF-A protocols evpn interconnect encapsulation vxlan
set routing-instances VRF-A interface irb.77
set routing-instances VRF-A route-distinguisher 172.0.0.114:77
set routing-instances VRF-A vrf-target target:7777:770000
set routing-options domain-path-id 65003:1:evpn
set routing-options router-id 172.0.0.114
set routing-options autonomous-system 65003
set routing-options forwarding-table chained-composite-next-hop ingress evpn
set protocols bgp group FABRIC-LEAF type internal
set protocols bgp group FABRIC-LEAF local-address 172.0.0.114
set protocols bgp group FABRIC-LEAF family evpn signaling
set protocols bgp group FABRIC-LEAF export EVPN-NH
set protocols bgp group FABRIC-LEAF neighbor 172.0.0.31
set protocols bgp group DCI type external
set protocols bgp group DCI multihop ttl 5
set protocols bgp group DCI local-address 172.0.0.114
set protocols bgp group DCI family inet unicast
set protocols bgp group DCI family evpn signaling
set protocols bgp group DCI peer-as 65254
set protocols bgp group DCI local-as 65003
set protocols bgp group DCI domain-path-id send
set protocols bgp group DCI domain-path-id receive
set protocols bgp group DCI neighbor 172.0.0.254
set protocols bgp group DCI vpn-apply-export
set protocols ospf area 0.0.0.0 interface lo0.0 passive
set protocols ospf area 0.0.0.0 interface et-0/0/4.0 interface-type p2p
set protocols ospf area 0.0.0.0 interface et-0/0/1.0 interface-type p2p
set protocols lldp port-id-subtype interface-name
set protocols lldp interface all
set switch-options vtep-source-interface lo0.0
'''



