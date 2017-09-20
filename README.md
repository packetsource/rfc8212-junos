# rfc8212-junos

Junos commit script shim to enforce the spirit of [RFC8212](https://tools.ietf.org/html/rfc8212)

Validates Junos post-inheritance candidate configuration to ensure that:

- EBGP peers explicitly specify an import and export policy at either
  group or neighbour level.

- EBGP peers running IP VPN AFI use "vpn-apply-export": a required knob to 
  ensure that the export policy is actually processed for NLRI in the
  VPN address family.

Two flavours:
- `rfc8212.slax`: strict implementation the replaces the absence of a policy with a deny-all
- `rfc8212-throw.slax`: no substitutions, just throw a commit error in absence of policy

# Deployment

- Copy `rfc8212.slax` to `/var/db/scripts/commit`
- Apply configuration change `set system scripts commit file rfc8212.slax`
- Ensure script available on all routing engines, `set system commits synchronize scripts`

# Example use

First-time running, `rfc8212.slax` will:
- Create a special policy __RFC8212_DEFAULT__ to encapsulate the default logic
- Enable transient changes for commit scripts. (Allowing it to substitute an absence of policy with a default)

````
adamc@router# commit            
warning: creating default EBGP policy for application to non-RFC8212 compliant EBGP peers
commit complete
````

Then, when committing a configuration with an EBGP peer without policy at either group or
neighbor level, `rfc8121.slax` will retrofit the default policy at the neighbor level and
allow the commit operation to conclude. 

````
[edit]
adamc@router# set protocols bgp group CUSTOMER type external 

[edit]
adamc@router# set protocols bgp group CUSTOMER neighbor 10.0.201.12 peer-as 65534 

[edit]
adamc@router# commit 
[edit protocols bgp group CUSTOMER neighbor 10.0.201.12]
  warning: EBGP export policy not found: using __RFC8212_DEFAULT__
[edit protocols bgp group CUSTOMER neighbor 10.0.201.12]
  warning: EBGP import policy not found: using __RFC8212_DEFAULT__
commit complete
````

The default policy is NOT visible in the candidate or committed configurations. It is implemented
as a transient change each time `by rfc8212.slax`. Despite not being visible in the configuration,
it *is* in effect.

````
adamc@router# show protocols bgp group CUSTOMER                         
type external;
neighbor 10.0.201.12 {
    peer-as 65534;
}

adamc@router# run show bgp neighbor 10.0.201.12 
Peer: 10.0.201.12+179 AS 65534 Local: 10.0.201.201 AS 8928 
  Group: CUSTOMER              Routing-Instance: master
  Forwarding routing-instance: master  
  Type: External    State: Connect        Flags: <TryConnect>
  Last State: Connect       Last Event: ConnectRetry
  Last Error: None
  Export: [ __RFC8212_DEFAULT__ ] Import: [ __RFC8212_DEFAULT__ ]
  Options: <Preference PeerAS Refresh>
  Holdtime: 90 Preference: 170
  Number of flaps: 0
````

Every subsequent commit where no explicit policy has been put in place will cause `rfc8212-slax` to
emit a warning and provide its default policy fillter as a transient configuration. It is therefore
advisable to `set system commits synchronize scripts` for consistent behaviour.

When a policy is put in place at either group or peer level, this will take precedence and
`rfc8212.slax` will not apply its default policy.

````
adamc@router# set protocols bgp group CUSTOMER neighbor 10.0.201.12 export AS-INTEROUTE 


adamc@router# commit 
[edit protocols bgp group CUSTOMER neighbor 10.0.201.12]
  warning: EBGP import policy not found: using __RFC8212_DEFAULT__
commit complete

adamc@router# run show bgp neighbor 10.0.201.12 
Peer: 10.0.201.12+179 AS 65534 Local: 10.0.201.201 AS 8928 
  Group: CUSTOMER              Routing-Instance: master
  Forwarding routing-instance: master  
  Type: External    State: Connect        Flags: <>
  Last State: Active        Last Event: ConnectRetry
  Last Error: None
  Export: [ AS-INTEROUTE ] Import: [ __RFC8212_DEFAULT__ ]
  Options: <Preference PeerAS Refresh>
  Holdtime: 90 Preference: 170
  Number of flaps: 0


adamc@router# set protocols bgp group CUSTOMER neighbor 10.0.201.12 import NO-ROUTES    

adamc@router# commit 
commit complete

adamc@router# run show bgp neighbor 10.0.201.12                                         
Peer: 10.0.201.12 AS 65534     Local: 10.0.201.201 AS 8928 
  Group: CUSTOMER              Routing-Instance: master
  Forwarding routing-instance: master  
  Type: External    State: Active         Flags: <>
  Last State: Idle          Last Event: Start
  Last Error: None
  Export: [ AS-INTEROUTE ] Import: [ NO-ROUTES ]
  Options: <Preference PeerAS Refresh>
  Holdtime: 90 Preference: 170
  Number of flaps: 0

````

No warranties, but best intentions. 