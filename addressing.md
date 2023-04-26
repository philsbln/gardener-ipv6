# Addressing considerations for Gardener IPv6 support

## Assigning addresses to Nodes

From K8S perspective, the infrastructure assigns addresses to Nodes. Whether this is done from within the *cloud-controller*, the *CNI*, or the infrastructure does not matter for the control plane. 

For security purposes, it might be advisable to separate node and pod address range.
This seems incompatible to how GCP implements ipv6 and may require some tricks on AWS. 

Recommendations:
- Reduce assumptions abot node addresses to a bare minimum
- Try to prrof-point that we don't need separation between node and pod address ranges.
- Have infrastructure assign nodes addresses using their own IPAM.
- Discover & use node addresses automatically.

## Assigning addresses to PODs

For PODs, there are various options how pod addresses are assigned to the nodes and pods

### POD address (ranges) to Nodes (on the control plane)

- Via the **kube-controller-manager**: this is the traditional one and used in Gardener
  - This is enabled when launching the *kube-controller-manager* with  ```--allocate-node-cidrs```
  - When a `v1.Node` resource is created, the *kube-controller-manager* sets the `PodCIDR` or `PodCIDRs`resource field to the prefix assigned to the node.
  - But where does the *kube-controller-manager* get its addresses from? It internally uses the [nodeipam GO package](https://pkg.go.dev/k8s.io/kubernetes/pkg/controller/nodeipam) to get the CIDR
    - The parameter ```--cidr-allocator-type CloudAllocator``` tells it to use the cloud plugin to derive the cidr from the infrastructure
    - The parameter ```--cidr-allocator-type RangeAllocator``` tells it to use a range of site ```node-cidr-mask-size-ipv6``` from the ```clusterCIDRs``` configured
  - Disadvantage: 
    - Limitations of the *nodeipam GO package* mentioned above
    - That approach does not go well with assigning a prefix per node and have the routing in the IaaS 
- Via a **controller part of the CNI** 
  - Depending on the anticipated CNI plugin on the node, there are various ways how the these ranges could be communicated:
    - Using the `PodCIDR` or `PodCIDRs` on the node object (Default in Gardener)
    - Using an CNI specific resources on the node object (Default way for Calico)
    - Using the API of the IaaS as ground truth (Default in AWS CNI)
- Via a **different controller** that sets the `PodCIDR` or `PodCIDRs`resource  

Recommendations: Decide whether we want 
  - A minimal solution => Set `PodCIDR` or `PodCIDRs` resource from MCM or webhook
  - A full-flex solution => Come up with gardener-specific resources and write our own CNI-IPAM plugin 

### POD addresses to PODs (on the Node)

- The **host-only-CNI** picks up `PodCIDR` or `PodCIDRs` and assigns from that ranges (Default in Gardener)
- The **CNI** picks up `PodCIDR` or `PodCIDRs` or a CNI specific replacement and manages it by its own means (Default in Calico)
- The **CNI** picks addresses from the IaaS control plane and manages it by its own means (AWS-CNI)
- Via a controller on the Node itself that updates the `v1.Node` ressource


## Addressing Scheme Ideas (WIP)

- we take one Cluster Prefix from the cloud infrastructure for Nodes and Pods (usually /64)
- we take one prefix per Node (length depends on infrastructure /64.../80.../96) from that Cluster prefix

| **Bits**    | 0...........64 |                      ..96 |                                      ..128 |
| :---------- | :------------- | :------------------------ | :----------------------------------------- |
| **General** | Cluster Prefix | Node ID (..96)            | Pool/Res Type (..104/112) + Res ID (..128) |
| **GoM**.    | Node/VM Prefix | 0       (..96)            | Pool/Res Type (..104/112) + Res ID (..128) |
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