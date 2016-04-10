##Summary
The Garden API contains assumptions that cause problems for pluggable container networking.

We propose to extract all networking functionality from Garden into a separate API server, name to-be-determined.  

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

## Proposed solution

A separate OS process, `cnetd` (container networking daemon), serves an HTTP API on `127.0.0.1`.  It has three sets of endpoints.

### OCI hook endpoints
Used exclusively by the OCI hook binary to drive network setup and teardown.

- `POST` to `/oci/hook/prestart` with
  ```json
  {
    "handle": "some-garden-container-handle",
    "spec": "some-network-spec",
    "pid": 12345
  }
  ```
  
- `POST` to `/oci/hook/poststop` with
  ```json
  {
    "handle": "some-garden-container-handle",
    "spec": "some-network-spec"
  }
  ```

### Garden legacy compatibility endpoints
Supports backwards compatibility with "legacy" Garden networking API calls.  Will be deprecated and removed once clients migrate to the new Network API

- `POST` to `/garden/containers/:handle/net/in` with
  ```json
  {
    "host_port": 54321,
    "container_port": 8080
  }
  ```

- `POST` to `/garden/containers/:handle/net/out` with a `garden.NetOutRule`

## Migration plan
1. Build a full network API server, add shims to Guardian to delegate to it
2. Switch the Executor & other Garden consumers over to the new API
3. Remove the shims and remove all networking features from Garden API
