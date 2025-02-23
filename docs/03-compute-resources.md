# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster across a single [compute zone](https://cloud.google.com/compute/docs/regions-zones/regions-zones).

> Ensure a default compute zone and region have been set as described in the [Prerequisites](01-prerequisites.md#set-a-default-compute-region-and-zone) lab.

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Virtual Private Cloud Network

In this section a dedicated [Virtual Private Cloud](https://cloud.google.com/compute/docs/networks-and-firewalls#networks) (VPC) network will be setup to host the Kubernetes cluster.

Create the `abell-kubernetes-tutorial` custom VPC network:

```
gcloud compute networks create abell-kubernetes-tutorial --subnet-mode custom
```

A [subnet](https://cloud.google.com/compute/docs/vpc/#vpc_networks_and_subnets) must be provisioned with an IP address range large enough to assign a private IP address to each node in the Kubernetes cluster.

Create the `kubernetes` subnet in the `abell-kubernetes-tutorial` VPC network:

```
gcloud compute networks subnets create kubernetes \
  --network abell-kubernetes-tutorial \
  --range 10.240.0.0/24
```

> The `10.240.0.0/24` IP address range can host up to 254 compute instances.

### Firewall Rules

Create a firewall rule that allows internal communication across all protocols:

```
gcloud compute firewall-rules create abell-kubernetes-tutorial-allow-internal \
  --allow tcp,udp,icmp \
  --network abell-kubernetes-tutorial \
  --source-ranges 10.240.0.0/24,10.200.0.0/16
```

Create a firewall rule that allows external SSH, ICMP, and HTTPS:

```
gcloud compute firewall-rules create abell-kubernetes-tutorial-allow-external \
  --allow tcp:22,tcp:6443,icmp \
  --network abell-kubernetes-tutorial \
  --source-ranges 0.0.0.0/0
```

> An [external load balancer](https://cloud.google.com/compute/docs/load-balancing/network/) will be used to expose the Kubernetes API Servers to remote clients.

List the firewall rules in the `abell-kubernetes-tutorial` VPC network:

```
gcloud compute firewall-rules list --filter="network:abell-kubernetes-tutorial"
```

> output

```
NAME                                    NETWORK                  DIRECTION  PRIORITY  ALLOW                 DENY  DISABLED
abell-kubernetes-tutorial-allow-external  kubernetes-the-hard-way  INGRESS    1000      tcp:22,tcp:6443,icmp        False
abell-kubernetes-tutorial-allow-internal  kubernetes-the-hard-way  INGRESS    1000      tcp,udp,icmp                Fals
```

### Kubernetes Public IP Address

Allocate a static IP address that will be attached to the external load balancer fronting the Kubernetes API Servers:

```
gcloud compute addresses create abell-kubernetes-tutorial \
  --region $(gcloud config get-value compute/region)
```

Verify the `abell-kubernetes-tutorial` static IP address was created in your default compute region:

```
gcloud compute addresses list --filter="name=('abell-kubernetes-tutorial')"
```

> output

```
NAME                     ADDRESS/RANGE   TYPE      PURPOSE  NETWORK  REGION    SUBNET  STATUS
abell-kubernetes-tutorial  XX.XXX.XXX.XXX  EXTERNAL                    us-west1          RESERVED
```

## Compute Instances

The compute instances in this lab will be provisioned using [Ubuntu Server](https://www.ubuntu.com/server) 20.04, which has good support for the [containerd container runtime](https://github.com/containerd/containerd). Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane:

```
for i in 0 1 2; do
  gcloud compute instances create abell-controller-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-2004-lts \
    --image-project ubuntu-os-cloud \
    --machine-type e2-standard-2 \
    --private-network-ip 10.240.0.1${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags abell-kubernetes-tutorial,controller
done
```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod-cidr` instance metadata will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.

Create three compute instances which will host the Kubernetes worker nodes:

```
for i in 0 1 2; do
  gcloud compute instances create abell-worker-${i} \
    --async \
    --boot-disk-size 200GB \
    --can-ip-forward \
    --image-family ubuntu-2004-lts \
    --image-project ubuntu-os-cloud \
    --machine-type e2-standard-2 \
    --metadata pod-cidr=10.200.${i}.0/24 \
    --private-network-ip 10.240.0.2${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags abell-kubernetes-tutorial,worker
done
```

### Verification

List the compute instances in your default compute zone:

```
gcloud compute instances list --filter="tags.items=abell-kubernetes-tutorial"
```

> output

```
NAME          ZONE        MACHINE_TYPE   PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP    STATUS
abell-controller-0  us-west1-c  e2-standard-2               10.240.0.10  XX.XX.XX.XXX   RUNNING
abell-controller-1  us-west1-c  e2-standard-2               10.240.0.11  XX.XXX.XXX.XX  RUNNING
abell-controller-2  us-west1-c  e2-standard-2               10.240.0.12  XX.XXX.XX.XXX  RUNNING
abell-worker-0      us-west1-c  e2-standard-2               10.240.0.20  XX.XX.XXX.XXX  RUNNING
abell-worker-1      us-west1-c  e2-standard-2               10.240.0.21  XX.XX.XX.XXX   RUNNING
abell-worker-2      us-west1-c  e2-standard-2               10.240.0.22  XX.XXX.XX.XX   RUNNING
```

## Configuring SSH Access

SSH will be used to configure the controller and worker instances. When connecting to compute instances for the first time SSH keys will be generated for you and stored in the project or instance metadata as described in the [connecting to instances](https://cloud.google.com/compute/docs/instances/connecting-to-instance) documentation.

Test SSH access to the `abell-controller-0` compute instances:

```
gcloud compute ssh abell-controller-0
```

If this is your first time connecting to a compute instance SSH keys will be generated for you. Enter a passphrase at the prompt to continue:

```
WARNING: The public SSH key file for gcloud does not exist.
WARNING: The private SSH key file for gcloud does not exist.
WARNING: You do not have an SSH key for gcloud.
WARNING: SSH keygen will be executed to generate a key.
Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
```

At this point the generated SSH keys will be uploaded and stored in your project:

```
Your identification has been saved in /home/$USER/.ssh/google_compute_engine.
Your public key has been saved in /home/$USER/.ssh/google_compute_engine.pub.
The key fingerprint is:
SHA256:nz1i8jHmgQuGt+WscqP5SeIaSy5wyIJeL71MuV+QruE $USER@$HOSTNAME
The key's randomart image is:
+---[RSA 2048]----+
|                 |
|                 |
|                 |
|        .        |
|o.     oS        |
|=... .o .o o     |
|+.+ =+=.+.X o    |
|.+ ==O*B.B = .   |
| .+.=EB++ o      |
+----[SHA256]-----+
Updating project ssh metadata...-Updated [https://www.googleapis.com/compute/v1/projects/$PROJECT_ID].
Updating project ssh metadata...done.
Waiting for SSH key to propagate.
```

After the SSH keys have been updated you'll be logged into the `abell-controller-0` instance:

```
Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-1042-gcp x86_64)
...
```

Type `exit` at the prompt to exit the `abell-controller-0` compute instance:

```
$USER@abell-controller-0:~$ exit
```
> output

```
logout
Connection to XX.XX.XX.XXX closed
```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
