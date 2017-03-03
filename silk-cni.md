Sketching out the behavior for a Silk CNI plugin

*Inspired by [Nyantec Narwhal](https://github.com/nyantec/narwhal)*

```
set -e -x -u
export namespace=dragonfruit
export id=dragonfruit
export container_ip=10.255.30.5

export container_interface=eth0
export host_interface=s-$id
export host_ip=169.254.0.1

ip link add $host_interface type veth peer name eth0

host_mac_addr="$(cat "/sys/class/net/$host_interface/address")"
container_mac_addr="$(cat "/sys/class/net/eth0/address")"

ip link set $host_interface arp off # disable arp
ip link set eth0 arp off # disable arp

sysctl -q -w "net.ipv4.conf.${host_interface}.rp_filter=1" # enable reverse-path filtering

ip link set eth0 netns $namespace # move eth0 into container
ip link set $host_interface up
ip -4 addr add $host_ip peer $container_ip/32 scope link dev $host_interface # set up PtP link on host
ip -4 neighbour replace $container_ip lladdr $container_mac_addr nud permanent dev $host_interface

sysctl -q -w "net.ipv6.conf.${host_interface}.disable_ipv6=1"
sysctl -q -w "net.ipv4.conf.${host_interface}.forwarding=1"  # enable host forwarding of packets

ip netns exec $namespace bash -c "
  set -e -x -u
  ip link set eth0 up
	ip -4 address add $container_ip peer $host_ip/32 dev eth0  # set up PtP inside container
	ip -4 neighbour replace $host_ip lladdr $host_mac_addr nud permanent dev eth0  # permanent ARP rule
	ip -4 route add default via $host_ip dev eth0  # default route to host-veth
	sysctl -q -w "net.ipv6.conf.eth0.disable_ipv6=1"
"
```
