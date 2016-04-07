##Summary
The Garden API makes assumptions that make container networking difficult,
especially for supporting 3rd party network plugins.

We propose that networking be configured immutably at startup and modeled a collection of named drivers.  Specifically:
- At container create time, the Garden client provides to the Garden server an
  ordered list of zero or more named network drivers to execute, along with any config needed for each driver
- Metrics & info about networks can be retrieved by network driver name.

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

type BridgeConfig struct {
  NetIn []PortMapping
  NetOut []NetOutRule
  Limits BandwidthLimits
}
```

- `Name` should be meaningful to the client and is used for subsequent lookups, e.g. "underlay-routable", "overlay", "private", etc.
- `Type` is the name of a network plugin available on the Garden server, e.g. "bridge", "calico", "ducati-vxlan".
- Per-network config can be captured via specialized types like `BridgeConfig`.  Other network types may have other config structs
- The ordering of the slice elements implies an order of operations of invoking the underlying network plugins.
We're trying to follow the [CNI spec](https://github.com/appc/cni/blob/master/SPEC.md) here.

### Look up info by name

**TBD...**

With are multiple networks, all the info & metrics calls should be keyed on network name:

```go
type ContainerInfo struct {
    State         string        
    Events        []string
    ContainerPath string      
    ProcessIDs    []string      
    Properties    Properties
    Networks      map[string]interface{}
}

type BridgeInfo struct {
    ContainerIP        string
    ExternalIP         string
    HostIP             string 
    MappedPorts        []PortMapping 
}
```

```go
type Metrics struct {
    MemoryStat  ContainerMemoryStat
    CPUStat     ContainerCPUStat
    DiskStat    ContainerDiskStat
    NetworkStat map[string]ContainerNetworkStat
}
```

- Also: what about network-related properties?
