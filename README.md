# Kubernetes-setup
# setup Kubernetes on multiple server
    here we go with two server 
        1. master
        2. child
 install on master server 
```
apt-get update && apt-get install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF > /etc/apt/sources.list.d/kubernetes.list  
deb http://apt.kubernetes.io/ kubernetes-xenial main  
EOF
apt-get update
apt-get install -y docker.io
apt-get install -y kubelet kubeadm kubectl kubernetes-cni
```
        
## first need to intiate a new cluster through kubeadm on master server

`kubeadm init`

after successfully run this command you show like 
    ![](https://i.imgur.com/9Cz04sg.png)
 now you can hit those three command which you get after `kubeam init` 
 
 > now you  get token and join commadn which you need to hit on child node or another server 
     >command you can see after above done .
        which is like example - `kubeadm join 167.99.96.34:6443 --token ovt8zj.57mcxpyjjrdkd2mm --discovery-token-ca-cert-hash sha256:d22dc887505d7425b4e071e4922f8e1a77aa02794653664015517620d61215ac`
        
 > now you can see your nodes by 
        `kubectl get nodes`
    after that you can see a status of nodes is not ready  so you need to set netwok layer for that 
        `kubectl apply -f https://docs.projectcalico.org/v2.6/getting-started/kubernetes/installation/hosted/kubeadm/1.6/calico.yaml`

> you can initiate pod directly without any replication controller but we can go for replication controller for multiple replicas and for load balancing .

> now create a yml file for replication cotroller.

```
# rc.yml file
apiVersion: v1
kind: ReplicationController
metadata:
  name: hello-rc
spec:
  replicas: 10
  selector:
    app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello-ctr
        image: nigelpoulton/pluralsight-docker-ci:latest
        ports:
        - containerPort: 8080
```

 > now run this file with
     > `kubectl create -f rc.yml`
     if you want to update yml file then after updation you need to hit 
     `kubectl apply -f rc.yml`
     
**Note** - each pod in replication controller have unique ip show when we had multiple server then each container failed then its create new for us . so for that we need to create service which manage all dynamic ip of each pods and bind with service ip address like - 

> **Manages pod through diffrent nodes by service**
    ![](https://i.imgur.com/dvifgN6.png)
    
## Now setup service on master node  


`kubectl expose rc hello-rc --name=hello-svc --target-port=8080 --type=NodePort`

**Note** - 
1. here hello-rc is my conainer and hello-svc is my service name which  initate for managing all pods.
 2. like if your app run on 3000 port then you need to set target port --target-port=3000 .

To check service details - 
    `kubectl describe svc hello-svc`
    which show like - 
    ![](https://i.imgur.com/8qM95xH.png)
    
    NodePort in this image is your actuall port whre your app run by service like here my current port 8080:30260 
    so app run on `chillNodeServer:30260`
**By deafult all pods run on child node so your pod response get on child server ip:port**

abstraction is Kubeam > service > Replication Controller > Pods > Container (docker or other)

# setup dashboard for kubernetes .
**step - 1 **  `kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml`
**step - 2** 
    By default, the dashboard will install with minimum user role privileges.
    To access the dashboard with full administrative permission, create a YAML file named dashboard-admin.yaml.
    `vi dashboard-admin.yaml`
    ```
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
   name: kubernetes-dashboard
   labels:
     k8s-app: kubernetes-dashboard
roleRef:
   apiGroup: rbac.authorization.k8s.io
   kind: ClusterRole
   name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system
    ```
 **step - 3** 
 `nohup kubectl proxy --address="your-server-ip" -p port --accept-hosts='^*$' &`
 nohup added for run a front end server in background 
  **step - 4** now hit url in browser
  `http://sever-ip:port/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/`


