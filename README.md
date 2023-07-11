### Consul Service Mesh Playground 

### Setup k8s clusters
```
$ /usr/bin/ssh root@devops 'bash -s' < install-k3s-1.sh > ~/.kube/config-k3s-1.yaml
$ /usr/bin/ssh root@carbon 'bash -s' < install-k3s-2.sh > ~/.kube/config-k3s-2.yaml
export KUBECONFIG=~/.kube/config-k3s-1.yaml
```
