# srl-bgp-safeguard
Agent that validates exported BGP routes before peering with the 'real' target IP

# Context
Cloudflare [recently shared an issue](https://blog.cloudflare.com/route-leaks-and-confirmation-biases/) where a missing BGP export policy caused advertisements of other people's prefixes. They suggest default EBGP rejection policies as required by [RFC8212](https://datatracker.ietf.org/doc/html/rfc8212), which is a good starting point.
However, given a fully programmable platform like #SRLinux, there is more we can do

# Workflow
This agent extends the system Yang model with a place for network engineers to 'request' a new BGP peering (intent). However, instead of directly applying this configuration, the agent first connects with an internal system to get approval for the prefixes to be exported:

1. Human engineer requests peering with 1.2.3.4:
```
/network-instance default protocols bgp safe-peering
neighbor 1.2.3.4
```
The CLI is exactly the same (referenced), but the parent path is 'safe-peering', a custom extension

2. The agent receives a config update event from the NDK, and peers with a configured internal route clearance server first
3. The BGP peering clearance server is a route reflector configured with RPKI validation. It validates all the exported routes, and reflects back the valid ones. By comparing the count of advertised prefixes versus received prefixes, the agent can check that all routes passed verification
4. The agent closes the BGP clearance server session, and modifies the local config to add the 'real' BGP peer IP:
```
/network-instance default protocols bgp
neighbor 1.2.3.4 !!! Provisioned through BGP safeguard on 2022-02-25 at 11:00am
```

## Resource Public Key Infrastructure (RPKI) route validator
In this setup, the RPKI validator/route reflector is implemented using static RPKI entries on an SR OS router. Those skilled in the art will appreciate that this could just as well use a dynamic RPKI source
