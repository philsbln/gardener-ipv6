# IPv6 for Gardener 

## Dual-Stack considered harmful

K8S networking is complex. Adding the choice between IPv4, IPv6, and IPds (IPv4+IPv6) increases the number of variants one can configure even more.
Before trying to enable dual-stack everywhere, keep some experiences from the IPv6 community in mind:
- Dual-stack is dual pain: you have to configure twice as many things compared to yust using IPv6.
- Dual-stack makes debugging hard: if something works, it is no indication for the netwok setup actually works. Chances are that IPv6 and IPv4 are both broken but fallback between them hides it.
- Dual-stack is hard to monitor: see above…
- IPv6 only clients can reach IPv4 only server using NAT64/DNS64. This works fine for almost all modern applications. Even for legacy applications that can't use IPv6 sockets, there are transition technologies, e.g., 464XLAT, that can be deployed without enabling dual-stack for everything.
- IPv4 only clients can reach IPv6 only services through dual-stack load balancers.

**So for all new deployments (and especially PoCs) going IPv6 only is highly recommended.**
In practice, we won't be able to avoid going dual-stack for all ingress load balancers and egress proxies, but we should really should avoid it within K8S clusters. We may need to do dual stack within K8S clusters only when migrating clusters from IPv4 to IPv6, but considering this should be a stretch goal. 

## Stay away from overlay routing

GCP and AWS shifted their address management model for IPv6 towards assigning not single addresses, but **larger prefixes (/96 on GCP, /80 on AWS) per vm/node**.
When using these prefixes to derive pod addresses, we need no additional routing and no NAT within the cluster to enable pod reachability. What still needs to be taken care of is NAT/load balancing to enable K8S services. 

In addition to that, it looks like much of the overlay code has not been tested well with IPv6 and much of their complexity was introduced to work around address shortage and overlapping address space. Therefore, just copying an overlay design to the IPv6 world may be harmful. 

## Addressing

Most K8S documentation does not cover the individual components that may or may not be involved in assigning IP addresses to Nodes and Pods. 

The different architectural decisions are now coverd in a separate [addressing document](addressing).
  


## Gardener Configuration Proposal

First, be aware that we have the question "do we want to do dual-stack" three times: For the Nodes (in case you want to deploy local NAT64 on the nodes), for the pods (in case you want IPv4 egress without NAT64 or IPv4 services), and for the services (only recommended during migrations towards IPv6 only).

We extend the existing fileds in the shoot specification `network` section to provide the cidrs for `nodes`, `pods` and `services`. 
First, they are enhanced to accept a tuple of values: One value for IPv4 and one for IPv6. 
We also introduce special key words to request different strategies for choosing the respective CIDRs. 
These will be available at runtime only and being configured by Gardener/K8S during the cluster deployment/reconciliation.
Please keep in mind that using these is intended for simple use-cases only – complex use-cases require direct configuration of the network plugin

The following keywords are currently anticipated
- **`v4private`** assign the default private networks for IPv4 
- **`v6global`** use the cloud provider's IPAM to reserve a global prefix for the range (optional: of at least that size, defaults to /64).
- **`v6private`** randomly generate a [RFC 4193](https://datatracker.ietf.org/doc/html/rfc4193) prefix for the range (optional: of at least that size between /48 and /112).
- **`custom`** do not configure from Gardener or kube-controller-manager – configuration is done by the network plugin. 
- **`+prefix/size`** modifier – request a prefix per node (optional: of at least that size)
- **`nov4`/`nov6`** no-op - can be specified for documentation purpose
- **`v6fromNode`/`v6fromNode`** (only for pods) assign a per-node pod cidr from the per-node cidr (optional: of at least that size). Requires `+nodeCidr` being specified on the nodes.
  

Example:

```yaml
  networking:
    type: calico # {calico,cilium}
    nodes: v6global/64+prefix/96
    pods: v6fromNode
    services: globalIPv6/64
```


### Issues: 
 - **Warning** Be careful to limit all "pool"-sizes to 16 bits, e.g., make sure that $clusterCidrMask - NodeCidrMaskSize < 16$. 
  [Nodeipam is broken in several ways](https://github.com/cilium/cilium/issues/20756) and used all over in k8s ecosystem. 
  Using (from an addressing perspective) sane config values results in wierd "CIDR range too large" errors. 
 - For some infrastructures like GoM it would be advisable to use a /64 per VM/node due to hardware limitations.
 - Generally using a /64 per VM/node is dangerous because this will limit the flexibility for a general IPv6 numbering scheme. Given a /36 per cloud region and that we want to stay on a nibble boundry for each aggregation level, we easily run out of bits for AZs, VPCs and VMs. If we have a flat network without VPCs, a /64 per VM is not a problem.   

## Collection of random related links
 - https://kubernetes.io/docs/concepts/services-networking/dual-stack/#enable-ipv4-ipv6-dual-stack
 - https://projectcalico.docs.tigera.io/networking/ipv6
 - https://projectcalico.docs.tigera.io/networking/ipv6-control-plane
 - https://github.com/kubernetes/kubernetes/blob/v1.25.0/pkg/controller/nodeipam/ipam/cloud_cidr_allocator.go
