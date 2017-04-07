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

vtepDeviceName="silk-vtep"
overlayIP="10.255.32.0"
overlayMAC="ee:ee:0a:ff:20:00" # based on overlay ip
vni="42" # don't use 1 because it will conflict with flannel at the moment
underlayDeviceName="$(ip route | grep default | awk '{print $5}')"  # the name of the underlay interface
underlayIP="$(ip addr list dev $underlayDeviceName | grep inet | awk '{print $2}' | head -n 1 | cut -d '/' -f1)" # this should be the underlay ip of your diego cell
destPort=4789 # this is the VXLAN standard, ref: https://tools.ietf.org/html/rfc7348

# create vtep, associate with underlay ip & vni & port
# example:
# ip link add silk-vtep address ee:ee:0a:ff:20:00 type vxlan id 42 local 10.244.0.145 dev wbiqee52f3p8-1 dstport 8888 nolearning gbp
ip link add $vtepDeviceName address $overlayMAC type vxlan id $vni local $underlayIP dev $underlayDeviceName dstport $destPort nolearning gbp

ip link set $vtepDeviceName up

ip -d link list $vtepDeviceName # to see the newly-created device

# assign overlay identity to vtep
ip addr add $overlayIP dev $vtepDeviceName

ip -d addr list $vtepDeviceName # to see newly assigned address

## Next steps:
# - disable IPv6?
# - add arp, fdb, & ip-routing rules
```

### References

- [Silk prototype `backend` package](https://github.com/cloudfoundry-incubator/cf-networking-release/tree/silk-v2/src/silk/backend)
