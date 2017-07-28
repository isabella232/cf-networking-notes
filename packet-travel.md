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

Create policy to allow connections on port 7777

```
cf allow-access proxy proxy --protocol tcp --port 7777
```

Start netcat listener on destination app instance

```
cf ssh proxy -i 1
nc -l 7777
```

Open connection on source app instance

```
cf ssh proxy -i 0
nc 10.255.132.2 7777
```

Start up tcpdump sessions on all devices in the pathway.

`bosh ssh` to both diego cells and do the following:

```
# host eth0 device on port 4789
tcpdump -T vxlan -v -XX -i eth0 dst port 4789

# silk-vtep device on port 4789
tcpdump -T vxlan -v -XX -i eth0 dst port 4789

# create a symlink for the network namespace of the container
mkdir /var/run/netns
ln -s /var/vcap/garden-cni/container-netns/$handle /var/run/netns/s-010255132002

# then you can listen on the container eth0 device on port 7777
ip netns exec s-010255132002 tcpdump -v -XX -i eth0 dst port 7777
```

From within the source app instance ssh session, send a message within the open netcat connection:

```
cf-networking with silk!
```

Hit enter.

### Source container eth0 device

```
diego-cell/c9d07dce-66a5-4f0a-88cc-b11cf205a138:~# ip netns exec s-010255099003 tcpdump -v -XX -i eth0 dst port 7777
tcpdump: listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
^C03:51:59.607085 IP (tos 0x0, ttl 64, id 39442, offset 0, flags [DF], proto TCP (6), length 77)
    10.255.99.3.56766 > 10.255.132.2.7777: Flags [P.], cksum 0xfd42 (incorrect -> 0x0fc9), seq 2356689519:2356689544, ack 2161316842, win 215, options [nop,nop,TS val 530639 ecr 679117], length 25
	0x0000:  aaaa 0aff 6303 eeee 0aff 6303 0800 4500  ....c.....c...E.
	0x0010:  004d 9a12 4000 4006 a395 0aff 6303 0aff  .M..@.@.....c...
	0x0020:  8402 ddbe 1e61 8c78 3a6f 80d3 13ea 8018  .....a.x:o......
	0x0030:  00d7 fd42 0000 0101 080a 0008 18cf 000a  ...B............
	0x0040:  5ccd 6366 2d6e 6574 776f 726b 696e 6720  \.cf-networking.
	0x0050:  7769 7468 2073 696c 6b21 0a              with.silk!.
```

### Source host VTEP device

```
diego-cell/c9d07dce-66a5-4f0a-88cc-b11cf205a138:~# tcpdump -T vxlan -v -XX -i silk-vtep
tcpdump: listening on silk-vtep, link-type EN10MB (Ethernet), capture size 262144 bytes
03:51:59.607108 IP (tos 0x0, ttl 63, id 39442, offset 0, flags [DF], proto TCP (6), length 77)
    10.255.99.3.56766 > 10.255.132.2.7777: Flags [P.], cksum 0xfd42 (incorrect -> 0x0fc9), seq 2356689519:2356689544, ack 2161316842, win 215, options [nop,nop,TS val 530639 ecr 679117], length 25
	0x0000:  eeee 0aff 8400 eeee 0aff 6300 0800 4500  ..........c...E.
	0x0010:  004d 9a12 4000 3f06 a495 0aff 6303 0aff  .M..@.?.....c...
	0x0020:  8402 ddbe 1e61 8c78 3a6f 80d3 13ea 8018  .....a.x:o......
	0x0030:  00d7 fd42 0000 0101 080a 0008 18cf 000a  ...B............
	0x0040:  5ccd 6366 2d6e 6574 776f 726b 696e 6720  \.cf-networking.
	0x0050:  7769 7468 2073 696c 6b21 0a              with.silk!.
```

### Source host eth0 device

```
diego-cell/c9d07dce-66a5-4f0a-88cc-b11cf205a138:~# tcpdump -T vxlan -v -XX -i eth0 dst port 4789
tcpdump: listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
03:51:59.607140 IP (tos 0x0, ttl 64, id 37823, offset 0, flags [none], proto UDP (17), length 127)
    diego-cell-0.node.dc1.cf.internal.36821 > diego-cell-1.node.dc1.cf.internal.4789: VXLAN, flags [I] (0x88), vni 1
IP (tos 0x0, ttl 63, id 39442, offset 0, flags [DF], proto TCP (6), length 77)
    10.255.99.3.56766 > 10.255.132.2.7777: Flags [P.], cksum 0x0fc9 (correct), seq 2356689519:2356689544, ack 2161316842, win 215, options [nop,nop,TS val 530639 ecr 679117], length 25
	0x0000:  4201 0a00 0001 4201 0a00 1010 0800 4500  B.....B.......E.
	0x0010:  007f 93bf 0000 4011 a291 0a00 1010 0a00  ......@.........
	0x0020:  200e 8fd5 12b5 006b 0000 8800 0001 0000  .......k........
	0x0030:  0100 eeee 0aff 8400 eeee 0aff 6300 0800  ............c...
	0x0040:  4500 004d 9a12 4000 3f06 a495 0aff 6303  E..M..@.?.....c.
	0x0050:  0aff 8402 ddbe 1e61 8c78 3a6f 80d3 13ea  .......a.x:o....
	0x0060:  8018 00d7 0fc9 0000 0101 080a 0008 18cf  ................
	0x0070:  000a 5ccd 6366 2d6e 6574 776f 726b 696e  ..\.cf-networkin
	0x0080:  6720 7769 7468 2073 696c 6b21 0a         g.with.silk!.
```

### Destination host eth0 device

```
diego-cell/7e9e15d5-454e-482e-a6f0-fc51e36b8941:~# tcpdump -T vxlan -v -XX -i eth0 dst port 4789
tcpdump: listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
03:51:59.622806 IP (tos 0x0, ttl 64, id 37823, offset 0, flags [none], proto UDP (17), length 127)
    diego-cell-0.node.dc1.cf.internal.36821 > diego-cell-1.node.dc1.cf.internal.4789: VXLAN, flags [I] (0x88), vni 1
IP (tos 0x0, ttl 63, id 39442, offset 0, flags [DF], proto TCP (6), length 77)
    10.255.99.3.56766 > 10.255.132.2.7777: Flags [P.], cksum 0x0fc9 (correct), seq 2356689519:2356689544, ack 2161316842, win 215, options [nop,nop,TS val 530639 ecr 679117], length 25
	0x0000:  4201 0a00 200e 4201 0a00 0001 0800 4500  B.....B.......E.
	0x0010:  007f 93bf 0000 4011 a291 0a00 1010 0a00  ......@.........
	0x0020:  200e 8fd5 12b5 006b 0000 8800 0001 0000  .......k........
	0x0030:  0100 eeee 0aff 8400 eeee 0aff 6300 0800  ............c...
	0x0040:  4500 004d 9a12 4000 3f06 a495 0aff 6303  E..M..@.?.....c.
	0x0050:  0aff 8402 ddbe 1e61 8c78 3a6f 80d3 13ea  .......a.x:o....
	0x0060:  8018 00d7 0fc9 0000 0101 080a 0008 18cf  ................
	0x0070:  000a 5ccd 6366 2d6e 6574 776f 726b 696e  ..\.cf-networkin
	0x0080:  6720 7769 7468 2073 696c 6b21 0a         g.with.silk!.
```

### Destination host VTEP device

```
diego-cell/7e9e15d5-454e-482e-a6f0-fc51e36b8941:~# tcpdump -T vxlan -v -XX -i silk-vtep
tcpdump: listening on silk-vtep, link-type EN10MB (Ethernet), capture size 262144 bytes
03:51:59.622887 IP (tos 0x0, ttl 63, id 39442, offset 0, flags [DF], proto TCP (6), length 77)
    10.255.99.3.56766 > 10.255.132.2.7777: Flags [P.], cksum 0x0fc9 (correct), seq 2356689519:2356689544, ack 2161316842, win 215, options [nop,nop,TS val 530639 ecr 679117], length 25
	0x0000:  eeee 0aff 8400 eeee 0aff 6300 0800 4500  ..........c...E.
	0x0010:  004d 9a12 4000 3f06 a495 0aff 6303 0aff  .M..@.?.....c...
	0x0020:  8402 ddbe 1e61 8c78 3a6f 80d3 13ea 8018  .....a.x:o......
	0x0030:  00d7 0fc9 0000 0101 080a 0008 18cf 000a  ................
	0x0040:  5ccd 6366 2d6e 6574 776f 726b 696e 6720  \.cf-networking.
	0x0050:  7769 7468 2073 696c 6b21 0a              with.silk!.
```

### Destination container eth0 device

```
diego-cell/7e9e15d5-454e-482e-a6f0-fc51e36b8941:~# ip netns exec s-010255132002 tcpdump -v -XX -i eth0 dst port 7777
tcpdump: listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
^C03:51:59.622918 IP (tos 0x0, ttl 62, id 39442, offset 0, flags [DF], proto TCP (6), length 77)
    10.255.99.3.56766 > 10.255.132.2.7777: Flags [P.], cksum 0x0fc9 (correct), seq 2356689519:2356689544, ack 2161316842, win 215, options [nop,nop,TS val 530639 ecr 679117], length 25
	0x0000:  eeee 0aff 8402 aaaa 0aff 8402 0800 4500  ..............E.
	0x0010:  004d 9a12 4000 3e06 a595 0aff 6303 0aff  .M..@.>.....c...
	0x0020:  8402 ddbe 1e61 8c78 3a6f 80d3 13ea 8018  .....a.x:o......
	0x0030:  00d7 0fc9 0000 0101 080a 0008 18cf 000a  ................
	0x0040:  5ccd 6366 2d6e 6574 776f 726b 696e 6720  \.cf-networking.
	0x0050:  7769 7468 2073 696c 6b21 0a              with.silk!.
```
