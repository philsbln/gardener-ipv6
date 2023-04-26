# Addressing considerations for Gardener IPv6 support

### Assigning adresses to Nodes

From K8S perspective, the infrastructure assignes addresses to Nodes. Whether this is done from within the *cloud-controller*, the *CNI*, or the infrastructure does not matter for the control plane. 

For security purpuses, it might be advisable to separate node and pod address range.
This seems incompatible to how GCP implements ipv6 and may require some tricks on AWS. 

Recommendations:
- Reduce assuptions abot node addresses to a bare minimum
- Try to prrof-point that we don't need a strict separation between node and pod address ranges.
- Have infrastructure assign nodes addresses using their own IPAM.
- Discover & use node addresses automatically.

### Assigning adresses to PODs

For PODs, there are various options how to get addresses assigned
- Via the **kube-controller-manager**: this is the traditional one and a good choice for an PoC
  - This is enabled when launching the *kube-controller-manager* with  ```--allocate-node-cidrs```
  - When a `v1.Node` ressource is created, the *kube-controller-manager* sets the `PodCIDR` or `PodCIDRs`resource field to the prefix assigned to the node.
  - But where does the *kube-controller-manager* get its addresses from? It internally uses the [nodeipam GO package](https://pkg.go.dev/k8s.io/kubernetes/pkg/controller/nodeipam) to get the CIDR
    - The parameter ```--cidr-allocator-type CloudAllocator``` tells it to use the cloud plugin to derive the cidr from the infrastructure
    - The parameter ```--cidr-allocator-type RangeAllocator``` tells it to use a range of site ```node-cidr-mask-size-ipv6``` from the ```clusterCIDRs``` configured
- Via a **different controller** that sets the `PodCIDR` or `PodCIDRs`resource  
- Via the **CNI** and CNI specific replacement for `PodCIDR` this is the default for *Calico*
- Via a controller on the Node itself that updates the `v1.Node` ressource

Once the `PodCIDR` has been added on the Node ressource, the CNI will pick up this information and configure the the node and pods accordingly  

## Addressing Scheme Ideas

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