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
A separate OS process, `cnetd` (container networking daemon), serves an HTTP API on `127.0.0.1`.  It exposes two categories of endpoints.


### Guardian-facing endpoints
Used exclusively Guardian or binaries that Guardian calls.

#### OCI hook endpoints
Called by the OCI hook binary to setup and teardown the network.
- `POST` to `/oci/hook/prestart` with
  ```json
  {
    "handle": "some-garden-container-handle",
    "spec": "some-network-spec",
    "pid": 12345
  }
  ```
  Will bind-mount the `pid`'s network namespace and then trigger network setup for the container.  That will execute any pre-configured CNI plugins with the `ADD` command.
  
- `POST` to `/oci/hook/poststop` with
  ```json
  {
    "handle": "some-garden-container-handle",
    "spec": "some-network-spec"
  }
  ```
  Will drive pre-configured CNI plugins with the `DEL` command, and then delete the network namespace bind-mount.
  
#### Garden legacy compatibility endpoints
Supports backwards compatibility with "legacy" Garden networking API calls.  Will be deprecated and removed once clients migrate to the new networking API (see below)

- `POST` to `/garden/containers/:handle/net/in`, same semantics as Garden NetIn.

- `POST` to `/garden/containers/:handle/net/out` with a `garden.NetOutRule`.  Same semantics as Garden NetOut.



### "User" facing Network API
- `POST` to `/containers/:handle/register` with
  ```json
  {
    "handle": "some-garden-container-handle",
    "networks": {
      "bridge": {},
      "overlay": {
        ... // some config
      }
    }
  }
  ```
  Before creating a container on Garden, a client should `register` the handle with the Network API.


## Migration plan
1. Build a full network API server, add shims to Guardian to delegate to it
2. Switch the Executor & other Garden consumers over to the new API
3. Remove the shims and remove all networking features from Garden API
