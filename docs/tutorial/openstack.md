# Getting Started with kops on OpenStack

**WARNING**: OpenStack support on kops is currently **beta**, which means that OpenStack support is in good shape. However, always some small things might change before we can really say that it is production ready.

The tutorial shown on this page works with `kops` v1.12 and above.

## Source your openstack RC
The Cloud Config used by the kubernetes API server and kubelet will be constructed from environment variables in the openstack RC file. The openrc.sh file is usually located under `API access`.

```bash
source openstack.rc
```

If you are authenticating by username,  `OS_DOMAIN_NAME` or `OS_DOMAIN_ID` must manually be set.
```bash
export OS_DOMAIN_NAME=<USER_DOMAIN_NAME>
```

## Environment Variables

It is important to set the following environment variables:

```bash
export KOPS_STATE_STORE=swift://<bucket-name> # where <bucket-name> is the name of the Swift container to use for kops state

```

If your OpenStack does not have Swift you can use any other VFS store, such as S3.

## Creating a Cluster

```bash
# to see your etcd storage type
openstack volume type list

# coreos (the default) + flannel overlay cluster in Default
kops create cluster \
  --cloud openstack \
  --name my-cluster.k8s.local \
  --state ${KOPS_STATE_STORE} \
  --zones nova \
  --network-cidr 10.0.0.0/24 \
  --image <imagename> \
  --master-count=3 \
  --node-count=1 \
  --node-size <flavorname> \
  --master-size <flavorname> \
  --etcd-storage-type <volumetype> \
  --api-loadbalancer-type public \
  --topology private \
  --bastion \
  --ssh-public-key ~/.ssh/id_rsa.pub \
  --networking weave \
  --os-ext-net <externalnetworkname>

# to update a cluster
kops update cluster my-cluster.k8s.local --state ${KOPS_STATE_STORE} --yes

# to delete a cluster
kops delete cluster my-cluster.k8s.local --yes
```

#### Optional flags
* `--os-kubelet-ignore-az=true` Nova and Cinder have different availability zones, more information [Kubernetes docs](https://kubernetes.io/docs/concepts/cluster-administration/cloud-providers/#block-storage)
* `--os-octavia=true` If Octavia Loadbalancer api should be used instead of old lbaas v2 api.
* `--os-dns-servers=8.8.8.8,8.8.4.4` You can define dns servers to be used in your cluster if your openstack setup does not have working dnssetup by default


# Compute and volume zone names does not match
Some of the openstack users do not have compute zones named exactly the same than volume zones. Good example is that there are several compute zones for instance `zone-1`, `zone-2` and `zone-3`. Then there is only one volumezone which is usually called `nova`. By default this is problem in kops, because kops assumes that if you are deploying things to `zone-1` there should be compute and volume zone called `zone-1`.

However, you can still get kops working in your openstack by doing following:

**Create cluster using your compute zones**

```
kops create cluster \
  ...
  --zones zone-1,zone-2,zone-3 \
  ...
```

**After you have initialized the configuration you need to edit configuration**

```
kops edit cluster my-cluster.k8s.local
```

Edit `ignore-volume-az` to `true` and `override-volume-az` according to your cinder az name.

Example (volume zone is called `nova`):

```
spec:
  ...
  cloudConfig:
    openstack:
      blockStorage:
        ignore-volume-az: true
        override-volume-az: nova
  ...
```

**Finally execute update cluster**

```
kops update cluster my-cluster.k8s.local --state ${KOPS_STATE_STORE} --yes
```

Kops should create instances to all three zones, but provision volumes from the same zone.

# Using external cloud controller manager
If you want use [External CCM](https://github.com/kubernetes/cloud-provider-openstack) in your installation, this section contains instructions what you should do to get it up and running.

Enable featureflag:

```
export KOPS_FEATURE_FLAGS=+EnableExternalCloudController
```

Create cluster without `--yes` flag (or modify existing cluster):

```
kops edit cluster <cluster>
```

Add following to clusterspec:

```
  cloudControllerManager:
    image: jesseh/occm:latest <- you can use this or compile your own
    logLevel: 2
```

Finally

```
kops update cluster --name <cluster> --yes
```

# Using OpenStack without lbaas
Some OpenStack installations does not include installation of lbaas component. That is why we have added very-experimental support of installing OpenStack kops without lbaas. You can install it using:

```
kops create cluster \
  --cloud openstack \
  ... (like usually)
  --api-loadbalancer-type=""
```

The biggest problem currently when installing without loadbalancer is that kubectl requests outside cluster is always going to first master. External loadbalancer is one option which can solve this issue.
