# ZeroTrustPoC

Aruba zero trust POC

This PoC was designed to test advanced zero trust policy with Aruba CX10000 data center switch.   The PoC demonstrates east-west security for a distributed application running on VMWare ESXi.   Policy is enforced on the Pensando enabled CX10000 for all flows in the datacenter.   The Poc also features testing of flow visibility by generating flow records and IPFIX for all IP flows in the data center

Th PoC is deployed using VMWare and a Google created test application named "Online Boutique"   The application consists of 11 different services, ships with a load generator and runs on Kubernetes.  To simulate non-container (Kubernetes) based deployments each Pod is deployed on a ESXi virtual machine and traffic is forced to a CX10k for policy evaluation and visibility generation (flow and IPFIX)

Hardware Requirements
<ul>
<li>Aruba CX10K (2-4) switches</li>
<li>Aruba CX Spine (83XX or 93XX) (1-2) switches</li>
<li>Spine/Leaf cabling</li>
<li>(2-3) servers to run ESXI </li>
<li>10g or 25g NICs</li>
</ul>

Software Requirements
<ul>
<li>ESXi 7.x or 8.x</li>
<li>VCenter</li>
<li>Aruba AFC</li>
<li>AMD Policy and Services Controller (PSM)</li>
<li>Kubernetes deployment</li>
<li>Git	</li>
</ul>
---
Steps
---

1. Cable the Spine/Leaf network as follows
2. Connect servers to the fabric- 1 Nic only
3. Install ESXi on servers
4. Install VCenter
5. Install AFC via VCenter - LINK
6. Install PSM via VCenter - Directions on the Aruba portal- Discover CX10Ks
7. Use AFC to deploy fabric, and configure switch pairs in VSX
8. Connect redundant link from servers - Configure LCAP
9. Build 20 virtual machines with Linux.  Ubuntu 20 worked well.   Use VMware clones to save time
- With clone - download and install (docker and K8s) or K3s in the clone!  **Do not run 'kubeadm init'**
- After clones boot change the 'machine-id' before 'kubeadm init'.  Modify the last two digits to make it unique.  For example...

> cat /etc/machine-id
>
> 19e2010d4ff249cf937373df45bd1e10

> sudo vim /etc/machine-id

10. Change hostname of the VMs to match microservice name (see example below)
11. Configure VMs for IP connectivity.   VMs will require access to the Internet

#### Example VMname and IP addressing
---
| Service               | IP Address         |
|-----------------------|--------------------|
| K8-master             | 172.16.30.10/24    |
| adservice             | 172.16.30.11/24    |
| cartservice           | 172.16.30.12/24    |
| checkoutservice       | 172.16.30.13/24    |
| currencyservice       | 172.16.30.14/24    |
| emailservice          | 172.16.30.15/24    |
| frontend              | 172.16.30.16/24    |
| loadgenerator         | 172.16.30.17/24    |
| paymentservice        | 172.16.30.18/24    |
| productcatalogservice | 172.16.30.19/24    |
| recommendationservice | 172.16.30.20/24    |
| redis-cart            | 172.16.30.21/24    |
| shippingservice       | 172.16.30.22/24    |

---

12. Install Kubernetes.  Any K8s should work.  To make life easier I highly recommend using k3s

####Install K3S
- k3s  - https://docs.k3s.io/quick-start

####Install K8S
- K8S example - https://www.letscloud.io/community/how-to-install-kubernetesk8s-and-docker-on-ubuntu-2004
- Initialize the K8-cluster from the Master Node.
>sudo kubeadm init --apiserver-advertise-address=172.16.30.10 --apiserver-cert-extra-sans=172.16.30.10 --pod-network-cidr=192.168.0.0/16

- Optionally install additional master nodes 

>>mkdir -p $HOME/.kube
>>sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
>>sudo chown $(id -u):$(id -g) $HOME/.kube/config

- Deploy the Calico CNI Plugin
>>kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml
- deploy custom resources from (this) git repo
>>kubectl apply -f custom-resources.yaml

####If you mess up K8S install 
1. Cleanup the K8-Cluster, on every single node.
>sudo kubeadm reset cleanup-node
>sudo rm -rf /etc/cni/net.d 

2. Reboot the nodes.

####Configure Kubernetes
 
 - Apply the custom resource file.

>kubectl apply -f custom-resources.yaml

- Join worker nodes to the cluster
-- Follow directions from kubeadm for worker nodes - save token
-- If you may have lost the tok en 
>kubeadm token create --print-join-command
- Once the cluster is Initialized 
>kubectl get nodes

8. Check the Node Status , it should be in ready state.

   
		   
10. Check the status of all the Nodes from Master, all the nodes should be in ready state.

> kubectl get nodes

![kubectl get nodes](/readmeIMG/kubectl%20get%20nodes.png)


11. Create the following Static routes in the CX10K's.

ip route < network from step13> <next-hop IP is IPv4 IP from Step12> 

ip route  192.168.4.0/24   172.16.30.21 vrf pod1 
ip route  192.168.33.0/24  172.16.30.10 vrf pod1
ip route  192.168.41.0/24  172.16.30.16 vrf pod1 
ip route  192.168.42.0/24  172.16.30.17 vrf pod1 
ip route  192.168.45.0/24  172.16.30.11 vrf pod1 
ip route  192.168.61.0/24  172.16.30.22 vrf pod1 
ip route  192.168.65.0/24  172.16.30.15 vrf pod1 
ip route  192.168.83.0/24  172.16.30.13 vrf pod1 
ip route  192.168.84.0/24  172.16.30.20 vrf pod1 
ip route  192.168.118.0/24  172.16.30.19 vrf pod1 
ip route  192.168.155.0/24  172.16.30.14 vrf pod1 
ip route  192.168.158.0/24  172.16.30.18 vrf pod1 
ip route  192.168.234.0/24  172.16.30.12 vrf pod1

15. Create LoadBalancer this is optional , we can use the NodeIP too.

>kubectl create  -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml

>> kubectl apply -f lb-pool.yaml
 
16. Apply the label to each node

>kubectl label node adservice type=adservice
kubectl label node cartservice type=cartservice
kubectl label node checkoutservice type=checkoutservice
kubectl label node currencyservice type=currencyservice
kubectl label node emailservice type=emailservice
kubectl label node loadgenerator type=loadgenerator
kubectl label node paymentservice type=paymentservice
kubectl label node productcatalogservice type=productcatalogservice
kubectl label node recommendationservice type=recommendationservice
kubectl label node redis-cart type=redis-cart
kubectl label node frontend type=frontend
kubectl label node shippingservice type=shippingservic

17. Deploy the Boutique APP 

>kubectl create -f boutique.yaml

18 .Check the status of the PODS.

> kubectl get pods -o wide

![get pods](/readmeIMG/kubectl%20get%20pods.png)

18.Access the APP from Browser by pointing to Endpoints:

 kubectl describe svc frontend
Name:              frontend
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          app=frontend
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.111.131.23
IPs:               10.111.131.23
Port:              http  80/TCP
TargetPort:        8080/TCP
Endpoints:         192.168.41.1:8080

19. If the Client is from remote network, make sure the static routes are advertised across the fabric.

20. Use Postman to Post the PSM IP collection and APP collection to PSM.

21. Edit the IP collection to make sure the Network is same as POD network from Step 13.

22. Use Postman to post the PSM security policy.

####Results
PSM Policy
![PSM Policy](/readmeIMG/sp-01.PNG)
top N with ElKS
![top n](/readmeIMG/top-n.PNG)
PSM Apps
![PSM Apps](/readmeIMG/apps.PNG)
PSM IP Collections
![IP Collections](/readmeIMG/ipcollections.PNG)
PSM Policy evaluation
![PSM Policy eval](/readmeIMG/polic-eval.PNG)