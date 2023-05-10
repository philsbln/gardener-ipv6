# Addressing considerations for Gardener IPv6 support

## Assigning addresses to Nodes

From K8S perspective, the infrastructure assigns addresses to Nodes. Whether this is done from within the *cloud-controller*, the *CNI*, or the infrastructure does not matter for the control plane.

For security purposes, it might be advisable to separate node and pod address range.
This seems incompatible to how GCP implements ipv6 and may require some tricks on AWS.

Recommendations:
- Reduce assumptions about node addresses to a bare minimum
- Try to proof-point that we don't need separation between node and pod address ranges.
- Have infrastructure assign nodes addresses using their own IPAM.
- Discover & use node addresses automatically.

## Assigning addresses to PODs

For PODs, there are various options how pod addresses are assigned.
From a kubernetes control plane perspctive there is no need to aggregate POD addresses on a a per-node basis.
Still, in order to achieve scalability, we need to perform a two-step approach that first assigns IP prefixes to the nodes and then have the nodes assign addresses frome these prefixes to pods.

### POD address (ranges) to Nodes (on the control plane)

- Via the **kube-controller-manager**: this is the traditional one and used in Gardener
  - This is enabled when launching the *kube-controller-manager* with  ```--allocate-node-cidrs```
  - When a `v1.Node` resource is created, the *kube-controller-manager* sets the `PodCIDR` or `PodCIDRs`resource field to the prefix assigned to the node.
  - But where does the *kube-controller-manager* get its addresses from? It internally uses the [nodeipam GO package](https://pkg.go.dev/k8s.io/kubernetes/pkg/controller/nodeipam) to get the CIDR
    - The parameter ```--cidr-allocator-type CloudAllocator``` tells it to use the cloud plugin to derive the cidr from the infrastructure
    - The parameter ```--cidr-allocator-type RangeAllocator``` tells it to use a range of site ```node-cidr-mask-size-ipv6``` from the ```clusterCIDRs``` configured

  Advantages:
    - Gardener already using this for IPv4
    - Pretty simple

  Disadvantages:
    - Functionality is likely to be infrastructure specific and, thus better handeled in the cloud-controller-manager
    - Limitations of the *nodeipam GO package* limit the pool size to 16bits
    - The approach does not go well with assigning a larger prefix per node and have the routing in the IaaS while only using parts of it for the PODs

- Via the **cloud-controller-manager**

  - Uses the same code as *kube-controller-manager* by default, but can get overridden by the cloud provider implementation.
  - AWS: not implemented (functionality is in the CNI)
  - GCP: TODO
  - Azure: TODO

  Advantages:
    - Seems the right place from achtitecture point of view
    - Integrates best with infrastructure

- Via a **controller that may be part of the CNI**
  - Depending on the anticipated CNI plugin on the node, there are various ways how the these ranges could be communicated:
    - Using the `PodCIDR` or `PodCIDRs` on the node object
    - Using an CNI specific resources on the node object (Default way for Calico)
    - Using the API of the IaaS as ground truth (Default in AWS CNI)
  - Main decision would be whether to use
    - IaaS provided CNI
    - Calico/Cilium
    - Integrate with MCM
    - Have our own CNI or a separate controller

  Advantages:
   - Fully flexible
   - Takes advantage of IaaS features

  Disadvantages:
   - Many variants possible that need careful triage
   - Most probably no portable solution available that we could reuse

Recommendations: Decide whether we want
  - A minimal solution => Set `PodCIDR` or `PodCIDRs` resource from MCM, webhook or custom controller
  - A full-flex solution => Come up with gardener-specific resources and write our own CNI plugin

### POD addresses to PODs (on the Node)

- The **host-only-CNI** picks up `PodCIDR` or `PodCIDRs` and assigns from that ranges (Default in Gardener)

  Advantages:
  - simple
  - portable

  Disadvantages:
  - No support for "IPv4 egress on IPv6 only clusters" yet

- The **CNI**, e.g., Calico or Cilium, picks up `PodCIDR` or `PodCIDRs` or a CNI specific replacement and manages it by its own means (Default in Calico)

  Advantages:
  - Well integrated with CNI

  Disadvantages:
  - No support for "IPv4 egress on IPv6 only clusters" yet

- An **IaaS specific CNI** picks addresses from the IaaS control plane and manages it by its own means (AWS-CNI)

  Advantages:
  - Well integrated with IaaS

  Disadvantages:
  - No not portable
  - May limit the way how we use Calico or Cilium for policies and services.

- We build our **own CNI plugin** to do address management.

  Advantages:
  - This does not prevent us from using Calico or Cilium for policies and services.
  - Fully flexible
  - Allows to build fancy "IPv4 egress on IPv6 only clusters" setups

  Disadvantages:
  - More things to maintain

Recommendations:
- If we decide to go with `PodCIDR` or `PodCIDRs` above, decide on CNI basis whether to use host-only-CNI or Calico/Cilium to to pick up the data
- If we want more flexibility, we need to write our own CNI plugin


## Addressing Scheme Ideas (WIP)

This is an approach how we could have a prefix per node provided by the IaaS while having a harmonized addressing scheme across all IaaS

- we take one Cluster Prefix from the cloud infrastructure for Nodes and Pods (usually /64)
- we take one prefix per Node (length depends on infrastructure /64.../80.../96) from that Cluster prefix

| **Bits**    | 0...........64 |                      ..96 |                                      ..128 |
| :---------- | :------------- | :------------------------ | :----------------------------------------- |
| **General** | Cluster Prefix | Node ID (..96)            | Res Type (..104/112) + Res ID (..128) |
| **GoM**.    | Node/VM        |                           | Res Type (..104/112) + Res ID (..128) |
| **AWS**     | Subnet         | Node ID (..80) + 0 (..96) | Res Type (..104/112) + Res ID (..128) |
| **GCP**     | Subnet         | Node ID (..96)            | Res Type (..104/112) + Res ID (..128) |

- that Node Prefix is further split per Ressource Type
  We use some IP encoding inspired by QUIC variants to make adresses shorter/ more readable and still
  allow large pools. So we support 255 16 bit pool types and 15 24bit pool types and 15 28bit pool types

| **Len** | **Prefix** | **Pool Type**           | **Usage** |
| ------: | :--------- | :---------------------- | :-------------------------------------------------------------- |
|      16 | `0000`     | Node addresses          | Used to address nodes / services on the node                    |
|      24 | `0d`       | Pod Range       | Fed into PodCIDRs to let CNI pick addresses                     ||
|      28 | `d`        | Per-Namespace Pod Range | Allow custom CNI IPAM to construct addresses (12b NS + 16b Pod) |

