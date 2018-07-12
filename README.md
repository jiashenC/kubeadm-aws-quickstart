# Deploy product to Kubernetes service on AWS.
This Readme is about how to deploy application to distributed Kubernetes 
service on AWS. I use `kubeadm` tool for the operation. 

## Prerequisites
**Docker**
```bash
sudo apt update
sudo apt install docker.io
```

**Kubeadm, Kubectl, Kubelet**
```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo cat >/etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
```

The version we successfully deploy are 
```bash
docker===1.13.1
kubectl kubeadm kubelet===1.10.3
```
Some one points out newer **Docker** version may fail the
deployment, but we haven't tested it out.

**AWS** 
We previously had trouble of successfully initializing
the `master` because AWS blocks the traffic, so
we open all ports to all traffic for both **inbound** and
**outbound**. 

## Deployment

### Create docker images on all nodes
Simply `make docker` on all nodes because the image pulling registry
has not been setup yet.

### Initialize a master
Dedicate 1 machine as `master` and initialize `master` on
that machine.
```bash
# Get the public address on AWS server.
master_ip=$(curl http://169.254.169.254/latest/meta-data/public-ipv4)
sudo kubeadm init --apiserver-advertise-address $master_ip
```
The only warning we get from **preflight** check is 
> [WARNING FileExisting-crictl]: crictl not found in system path

If you see other warnings, you should clean the config 
```bash
sudo kubeadm reset
```
then try to first fix the warning and then redo the 
initialization step.  

### Save command for later use
If you successfully initialize a `master`, you will see a command similar as
```bash
kubeadm join <ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<sha256>
```
You should save this command, because you need this to join `node` to the Kubernetes cluster.

### Configure `kubectl`
```bash
sudo cp /etc/kubernetes/admin.conf $HOME/
sudo chown $(id -u):$(id -g) $HOME/admin.conf
export KUBECONFIG=$HOME/admin.conf
```

### Configure pod network
```bash
# use third party plugin for configuring VPC network
sudo sysctl net.bridge.bridge-nf-call-iptables=1
export kubever=$(kubectl version | base64 | tr -d '\n')
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$kubever"

# set proxy server config
master_ip=$(curl http://169.254.169.254/latest/meta-data/public-ipv4)
export no_proxy=$master_ip
```

After this step, you should be able to see your `master` is up and is **Ready**.
```bash
kubectl get nodes
```

### Join `node`
Dedicate several machines as `node` and join them to Kubernetes cluster we just created. 
Join the cluster by using the previously saved command
```bash
kubeadm join <ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<sha256>
```

Update proxy server config on those `node`
```bash
master_ip=$(curl http://169.254.169.254/latest/meta-data/public-ipv4)
export no_proxy=$master_ip
```

### Start pod on `master`
```bash
kubectl create -f master.yml
kubectl create -f db.yml
```

After this step, you should be able to see pods are created
```bash
kubectl get pods
```

The status of pod `initdb` should be `completed`. Other pods should be `running`.

### Start replicas from `master`
```bash
kubectl create -f replica.yml
```

### Test the service
You can access our service under [https://master_ip:30443](https://0.0.0.0:30443).

## Addon

### Expose kubernetes-dashboard serivce
Expose `kubernetes-dashboard` service helps to manage all `pods`, `services`, `deployments` etc. 
* Start the  `kubernetes-dashboard` service. 
	```bash
	kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
	```
	
* Expose traffic through `NodePort` service.
	```bash
	kubectl -n kube-system edit service kubernetes-dashboard
	```
	
* Change `type` from `Cluster` to `NodePort`.
* Access the dashboard at `<node ip>`:`<node port>`. The `node ip` is not necessary the ip of `master` node when you have multiple nodes for `k8s` cluster. 
* Use token to access dashboard. Token is stored in `secret`.
	```bash
	kubectl -n kube-system describe secret namespace-controller-token-*
	```

## TroubleShooting

### **kube-dns** pending
`kube-dns` service is important for different services inside `k8s` cluster to figure out services ip. Before you start the deployment, you need to make sure it works. There are 2 factors could cause `kube-dns` stay at pending. 

1. Fix `cgroup` setting.
    * Add `Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=cgroupfs"` to `/etc/systemd/system/kubelet.service.d/10-kubeadm.conf`.
	* Restart the `kubelet` service and apply the change.

		```bash
		sudo systemctl deamon-reload
		sudo systemctl restart kubelet
		```
	
2. Clean `weave-net` setting.

	```bash
	sudo weave launch
	sudo weave reset
	sudo rm /opt/cni/bin/weave-*
	```
	
### **AWS** configuration

1. Add **AWS** credential first by using client tool.
2. Add below configuration to `/etc/kubernetes/cloud-config.conf` file. You also need to tag all **AWS** EC2 instances with `key`:`KubernetesCluster`, `value`:`k8s`(in my case). Otherwise, **AWS** is not able to find the instances to start the cluster.
	```bash
	[Global]
	KubernetesClusterTag=k8s
	KubernetesClusterID=k8s
	```
	
3. Provide `cloud-provider` flag to cluster.
	```bash
	apiVersion: kubeadm.k8s.io/v1alpha1
	kind: MasterConfiguration
	cloudProvider: aws
	kubernetesVersion: 1.10.3
	```

4. Create new `kubelet` conf `/etc/systemd/system/kubelet.service.d/20-cloud-provider.conf` file with **AWS** tag. 
	```bash
	Environment="KUBELET_EXTRA_ARGS=--cloud-provider=aws --cloud-config=/etc/kubernetes/cloud-config.conf
	```
	
5. Make sure the role attached to instances have necessary permission.
	```bash
	{
	  "Version": "2012-10-17",
	  "Statement": [
	    {
	      "Effect": "Allow",
	      "Action": "s3:*",
	      "Resource": [
	        "arn:aws:s3:::kubernetes-*"
	      ]
	    },
	    {
	      "Effect": "Allow",
	      "Action": "ec2:Describe*",
	      "Resource": "*"
	    },
	    {
	      "Effect": "Allow",
	      "Action": "ec2:AttachVolume",
	      "Resource": "*"
	    },
	    {
	      "Effect": "Allow",
	      "Action": "ec2:DetachVolume",
	      "Resource": "*"
	    },
	    {
	      "Effect": "Allow",
	      "Action": ["ec2:*"],
	      "Resource": ["*"]
	    },
	    {
	      "Effect": "Allow",
	      "Action": ["elasticloadbalancing:*"],
	      "Resource": ["*"]
	    }  ]
	} 
	```
