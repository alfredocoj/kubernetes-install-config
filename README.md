# kubernetes-install-config
Kubernetes installation and configuration tutorial.

## Instalação do Kubernetes no Debian 9 (Strech)

Kubernetes Cluster Basic Architecture:

- Node Master: 
    
    * IP: 192.168.6.1
    * CPU: 4
    * Memory: 10GB
    * SO: Debian 9 (Strech)

- Node Agent 01:

    * IP: 192.168.6.2
    * CPU: 17
    * Memory: 50GB
    * SO: Debian 9 (Strech)

- Node Agent 02:

    * IP: 192.168.6.3
    * CPU: 9
    * Memory: 50GB
    * SO: Debian 9 (Strech)


### Configuration of Hostname

In Node Master:

```
sudo hostnamectl set-hostname k8s-master
```

In Node 01:

```
sudo hostnamectl set-hostname k8s-node-01
```

In Node 02:

```
sudo hostnamectl set-hostname k8s-node-02
```

Once the correct hostname has been configured on each host, populate each node (master and agent nodes) with the configured values.
```
nano /etc/hosts
```

and add
```
192.168.6.1 k8s-master 
192.168.6.2 k8s-node-01 
192.168.6.3 k8s-node-02
```

### Prerequisites for master and nodes

Update system packages to the latest version on all nodes:
```
sudo apt-get update
sudo apt-get upgrade
```

Add user to manage Kubernetes cluster:
```
sudo useradd -s /bin/bash -m k8s-admin
sudo passwd k8s-admin
sudo usermod -aG sudo k8s-admin
echo "k8s-admin ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/k8s-admin
su - k8s-admin
sudo su -
    
root@k8s-hostname:~#
```
- Installing the docker-ce

```
sudo apt-get remove docker docker-engine docker.i
sudo apt-get install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository \
 "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
 $(lsb_release -cs) \
 stable"
sudo apt-get update
apt-get install -y docker-ce=18.06.0~ce~3-0~debian
sudo usermod -aG docker k8s-admin

```

run these commands to verify that user k8-admin is in the docker group

```

root@k8s-hostname:~# su - k8s-admin
k8s-admin@k8s-hostname:~$ id -nG
```
The command `id -nG` should return something like the following:

```
k8s-admin root docker
```


* Configuring Cgroup Drivers

Setup daemon.
```
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

mkdir -p /etc/systemd/system/docker.service.d
```
Then restart the daemon and docker service.

```
# Restart docker.
systemctl daemon-reload
systemctl restart docker

```

Or do it this way:

```
mkdir -p /etc/systemd/system/docker.service.d
cd /etc/systemd/system/docker.service.d
cat service-overrides.conf 
[Service]
EnvironmentFile=-/etc/default/docker
EnvironmentFile=-/etc/default/docker-storage
ExecStart=
ExecStart=/usr/bin/dockerd $OPTIONS \
        $DOCKER_STORAGE_OPTIONS --exec-opt native.cgroupdriver=systemd

```

Then restart the daemon and docker service.

```
# Restart docker.
systemctl daemon-reload
systemctl restart docker

```


```
cat /etc/systemd/system/kubelet.service.d/10-kubeadm.conf 
# Note: This dropin only works with kubeadm and kubelet v1.11+
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
EnvironmentFile=-/etc/default/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS --cgroup-driver=systemd
```
### Installing and configuring Master

Nthis example the master is the server 192.168.6.1.

```

cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF

curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

sudo apt update

sudo apt install kubectl kubelet kubeadm kubernetes-cni
```

Confirm that the packages have been installed:

```

$ which kubelet
/usr/bin/kubelet
$ which kubeadm
/usr/bin/kubeadm

```

Official documentation recommends that swap mode be disabled.

```
sudo swapoff -a
```

Run the following command to update fstab so that swapping will be disabled after reboot.

```
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
``` 
*Obs.:* The above command comments all swap entries in the file `/etc/fstab`.


Para iniciar o cluster execute com o usuário k8-admin

```
kubeadm init
```

If all went well, the above command should print something like:

```
---
Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 192.168.2.2:6443 --token 9y4vc8.h7jdjle1xdovrd0z --discovery-token-ca-cert-hash sha256:cff9d1444a56b24b4a8839ff3330ab7177065c90753ef3e4e614566695db273c
```

If something goes wrong, try resetting the cluster.
```
# kubeadm reset -f

# iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
```

Do what you ask for when returning the command `kebeadm init`:

```
su - k8s-admin
mkdir -p $HOME/.k8s
sudo cp -i /etc/kubernetes/admin.conf $HOME/.k8s/config
sudo chown $(id -u):$(id -g) $HOME/.k8s/config
export KUBECONFIG=$HOME/.k8s/config
echo "export KUBECONFIG=$HOME/.k8s/config" | tee -a ~/.bashrc
```

- Creating a POD Network for the Cluster (Network Type: Weave Net POD Network)
 
Run the following command with user k8-admin (to switch users, run  `su - k8s-admin`):

```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

To verify that the Weave network has been created, run:

```
kubectl get pod -n kube-system | grep weav
```

### Installing and Configuring kuberneste on Agent Nodes

In this example, the agent nodes are 192.168.6.2, 192.168.6.3. Perform the following steps on both servers.



First, make sure that the **Prerequisites for master and nodes** step has been performed for all nodes.


Then run the following commands to install `kubectl`, `kubelet`, `kubeadm`, `kubernetes-cni`.

```

cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF

curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

sudo apt update

sudo apt install kubectl kubelet kubeadm kubernetes-cni
```

Confirm that the packages have been installed:

```

$ which kubelet
/usr/bin/kubelet
$ which kubeadm
/usr/bin/kubeadm

```

All set up, it is necessary to join each node to the cluster created on the master (note that the following command is output from the command `kubeadm init` executed on the master node)
```
sudo kubeadm join 192.168.6.1:6443 --token mu0y4k.cbbs5f0pskuytuhb --discovery-token-ca-cert-hash sha256:790e9cdacab9e2158d31d7ed89390f089e499587380a66b8db66d03915b6aba9
```

The command output should look something like this:

```
---
[tlsbootstrap] Waiting for the kubelet to perform the TLS Bootstrap...
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "k8s-node-02" as an annotation

This node has joined the cluster:
* Certificate signing request was sent to master and a response
  was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the master to see this node join the cluster.
```

### Validating Clustering
Access the master node server, and with user k8-admin, run the following command:


```
k8s-admin@k8s-master:~$ kubectl get nodes
```

The return should be something like:

```
NAME          STATUS    ROLES     AGE       VERSION
k8s-master    Ready     master    35m       v1.11.0
k8s-node-01   Ready     <none>    2m        v1.11.0
k8s-node-02   Ready     <none>    1m        v1.11.0
```

On both nodes, Weave Net must have been configured. It can be verified by running the commands on each agent nodes.

Node 1:
```
root@k8s-node-01:~# ip ad | grep weave


6: weave: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1376 qdisc noqueue state UP group default qlen 1000
    inet 10.44.0.0/12 brd 10.47.255.255 scope global weave
9: vethwe-bridge@vethwe-datapath: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1376 qdisc noqueue master weave state UP group default

```

Node 2:
```
root@k8s-node-02:~# ip ad | grep weave
6: weave: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1376 qdisc noqueue state UP group default qlen 1000
    inet 10.47.0.0/12 brd 10.47.255.255 scope global weave
9: vethwe-bridge@vethwe-datapath: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1376 qdisc noqueue master weave state UP group default

```

### Subindo uma aplicação para teste

Creating a pod test to verify that the cluster runs as expected:

```
k8s-admin@k8s-master:~$ kubectl create namespace test-namespace
namespace/test-namespace created
```

Create the deploy settings file `http-app-deployment.yml` as follows:

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: http-app
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: http-app
    spec:
      containers:
      - name: http-app
        image: katacoda/docker-http-server:latest
        ports:
```

Creating a namespace (a deployment organizer):

```
k8s-admin@k8s-master:~$ kubectl create namespace test-namespace
    namespace/test-namespace created
```

After the namespace is created, create a pod using the deployment object defined earlier. -n is used to specify the namespace. We expect three pods to be created as our replica value is 3.

```
k8s-admin@k8s-master:~$ kubectl create -n test-namespace -f http-app-deployment.yml
    deployment.extensions/http-app created
```

Confirming the creation of deployment configurations:

```
k8s-admin@k8s-master:~$ kubectl -n test-namespace get deployments
NAME       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
http-app   3         3         3            3           1m

$ kubectl -n test-namespace get pods
NAME                       READY     STATUS    RESTARTS   AGE
http-app-97f76fcd8-68pxg   1/1       Running   0          1m
http-app-97f76fcd8-f9bdk   1/1       Running   0          1m
http-app-97f76fcd8-vgmq7   1/1       Running   0          1m
```

With the deployment created, we can use `kubectl` to create a service that exposes Pods on a specific port. An alternative method is to define a Service object with YAML. Below is our definition of service.

```
k8s-admin@k8s-master:~$ cat http-app-service.yml 
apiVersion: v1
kind: Service
metadata:
  name: http-app-svc
  labels:
    app: http-app
spec:
  type: NodePort
  ports:
  - port: 80
    nodePort: 30080
  selector:
    app: http-app
```

Creating service:

```
k8s-admin@k8s-master:~$ kubectl -n test-namespace create -f http-app-service.yml 
    service/http-app-svc created
```

The service will be available on IP Cluster and port 30080. To obtain IP CLUSTER, run:
```
k8s-admin@k8s-master:~$ kubectl -n test-namespace get svc
NAME           TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
http-app-svc   NodePort   10.5.45.208   <none>        80:30080/TCP   1m

```

### After installation

Enable shell autocomplete for kubectl commands by running the command:
```
source <(kubectl completion bash)
```

To add autocomplete kubectl to your profile, it is automatically loaded in the future.

```
echo "source <(kubectl completion bash)" >> ~/.bashrc
```


### Kubernetes Dashboard Installation

### 1. Deploy the Kubernetes Dashboard to the cluster:

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```

Result:

```
secret "kubernetes-dashboard-certs" created
serviceaccount "kubernetes-dashboard" created
role "kubernetes-dashboard-minimal" created
rolebinding "kubernetes-dashboard-minimal" created
deployment "kubernetes-dashboard" created
service "kubernetes-dashboard" created
```

### 2. Deploy heapster to enable container cluster monitoring and performance analysis on the cluster:
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/heapster.yaml
```

Result:
```
serviceaccount "heapster" created
deployment "heapster" created
service "heapster" created
```

3. Deploy the influxdb backend to the heapster in the cluster:

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/influxdb.yaml
```

Result:

```
deployment "monitoring-influxdb" created
service "monitoring-influxdb" created
```

4.Create the heapster cluster role binding for the dashboard:

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/rbac/heapster-rbac.yaml
```

Result:

```
clusterrolebinding "heapster" created
```

5. fsefd

To run **heapster** you need to change the deployment configuration file. Change as follows:

```

"containers": [
          {
            "name": "heapster",
            "image": "k8s.gcr.io/heapster-amd64:v1.5.4",
            "command": [
              "/heapster",
              "--source=kubernetes:https://kubernetes.default?useServiceAccount=true&kubeletHttps=true&kubeletPort=10250&insecure=true",
              "--sink=influxdb:http://monitoring-influxdb.kube-system.svc:8086",
              "--metric_resolution=30s"
            ],
            "resources": {},
            "terminationMessagePath": "/dev/termination-log",
            "terminationMessagePolicy": "File",
            "imagePullPolicy": "IfNotPresent"
          }
        ]

```

And then create a role that allows heapster to operate and link it to the heapster service account:

```sh

$ cat role-heapster.yaml


kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: node-stats-full
rules:
- apiGroups: [""]
  resources: ["nodes/stats"]
  verbs: ["get", "watch", "list", "create"]
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: heapster-node-stats
subjects:
- kind: ServiceAccount
  name: heapster
  namespace: kube-system
roleRef:
  kind: ClusterRole
  name: node-stats-full
  apiGroup: rbac.authorization.k8s.io
```

Run the command `kubectl apply -f role-heapster.yaml` and it will return something like:

```sh
clusterrole.rbac.authorization.k8s.io/node-stats-full created
clusterrolebinding.rbac.authorization.k8s.io/heapster-node-stats created
```

Now just follow the next steps.

References:

[Fonte 1](https://github.com/awslabs/amazon-eks-ami/issues/128)

[Fonte 2](https://github.com/kubernetes/dashboard/issues/3147)


### Step 2: Create an eks-admin Service Account and Cluster Role Linking

1. Create a file called eks-admin-service-account.yaml with the text below. This manifest defines a service account and cluster role binding called eks-admin.

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: eks-admin
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: eks-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: eks-admin
  namespace: kube-system
```

Apply the service account and cluster role binding to the cluster:
```
kubectl apply -f eks-admin-service-account.yaml
```

Resultado:
```
serviceaccount "eks-admin" created
clusterrolebinding.rbac.authorization.k8s.io "eks-admin" created
```


### Step 3: Connect to the Dashboard


1.Retrieve an authentication token for the eks-admin service account. Copy the `<authentication_token>` value from the output. You use this token to connect to the dashboard.

```

kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep eks-admin | awk '{print $1}')

```

Result:
```
Name:         eks-admin-token-b5zv4
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=eks-admin
              kubernetes.io/service-account.uid=bcfe66ac-39be-11e8-97e8-026dce96b6e8

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:      <authentication_token>
```

Start the kubectl proxy.


```
kubectl proxy  --accept-hosts='^*$'
```


3. Command to allow remote machine access (commands must be executed on your computer):

```
ssh -L 8001:localhost:8001 k8s-admin@192.168.6.1

```

4. Open the following link with a web browser to access the dashboard endpoint:
http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/login

5. Select token, paste output `<authentication_token>` from the previous command in the Token field and select SIGN IN.

And ready!!


### Step 4: Enable Unsafe Mode on K8s Dashboard

To expose the kubernetes dashboard service (not advisable for production), follow the steps below:

1. Then configure the "NodePort" way to access it. Change 'type: ClusterIP' to 'type: NodePort' and save:
```
KUBE_EDITOR="nano" kubectl -n kube-system edit service kubernetes-dashboard
```

2. Discover the node and the exposed port where the panel is running:
```
kubectl -n kube-system describe pod kubernetes-dashboard-*
kubectl -n kube-system get service kubernetes-dashboard
```
3. Grant Cluster Administrator Permissions to Dashboard:
- Example of administrative user creation:
```
kubectl create clusterrolebinding add-on-cluster-admin --clusterrole=cluster-admin --serviceaccount=kube-system:kubernetes-dashboard
```


4. Access Dashboard via the link - `http: //:` that you got as part of step 2
Example: `https://192.168.6.3:31843`





### REFERENCES

[Source 1 - Main: Configuring 3 nodes in cluster kubernetes](https://computingforgeeks.com/how-to-setup-3-node-kubernetes-cluster-on-ubuntu-18-04-with-weave-net-cni/)


[Source 2 - Installing Kubernetes on CentOS, Ubuntu and Debian](https://www.itzgeek.com/how-tos/linux/centos-how-tos/how-to-install-kubernetes-on-centos-7-ubuntu-18-04-16-04-debian-9.html)

[Source 3 - Instalando e configurando um deploy com kuberntes em 5 min](https://codeburst.io/getting-started-with-kubernetes-deploy-a-docker-container-with-kubernetes-in-5-minutes-eb4be0e96370)

[Source 4 - Instalando KubeAdmin](https://v1-12.docs.kubernetes.io/docs/setup/independent/install-kubeadm/)

[Source 5 - Dashboard Tutorial](https://docs.aws.amazon.com/pt_br/eks/latest/userguide/dashboard-tutorial.html)

[Source 6 - Dashboard Tutorial Bug1](https://github.com/kubernetes/dashboard/issues/2735#issuecomment-402792561)

[Source 7 - Dashboard Tutorial Bug2](https://stackoverflow.com/questions/39864385/how-to-access-expose-kubernetes-dashboard-service-outside-of-a-cluster)

[Source 8 - Criando usuário documentação oficial](https://github.com/kubernetes/dashboard/wiki/Creating-sample-user)

[Source 9 - Habilitar o modo inseguro em dashboard K8s](https://github.com/kubernetes/dashboard/issues/2034)

[Source 10 - Desabilitar o modo swap em nodes Kubernetes](https://docs.platform9.com/support/disabling-swap-kubernetes-node/)

[Source 11 - Modo inseguro no docker](https://github.com/Juniper/contrail-docker/wiki/Configure-docker-service-to-use-insecure-registry)

[Source 12 - CGROUP DRIVER DOCKE](https://kubernetes.io/docs/setup/cri/)

[Source 13 - CGROUP DRIVER DOCKE - resolvendo bug](https://github.com/openshift/origin/issues/18776)

[Source 14 - Troubleshooting do kubeadm](https://kubernetes.io/docs/setup/independent/troubleshooting-kubeadm/)
