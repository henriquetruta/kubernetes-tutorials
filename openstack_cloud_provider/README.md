# Setting OpenStack as a Kubernetes Cloud Provider

Here, we'll show how to set up OpenStack as Kubernetes cloud provider. The main reason I've used this was to make it possible to create LoadBalancer services without being attached to a public cloud provider such as Google or Amazon.
With OpenStack as cloud providers, Kubernetes is able to request an LB from OpenStack LBaaS API.

This tutorial assumes you have:

* A running Kubernetes cluster. If you don't, follow these [steps](https://github.com/henriquetruta/kubernetes-tutorials/tree/master/kubeadm)
* SSH access to the kubernetes cluster nodes (master and slaves)
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
* `subnet-id` is the id of the subnet you want to create your loadbalancer on. Get it in Network > Networks and click on the respective network to get its subnets

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

### Checking if it worked

There are several ways to see if your changes were successfully applied. One of them is to look for the processes running in the nodes. For example, in the any node you can see whether the `kubelet` is running with the `cloud` properties you set. Check it with:

```bash
$ ps xau | grep /usr/bin/kubelet
root     21647  4.5  4.5 421688 93740 ?        Ssl  15:06   0:08 /usr/bin/kubelet --kubeconfig=/etc/kubernetes/kubelet.conf --require-kubeconfig=true --cloud-provider=openstack --cloud-config=/etc/kubernetes/cloud.conf --pod-manifest-path=/etc/kubernetes/manifests --allow-privileged=true --network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin --cluster-dns=10.96.0.10 --cluster-domain=cluster.local --authorization-mode=Webhook --client-ca-file=/etc/kubernetes/pki/ca.crt --cadvisor-port=0
```

In this case, you can see that both flags were provided. `kubelet` must see those changes in all nodes of your cluster.

To see the changes in `kube-controller-manager` you can see by looking at the process, or you can also investigate the output of:

```bash
kubectl describe po kube-controller-manager -n kube-system
```

Again, take a look if your properties were properly applied. If so, congrats! Your kubernetes cluster now has OpenStack as its cloud provider.

### Testing your cloud provider

First, create a deployment, i.e. run an application on your Kubernetes cluster. At this repo, you have an application called `microbot` that basically just prints the name of the pod that served the request. The files are available in this repo. Create it with:

```bash
kubectl create -f resources/microbot_deploy.yaml
```

Wait for all pods to be ready. And then, you can expose this application, i.e. create a service that will make this application available outside of the cluster. This service will be created with the `LoadBalancer` type, which is only possible as we have a cloud provider in our Kubernetes cluster.

What happens here is that Kubernetes delegates the creation of the Load Balancer to the cloud provider API, that in this case is to Neutron's LBaaS API.

To expose this service, we have two options. It can be in a private or public network.

#### Private LB

The first is creating the Load Balancer in the private network, with the `subnet-id` provided in the `cloud.conf` file. To do so, just create it with the following command:

```bash
kubectl create -f resources/microbot_svc_private.yaml
```

This file is very straightforward if you know the service concept. We are basically making the deployment available at the port 80, which will have requests handled by OpenStack's LB. List the services and wait for it to get an External IP.

Once it gets an external IP, you can curl it and see which pod has served the request. Remember that as this is an internal LB, you should be in a point where this network is reachable.

```bash
$ kubectl get svc microbot
NAME       CLUSTER-IP     EXTERNAL-IP        PORT(S)        AGE
microbot   10.106.83.25   10.11.4.52         80:30080/TCP   1m

$ kubectl get po
NAME                        READY     STATUS    RESTARTS   AGE
microbot-4136705477-c45fs   1/1       Running   0          19m
microbot-4136705477-l9m52   1/1       Running   0          19m
microbot-4136705477-mdx7w   1/1       Running   0          19m

$ curl 10.11.4.52
<!DOCTYPE html>
<html>
  <style type="text/css">
    .centered
      {
      text-align:center;
      margin-top:0px;
      margin-bottom:0px;
      padding:0px;
      }
  </style>
  <body>
    <p class="centered"><img src="microbot.png" alt="microbot"/></p>
    <p class="centered">Container hostname: microbot-4136705477-l9m52</p>
  </body>
</html>
```

And you can see that `microbot-4136705477-l9m52` is in our pods list.

#### Public LB

The other option to expose the service, is to make it publicly available, which happens if you have free Floating IPs in your OpenStack project. If you do have, you can create the service using the `microbot_svc_public` file. The only difference here is that we tell Kubernetes that we don't want a service that is only internal and we specify the public network to expose the service. If you open this file, you'll see the two new fields, passed through annotations:

```yaml
$ cat resources/microbot_svc_public.yaml
...
  annotations:
    service.beta.kubernetes.io/openstack-internal-load-balancer: "false"
    loadbalancer.openstack.org/floating-network-id: "9cdf8226-fd6c-499a-994e-d12e51a498af"
...
```

> Note: This is only available in Kubernetes versions >=1.8, which in the time I'm writing this is in 1.8.0 alpha 3.

Here, you just need to edit the `floating-network-id` field to have the ID of your public network, and then run:

```bash
$ kubectl create -f resources/microbot_svc_public.yaml
service "microbot" created

# Look for the Load Balancer Ingress field, which is the same as the External IP
$ kubectl describe svc microbot | grep Ingress
LoadBalancer Ingress: 10.11.4.52, 150.46.85.53
```

You can see here that my Load Balancer got two IP addresses. The first, private, and the second, a public one. If you `curl` either of them, you should get the same result.

And that's it! You exposed your service as Load Balancer, using OpenStack as cloud provider!

## External cloud-providers

Kubernetes community is moving towards removing the cloud providers specific code from the [main tree](https://github.com/kubernetes/kubernetes/tree/master/pkg/cloudprovider/providers) as it's now. The idea here is to have each cloud provider maintaining its code in a separate repository. What's going to change for the kubernetes admin is that instead of passing `cloud-provider=<provider>` it will pass it `cloud-provider=external` and pass the credentials and other attributes to a new component called `cloud-controller-manager`.

At the moment I'm writing this, there is no support for OpenStack as an external cloud provider yet. I've described my efforts on [this issue](https://github.com/kubernetes/kubernetes/issues/52276).

If you want to keep track this migration to external cloud providers, which will happen at most in release 1.9, follow it [here](https://github.com/kubernetes/kubernetes/issues?utf8=%E2%9C%93&q=is%3Aissue%20is%3Aopen%20label%3Aarea%2Fcloudprovider%20) and [here](https://docs.google.com/document/d/1m4Kvnh_u_9cENEE9n1ifYowQEFSgiHnbw43urGJMB64/edit#heading=h.rkwrsiw228ev).

## Issues

I've found some problems during this, which I intend to solve in a short future

* Using OpenStack as `external`, which will be the default soon
* We should have a way of telling Kubernetes we don't want both private and public IPs, but only public
* Documentation is still poor (feature in development, I suppose)
* (Federation) Both External IPs, private and public are written in DNS zone
* (Federation) When service is unhealthy, DNS records aren't properly managed
* (Federation) Annotation of floating-network should be passed to federation control plane, which means that it can't work if we have more than one OpenStack cluster, as all clusters would share the same network-id
* (Federation) Annotation of floating-network should be passed to all clusters in federation, even if they're not OpenStack

Thanks! If anything, contact me at henriquecostatruta@gmail.com or `htruta` in Kubernetes Slack.
