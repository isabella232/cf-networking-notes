In a separate document, we explained [why we want to extract networking from Garden](why-extract-api.md).  Here we explain how.

## Proposed approach
A separate OS process, tentatively named `conetd` (container networking daemon), serves an HTTP API on a UNIX socket.  It exposes two categories of endpoints.

### Guardian-facing endpoints
Used exclusively Guardian or binaries that Guardian calls.

#### OCI hook generation
Called by Garden to get the prestart and post-stop OCI hooks used for network setup and teardown.
- `POST` to `/oci/hook` with a Garden `ContainerSpec`.  The handler will utilize the `ContainerSpec` `Properties` hash key `network`.  The response is a set of OCI hooks, e.g.

  ```json
  
  "hooks" : {
        "prestart": [
            {
                "path": "/var/vcap/packages/ducati/bin/oci-hook",
                "args": [ "oci-hook", "--handle=some-container-handle", "--action=up", "--vni=12345"],
                "env":  [ "key1=value1" ]
            },
        ],
        "poststop": [
            {
                "path": "/var/vcap/packages/ducati/bin/oci-hook",
                "args": ["oci-hook", "--handle=some-container-handle", "--action=down" ]
            }
        ]
    }
  ```
  
#### Garden legacy compatibility endpoints
Supports backwards compatibility with "legacy" Garden networking API calls.  Will be deprecated and removed once clients migrate to the new networking API (see below)

- `POST` to `/garden/containers/:handle/net/in`, same semantics as Garden NetIn.

- `POST` to `/garden/containers/:handle/net/out` with a `garden.NetOutRule`.  Same semantics as Garden NetOut.



### "User-facing" Network API
Used by current Garden clients (Diego, Concourse, BOSH-lite) to define networks & policy.

- `POST` to `/networks` with
  ```json
  {
      "name": "my-overlay",
      "type": "vxlan",
      "config": {
        "some": "config"
      }
  }
  ```

  To create a container that will be attached to this network, the Garden client must set the `network` property on a `ContainerSpec` to be an array of networks -- the container will be attached to each of them.
  
- `PUT` to `/containers/:handle/policy` with
  ```json
  {
    "networks": {
      "my-bridge-network": {
        "some": "policy document"
      },
      "my-overlay": {
        "policies": [
          { "something": "tbd" }
        ]
      }
    }
  }
  ```

  At any time while the container is alive, the client may update policy via a `PUT` to this endpoint.
  
## The evolution of network policy

In order to move quickly, we should let the network policy language remain mostly "legacy" for now, and slowly evolve it towards something better.  I'd rather not block the extraction effort on the perfection of the policy language.

Therefore, the first iteration of this extraction effort will have the `bridge` policy document closely resemble existing Garden `NetIn` and `NetOut` policy documents.  We can then evolve towards something better.

**TBD: Limits & metrics...**

## Migration plan
1. Build a full network API server, add shims to Guardian to delegate to it
2. Switch the Executor & other Garden consumers over to the new API
3. Remove the shims and remove all networking features from Garden API
