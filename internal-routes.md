# Proposal for Internal Routes

## Overview

In the initial proposal for application networking we had outlined a basic service discovery mechanism: a DNS name for apps in the format “appname.spacename.orgname.cloudfoundry”. This will be referred to as Predefined Names. Developers need some basic service discovery to connect their applications independent of a container’s IP address; IP addresses should be left for the platform to control. Though necessary, we hope to provide the most basic service discovery we can while focusing on network connectivity. Service discovery has many existing solutions but container connectivity in Cloud Foundry has very few.

Now that the Container Networking team has dived into the work on Predefined Names  this it isn’t clear that this really is the simplest thing. Another solution exists which is more complex to describe but easier to implement, simpler to use, and more congruous with existing concepts: Internal Routes.

## User stories for Predefined Names
> *As an operator, I can define a property in the release manifest to be the top level domain of internal hostnames, with a pre-specified value of “cloudfoundry.”*

> *As a developer, when I push an app named inventory to the Checkout space, in the E_Commerce org, then I can do a DNS lookup from another app in cloudfoundry for “inventory.checkout.ecommerce.cloudfoundry” and get the IP addresses of inventory’s containers.*



## User stories for Internal Routes
> *As an operator, I can define a property in the release manifest to be the top level domain of internal hostnames, with a pre-specified value of “cloudfoundry.”*

> *As an developer, I can run “cf create-internal-route cf create-route my-space example.com*

> *As developer I can run “cf create-create-internal-route inventory inventory”*

> *As an operator I can define the internal domain, to replace .cloudfoundry*

## Comparison
As you see, there are certainly fewer user stories in the Predefined Names track but also a few problems that arise:
* Name database size and synchronization issues scale directly with number of applications
* Cloud controller would need to be polled or hit frequently to figure out if a particular app/space/org combination exists whenever another app tries to reach it
* The validations of an Cloud Foundry Org and Space are not the same as the constraints of a DNS name, so we’d have to map more permissive CF names to the less permissive DNS equivalent and collisions in that mapping would need to be resolved.

By contrast, Internal routes would:
* Limit the size of number of the DNS entries that would need to be stored and synchronized
* Open the possibility of storing state outside of Cloud Controller, once the CLI has determined the app ID of a DNS name it would not need to query for more information
