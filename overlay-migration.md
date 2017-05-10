# Changing the overlay network with zero downtime

Exploring allowing operators to have zero downtime when redeploying with a different network.
Currently, when the network changes, the existing cells will not be able to add routes to the new cell.

As a first step to allow operators to change the network, we can have the cell
filter out routes to subnets that are not reachable (subnet is not within the
overlay network configured locally).  This is not a zero downtime migration,
because the cells on the old overlay network will not be able to reach the
cells on the new overlay network, and vice-versa, during the deploy.

In the future, if we want to support a zero downtime migration, we can have the
cells program routes for muliple networks.

We have not yet fully explored how to store or distribute the state when
multiple networks exist, but here are some notes for how the routes can be
programmed on the cell:

Instead of using the subnet prefix length of the overlay network for the VTEP's overlay IP address,
we will use a /32 for the IP address and manually add the route(s) for the overlay network.

```
ip route add $overlayNetwork dev silk-vtep  proto kernel  scope link  src $localVTEPIP
```

When there are multiple networks, there will be a route to each.


For example, here is how you can set up routes to cells on two networks:

```
# local cell
underlayIP=10.244.0.146
localVTEPIP=10.255.39.0

# overlay network 1
overlayNetwork=10.255.0.0/16

# remote cell on overlay network 1
remoteSubnet=10.255.209.0/24
remoteVTEPIP=10.255.209.0
remoteVTEPMAC=ee:ee:0a:ff:d1:00
remoteUnderlayIP=10.244.0.139

ip route add $overlayNetwork dev silk-vtep  proto kernel  scope link  src $localVTEPIP
ip route add $remoteSubnet via $remoteVTEPIP dev silk-vtep src $localVTEPIP scope global
ip neigh replace to $remoteVTEPIP dev silk-vtep lladdr $remoteVTEPMAC
bridge fdb replace $remoteVTEPMAC dev silk-vtep dst $remoteUnderlayIP

# overlay network 2
overlayNetwork=10.254.0.0/16

# remote cell on overlay network 2
remoteSubnet=10.254.209.0/24
remoteVTEPIP=10.254.209.0
remoteVTEPMAC=ee:ee:0a:fe:d1:00
remoteUnderlayIP=10.244.0.139

ip route add $overlayNetwork dev silk-vtep  proto kernel  scope link  src $localVTEPIP
ip route add $remoteSubnet via $remoteVTEPIP dev silk-vtep src $localVTEPIP scope global
ip neigh replace to $remoteVTEPIP dev silk-vtep lladdr $remoteVTEPMAC
bridge fdb replace $remoteVTEPMAC dev silk-vtep dst $remoteUnderlayIP
```

After this, the routing tables look as follows:

```
ip route list dev silk-vtep
# 10.254.0.0/16  proto kernel  scope link  src 10.255.39.0
# 10.254.209.0/24 via 10.254.209.0  src 10.255.39.0
# 10.255.0.0/16  proto kernel  scope link  src 10.255.39.0
# 10.255.209.0/24 via 10.255.209.0  src 10.255.39.0

ip neigh list dev silk-vtep
# 10.255.209.0 lladdr ee:ee:0a:ff:d1:00 PERMANENT
# 10.254.209.0 lladdr ee:ee:0a:fe:d1:00 PERMANENT

bridge fdb show dev silk-vtep
# ee:ee:0a:fe:d1:00 dst 10.244.0.139 self permanent
# ee:ee:0a:ff:d0:00 dst 10.244.0.138 self permanent
# ee:ee:0a:ff:d1:00 dst 10.244.0.139 self permanent
```
