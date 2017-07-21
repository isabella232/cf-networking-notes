


Application container connecting to another application container on a different host (diego cell)

1. Application sends from container to destination
2. Default route sends traffic via 169.254.0.1  through the "eth0" device inside the container
3. ARP is disabled, but a permanent neighbor rule associates the 169.254.0.1 with the host MAC address (beginning with `aa:aa:`)
4. Packet has source IP matching eth0 device of the container
5. Packet reaches the host side of the veth device
6. Route for the matching destination subnet has the VTEP device of the source and destination host
7. ARP rule for the destination VTEP resolves it's MAC address
8. FDB rule for the destination MAC resolves to the outgoing VTEP device and destination underlay IP address (the IP address of eth0 on the destination host)
9. VTEP encapulates traffic with the destination and sends it as UDP on port 4789 to the destination underlay IP
10. On the destination VM, the packet is received on eth0, port 4789


## Inside the Source Application Container

### Default route

From inside the container:

```bash
ip route

# output
default via 169.254.0.1 dev eth0  src 10.255.108.184
169.254.0.1 dev eth0  proto kernel  scope link  src 10.255.108.184
```

### ARP is disabled

From inside the container:

```bash
ip link show dev eth0

# output
4054: eth0@if4055: <BROADCAST,MULTICAST,NOARP,UP,LOWER_UP> mtu 1410 qdisc noqueue state UP mode DEFAULT group default
    link/ether ee:ee:0a:ff:6c:b8 brd ff:ff:ff:ff:ff:ff
```

MAC address of container side of veth pair is based on the container's IP address, with prefix `ee:ee`.

(Note "NOARP")

### Permanent neighbor rule

From inside the container:

```bash
ip neigh show

# output
169.254.0.1 dev eth0 lladdr aa:aa:0a:ff:6c:b8 PERMANENT
```

MAC address of host side of veth pair is based on the container's IP address, with prefix `aa:aa`.

### IP Address for eth0

```bash
ip addr show dev eth0

# output
4054: eth0@if4055: <BROADCAST,MULTICAST,NOARP,UP,LOWER_UP> mtu 1410 qdisc noqueue state UP group default
    link/ether ee:ee:0a:ff:6c:b8 brd ff:ff:ff:ff:ff:ff
    inet 10.255.108.184 peer 169.254.0.1/32 scope link eth0
       valid_lft forever preferred_lft forever<Paste>
```

## On the Host of the Source Container

## Veth device

Veth device is named based on the Container IP address.
In this example, 10.255.108.184 translates to s-010255108184

```bash
ip addr show dev s-010255108184

# output
4055: s-010255108184@if4054: <BROADCAST,MULTICAST,NOARP,UP,LOWER_UP> mtu 1410 qdisc noqueue state UP group default
    link/ether aa:aa:0a:ff:6c:b8 brd ff:ff:ff:ff:ff:ff
    inet 169.254.0.1 peer 10.255.108.184/32 scope link s-010255108184
       valid_lft forever preferred_lft forever
```

## Route for destination

The route is in the form:

```bash
$destinationOverlaySubnet via $destinationVTEPOverlayIP dev silk-vtep  src $sourceVTEPOverlayIP
```

In this example, the destination is another application container with IP address: 10.255.43.6
A matching route for the destination cell's subnet will be determined based on the the configured `subnet_prefix_length`.
For example, with the default `subnet_prefix_length` of `/24`, the route will be for `10.255.43.0`:

```bash
ip route list 10.255.43.0/24

# output
10.255.43.0/24 via 10.255.43.0 dev silk-vtep  src 10.255.108.0
```

## ARP rule for VTEP

The ARP rule for the destination VTEP IP address determines the destination
which MAC address that will be written into the ethernet frame for packets that
are detined to that IP address:

```
ip neigh show to 10.255.43.0

# output
10.255.43.0 dev silk-vtep lladdr ee:ee:0a:ff:2b:00 PERMANENT
```

## FDB rule for the destination MAC address

The FDB rule for the destination MAC address resolves the destination host's underlay IP address,
and the device name of the VXLAN Tunnel Endpoint (VTEP) outgoing interface (`silk-vtep` in this case).

```
bridge fdb show | grep ee:ee:0a:ff:2b:00

# output
ee:ee:0a:ff:2b:00 dev silk-vtep dst 10.0.16.14 self permanent
```

## VTEP device

The

```bash
ip -d link show silk-vtep

# output
865: silk-vtep: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1410 qdisc noqueue state UNKNOWN mode DEFAULT group default
    link/ether ee:ee:0a:f4:7e:00 brd ff:ff:ff:ff:ff:ff promiscuity 0
    vxlan id 1 local 10.0.32.12 dev eth0 srcport 0 0 dstport 4789 nolearning ageing 300 gbp
```

Note: you may need a more recent version of `iproute` to see that GBP is enabled:

```bash
curl -o /tmp/iproute2.deb -L http://mirrors.kernel.org/ubuntu/pool/main/i/iproute2/iproute2_4.3.0-1ubuntu3_amd64.deb
curl -o /tmp/libmnl0.deb -L http://mirrors.kernel.org/ubuntu/pool/main/libm/libmnl/libmnl0_1.0.3-5_amd64.deb
dpkg -i /tmp/*.deb
```












## Packet capture from all devices for a request across hosts



### Source sends packet on port 4789

```
# tcpdump -T vxlan -v -XX -i eth0 dst port 4789
23:29:09.212241 IP (tos 0x0, ttl 64, id 57022, offset 0, flags [none], proto UDP (17), length 110)
    diego-cell-0.node.dc1.cf.internal.37151 > diego-cell-1.node.dc1.cf.internal.4789: VXLAN, flags [I] (0x88), vni 1
IP (tos 0x0, ttl 63, id 56269, offset 0, flags [DF], proto TCP (6), length 60)
    10.255.45.3.58492 > 10.255.115.40.http-alt: Flags [S], cksum 0x7b1c (correct), seq 1911844672, win 27400, options [mss 1370,sackOK,TS val 306033805 ecr 0,nop,wscale 7], length 0
	0x0000:  4201 0a00 0001 4201 0a00 100f 0800 4500  B.....B.......E.
	0x0010:  006e debe 0000 4011 57a5 0a00 100f 0a00  .n....@.W.......
	0x0020:  200d 911f 12b5 005a 0000 8800 0001 0000  .......Z........
	0x0030:  0100 eeee 0aff 7300 eeee 0aff 2d00 0800  ......s.....-...
	0x0040:  4500 003c dbcd 4000 3f06 a9c5 0aff 2d03  E..<..@.?.....-.
	0x0050:  0aff 7328 e47c 1f90 71f4 6f40 0000 0000  ..s(.|..q.o@....
	0x0060:  a002 6b08 7b1c 0000 0204 055a 0402 080a  ..k.{......Z....
	0x0070:  123d b48d 0000 0000 0103 0307            .=..........
```

### Destination receives packet on port 4789

```
# tcpdump -T vxlan -v -XX -i eth0 dst port 4789
23:29:09.211784 IP (tos 0x0, ttl 64, id 57022, offset 0, flags [none], proto UDP (17), length 110)
    diego-cell-0.node.dc1.cf.internal.37151 > diego-cell-1.node.dc1.cf.internal.4789: VXLAN, flags [I] (0x88), vni 1
IP (tos 0x0, ttl 63, id 56269, offset 0, flags [DF], proto TCP (6), length 60)
    10.255.45.3.58492 > 10.255.115.40.http-alt: Flags [S], cksum 0x7b1c (correct), seq 1911844672, win 27400, options [mss 1370,sackOK,TS val 306033805 ecr 0,nop,wscale 7], length 0
	0x0000:  4201 0a00 200d 4201 0a00 0001 0800 4500  B.....B.......E.
	0x0010:  006e debe 0000 4011 57a5 0a00 100f 0a00  .n....@.W.......
	0x0020:  200d 911f 12b5 005a 0000 8800 0001 0000  .......Z........
	0x0030:  0100 eeee 0aff 7300 eeee 0aff 2d00 0800  ......s.....-...
	0x0040:  4500 003c dbcd 4000 3f06 a9c5 0aff 2d03  E..<..@.?.....-.
	0x0050:  0aff 7328 e47c 1f90 71f4 6f40 0000 0000  ..s(.|..q.o@....
	0x0060:  a002 6b08 7b1c 0000 0204 055a 0402 080a  ..k.{......Z....
	0x0070:  123d b48d 0000 0000 0103 0307            .=..........
```

### Decapsulated packet on silk-vtep device

```
# tcpdump -T vxlan -v -XX -i silk-vtep
23:31:18.729654 IP (tos 0x0, ttl 63, id 56083, offset 0, flags [DF], proto TCP (6), length 60)
    10.255.45.3.58582 > 10.255.115.40.http-alt: Flags [S], cksum 0xd1d5 (correct), seq 596305947, win 27400, options [mss 1370,sackOK,TS val 306066184 ecr 0,nop,wscale 7], length 0
	0x0000:  eeee 0aff 7300 eeee 0aff 2d00 0800 4500  ....s.....-...E.
	0x0010:  003c db13 4000 3f06 aa7f 0aff 2d03 0aff  .<..@.?.....-...
	0x0020:  7328 e4d6 1f90 238a e81b 0000 0000 a002  s(....#.........
	0x0030:  6b08 d1d5 0000 0204 055a 0402 080a 123e  k........Z.....>
	0x0040:  3308 0000 0000 0103 0307                 3.........
```


