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

## K8S Network Architecture dissected

Most K8S documentation does not cover the individual components hat may or may not be involved in assigning IP addresses to Nodes and Pods. 

### Assigning adresses to Nodes

From K8S perspective, the infrastructure assignes addresses to Nodes. Whether this is done from within the *cloud-controller*, the *CNI*, or the infrastructure does not matter for the control plane. 

Recommendations:
- let infrastructure assign addresses to nodes based on their own IPAM
- For a PoC on AWS, create a VPC with AWS assigned /56, make an IPv6 only subnet and request a /80 prefix per VM

### Assigning adresses to PODs

For PODs, there are various options how to get addresses assigned
- Via the **kube-controller-manager**: this is the traditional one and a good choice for an PoC
  - This is enabled when launching the *kube-controller-manager* with  ```--allocate-node-cidrs```
  - When a `v1.Node` ressource is created, the *kube-controller-manager* sets the `PodCIDR` or `PodCIDRs`resource field to the prefix assigned to the node.
  - But where does the *kube-controller-manager* get its addresses from? It internally uses the [nodeipam GO package](https://pkg.go.dev/k8s.io/kubernetes/pkg/controller/nodeipam) to get the CIDR
    - The parameter ```--cidr-allocator-type CloudAllocator``` tells it to use the cloud plugin to derive the cidr from the infrastructure
    - The parameter ```--cidr-allocator-type RangeAllocator``` tells it to use a range of site ```node-cidr-mask-size-ipv6``` from the ```clusterCIDRs``` configured
    - **warning** [nodeipam is broken in several ways](https://github.com/cilium/cilium/issues/20756) 
      Using sane config values results in wierd "CIDR range too large" error. 
      Make sure make sure that $clusterCidrMask - NodeCidrMaskSize < 16$
- Via a **different controller** that sets the `PodCIDR` or `PodCIDRs`resource  
- Via the **CNI** and CNI specific replacement for `PodCIDR` this is the default for *Calico*
- Via a controller on the Node itself that updates the `v1.Node` ressource

Once the `PodCIDR` has been added on the Node ressource, the CNI will pick up this information and configure the the node and pods accordingly  

### Preventing NAT for PODs

When routing the POD v6 addressess, it does not make sense to hide them behind the Node IP.  The switch controlling that looks like a wierd side effect… make shure IPv6 range ist *not* passed to `--cluster-cidr`of the kube proxy. 

### Assigning adresses to Services

- There is a IPv6 related bug in *kube-apiserver* that keeps crashing if ```service-cidr``` is larger than ```/112```.
- It may be useful to use global ipv6 address space here to enable cluster to cluster exposure of services
- For a PoC, using [ULA/RFC 4193](https://datatracker.ietf.org/doc/html/rfc4193) is totally valid.

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


## Addressing Scheme Ideas

- we take one Cluster Prefix from the cloud infrastructure for Nodes and Pods (usually /64)
- we take one prefix per Node (length depends on infrastructure /80.../96) from that Cluster prefix

| **Bits**    | 0...........64 |                      ..96 |                                      ..128 |
| :---------- | :------------- | :------------------------ | :----------------------------------------- |
| **General** | Cluster Prefix | Node ID (..96)            | Pool/Res Type (..104/112) + Res ID (..128) | 
| **AWS**     | Subnet         | Node ID (..80) + 0 (..96) | Pool/Res Type (..104/112) + Res ID (..128) | 
| **GCP**     | Subnet         | Node ID (..96)            | Pool/Res Type (..104/112) + Res ID (..128) | 

- that Node Prefix is further split per Pool/Ressource Type 
  We use some IP encoding inspired by QUIC variants to make adresses shorter/ more readable and still 
  allow large pools. So we support 255 16 bit pool types and 15 24bit pool types and 15 28bit pool types

| **Len** | **Prefix** | **Pool Type**           | **Usage** |
| ------: | :--------- | :---------------------- | :-------------------------------------------------------------- |
|      16 | `0000`     | Node addresses          | Used to address nodes / services on the node                    |
|      24 | `0d`       | Default Pod Range       | Fed into PodCIDRs to let CNI pick addresses                     |
|      24 | `0f`       | Services                | Used as service-prefix on the Pseudo-Node with NodeID 0000      |
|      28 | `d`        | Per-Namespace Pod Range | Allow custom CNI PIAM to construct addresses (12b NS + 16b Pod) |

- For Services, we either use a pseudo-node and use its prefix for services or a 

## Collection of random related links
 - https://kubernetes.io/docs/concepts/services-networking/dual-stack/#enable-ipv4-ipv6-dual-stack
 - https://projectcalico.docs.tigera.io/networking/ipv6
 - https://projectcalico.docs.tigera.io/networking/ipv6-control-plane
 - https://github.com/kubernetes/kubernetes/blob/v1.25.0/pkg/controller/nodeipam/ipam/cloud_cidr_allocator.go
