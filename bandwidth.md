### Goal:

* We want to shape both (container ingress / host egress) and (container egress
  / host ingress) traffic.

### Given:

* Linux can properly throttle egress traffic. Throttling delays packet traffic
  but does not drop packet traffic. Throttling is desired over dropping. We can
  easily shape (container ingress / host egress) on the host veth pair in the
  host namespace.
* Linux generally can't directly perform traffic shaping on ingress data
  without dropping packets, and we cannot touch the container namespace. We
  need to find a way to shape i.e. throttle (container egress / host ingress)
  without touching the container.

### Test setup:
```
cf push proxy -i 2
cf allow-access proxy proxy --protocol tcp --port 9000

# discover IP address of sink
cf ssh proxy -i 0 -c 'ip addr'
sink_ip=10.255.247.3

# set up sink to receive in a loop
cf ssh proxy -i 0 -c 'while true; do nc -l 9000 > /dev/null; done'

# set up source to send in a loop
cf ssh proxy -i 1 -c "while true; do time cat /dev/urandom | head -c 20000 | nc $sink_ip 9000; done"

# observe that source sends full message very quickly, "real" time is < 0.01 seconds

bosh ssh diego-cell  # to cell holding the source container
sudo su

export network_host_iface=s-010255247002 # network device of source container

export network_ifb_iface=test-ifb
export container_iface_mtu=1450

# setup ifb device for traffic shaping
ip link add ${network_ifb_iface} type ifb
ip link set ${network_ifb_iface} netns 1
ifconfig ${network_ifb_iface} mtu ${container_iface_mtu}
ip link set ${network_ifb_iface} up

tc qdisc add dev ${network_host_iface} ingress handle ffff:
tc filter add dev ${network_host_iface} parent ffff: protocol all u32 match ip src 0.0.0.0/0 action mirred egress redirect dev ${network_ifb_iface}

# to throttle egress from the source container
export RATE=100000
export BURST=100000
tc qdisc add dev ${network_ifb_iface} root tbf rate ${RATE}bit burst ${BURST} latency 100ms

export RATE=80000
export BURST=80000
tc qdisc add dev ${network_ifb_iface} root tbf rate ${RATE}bit burst ${BURST} latency 100ms

export RATE=160000
export BURST=160000
tc qdisc add dev ${network_ifb_iface} root tbf rate ${RATE}bit burst ${BURST} latency 100ms

# to throttle ingress to the source container
# stop the source loop, and instead just test via
cf ssh proxy -i 0 -c 'curl http://httpbin.org/stream/100000'

# and then control ingress bandwidth from the host diego cell via
tc qdisc add dev ${network_host_iface} root tbf rate ${RATE}bit burst ${BURST} latency 100ms
```

### Explaination:

* For container ingress (host egress to container), we can easily limit with the normal
  throttling of egress traffic from the host.
  Traffic then looks like: `outside -> host eth0 -> L3 forwarding via host router -> host-side container veth -> container eth0`


* For container egress (host ingress from container):
  Sykes' solution is to use an intermediate IFB device to properly shape
  (container egress / host ingress) traffic. We mirror host ingress as ifb
  egress, then throttle ifb egress.


  Silk today: container eth0 -> host-side container veth -> L3 forwarding via host router -> eth0 -> outside

  Silk + bandwidth controls: container eth0 -> host-side container veth -> ifb (as egress) throttled -> L3 forwarding via host router -> eth0 -> outside

[tc](http://man7.org/linux/man-pages/man8/tc.8.html)
[tc filter ...](http://man7.org/linux/man-pages/man8/tc-u32.8.html)
[tc ... action mirred](http://man7.org/linux/man-pages/man8/tc-mirred.8.html)
