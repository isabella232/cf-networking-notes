In a separate document, we explained [why we want to extract networking from Garden](why-extract-api.md).  Here we explain how.

## Proposed approach
A separate OS process, tentatively named `conetd` (container networking daemon), serves an HTTP API on `127.0.0.1`.  It exposes two categories of endpoints.

### Guardian-facing endpoints
Used exclusively Guardian or binaries that Guardian calls.

#### OCI hook generation
Called by Garden to get the prestart and post-stop OCI hooks used for network setup and teardown.
- `GET` to `/oci/hook/:handle`, response looks like

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

Used by current Garden clients (Diego, Concourse, BOSH-lite)

- `POST` to `/containers/:handle/register` with
  ```json
  {
    "networks": {
      "bridge": {
        "some": "config"
      },
      "overlay": {
        "vni": 1234,
        "some": "other config"
      }
    }
  }
  ```
  
  Before creating a container on Garden, a client should `register` the handle with the Network API.

- `PUT` to `/containers/:handle/policy` with
  ```json
  {
    "networks": {
      "bridge": {
        "expose": [ 
          { "host": 0, "container": 0 },
          { "host": 0, "container": 80 }
        ],
        "policies": [
          "system.allow-legacy-db",
          "builtin.deny-private-address-ranges",
          "builtin.allow-all"
        ]
      },
      "overlay": {
        "policies": [
          "builtin.allow-self",
          "builtin.allow-space",
          "builtin.deny-all"
        ]
      }
    }
  }
  ```
  responds with the same data, except any `0`s in the netin spec get replaced with dynamically-allocated ports.
  
  At any time while the container is alive, the client may update policy via a `PUT` to this endpoint.

**TBD: Limits & metrics...**

## Migration plan
1. Build a full network API server, add shims to Guardian to delegate to it
2. Switch the Executor & other Garden consumers over to the new API
3. Remove the shims and remove all networking features from Garden API
