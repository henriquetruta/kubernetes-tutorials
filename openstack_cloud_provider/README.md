# Setting OpenStack as a Kubernetes Cloud Provider

Here, we'll show how to set up OpenStack as Kubernetes cloud provider. The main I've used this was to make it possible to create LoadBalancer services without being attached to a public cloud provider such as Google or Amazon.
With OpenStack as cloud providers, Kubernetes is able to request an LB from OpenStack LBaaS API.

# TODO explicar CP

This tutorial assumes you have:

* A running Kubernetes cluster. If you don't, follow these [steps](https://github.com/henriquetruta/kubernetes-tutorials/tree/master/kubeadm)
* SSH access to the kuberentes cluster nodes (master and slaves)
* Access to an OpenStack cloud, and this cloud is reachable by the Kubernetes nodes, i.e. they can access Keystone to authenticate and are able to communicate with Neutron
* A basic understanding of OpenStack resources, such as projects, domains, load balancers
* (Optional): If you want to use Floating IPs to expose your LoadBalancers, you must use a Kubernetes version >= 1.8

## Hands-on

### Conf file

The first step is creating a file that will tell Kubernetes where is your OpenStack auth endpoint and your credentials. We'll call this file `cloud.conf`. This is an example of how this file should look like:

```conf
[Global]
username=user
password=pass
auth-url=https://<openstack_endpoint>:5000/v3
tenant-id=c869168a828847f39f7f06edd7305637
domain-id=2a73b8f597c04551a0fdc8e95544be8a

[LoadBalancer]
subnet-id=6937f8fa-858d-4bc9-a3a5-18d2c957166a
```

Where:

* `username` and `password` speak for themselves
* `auth-url` is the keystone endpoint. Find it in horizon in Access and Security > API Access > Credentials
* `tenant-id` is the id of the project (tenant) you want to create your resources on
* `domain-id`is the id of the domain your user belongs to
* `subnet-id` is the id of the subnet you want to create your loadabalancer on. Get it in Network > Networks and click on the respective network to get its subnets

There are other sections we're not going to cover here, such as the one that provides volumes to Kubernetes via Cinder:

```conf
[BlockStorage]
bs-version=v2
```

At the moment I'm writing this, I haven't found any official    documentation explaining those variables and others, so you better trust me.

Then, copy this file to your master node at the `/etc/kubernetes` dir.

```bash
scp cloud.conf user@master-ip:/etc/kubernetes/
```

### Restarting kube-controller-manager

Once you have the credentials file, you must update the `kube-controller-manager` manifest to point it to your credentials file and tell it you want OpenStack as cloud provider.

To do so, SSH to your master node and edit the following file:

```bash
# Use vim or any other editor you like
vim /etc/kubernetes/manifests/kube-controller-manager.yaml
```

There, add two parameters to the `spec/containers/command` section: `cloud-provider` and `cloud-config`. The section will look like:

```yaml
...
spec:
  containers:
  - command:
    - kube-controller-manager
    - --use-service-account-credentials=true
    - --controllers=*,bootstrapsigner,tokencleaner
    - --root-ca-file=/etc/kubernetes/pki/ca.crt
    - --service-account-private-key-file=/etc/kubernetes/pki/sa.key
    - --address=127.0.0.1
    - --kubeconfig=/etc/kubernetes/controller-manager.conf
    - --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
    - --cluster-signing-key-file=/etc/kubernetes/pki/ca.key
    - --leader-elect=true
    - --cloud-provider=openstack
    - --cloud-config=/etc/kubernetes/cloud.conf
...
```

Now, you should apply this same change in `kubelet`, as the next section explains.

### Restarting kubelet

`kubelet` is a daemon that runs in every Kubernetes node. So, this step must be made in *all nodes of the cluster*, including the master node.

The `kubelet` runs as a `systemd` process. You can see the place its conf file is located by running:

```bash
$ systemctl cat kubelet
# /lib/systemd/system/kubelet.service
[Unit]
Description=kubelet: The Kubernetes Node Agent
Documentation=http://kubernetes.io/docs/

[Service]
ExecStart=/usr/bin/kubelet
Restart=always
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target

# /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--kubeconfig=/etc/kubernetes/kubelet.conf --require-kubeconfig=true"
Environment="KUBELET_SYSTEM_PODS_ARGS=--pod-manifest-path=/etc/kubernetes/manifests --allow-privileged=true"
Environment="KUBELET_NETWORK_ARGS=--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin"
Environment="KUBELET_DNS_ARGS=--cluster-dns=10.96.0.10 --cluster-domain=cluster.local"
Environment="KUBELET_AUTHZ_ARGS=--authorization-mode=Webhook --client-ca-file=/etc/kubernetes/pki/ca.crt"
Environment="KUBELET_CADVISOR_ARGS=--cadvisor-port=0"
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_SYSTEM_PODS_ARGS $KUBELET_NETWORK_ARGS $KUBELET_DNS_ARGS $KUBELET_AUTHZ_ARGS
...

```

You need to edit in this case the following file:

```bash
vim /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```

Edit the `KUBELET_KUBECONFIG_ARGS` line. In the end, it should have the `cloud` arguments, like the following:

```bash
Environment="KUBELET_KUBECONFIG_ARGS=--kubeconfig=/etc/kubernetes/kubelet.conf --require-kubeconfig=true --cloud-provider=openstack --cloud-config=/etc/kubernetes/cloud.conf"
```

Save the file and restart `kubelet` with the following commands:

```bash
systemctl daemon-reload
systemctl restart kubelet
```

### Testing your cloud provider

## External cloud-providers