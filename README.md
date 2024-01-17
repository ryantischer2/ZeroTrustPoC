# ZeroTrustPoC

1.   Aruba zero trust POC hardware
  	- Aruba CX10K (2-4) switches
  	- Aruba CX Spine (83XX or 93XX) (1-2) switches
     	- Spine/Leaf cabling
  	- (2-3) servers to run ESXI 
    		-10g or 25g NICs
2.  Software Requirements
	- ESXi 7.x or 8.x
	- VCenter
	- Aruba AFC
	- AMD Policy and Services Controller (PSM)
	- Kubernetes deployment
	- Git	

1.   Cable the Spine/Leaf network as follows
2.   Connect servers to the fabric- 1 Nic only
3.   Install ESXi on servers
4.   Install VCenter
5.   Install AFC via VCenter - LINK
6.   Install PSM via VCenter - Directions on the Aruba portal
  	- Discover CX10Ks
7.   Use AFC to deploy fabric, and configure switch pairs in VSX
8.   Connect redundant link from servers - Configure
9.   Build 20 virtual machines with Linux.  Ubuntu 20 worked well.   Use VMware Templates to save time
   	- If using templates or cloning - download and install (docker and K8s) or K3s - Do not run 'kubeadm init'
     	- If cloning VM you need to change the 'machine-id' before 'kubeadm init'.  Modify the last two digits to make it unique 
      		
		cat /etc/machine-id

		19e2010d4ff249cf937373df45bd1e10

		sudo vim /etc/machine-id
10.  Change hostname of the VMs to match microservice name (see example below)
11.  Configure VMs for IP connectivity.   VMs will require access to the Internet

Example
---------------------------------
K8-master	            172.16.30.10/24	
adservice	            172.16.30.11/24	
cartservice	            172.16.30.12/24	
checkoutservice	        172.16.30.13/24	
currencyservice	        172.16.30.14/24	
emailservice	        172.16.30.15/24	
frontend	            172.16.30.16/24	
loadgenerator	        172.16.30.17/24	
paymentservice	        172.16.30.18/24	
productcatalogservice	172.16.30.19/24	
recommendationservice	172.16.30.20/24	
redis-cart	            172.16.30.21/24	
shippingservice	        172.16.30.22/24	
---------------------------------

12.  Install Kubernetes.  Any "install' or distribution should work.  Highly recommend using k3s
	- k8S example - https://www.letscloud.io/community/how-to-install-kubernetesk8s-and-docker-on-ubuntu-2004
	- k3s  - https://docs.k3s.io/quick-start


1. Cleanup the K8-Cluster, on every single node.
   
    sudo kubeadm reset cleanup-node
	sudo rm -rf /etc/cni/net.d 
	
2. Reboot the nodes.

3.  Initialize the K8-cluster from the Master Node.

       sudo kubeadm init --apiserver-advertise-address=172.16.30.10 --apiserver-cert-extra-sans=172.16.30.10 --pod-network-cidr=192.168.0.0/16
	   
4. Once the cluster is Initialized 
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

5. Deploy the Calico CNI Plugin

kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml

6. Create the Custom Resource for Calico , 
       wget https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml

For this Demo the Encapsulation is None and blockSize is 24 

****************************************************************************************************************************   
# This section includes base Calico installation configuration.
# For more information, see: https://docs.tigera.io/calico/latest/reference/installation/api#operator.tigera.io/v1.Installation
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Configures Calico networking.
  calicoNetwork:
    # Note: The ipPools section cannot be modified post-install.
    ipPools:
    - blockSize: 24
      cidr: 192.168.0.0/16
      encapsulation: None
      natOutgoing: Enabled
      nodeSelector: all()

---

# This section configures the Calico API server.
# For more information, see: https://docs.tigera.io/calico/latest/reference/installation/api#operator.tigera.io/v1.APIServer
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
*************************************************************************************************************************************
 
7. Apply the custom resource file.
    
	   kubectl apply -f custom-resources.yaml
	   
8. Check the Node Status , it should be in ready state.
    
	  kubectl get nodes

9. Join the other worker Nodes to K8 cluster using the token, printed from output of Step 3. or print new one.
	  
      kubeadm token create --print-join-command
		   
10. Check the status of all the Nodes from Master, all the nodes should be in ready state.

 kubectl get nodes
NAME                    STATUS   ROLES           AGE     VERSION
adservice               Ready    <none>          7m24s   v1.28.2
cartservice             Ready    <none>          7m24s   v1.28.2
checkoutservice         Ready    <none>          7m24s   v1.28.2
currencyservice         Ready    <none>          7m24s   v1.28.2
emailservice            Ready    <none>          7m24s   v1.28.2
frontend                Ready    <none>          7m25s   v1.28.2
k8-master               Ready    control-plane   17m     v1.28.2
loadgenerator           Ready    <none>          7m24s   v1.28.2
paymentservice          Ready    <none>          3m40s   v1.28.2
productcatalogservice   Ready    <none>          7m24s   v1.28.2
recommendationservice   Ready    <none>          3m45s   v1.28.2
redis-cart              Ready    <none>          3m44s   v1.28.2
shippingservice         Ready    <none>          3m57s   v1.28.2


11. Install the Calicotool

     
	wget -O calicoctl https://github.com/projectcalico/calico/releases/latest/download/calicoctl-linux-amd64
	chmod +x calicoctl
	sudo mv calicoctl /usr/local/bin/
	export KUBECONFIG=$HOME/.kube/config
	export DATASTORE_TYPE=kubernetes


12. Check the Status of Nodes 

calicoctl get nodes -o wide

NAME                    ASN       IPV4              IPV6
adservice               (64512)   172.16.30.11/24
cartservice             (64512)   172.16.30.12/24
checkoutservice         (64512)   172.16.30.13/24
currencyservice         (64512)   172.16.30.14/24
emailservice            (64512)   172.16.30.15/24
frontend                (64512)   172.16.30.16/24
k8-master               (64512)   172.16.30.10/24
loadgenerator           (64512)   172.16.30.17/24
paymentservice          (64512)   172.16.30.18/24
productcatalogservice   (64512)   172.16.30.19/24
recommendationservice   (64512)   172.16.30.20/24
redis-cart              (64512)   172.16.30.21/24
shippingservice         (64512)   172.16.30.22/24

13. Get the pods network block from every VM and update the excell sheet.

elk@k8-master:~$ ip route | grep black
blackhole 192.168.33.0/24 proto bird

14. Create the following Static routes in the CX10K's.

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

kubectl create  -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml

vim lb-pool.yaml
*******************************************************************
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: nat
  namespace: metallb-system
spec:
  addresses:
    - 172.16.30.245-172.16.30.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: empty
  namespace: metallb-system
********************************************************************* 
 kubectl apply -f lb-pool.yaml
 
16. Apply the label to each node

kubectl label node adservice type=adservice
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

16. Deploy the Boutique APP 

kubectl create -f boutique.yaml

17.Check the status of the PODS.

kubectl get pods -o wide
NAME                                     READY   STATUS    RESTARTS   AGE     IP              NODE                    NOMINATED NODE   READINESS GATES
adservice-55b7f5c45b-h7r5z               1/1     Running   0          7m29s   192.168.45.1    adservice               <none>           <none>
cartservice-7d45657c79-wn67f             1/1     Running   0          7m30s   192.168.234.1   cartservice             <none>           <none>
checkoutservice-c5c6b7c78-b7knv          1/1     Running   0          7m31s   192.168.83.1    checkoutservice         <none>           <none>
currencyservice-6b47794c8-w42ql          1/1     Running   0          7m30s   192.168.155.1   currencyservice         <none>           <none>
emailservice-ffc6c99cc-slf6k             1/1     Running   0          7m31s   192.168.65.1    emailservice            <none>           <none>
frontend-68b9b7b9dc-7tnt9                1/1     Running   0          7m30s   192.168.41.1    frontend                <none>           <none>
loadgenerator-9d4888654-2dwd2            1/1     Running   0          7m30s   192.168.42.1    loadgenerator           <none>           <none>
loadgenerator-9d4888654-gzvqx            1/1     Running   0          7m30s   192.168.42.2    loadgenerator           <none>           <none>
loadgenerator-9d4888654-wr2gq            1/1     Running   0          7m30s   192.168.42.3    loadgenerator           <none>           <none>
paymentservice-68bddd4c7f-28t6k          1/1     Running   0          7m30s   192.168.158.2   paymentservice          <none>           <none>
productcatalogservice-76f98fcfcf-vzvzq   1/1     Running   0          7m30s   192.168.118.1   productcatalogservice   <none>           <none>
recommendationservice-cb455454-gngjt     1/1     Running   0          7m30s   192.168.84.2    recommendationservice   <none>           <none>
redis-cart-55c5697d86-rmvmd              1/1     Running   0          7m30s   192.168.4.2     redis-cart              <none>           <none>
shippingservice-5977dbc67c-xz7lh         1/1     Running   0          7m30s   192.168.61.1    shippingservice         <none>           <none>

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

20. Post the IP collection and APP collection to PSM.

21. Edit the IP collection to make sure the Network is same as POD network from Step 13.

22. post the secutiy policy.

