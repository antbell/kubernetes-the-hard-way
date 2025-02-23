# Cleaning Up

In this lab you will delete the compute resources created during this tutorial.

## Compute Instances

Delete the controller and worker compute instances:

```
gcloud -q compute instances delete \
  abell-controller-0 abell-controller-1 abell-controller-2 \
  abell-worker-0 abell-worker-1 abell-worker-2 \
  --zone $(gcloud config get-value compute/zone)
```

## Networking

Delete the external load balancer network resources:

```
{
  gcloud -q compute forwarding-rules delete abell-kubernetes-forwarding-rule \
    --region $(gcloud config get-value compute/region)

  gcloud -q compute target-pools delete abell-kubernetes-target-pool

  gcloud -q compute http-health-checks delete abell-kubernetes

  gcloud -q compute addresses delete abell-kubernetes-tutorial
}
```

Delete the `abell-kubernetes-tutorial` firewall rules:

```
gcloud -q compute firewall-rules delete \
  abell-kubernetes-tutorial-allow-nginx-service \
  abell-kubernetes-tutorial-allow-internal \
  abell-kubernetes-tutorial-allow-external \
  abell-kubernetes-tutorial-allow-health-check
```

Delete the `abell-kubernetes-tutorial` network VPC:

```
{
  gcloud -q compute routes delete \
    abell-kubernetes-route-10-200-0-0-24 \
    abell-kubernetes-route-10-200-1-0-24 \
    abell-kubernetes-route-10-200-2-0-24

  gcloud -q compute networks subnets delete kubernetes

  gcloud -q compute networks delete abell-kubernetes-tutorial
}
```

Delete the `abell-kubernetes-tutorial` compute address:

```
gcloud -q compute addresses delete abell-kubernetes-tutorial \
  --region $(gcloud config get-value compute/region)
```
