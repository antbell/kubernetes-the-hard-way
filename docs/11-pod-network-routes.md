# Provisioning Pod Network Routes

Pods scheduled to a node receive an IP address from the node's Pod CIDR range. At this point pods can not communicate with other pods running on different nodes due to missing network [routes](https://cloud.google.com/compute/docs/vpc/routes).

In this lab you will create a route for each worker node that maps the node's Pod CIDR range to the node's internal IP address.

> There are [other ways](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) to implement the Kubernetes networking model.

## The Routing Table

In this section you will gather the information required to create routes in the `abell-kubernetes-tutorial` VPC network.

Print the internal IP address and Pod CIDR range for each worker instance:

```
for instance in abell-worker-0 abell-worker-1 abell-worker-2; do
  gcloud compute instances describe ${instance} \
    --format 'value[separator=" "](networkInterfaces[0].networkIP,metadata.items[0].value)'
done
```

> output

```
10.240.0.20 10.200.0.0/24
10.240.0.21 10.200.1.0/24
10.240.0.22 10.200.2.0/24
```

## Routes

Create network routes for each worker instance:

```
for i in 0 1 2; do
  gcloud compute routes create abell-kubernetes-route-10-200-${i}-0-24 \
    --network abell-kubernetes-tutorial \
    --next-hop-address 10.240.0.2${i} \
    --destination-range 10.200.${i}.0/24
done
```

List the routes in the `abell-kubernetes-tutorial` VPC network:

```
gcloud compute routes list --filter "network: abell-kubernetes-tutorial"
```

> output

```
NAME                            NETWORK                  DEST_RANGE     NEXT_HOP                  PRIORITY
default-route-1606ba68df692422  abell-kubernetes-tutorial  10.240.0.0/24  kubernetes-the-hard-way   0
default-route-615e3652a8b74e4d  abell-kubernetes-tutorial  0.0.0.0/0      default-internet-gateway  1000
abell-kubernetes-route-10-200-0-0-24  abell-kubernetes-tutorial  10.200.0.0/24  10.240.0.20               1000
abell-kubernetes-route-10-200-1-0-24  abell-kubernetes-tutorial  10.200.1.0/24  10.240.0.21               1000
abell-kubernetes-route-10-200-2-0-24  abell-kubernetes-tutorial  10.200.2.0/24  10.240.0.22               1000
```

Next: [Deploying the DNS Cluster Add-on](12-dns-addon.md)
