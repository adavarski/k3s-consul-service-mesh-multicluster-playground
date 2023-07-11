## Consul Service Mesh Playground 

### Objective: 
In this playground, a Consul Cluster(one node for simplicity) is configured on Kubernetes cluster(k3s-1) and second
Consul Cluster(one node for simplicity) on onother Kubernetes cluster (k3s-2). Using Mesh Gateway, it is possible to unify and configure the
mTLS environment communication between different clusters and the traffic path between services. In addition, data in mTLS is not
thawed in the gateway, so data is transmitted and received safely between the two clusters. Each cluster of Consul is identified by the
name (DC-1 & DC-2). Configure a sidecar for each application. To configure Mesh Gateway, all sidecars must be configured with Envoy.
To configure Mesh Gateway, Sidecar and Consul need to communicate with TLS. Application services in each environment are
connected with a Mesh Gateway.


<img src="pictures/consul-mesh-overview.png?raw=true" width="1000">

<img src="pictures/consul-mesh-mesh-gateway.png?raw=true" width="1000">

### Setup k8s clusters
```
### Note: Two servers with IPs 192.168.1.99 & 100 with ssh & sudo configured 
$ /usr/bin/ssh root@192.168.1.99  'bash -s' < install-k3s-1.sh > ~/.kube/config-k3s-1.yaml
$ /usr/bin/ssh root@192.168.1.100 'bash -s' < install-k3s-2.sh > ~/.kube/config-k3s-2.yaml
```

Note: Networking must works between these 2 environment (k8s clusters). You can make this work however you like, depending on your own configuration, whether it’s VPC peering or VPN. For production for example between Azure AKS, GCP GKE we has to have connectivity between environments (k8s clusters) setuped via VPN (IPsec tunnels : Azure AKS <=> GKE GCP. We have to use only secured communications between 2 environments, so VPN/VPC Peering/etc. must be setup. For this playground we have connectivity between k3s-1 & k3s-2 (k8s clusters in shared network).

Note: Ports used by this setup (Ref: https://www.consul.io/docs/install/ports)

Port for the Federation
```
Consul Server & Agent
- 8301 : Gassip protocol used to check each other's status.
Consul Server
- 8500 : Port for API communication of Server.8501 : It communicates with Mesh Gateway by TLS.
- 8300 : Agent performs RPC communication with the server.
- 8600 : Used for Consul DNS.
Consul Agent
- 21000~21255: This is the port range to which the Sidecar service is assigned.
```

### Setup Consul Mesh

```
$ sudo apt install consul
$ consul keygen
GoIofxO1Xy0j2S7gMTyI8ewhQQe0XuChrKul+MzaDmA=

### Setup Consul on k8s-1 cluster
$ export KUBECONFIG=~/.kube/config-k3s-1.yaml
$ kubectl create ns consul
$ kubectl create secret generic consul-gossip-encryption-key –from-literal=key=GoIofxO1Xy0j2S7gMTyI8ewhQQe0XuChrKul+MzaDmA=
$ helm repo add hashicorp https://helm.releases.hashicorp.com
$ helm install -f consul-values-k3s-1.yaml consul hashicorp/consul -n consul
$ kubectl get secret consul-federation -n consul -o yaml > consul-federation-secret.yaml
$ kubectl get po -n consul
NAME                                           READY   STATUS    RESTARTS   AGE
consul-webhook-cert-manager-55b86459b9-f2q7v   1/1     Running   0          2m58s
consul-connect-injector-87c99bbdb-whcg6        1/1     Running   0          2m58s
consul-server-0                                1/1     Running   0          2m57s
consul-client-6tltm                            1/1     Running   0          2m58s
consul-mesh-gateway-cb55d6699-lx9h4            1/1     Running   0          2m58s
### Note: Setup/Apply needed proxy (all k8s clusters) -> (none/local/remote)
$ kubectl apply -f proxy-defaults.yaml

### Setup Consul on k8s-2 cluster
$ scp root@192.168.1.99:~/consul-federation-secret.yaml .
$ export KUBECONFIG=~/.kube/config-k3s-2.yaml
$ kubectl create ns consul
$ kubectl apply -f ./consul-federation-secret.yaml
$ helm install -f consul-values-k3s-2.yaml consul hashicorp/consul -n consul
$ kubectl get po -n consul
NAME                                           READY   STATUS    RESTARTS   AGE
consul-webhook-cert-manager-85d466975f-7nfx9   1/1     Running   0          81s
consul-server-0                                1/1     Running   0          81s
consul-client-jsdm5                            1/1     Running   0          81s
consul-connect-injector-6bd7cbd474-p5276       1/1     Running   0          81s
consul-mesh-gateway-77544ccb56-xhq4r           1/1     Running   0          81s
### Note: Setup/Apply needed proxy (all k8s clusters) -> (none/local/remote)
$ kubectl apply -f proxy-defaults.yaml

```
### Deploy apps

```
$ export KUBECONFIG=~/.kube/config-k3s-1.yaml
$ kubectl apply -f counting-deployment.yaml
$ export KUBECONFIG=~/.kube/config-k3s-2.yaml
$ kubectl apply -f dashboard-deployment.yaml

```

### Check Consul Service Mesh:
```
$ kubectl exec statefulset/consul-server -n consul -- consul catalog services -datacenter dc-1
consul
counting
counting-sidecar-proxy
mesh-gateway
$ kubectl exec statefulset/consul-server -n consul -- consul catalog services -datacenter dc-2
consul
dashboard-service
dashboard-service-sidecar-proxy
mesh-gateway

$ kubectl exec statefulset/consul-server -n consul -- consul members -wan
Node                  Address          Status  Type    Build   Protocol  DC    Partition  Segment
consul-server-0.dc-1  10.42.0.13:8302  alive   server  1.16.0  2         dc-1  default    <all>
consul-server-0.dc-2  10.42.0.11:8302  alive   server  1.15.1  2         dc-2  default    <all>

$ kubectl exec statefulset/consul-server -n consul -- sh -c 'curl --silent --insecure https://localhost:8501/v1/catalog/service/mesh-gateway | jq ".[].ServiceTaggedAddresses.wan"'
{
  "Address": "192.168.1.100",
  "Port": 31001
}
$ export KUBECONFIG=~/.kube/config-k3s-2.yaml
$ kubectl exec statefulset/consul-server -n consul -- sh -c 'curl --silent --insecure https://localhost:8501/v1/catalog/service/mesh-gateway | jq ".[].ServiceTaggedAddresses.wan"'
{
  "Address": "192.168.1.99",
  "Port": 31001
}
```
Check Consul Mesh via Consul UI (Note: Not exposed as Load Balancer, because of security)
```
% kubectl port-forward -n consul service/consul-server 8500:8500
```

Browser:

<img src="pictures/k3s-consul-mesh-dc-1-services.png?raw=true" width="1000">

<img src="pictures/k3s-consul-mesh-dc-2-services.png?raw=true" width="1000">


Check apps
```
$ export KUBECONFIG=~/.kube/config-k3s-1.yaml
$ kubectl port-forward  service/counting 8080:80
$ curl localhost:8080
{"count":13,"hostname":"counting-service-74bddb55c9-8pc5v"}

$ export KUBECONFIG=~/.kube/config-k3s-2.yaml
$ kubectl port-forward  service/dashboard-service 8080:80
```
Browser: 
<img src="pictures/k3s-consul-mesh-counting-UI.png?raw=true" width="1000">

### Playground summary:

<img src="pictures/consul-mesh-POC.png?raw=true" width="1000">







