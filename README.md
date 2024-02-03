# Canary Deployment with Istio

## Kind cluster initiation
If you don't already have kind installed, follow the [link](https://kind.sigs.k8s.io/docs/user/quick-start/)

```bash
kind create cluster --config=kind-cluster.yaml
```

## Install Istio Ingress Gateway CRD
```bash
kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
  { kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref=v0.8.0" | kubectl apply -f -; }
```

## Istio Initializing
```bash
curl -L https://istio.io/downloadIstio | sh -
# export the istio path
istioctl install --set profile=demo -y
```
### Install and Configure MetalLB

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml
```
- Setup address pool for LB
  
  ``` bash
  docker network inspect -f '{{ (index .IPAM.Config 0).Gateway }}' kind 
  ```
  then configure `metallb-conf.yaml` accordingly and run `k apply`

### Install RabbitMQ CRDs
```bash
kubectl apply -f "https://github.com/rabbitmq/cluster-operator/releases/latest/download/cluster-operator.yml"
```
#### Configure RMQ
```bash
k exec -it rabbitmq-server-0 -- bash

rabbitmqctl add_user admin admin

rabbitmqctl set_permissions --vhost / admin '.*' '.*' '.*'

rabbitmqctl set_user_tags admin administrator
```
Or get the default password from cli
```bash
# Get Username
kubectl get secret rabbitmq-default-user -o jsonpath="{.data.username}" | base64 --decode
# Get Password
kubectl get secret rabbitmq-default-user -o jsonpath="{.data.password}" | base64 --decode
```
### Install Mongo
```bash
k apply -f -r k8s-config/mongod
```
#### Configure Mongo ReplicaSet
- Drop to mongo shell
  `k exec -it mongo-0 -- mongosh`
```bash
rs.initiate()
var cfg = rs.conf()

cfg.members[0].host="mongo-0.mongo.default.svc.cluster.local:27017"
rs.reconfig(cfg)
rs.add("mongo-1.mongo.default.svc.cluster.local:27017")
rs.add("mongo-2.mongo.default.svc.cluster.local:27017")

```
- Check the replication status
  `rs.status()`


### Installing the application via helm

Update `egcom/applications/templates/egcom-cm.yaml ` ConfigMap values with the Grafana and RMQ creds.

```bash
k create ns egcom

k label namespace default istio-injection=enabled

helm install egcom ./applications -n egcom

```

### Test your setup


```
~ ❯ k get gtw -n egcom
NAME                CLASS   ADDRESS        PROGRAMMED   AGE
apigw-gateway       istio   172.18.0.103   True         31h
inventory-gateway   istio   172.18.0.104   True         31h
product-gateway     istio   172.18.0.102   True         31h
review-gateway      istio   172.18.0.101   True         31h
```
### Postman Collection

`baseURL` is the address of the apigw-gateway

[<img src="https://run.pstmn.io/button.svg" alt="Run In Postman" style="width: 128px; height: 32px;">](https://app.getpostman.com/run-collection/10587991-172b56a8-a87f-4ec8-93ee-24b29544c1a2?action=collection%2Ffork&source=rip_markdown&collection-url=entityId%3D10587991-172b56a8-a87f-4ec8-93ee-24b29544c1a2%26entityType%3Dcollection%26workspaceId%3D82d5477e-9493-4000-b370-675b650112cb)