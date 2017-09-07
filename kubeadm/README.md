# Creating a Kubernetes cluster using `kubeadm`

Here, we'll show the easiest way  to create a single node Kubernetes cluster on Ubuntu 16.04. This can be dones in VMs or physical machines.

## Notes

* `kubeadm` is in alpha state and is not supposed to be used in production. This cluster should be used for tests/development.
* In production, it is recommended that the master node be exclusive for the k8s pods, and not for regular pods, as we'll do here with `taint`

## Hands-on

SSH to your machine. And preferrably run all the following commands as `root` user. Just do:

`sudo su -`

### Install some packages

Add some source repos and install packages such as `kubelet`, `docker` and `kubeadm` itself. Just run the following lines:

```bash
apt-get update && apt-get install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y docker.io kubelet kubeadm kubectl kubernetes-cni 
```

### Run `kubeadm`

Run:

```bash
kubeadm init
```


Wait a couple minutes and then your cluster is _almost_ ready.

In most cases, the default configuration of `init` should work. If you want to install a verstion different than the stable, you must provide the option `--kubernetes-version` and provide a valid version. See [releases page](https://github.com/kubernetes/kubernetes/releases).

For example, to install the version `v1.8.0-alpha.3`, just run it with:

```bash
kubeadm init --kubernetes-version='v1.8.0-alpha.3'
```

The output of `init` should look like this:

#### Accessing the cluster

Once completed the `init`, the cluster should be ready and accessible via [`kubectl`](https://kubernetes.io/docs/user-guide/kubectl-overview/), the Kubernetes CLI. By default, it looks for credentials in `~/.kube/config` file. If you don't have anything in there, you should hit an error. 

The easisest way to get a credentials file is copying the `admin.conf` created by `kubeadm` and place it in the default location. Do it with:

```bash
mkdir ~/.kube
cp /etc/kubernetes/admin.conf ~/.kube/config
```

Now, you should be able to run commands such as `kubectl get pods --all-namespaces`.

### Tainting the Master node

As stated before, master node should not receive nothing but the main Kubernetes pods, such as `api-server`,  `kube-controller-manager`, `cni`, `kube-dns` and others. But as this is a dev environment, there should be no problem in marking this node as schedulable, so you can have a single node fully working cluster. 

Enable scheduling of regular pods in the master with:

```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```

### Installing a CNI plugin

Kubernetes needs a network controller, that is self-hosted and installed as a plugin. You must install a pod network add-on so that your pods can communicate with each other. Here, we'll use Calico. To install it in a single command, just run:

```bash
kubectl apply -f https://docs.projectcalico.org/v2.5/getting-started/kubernetes/installation/hosted/kubeadm/1.6/calico.yaml
```

This command works for clusters of version >= 1.6. If your version is older, look for it in the [calico page](https://docs.projectcalico.org/v2.5/getting-started/kubernetes/installation/hosted/kubeadm)

If you want to see other Network plugins, look [here](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#pod-network).

### Check if it works

Once you've installed the network plugin, you now only need to check whether all pods are running correctly. Monitor it with:

```bash
watch kubectl get pods --all-namespaces
```

And wait for all pods to be `RUNNING`.

### Optional: Join other nodes

If you want more than one node in your cluster, join others after you finish the instalation of the master. In the output of `kubeadm init` you should see the `join` command. More info [here](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#joining-your-nodes).

### Conclusion

And that's it! You have a running Kubernetes cluster. To more details on this process, check the [official documentation](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/).