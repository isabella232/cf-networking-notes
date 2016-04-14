In a separate document, we explained [why we want to extract networking from Garden](why-extract-api.md).  Here we explain how.

## Proposed approach
A separate OS process, tentatively named `conetd` (container networking daemon), serves an HTTP API on a UNIX socket.  It exposes two categories of endpoints -- one used by "clients" (e.g. Diego, Concourse), and another used by Guardian itself.


### "User-facing" Network API
Used by current Garden clients (Diego, Concourse, BOSH-lite) to define networks & policy.

Before creating a container, we define one or more networks to which it may be attached:

- `POST` to `/networks` with

  ```json
  {
      "name": "my-overlay",
      "type": "vxlan",
      "config": {
        "some": "config"
      },
      "default_policy": {
        "some": "policy document"
      }
  }
  ```

Then, define a container's network config:
- `PUT` to `/containers/:handle/config` with
```json
{
  "attach_to": [
    "my-bridge",
    "my-overlay"
  ],
  "more": "config here..."
}
```

Then create the container via the Garden API as usual.  Guardian will configure OCI hooks similarly to how it does today: it sets the hook binary, passing the container handle and `action` (`up` or `down`) as flags.  The hook binary will be a simple HTTP client to a private endpoint on `conetd`, passing in the container handle and action, enabling `conetd` to do the appropriate work using the already-created `config`.

  
At any time while the container is alive, the client may update policy on a particular container:
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

### Guardian-facing endpoints

**Garden legacy compatibility endpoints**
Supports backwards compatibility with "legacy" Garden networking API calls.  Will be deprecated and removed once clients fully migrate to the new networking API (see below)

- `POST` to `/garden/containers/:handle/net/in`, same semantics as Garden NetIn.

- `POST` to `/garden/containers/:handle/net/out` with a `garden.NetOutRule`.  Same semantics as Garden NetOut.



  
## The evolution of network policy

In order to move quickly, we should let the network policy language remain mostly "legacy" for now, and slowly evolve it towards something better.  I'd rather not block the extraction effort on the perfection of the policy language.

Therefore, the first iteration of this extraction effort will have the `bridge` policy document closely resemble existing Garden `NetIn` and `NetOut` policy documents.  We can then evolve towards something better.

**TBD: Limits & metrics...**

## Migration plan
1. Build a full network API server, add shims to Guardian to delegate to it
2. Switch the Executor & other Garden consumers over to the new API
3. Remove the shims and remove all networking features from Garden API
