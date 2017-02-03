# reorganizing iptables egress rules

## Problem
Existing large deployments may have 10k+ iptables rules per host, and this causes
performance problems when trying to modify policy.

## Goals
- reduce total number of rules
- support dynamic egress with source granularity of app, space or global

## Approach
- leverage container marks for egress control
- accept (and thus terminate matching) as early as possible
- chains should reflect logical structure: `global > spaces > apps > containers`

## Requirements
Desired ASG rules should be communicated to the enforcement system with info
about their granularity.  For example, if the policy-server internal API response
included data like
```json
{
  "global_policies": [
     { "addresses": [
        "0.0.0.0-9.255.255.255",
        "11.0.0.0-169.253.255.255",
        "169.255.0.0-172.15.255.255",
        "172.32.0.0-192.167.255.255",
        "192.169.0.0-255.255.255.255"
        ]
     }
  ],
  "space_policies": [
    { "source": "some-space-guid", "destination": { "addresses": ... } }
  ]
}
```
Then the enforcement mechanism could ensure that, for example, the global policies were written
only once but applied to all containers.

## Example rule set
```
-A FORWARD -s 10.255.100.0/24 -j cfn-fw

# set app mark for each source container
-A cfn-fw -s 10.255.100.100/32 -j MARK --set-xmark 0xABCD
-A cfn-fw -s 10.255.100.101/32 -j MARK --set-xmark 0xABCD
-A cfn-fw -s 10.255.100.102/32 -j MARK --set-xmark 0xABCE
-A cfn-fw -s 10.255.100.103/32 -j MARK --set-xmark 0xABCF

# allow outgoing related established
-A cfn-fw -m state --state RELATED,ESTABLISHED -j ACCEPT

# allow outgoing matching global egress rules
-A cfn-fw -j cfn-fw-global-egress

# based on mark, jump to per-app egress chain
-A cfn-fw -m mark --mark 0xABCD -j cfn-fw-app-APP-GUID1
-A cfn-fw -m mark --mark 0xABCE -j cfn-fw-app-APP-GUID2
-A cfn-fw -m mark --mark 0xABCF -j cfn-fw-app-APP-GUID3

# per-app egress chains
-A cfn-fw-app-APP-GUID1 -j cfn-fw-space-SPACE-GUIDA
-A cfn-fw-app-APP-GUID1 -d 192.168.4.0/24 -m tcp -j ACCEPT
-A cfn-fw-app-APP-GUID1 -d 192.168.5.0/24 -m tcp -j ACCEPT

-A cfn-fw-app-APP-GUID2 -j cfn-fw-space-SPACE-GUIDA
-A cfn-fw-app-APP-GUID2 -d 10.10.30.0/24 -m tcp -j ACCEPT
-A cfn-fw-app-APP-GUID2 -d 192.168.9.0/24 -m tcp -j ACCEPT

-A cfn-fw-app-APP-GUID3 -j cfn-fw-space-SPACE-GUIDB
-A cfn-fw-app-APP-GUID3 -d 10.10.30.0/24 -m tcp -j ACCEPT
-A cfn-fw-app-APP-GUID3 -d 192.168.9.0/24 -m tcp -j ACCEPT

# per-space egress chains, resulting from space ASGs
-A cfn-fw-space-SPACE-GUIDA -d 10.20.0.0/16 -m tcp -j ACCEPT
-A cfn-fw-space-SPACE-GUIDA -d 10.30.0.0/16 -m tcp -j ACCEPT

-A cfn-fw-space-SPACE-GUIDB -d 10.30.0.0/16 -m udp -j ACCEPT

# global egress rules, resulting from global ASGs
-A cfn-fw-global-egress -d 0.0.0.0/32 -m udp --dport 53 -j ACCEPT
-A cfn-fw-global-egress -d 10.200.10.0/24 -m tcp --dport 3306 -j ACCEPT
```
