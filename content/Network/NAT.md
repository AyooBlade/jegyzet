NAT (Network Address Translation)

Topics:
[[#Nat basics]]  
[[#Static NAT]]  
[[#Dynamic NAT]]  
[[#PAT (Port Address Translation)]]  
[[#NAT & PAT comparison]]  
[[#Advantages & Disadvantages]]  
[[#Configure Static NAT]]  
[[#Configure Dynamic NAT]]  

## Why?

The transition to IPv6-only networks is still ongoing.  

## Nat basics
NAT provides the translation of private addresses to public addresses.  

![[Pasted image 20260209180109.png]]

This allows a device with private IPv4 address to access resources outside of their private   network. NAT, combined with private IPV4 addresses, has been the primary method of   preserving public IPV4 addresses. One public IPV4 address can be shared by hundreds of  devices, each configurated with unique private IPV4 address.  

==Security==: It hides internal IPv4 addresses from outside networks.  

==NAT pool==: NAT-enabled routers configurated public IPV4 addresses.  

NAT-enabled router translates the internal IPv4 address of the device to have a public IPv4 address from the provided pool of addresses.  

Configure the NAT on the border router always.  

![[Pasted image 20260209181901.png]]


==Four types of addresses:==  
	Inside local address  
	Inside global address  
	Outside local address  
	Outside global address  

**Inside address:** The address of the device which is being translated by NAT.  
**Outside address:** The address of the destination device  
**Local address:** Any address that appears on the inside portion of the network.  
**Global address:** Any address that appears on the outside portion of the network.  


## Types of NAT

### Static NAT

Uses a one-to-one mapping of local and global addresses.  

![[Pasted image 20260209185205.png]]

It's useful for web servers or devices that must have a consistent address that is accessible from the internet.  


### Dynamic NAT

Uses a pool of public addresses and assigns them on a first-come, first-served basics.

![[Pasted image 20260209185648.png]]


### PAT (Port Address Translation)

NAT overload.  
Maps multiple private IPv4 addresses to a single public IPv4 address or a few.
(*Your home router does this too*)  

Each private address is also tracked by a port number.  
When a NAT router receives a packet from the client, it uses its source port number to uniquely identify the specific NAT translation.  

PAT ensures that devices use a different TCP port number for each session with a server on the internet. When a response comes back from the server, the source port, which becomes the destination port number on the return trip, determines to which device the router forwards the packets.  

![[Pasted image 20260209191533.png]]

#### Next Available Port

The router picks the next free port number and sends the traffic using it, while  remembering which internal device it belongs to.  
When the reply comes back on that port, the router delivers it to the correct device.  

![[Pasted image 20260209192638.png]]

### NAT & PAT comparison

| NAT                                                                                           | PAT                                                                                                    |
| --------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| One-to-one mapping between Inside Local and Inside Global addresses                           | One Inside Global address can be mapped to many Inside Local addresses                                 |
| Uses only IPv4 addresses in translation process                                               | Uses IPv4 addresses and TCP or UDP source port numbers in translation process                          |
| A unique Inside Global address is required for each inside host accessing the outside network | A single unique Inside Global address can be shared by many inside hosts accessing the outside network |

## Advantages & Disadvantages

#### Advantages:

With NAT overload (PAT), internal hosts can share a single public IPv4 address for all external communications.    

NAT increases the flexibility of connections to the public network. Multiple pools, backup pools, and load-balancing pools can be implemented to ensure reliable public network connections.  

#### Disadvantages:

Network performance slower because the translation of each IPv4 address within the packet headers takes time.  

==Carrier Grade NAT (CGN)==  
As public IPv4 addresses run out, ISPs increasingly give customers private IPv4 addresses instead of public ones.  
This cause traffic to be translated twice: once on the customer's router and again inside the ISP's network.  
This adds delay and complexity to connections.  

Applications that use physical addresses, instead of a qualified domain name, don't reach destinations that are translated across the NAT router.  

It becomes much more difficult to trace packets that undergo numerous packet address changes over multiple NAT hops, making troubleshooting challenging.  

Nat also complicates the use of tunneling protocols, such as IPsec, because NAT modifies values in the headers.  



## Configure
### Configure Static NAT:

##### Configure

1) Create a mapping between the inside local address and the inside global addresses.  

```
R2(config)# ip nat inside source static 192.168.10.254 209.165.201.5
```

2) Interfaces participating in the translation are configured as inside or outside relative to NAT.   

```
R2(config)# interface serial 0/1/0
R2(config-if)# ip address 192.168.1.2 255.255.255.252
R2(config-if)# ip nat inside
R2(config-if)# exit
R2(config)# interface serial 0/1/1
R2(config-if)# ip address 209.165.200.1 255.255.255.252
R2(config-if)# ip nat outside
```


##### Verify

```
R2# show ip nat translations
```
```
Pro  Inside global       Inside local       Outside local     Outside global
---  209.165.201.5       192.168.10.254     ---               ---
Total number of translations: 1
```


```
R2# show ip nat statistics
```
```
Total active translations: 1 (1 static, 0 dynamic; 0 extended)
Outside interfaces:
  Serial0/1/1
Inside interfaces:
  Serial0/1/0
Hits: 4  Misses: 1
**(output omitted)**
```


### Configure Dynamic NAT

##### Configure

1) Define the pool of addresses.  
   Netmask indicates which address bits belong to the network and which bits belong to the host for that range of addresses.  

```
R2(config)# ip nat pool NAT-POOL1 209.165.200.226 209.165.200.240 netmask 255.255.255.224
```

2) Configure a standard [[ACL]] to identify (permit) only those addresses that are to be translated.   

```
R2(config)# access-list 1 permit 192.168.0.0 0.0.255.255
```

3) Bind the ACL to the pool.
   how: Router(config)# **ip nat inside source list** {*access-list-number | access-list-name*} **pool** *pool-name*  

```
R2(config)# ip nat inside source list 1 pool NAT-POOL1
```

4) configure the inside interface  

```
R2(config)# interface serial 0/1/0
R2(config-if)# ip nat inside
```

5) configure the outside interface  

```
R2(config)# interface serial 0/1/1
R2(config-if)# ip nat outside
```

##### Verify

```
R2# show ip nat translations

Pro Inside global      Inside local       Outside local      Outside global
--- 209.165.200.228    192.168.10.10      ---                ---
--- 209.165.200.229    192.168.11.10      ---                ---
R2#
```


```
R2# show ip nat translation verbose

Pro Inside global      Inside local       Outside local      Outside global
tcp 209.165.200.228    192.168.10.10      ---                ---
    create 00:02:11, use 00:02:11 timeout:86400000, left 23:57:48, Map-Id(In): 1,
    flags:
none, use_count: 0, entry-id: 10, lc_entries: 0
tcp 209.165.200.229    192.168.11.10      ---                ---
    create 00:02:10, use 00:02:10 timeout:86400000, left 23:57:49, Map-Id(In): 1,
    flags:
none, use_count: 0, entry-id: 12, lc_entries: 0
R2#
```

```
R2# clear ip nat translation *
```


```
R2# show ip nat statistics

Total active translations: 4 (0 static, 4 dynamic; 0 extended)
Peak translations: 4, occurred 00:31:43 ago
Outside interfaces:
  Serial0/1/1
Inside interfaces:
  Serial0/1/0
Hits: 47  Misses: 0
CEF Translated packets: 47, CEF Punted packets: 0
Expired translations: 5
Dynamic mappings:
-- Inside Source
[Id: 1] access-list 1 pool NAT-POOL1 refcount 4
pool NAT-POOL1: netmask 255.255.255.224
start 209.165.200.226 end 209.165.200.240
type generic, total addresses 15, allocated 2 (13%), misses 0
(output omitted)
R2#
```


```
R2# show running-config | include NAT

ip nat pool NAT-POOL1 209.165.200.226 209.165.200.240 netmask 255.255.255.224
ip nat inside source list 1 pool NAT-POOL1
```
