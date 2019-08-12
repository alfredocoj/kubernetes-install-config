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

Confirme se os pacotes foram instalados:

```

$ which kubelet
/usr/bin/kubelet
$ which kubeadm
/usr/bin/kubeadm

```

Tudo configurado, é necessário juntar cada nó ao cluster criado no master (observe que o comando a seguir, é saída do comando 'kubeadm init' executado no nó master)

```
sudo kubeadm join 192.168.6.1:6443 --token mu0y4k.cbbs5f0pskuytuhb --discovery-token-ca-cert-hash sha256:790e9cdacab9e2158d31d7ed89390f089e499587380a66b8db66d03915b6aba9
```

A saída do comando, deve ser algo parecido com isso:

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

### Validando Clusterização
Acesso o servidor do nó master, e com o usuário k8-admin, execute o seguinte comando:


```
k8s-admin@k8s-master:~$ kubectl get nodes
```

O retorno deve ser algo do tipo:

```
NAME          STATUS    ROLES     AGE       VERSION
k8s-master    Ready     master    35m       v1.11.0
k8s-node-01   Ready     <none>    2m        v1.11.0
k8s-node-02   Ready     <none>    1m        v1.11.0
```

Nos dois nós, o Weave Net deve ter sido configurado. Pode ser verificado executando os comandos em cada nós agente.

Nó 1:
```
root@k8s-node-01:~# ip ad | grep weave


6: weave: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1376 qdisc noqueue state UP group default qlen 1000
    inet 10.44.0.0/12 brd 10.47.255.255 scope global weave
9: vethwe-bridge@vethwe-datapath: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1376 qdisc noqueue master weave state UP group default

```

Nó 2:
```
root@k8s-node-02:~# ip ad | grep weave
6: weave: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1376 qdisc noqueue state UP group default qlen 1000
    inet 10.47.0.0/12 brd 10.47.255.255 scope global weave
9: vethwe-bridge@vethwe-datapath: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1376 qdisc noqueue master weave state UP group default

```

### Subindo uma aplicação para teste

Criando um teste pod para verficar que o cluster rode como o esperado:

```
k8s-admin@k8s-master:~$ kubectl create namespace test-namespace
namespace/test-namespace created
```

Crie o arquivo de configurações de deploy http-app-deployment.yml, como se segue:

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

Criando um namespace (um organizador de deploys):

```
k8s-admin@k8s-master:~$ kubectl create namespace test-namespace
    namespace/test-namespace created
```

Depois que o namespace for criado, crie um pod usando o objeto de implementação definido anteriormente. -n é usado para especificar o espaço de nomes. Esperamos que três pods sejam criados, já que nosso valor de réplicas é 3.

```
k8s-admin@k8s-master:~$ kubectl create -n test-namespace -f http-app-deployment.yml
    deployment.extensions/http-app created
```

Confirmando a criação das configurações de deployment:

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

Com a implantação criada, podemos usar o kubectl para criar um serviço que expõe os Pods em uma porta específica. Um método alternativo é definir um objeto Service com YAML. Abaixo está nossa definição de serviço.

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

Criando serviço:

```
k8s-admin@k8s-master:~$ kubectl -n test-namespace create -f http-app-service.yml 
    service/http-app-svc created
```

O serviço estará diponível no Cluster IP e porta 30080. Para obter o CLUSTER IP, execute:
```
k8s-admin@k8s-master:~$ kubectl -n test-namespace get svc
NAME           TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
http-app-svc   NodePort   10.5.45.208   <none>        80:30080/TCP   1m

```

### Após instalação

Ative o preenchimento automático de shell para comandos kubectl, executando o comando:
```
source <(kubectl completion bash)
```

Para adicionar autocompletar o kubectl ao seu perfil, ele é automaticamente carregado no futuro.

```
echo "source <(kubectl completion bash)" >> ~/.bashrc
```


### Instalação do Dashboard do Kubernetes

### 1. Implante o painel do Kubernetes no cluster:

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```

Resultado:

```
secret "kubernetes-dashboard-certs" created
serviceaccount "kubernetes-dashboard" created
role "kubernetes-dashboard-minimal" created
rolebinding "kubernetes-dashboard-minimal" created
deployment "kubernetes-dashboard" created
service "kubernetes-dashboard" created
```

### 2. Implante o heapster para habilitar o monitoramento e análise de desempenho do cluster do contêiner no cluster:
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/heapster.yaml
```

Resultado:
```
serviceaccount "heapster" created
deployment "heapster" created
service "heapster" created
```

3. Implante o back-end influxdb para o heapster no cluster:

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/influxdb.yaml
```

Resultado:

```
deployment "monitoring-influxdb" created
service "monitoring-influxdb" created
```

4.Crie a vinculação da função do cluster do heapster para o painel:

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/rbac/heapster-rbac.yaml
```

Resultado:

```
clusterrolebinding "heapster" created
```

5. fsefd

Para funcionar o **heapster** é necessário alterar o arquivo de configuração do deployment. Altere conforme se segue:

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

E, em seguida, crie um papel que permita ao heapster operar e vinculá-lo à conta de serviço do heapster:

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

Execute o comando `kubectl apply -f role-heapster.yaml` e o mesmo retornará algo como:

```sh
clusterrole.rbac.authorization.k8s.io/node-stats-full created
clusterrolebinding.rbac.authorization.k8s.io/heapster-node-stats created
```

Agora é só seguir as próximas etapas.

_Referências:_ [Fonte 1](https://github.com/awslabs/amazon-eks-ami/issues/128), [Fonte 2](https://github.com/kubernetes/dashboard/issues/3147)


### Etapa 2: Criar uma conta de serviço do eks-admin e a vinculação de função do cluster

1. Crie um arquivo chamado eks-admin-service-account.yaml com o texto abaixo. Este manifesto define uma conta de serviço e uma vinculação da função do cluster chamadas eks-admin.

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

Aplique a conta de serviço e a vinculação da função do cluster ao cluster:
```
kubectl apply -f eks-admin-service-account.yaml
```

Resultado:
```
serviceaccount "eks-admin" created
clusterrolebinding.rbac.authorization.k8s.io "eks-admin" created
```


### Etapa 3: Conectar-se ao painel


1.Recupere um token de autenticação para a conta de serviço eks-admin. Copie o valor <authentication_token> da saída. Você usa esse token para se conectar ao painel.

```

kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep eks-admin | awk '{print $1}')

```

Resultado:
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

Inicie a kubectl proxy.


```
kubectl proxy  --accept-hosts='^*$'
```


3. Comando para liberar o acesso de máquina remota (comanda deve ser executado no seu computador):

```
ssh -L 8001:localhost:8001 k8s-admin@192.168.6.1

```

4. Abra o link a seguir com um navegador da web para acessar o endpoint do painel: http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/login

5. Selecione Token, cole a saída `<authentication_token>` do comando anterior no campo Token e selecione SIGN IN (FAZER LOGIN).

E pronto!!


### Etapa 4: Habilitar o modo inseguro em dashboard K8s

Para expor o serviço do dashboard do kubernetes (não aconselhável para produção), siguir os passos abaixo:

1. Em seguida, configure a maneira "NodePort" de acessá-lo. Altere 'type: ClusterIP' para 'type: NodePort' e salve:
```
KUBE_EDITOR="nano" kubectl -n kube-system edit service kubernetes-dashboard
```

2. Descubra o nó e a porta exposta onde o painel está sendo executado:
```
kubectl -n kube-system describe pod kubernetes-dashboard-*
kubectl -n kube-system get service kubernetes-dashboard
```
3. Conceder permissões de administrador de cluster ao Painel:
- Exemplo de criação de usuário administrativo:
```
kubectl create clusterrolebinding add-on-cluster-admin --clusterrole=cluster-admin --serviceaccount=kube-system:kubernetes-dashboard
```


4. Acesse o Dashboard pelo link - `http: //:` que você obteve como parte da etapa 2
Exemplo: `https://192.168.6.3:31843`





### REFERÊNCIAS

[Fonte1 - Principal: Configurando 3 nodes em cluster kuberntes](https://computingforgeeks.com/how-to-setup-3-node-kubernetes-cluster-on-ubuntu-18-04-with-weave-net-cni/)


[Fonte2 - Instalando Kubernetes no CentOS, Ubuntu e Debian](https://www.itzgeek.com/how-tos/linux/centos-how-tos/how-to-install-kubernetes-on-centos-7-ubuntu-18-04-16-04-debian-9.html)

[Fonte3 - Instalando e configurando um deploy com kuberntes em 5 min](https://codeburst.io/getting-started-with-kubernetes-deploy-a-docker-container-with-kubernetes-in-5-minutes-eb4be0e96370)

[Fonte4 - Instalando KubeAdmin](https://v1-12.docs.kubernetes.io/docs/setup/independent/install-kubeadm/)

[Fonte5 - Dashboard Tutorial](https://docs.aws.amazon.com/pt_br/eks/latest/userguide/dashboard-tutorial.html)

[Fonte6 - Dashboard Tutorial Bug1](https://github.com/kubernetes/dashboard/issues/2735#issuecomment-402792561)

[Fonte7 - Dashboard Tutorial Bug2](https://stackoverflow.com/questions/39864385/how-to-access-expose-kubernetes-dashboard-service-outside-of-a-cluster)

[Fonte8 - Criando usuário documentação oficial](https://github.com/kubernetes/dashboard/wiki/Creating-sample-user)

[Fonte9 - Habilitar o modo inseguro em dashboard K8s](https://github.com/kubernetes/dashboard/issues/2034)

[Fonte10 - Desabilitar o modo swap em nodes Kubernetes](https://docs.platform9.com/support/disabling-swap-kubernetes-node/)

[Fonte11 - Modo inseguro no docker](https://github.com/Juniper/contrail-docker/wiki/Configure-docker-service-to-use-insecure-registry)

[Fonte 12 - CGROUP DRIVER DOCKE](https://kubernetes.io/docs/setup/cri/)

[Fonte 13 - CGROUP DRIVER DOCKE - resolvendo bug](https://github.com/openshift/origin/issues/18776)

[Fonte 14 - Troubleshooting do kubeadm](https://kubernetes.io/docs/setup/independent/troubleshooting-kubeadm/)
