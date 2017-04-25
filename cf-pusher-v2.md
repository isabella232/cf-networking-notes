# Thoughts on a better cf-pusher
**using the CC API v3**

In order to run scaling tests efficiently, we need to automate
the create of a large number of applications.  Here are the current
test parameters for our scaling tests:

```
{
    "proxy_instances": 2,
    "proxy_applications": 100,
    "test_app_instances": 2,
    "test_applications": 100,
    "test_app_registry_ttl_seconds": 180,
    "policy_update_wait_seconds": 10,
    "extra_listen_ports": 2,
    "sample_percent":20,
    "prefix":"scale-",
    "concurrency": 20,
    "asg_size": 5
}
```

The most interesting ones are the number of proxy and tick applications and instances.

The number of applications and the number of instances per application need to be
scaled indepedently to test different aspects of our system.

While we want to be able to have N "tick" applications, for example,
all of these applications share the same bits, so it would be natural to use
a single droplet rather than pushing the bits N different times.


## Workflow overview

The goal is a declarative "convergence" of the live system to a desired state.  To converge, we:

- destroy and recreate the org
- for each app in [proxy, tick, registry]
  - create a "prototype" app: `cf push --no-start $app -i 1`
  - discover the droplet ID of the prototype
  - for i := 1 to N-1 desired apps:
    - create a new app object
    - copy the droplet of the prototype to the new app
    - set the copied droplet as the current droplet for the app
    - create a route
    - create a route mapping to the app
    - scale the app (or do this in a separate control loop)
    - start the app


## API details

to get current droplet for app:

```
cf curl /v3/apps/d4cf0f68-d498-4f33-bc25-d6cd37d3cbe8/relationships/current_droplet
```

returns
```json
{
   "data": {
      "guid": "9144ac1e-1d57-453f-9045-99b72be8397d"
   },
   "links": {
      "self": {
         "href": "https://api.bosh-lite.com/v3/apps/d4cf0f68-d498-4f33-bc25-d6cd37d3cbe8/relationships/current_droplet"
      },
      "related": {
         "href": "https://api.bosh-lite.com/v3/apps/d4cf0f68-d498-4f33-bc25-d6cd37d3cbe8/droplets/current"
      }
   }
}
```

to discover the space for that app:
```
cf curl /v3/apps/d4cf0f68-d498-4f33-bc25-d6cd37d3cbe8
```
returns json including

```json
{
   "guid": "d4cf0f68-d498-4f33-bc25-d6cd37d3cbe8",
   "links": {
      "space": {
         "href": "https://api.bosh-lite.com/v2/spaces/ad6ac153-7464-478a-b1f4-5e8159f4e2be"
      }
    }
}
```

to create a new app with the API:

```
cf curl /v3/apps -X POST -d '{
  "name": "synthetic-proxy",
  "relationships": {
    "space": { "data": { "guid": "ad6ac153-7464-478a-b1f4-5e8159f4e2be" } }
  },
  "environment_variables": {
    "GOPACKAGENAME": "example-apps/proxy"
  }
}'
```
the response will include json like
```json
{
   "guid": "060e418b-a4a9-4144-a3c3-4dac66d0e47a",
   "name": "synthetic-proxy",
   "state": "STOPPED",
   "created_at": "2017-04-25T20:53:11Z",
   "updated_at": "2017-04-25T20:53:11Z",
   "lifecycle": {
      "type": "buildpack",
      "data": {
         "buildpacks": [],
         "stack": "cflinuxfs2"
      }
   }
 }
```

to copy the first app's droplet into the new app:

```
cf curl /v3/droplets/9144ac1e-1d57-453f-9045-99b72be8397d/copy -X POST -d '{
  "relationships": { "app": { "guid": "060e418b-a4a9-4144-a3c3-4dac66d0e47a" } }
}'
```

the response includes

```json
{
   "guid": "43151587-2ba7-45a6-9b26-d8288f6844a1",
   "state": "COPYING"
}
```

then set that new droplet as the current droplet for the app:

```
cf curl v3/apps/060e418b-a4a9-4144-a3c3-4dac66d0e47a/relationships/current_droplet -X PATCH -d '{
  "data": { "guid": "43151587-2ba7-45a6-9b26-d8288f6844a1" }
}'
```

then start the app
```
cf curl /v3/apps/060e418b-a4a9-4144-a3c3-4dac66d0e47a/start -X PUT
```

this creates a process for the app.


to list domains (so we can find the apps domain, a.k.a. first shared domain):
```
cf curl /v2/shared_domains
```
returns
```json
{
   "total_results": 1,
   "total_pages": 1,
   "prev_url": null,
   "next_url": null,
   "resources": [
      {
         "metadata": {
            "guid": "55e0e895-e0bf-407f-bcdd-f0818aee834f",
            "url": "/v2/shared_domains/55e0e895-e0bf-407f-bcdd-f0818aee834f",
            "created_at": "2017-04-18T23:19:27Z",
            "updated_at": "2017-04-18T23:19:27Z"
         },
         "entity": {
            "name": "bosh-lite.com",
            "router_group_guid": null,
            "router_group_type": null
         }
      }
   ]
}
```

to create a route
```
cf curl /v2/routes -X POST -d '{
  "domain_guid": "55e0e895-e0bf-407f-bcdd-f0818aee834f",
  "space_guid": "ad6ac153-7464-478a-b1f4-5e8159f4e2be",
  "host": "synthetic-proxy"
}'
```
responds with
```json
{
   "metadata": {
      "guid": "de5400dc-e167-4eac-ae67-60fd55005f91",
      "url": "/v2/routes/de5400dc-e167-4eac-ae67-60fd55005f91",
      "created_at": "2017-04-25T21:08:41Z",
      "updated_at": "2017-04-25T21:08:41Z"
   },
   "entity": {
      "host": "synthetic-proxy",
      "path": "",
      "domain_guid": "55e0e895-e0bf-407f-bcdd-f0818aee834f",
      "space_guid": "ad6ac153-7464-478a-b1f4-5e8159f4e2be",
      "service_instance_guid": null,
      "port": null,
      "domain_url": "/v2/shared_domains/55e0e895-e0bf-407f-bcdd-f0818aee834f",
      "space_url": "/v2/spaces/ad6ac153-7464-478a-b1f4-5e8159f4e2be",
      "apps_url": "/v2/routes/de5400dc-e167-4eac-ae67-60fd55005f91/apps",
      "route_mappings_url": "/v2/routes/de5400dc-e167-4eac-ae67-60fd55005f91/route_mappings"
   }
}
```

to map a route to the app
```
cf curl /v3/route_mappings -X POST -d '{
  "relationships": {
      "app": {
        "guid": "060e418b-a4a9-4144-a3c3-4dac66d0e47a"
      },
      "route": {
        "guid": "de5400dc-e167-4eac-ae67-60fd55005f91"
      }
    }
}'
```


to scale the app


```
cf curl /v3/apps/060e418b-a4a9-4144-a3c3-4dac66d0e47a/processes/web/scale -X PUT -d '{
  "instances": 3
}'
```
