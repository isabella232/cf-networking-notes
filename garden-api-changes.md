##Summary
The Garden API contains assumptions that cause problems for pluggable container networking.

We propose that networking be configured immutably at startup and modeled as a collection of named drivers:
- At container create time, the Garden client provides to the Garden server an
  ordered list of zero or more named network plugins to execute, along with any config needed for each
- Metrics & info about networks can be retrieved by network plugin name.

##Problem
In the magical world of pluggable container networking, a container may end up with more than one 
network interface (below, left).  Alternately, the multiple network interfaces may be hidden in a 
dedicated “routing” network namespace, while the container only sees a single network interface (below, right).

![diagram](interface-topology.png)


In both cases, the Garden API poorly models the underlying reality.
The following parts of the API assume a single network interface, and/or that the interface may be
configured with port-mapping:

- Inputs / ContainerSpec
  - Subnet specification in Network field
  - Limits -> Bandwidth
  - NetIn (Exposing ports into the container)
  - NetOut (Allowing outbound traffic)
- Outputs
  - Metrics -> NetworkStat
  - Info (ContainerInfo struct)
    - ExternalIP
    - HostIP
    - ContainerIP
    - MappedPorts

## Proposed solution

### Configuration at creation
All configuration of networking happens at container creation time, via a new `NetworkSpecs` field on the ContainerSpec struct:
```go
type ContainerSpec struct {
  ...
  NetworkSpecs []NetworkSpec
}

type NetworkSpec struct {
  Name string
  Type string
  Config interface{}
}
```


- `Name` should be meaningful to the client and is used for subsequent lookups for metrics & info, e.g. "underlay-routable", "overlay", "private", etc.
- `Type` is the name of a network plugin available on the Garden server, e.g. "bridge", "calico", "ducati-vxlan".
- Per-network config can be captured via specialized types, e.g.  
  ```go
  type BridgeConfig struct {
    Subnet string
    NetIn  []PortMapping
    NetOut []NetOutRule
    Limits BandwidthLimits
  }
  ```
  or
  ```go
  type VXLANConfig struct {
    VNI      int
    PolicyID string
    Limits   BandwidthLimits
  }
  ```
  
- A client may specify multiple networks of the same `Type` as long as they have distinct `Name`s
- The ordering of the slice elements implies an order of operations of invoking the underlying network plugins.
We're trying to follow the [CNI spec](https://github.com/appc/cni/blob/master/SPEC.md) here.

### Look up metrics and info by name

Metrics are available separately for each network
```go
type Metrics struct {
    MemoryStat  ContainerMemoryStat
    CPUStat     ContainerCPUStat
    DiskStat    ContainerDiskStat
    NetworkStat map[string]ContainerNetworkStat  // keyed by network name
}
```

Similarly for Info:
```go
type ContainerInfo struct {
    State         string        
    Events        []string
    ContainerPath string      
    ProcessIDs    []string      
    Properties    Properties
    Network       map[string]interface{}
}
```

We will have custom `Info` structs for each network `Type`:
```go
type BridgeInfo struct {
    ContainerIP        string
    ExternalIP         string
    HostIP             string 
    MappedPorts        []PortMapping 
}

type VXLANInfo struct {
    ContainerIP        string
    ContainerMAC       string
    HostIP             string
    VNI                int
}
```

## TBD:
Garden properties?

Config or metrics not tied to a particular plugin, e.g.
- Container IP (in the single-interface-per-container world)   
- default route
- aggregate bandwidth limit

Maybe we reserve a "network" name, e.g. `global` or `root` for that stuff?

Or maybe every collection changes to also contain a "global" field, e.g.:
```go
type AllNetworkSpecs struct {
  Global NetworkGlobalSpec
  Networks []NetworkSpec
}
```
and
```go
type NetworkStat struct {
  Global NetworkGlobalStat
  Networks map[string]ContainerNetworkStat
}
```
