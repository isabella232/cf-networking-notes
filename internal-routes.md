# Proposal for Internal Routes

## Overview
The container networking team is very close to a working overlay network that applications can have level 3 connectivity with each other. However, there is no mechanism for developers to discover the IP address of a particular app on that overlay network. We propose a basic DNS based service

## User stories

> GIVEN I have an app my-frontend-app
> AND the shared internal domain is called `.mycloudfoundry`
> WHEN I run `cf map-internal-route my-frontend-app frontend`
> THEN on the overlay network DNS resolves `frontend.mycloudfoundry` to the IP addresses of my app instances

> GIVEN I have a property set on my bosh manifest for `connet.shared_internal_domain: mycloudfoundry`
> WHEN I deploy Cloud Foundry
> THEN applications with internal routes are accessible at `*.mycloudfoundry`

## About domains

Internal routes parallels normal application routes. This proposal has avoided talking about a key feature of external routes: user provided domains scoped to an org. For the initial effort the commands are assumed to only have a single domain: the shared domain. We are interested in feedback on whether or not this is acceptable scope to cut.

## Implementation concerns

### How would a user interact with this?
The user interface could be a plugin for the CLI. This plugin could resolve app names to app guids and then POST the desired DNS name to app guid mapping to cloud foundry and report errors or successful mappings.

### Where is this stored?
Some component would need to maintain a table of DNS names and their associated applications. This component would either need to push the entirety of this table down to each cell or remain available for queries by Ducati Daemons running on each cell as they receive DNS resolutions with a top level domain of the internal domain. Where it is stored affects where the CLI plugin must POST.

### What about DNS caching and load balancing?
The Ducati Daemon could be extended to rotate the IP address order it returns in response to DNS requests to perform some basic load balancing. Another option would be to return virtual IPs and balance them to the real IP addresses. Either way, DNS cache busting is a problem but it is possible to solve it
