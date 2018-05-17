# Prototyping Silk cross-host connectivity

Some notes from exploring this on a bosh-lite diego-cell:

```bash
#!/bin/bash

set -e -x
set -o pipefail

# get a more recent version of iproute2 so we can do stuff with GBP
curl -o /tmp/iproute2.deb -L http://mirrors.kernel.org/ubuntu/pool/main/i/iproute2/iproute2_4.3.0-1ubuntu3_amd64.deb
curl -o /tmp/libmnl0.deb -L http://mirrors.kernel.org/ubuntu/pool/main/libm/libmnl/libmnl0_1.0.3-5_amd64.deb
dpkg -i /tmp/*.deb


ipToMACAddr() {
	printf "ee:ee:%02x:%02x:%02x:%02x\n" $(echo $1 | cut -d '.' -f1) $(echo $1 | cut -d '.' -f2) $(echo $1 | cut -d '.' -f3) $(echo $1 | cut -d '.' -f4)
}

getIPForDevice() {
	deviceName="$1"
	ip addr list dev $deviceName | grep inet | awk '{print $2}' | head -n 1 | cut -d '/' -f1
}

setup_vtep() {
	overlayIP="$1" # e.g. 10.255.32.0

	vtepDeviceName="silk-vtep"
	overlayMAC=$(ipToMACAddr $overlayIP)
	vni="42" # don't use 1 because it will conflict with flannel at the moment
	underlayDeviceName="$(ip route | grep default | awk '{print $5}')"  # the name of the underlay interface
	underlayIP="$(getIPForDevice $underlayDeviceName)"
	destPort=4789 # this is the VXLAN standard, ref: https://tools.ietf.org/html/rfc7348

	# create vtep, associate with underlay ip & vni & port
	# example:
	# ip link add silk-vtep address ee:ee:0a:ff:20:00 type vxlan id 42 local 10.244.0.145 dev wbiqee52f3p8-1 dstport 8888 nolearning gbp
	ip link add $vtepDeviceName address $overlayMAC type vxlan id $vni local $underlayIP dev $underlayDeviceName dstport $destPort nolearning gbp

	ip link set $vtepDeviceName up

	# ip -d link list $vtepDeviceName # to see the newly-created device

	# assign overlay identity to vtep
	ip addr add $overlayIP/16 dev $vtepDeviceName

	ip -d addr list $vtepDeviceName # to see newly assigned address
}

add_remote_host() {
	remoteVTEPOverlayIP="$1" # e.g. 10.255.48.0
	remoteUnderlayIP="$2" # e.g. 10.244.0.48

	vtepDeviceName="silk-vtep"
	remoteVTEPOverlayMAC=$(ipToMACAddr $remoteVTEPOverlayIP)
	remoteOverlaySubnet="$remoteVTEPOverlayIP/24"
	localVTEPOverlayIP="$(getIPForDevice $vtepDeviceName)"

	echo "adding route to remote subnet"
	# upsert ip route: remoteSubnet via remoteVTEPOverlayIP
	# overlay L3 subnet --> overlay L3 router for that subnet
	ip route add $remoteOverlaySubnet via $remoteVTEPOverlayIP dev $vtepDeviceName src $localVTEPOverlayIP scope global
	# e.g. ip route add 10.255.2.0/24 via 10.255.2.0 dev silk.1

	echo "adding ARP rule"
	# upsert arp rule: remoteVTEPOverlayIP --> remoteVTEPOverlayMAC
	# overlay router L3 --> overlay router L2
	ip neigh replace to $remoteVTEPOverlayIP dev $vtepDeviceName lladdr $remoteVTEPOverlayMAC

	echo "adding FDB rule"
	# upsert fdb rule: remoteVTEPOverlayMAC --> remoteUnderlayIP
	# overlay router L2 --> underlay router L3
	bridge fdb replace $remoteVTEPOverlayMAC dev $vtepDeviceName dst $remoteUnderlayIP
}
```

e.g. on host A (underlay 10.244.0.144)
```
setup_vtep 10.255.144.0
add_remote_host 10.255.138.0 10.244.0.138
```

and on host B (underlay 10.244.0.138)
```
setup_vtep 10.255.138.0
add_remote_host 10.255.144.0 10.244.0.144
```

We then see
```bash
ip neigh list dev silk-vtep
# 10.255.144.0 lladdr ee:ee:0a:ff:90:00 PERMANENT # one per remote host

bridge fdb list dev silk-vtep
# ee:ee:0a:ff:90:00 dst 10.244.0.144 self permanent # one per remote host

ip route list dev silk-vtep
# 10.255.0.0/16  proto kernel  scope link  src 10.255.138.0 # this is added automatically by ip addr add
# 10.255.144.0/24 via 10.255.144.0  src 10.255.138.0 # one per remote host
```

With this setup, it is now possible to ping across hosts, e.g. on `10.255.138.0` I can reach `10.255.144.0`

Then follow the instructions in [silk-cni.md](silk-cni.md) to set up containers on each host.  At that point, we can

Host 1:
```
ip netns exec dragonfruit nc -l -p 9999
```

Host 2:
```
echo hello from apricot | nc $ipOfDragonfruit 9999
```


### References

- [Silk prototype `backend` package](https://github.com/cloudfoundry-incubator/cf-networking-release/tree/silk-v2/src/silk/backend)
