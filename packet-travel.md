


Application container connecting to another application container on a different host (diego cell)

1. Application sends from container to destination
2. Default route sends traffic via 169.254.0.1  through the "eth0" device inside the container
3. ARP is disabled, but a permanent neighbor rule associates the 169.254.0.1 with the host MAC address (beginning with `aa:aa:`)
4. Packet has source IP matching eth0 device of the container
5. Packet reaches the host side of the veth device
6. Route for the matching destination subnet has the VTEP device of the source and destination host
7. Neighbor rule for the destination VTEP resolves it's MAC address
8. ARP rule for the destination MAC resolves it's underlay IP address (the IP address of eth0 on the destination host)
9. VTEP encapulates traffic with the destination 


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

## Neighbor rule for VTEP

The neighbor rule for the destination VTEP IP address resolves it's MAC address,
which is based on it's IP with an `ee:ee:` prefix:

```
ip neigh show to 10.255.43.0

# output
10.255.43.0 dev silk-vtep lladdr ee:ee:0a:ff:2b:00 PERMANENT
```

## ARP rule for the destination MAC address

The ARP rule for the destination MAC address resolves the destination host's underlay IP address:

```
bridge fdb show | grep ee:ee:0a:ff:2b:00

# output
ee:ee:0a:ff:2b:00 dev silk-vtep dst 10.0.16.14 self permanent
```

## VTEP device

The

```bash
ip -d link silk-vtep

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
