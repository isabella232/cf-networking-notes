##Summary
The Garden API contains assumptions that cause problems for pluggable container networking.

We propose to extract all networking functionality from Garden into a separate API server, name to-be-determined.  

This document explains why.  A separate document explains [how we propose to extract it](how-extract-api.md)

##Problem
Both the Garden API and it's current implementation (Guardian) are problematic for a world where a container may be reachable over multiple networks of various types.

### API Limitations
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


### Limitations of Guardian
Although we could address some of the above issues by adding a notion of multiple networks to the existing Garden API, there are additional issues in the current implementation (Guardian) which hampers our ability to support 3rd party networking plugins.

Guardian has partially externalized its own Networking behavior.  Network setup and teardown are implemented via `runc` prestart and poststop hooks.  However, key networking features like `NetIn`, `NetOut`, `Info` and `Metrics` are still implemented in-process, and there are no immediate plans to externalize them.
