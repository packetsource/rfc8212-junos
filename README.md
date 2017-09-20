# rfc8212-junos

Junos commit script shim to enforce the spirit of [RFC8212](https://tools.ietf.org/html/rfc8212)

Validates Junos post-inheritance candidate configuration to ensure that:

- EBGP peers running IP VPN use "vpn-apply-export": a required knob to 
  ensure that the export policy is actually processed for NLRI in the
  VPN address family
- EBGP peers generally have an export policy defined either at neighbour
  level or group level.

# Deployment

- Copy `rfc8212.slax` to `/var/db/scripts/commit`
- Apply configuration change `set system scripts commit file rfc8212.slax`

Example:
````
adamc@router# show protocols bgp group CUSTOMER neighbor 10.0.201.220  
family inet {
    unicast {
        prefix-limit {
            maximum 3;
        }
    }
}
peer-as 64500;

[edit]
adamc@router# commit 
[edit protocols bgp group CUSTOMER neighbor 10.0.201.220]
  EBGP peers must have an export policy. See RFC 8212.
error: 1 error reported by commit scripts
error: commit script failure

[edit]
adamc@router# set protocols bgp group CUSTOMER neighbor 10.0.201.220 export AS-INTEROUTE 

[edit]
adamc@router# commit 
commit complete

````
