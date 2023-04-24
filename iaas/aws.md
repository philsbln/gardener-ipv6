# IPv6 at AWS

## Prefix Assigment model

- Each node gets a /80 assigned
- The [amazon-vpc-cni](https://github.com/aws/amazon-vpc-cni-k8s/blob/e864016e80cb311bb2f8fc4338ef79e9b3a53e47/pkg/ipamd/ipamd.go) in the node fetches the prefix from the AWS API and starts mandging in its integrated ipamd.
- The AWS cni plugin asks local ipamd to assign an address. 
