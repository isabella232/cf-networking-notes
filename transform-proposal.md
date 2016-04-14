## Proposal - LRP and Task Transformer

Attempting to integrate new projects with Cloud Foundry and Diego would be
much simpler if there was a way to inject some data into the tasks and desired
LRPs created by the ☆NSYNC component of the bridge.

I'd like to propose a few new hooks to the current flow from the CC bridge to
the Garden API that could make this simpler. These include:

1. Introduction of a generic properties collection on the Task and DesiredLRP models
2. Propagation of the properties through the Executor and to the Garden
   ContainerSpec
3. Introduction of an ☆NSYNC transformer that can augment the Tasks and
   DesiredLRPs created by the bridge before they are sent across the BBS
   "desire" API

### Generic Properties

A Diego LRP contains two bits of data that are passed through the running
system without impacting diego code directly: the `annotation` field and the
`routes` map.

The first of these was probably intended to allow humans to add some text that
describes the LRP purpose but, it's currently used to hold an ETag that is used
by the route emitter. Unfortunately, it's a single field and only allows one bit
of data so it's hard to consider it extensible.

The second is the `routes` map. This field is a map that uses string keys but
can carry any data at all. It's opaque and can be dynamically updated but it's
intended to be used by code that polls the BBS to maintain routes to actual
LRPs. While useful, it's intended to be domain specific.

Tasks and desired LRPs should be extended to include opaque key:value
properties. This metadata could be used to carry arbitrary data through the
diego system into their specific domains. In the case of the container
networking team, we would like to be able to carry at least the application
guid and the network ID through the system. These two pieces of information
currently have no relevance to Diego but they're quite important to us.

A complementary feature could be the introduction of labels or tags on tasks
and LRPs that could be used as part of selectors on the BBS.

### Property Propagation

Unlike Diego, the Garden API has had support for arbitrary properties for
quite some time and at one point, Diego used them extensively.

The current iteration of the external networker code is using a container
property to hold its configuration. When a container is created, the
properties from the container spec are used to acquire and propagate external
networking information to the plugin that generates runc *prestart* and
*poststop* hook configurations for the container.

If the Diego executor were to propagate properties from the LRP to garden at
the time of container creation, there would be no need for custom code in the
executor.

### Transformer

If properties were supported on tasks and LRPs and could be propagated to
garden, the last point of integration would be the generation/addition of
these properties. Ideally this would be done via a plugin model that could be
bosh deployed with Diego.

One way to implement this would be to call a set of plugins to *transform* the
tasks and LRPs created by ☆NSYNC or all tasks and LRPs that are desired on the
BBS. While the BBS would be able to catch "everything", ☆NSYNC would only hook
into Cloud Foundry specific items.

If ☆NSYNC or the BBS are configured appropriately, they would make an http
call over a UNIX domain socket to the configured plugin. The contract would be
very simple: POST the JSON serialization of the task or LRP to a
`/transform/task` or `/transform/desired_lrp` route. The response would
contain the JSON serialization of the augmented (or unchanged) task or LRP.
