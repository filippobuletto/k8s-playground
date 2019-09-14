# Kubernetes Playground

This project contains a `Vagrantfile` to provisioning a 3 nodes Kubernetes cluster using `VirtualBox` and `Ubuntu 16.04`.

## Prerequisites

You need the following installed to use this playground.

- `Vagrant`, version 2.2 or better.
- `VirtualBox`, tested with Version 6.0
- Internet access, this playground pulls Vagrant boxes from the Internet as well
as installs Ubuntu application packages from the Internet.
- Tested on **Windows 10** Build 17763, **macOS Mojave** 10.14.6

## Bringing Up The cluster

To bring up the cluster, clone this repository to a working directory.

```bash
git clone https://github.com/filippobuletto/k8s-playground.git
```

Change into the working directory and `vagrant up`

```bash
cd k8s-playground
vagrant up
```

Vagrant will start three machines. Each machine will have a NAT-ed network
interface, through which it can access the Internet, and a `private-network`
interface in the subnet 192.168.205.0/24. The private network is used for
intra-cluster communication.

The machines created are:

| NAME | IP ADDRESS | ROLE |
| --- | --- | --- |
| k8s-head | `192.168.205.10` | Cluster Master |
| k8s-node-1 | `192.168.205.11` | Cluster Worker |
| k8s-node-2 | `192.168.205.12` | Cluster Worker |

As the cluster brought up the cluster master (**k8s-head**) will perform a `kubeadm init` and the cluster workers will perform a `kubeadmin join`.

### Import `kubeconfig` file

In order for `kubectl` to find and access a Kubernetes cluster, it needs a `kubeconfig` file, which is created by vagrant during master provisioning: `admin.conf`.

Move and rename the file to the default location: `%USERPROFILE%\.kube\config`.

Check that `kubectl` is properly configured by getting the cluster state: `kubectl cluster-info`.

## Starting A Sample Service / Deployment

The following is an example of a Deployment. It creates a ReplicaSet to bring up three `nginx` Pods:

```bash
kubectl apply -f https://k8s.io/examples/controllers/nginx-deployment.yaml
```

Get the description of the Deployment:

```bash
kubectl describe deployment
```

Looking at the Pods created:

```bash
kubectl get pods
```

Create a Service object that exposes the deployment:

```bash
kubectl expose deployment nginx-deployment --name=web-service --type=NodePort
```

Display information about the Service:

```bash
kubectl get service web-service
```

Access the service from the node port, ex: [http://192.168.205.10:31751/](http://192.168.205.10:31751/).

To delete the Service, enter this command:

```bash
kubectl delete service web-service
```

To delete the Deployment, the ReplicaSet, and the Pods that are running the Hello World application, enter this command:

```bash
kubectl delete deployment nginx-deployment
```

## Enable Ingress Controller

### Install load-balancer: MetalLB

As per [https://metallb.universe.tf/installation/](https://metallb.universe.tf/installation/):

```bash
kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.8.1/manifests/metallb.yaml
```

Apply [Layer 2 configuration](https://metallb.universe.tf/configuration/#layer-2-configuration):

```bash
kubectl apply -f resources/metallb.yaml
```

Or create this ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.205.11-192.168.205.12
```

### Install the NGINX Ingress Controller

The following **Mandatory Command** is required:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml
```

Create the `ingress-nginx` LoadBalancer Service:

```bash
kubectl apply -f resources/service-loadbalancer.yaml
```

Or create this Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  externalTrafficPolicy: Local
  type: LoadBalancer
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
    - name: https
      port: 443
      targetPort: 443
      protocol: TCP
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
```

### Ingress example

Expose the previous example deployment `nginx-deployment` to the host `example.kube`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  ports:
  - port: 9376
    protocol: TCP
    targetPort: 80
    name: web
  selector:
    app: nginx
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    kubernetes.io/ingress.class: "nginx"
    ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: example.kube
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx-service
          servicePort: web
```

Map the host `example.kube` to a cluster worker's ip like:

```bash
echo "192.168.205.11 example.kube" | sudo tee -a /etc/hosts > /dev/null
```

```powershell
If ((Get-Content "$($env:windir)\system32\Drivers\etc\hosts" ) -notcontains "192.168.205.11 example.kube")
 {ac -Encoding UTF8  "$($env:windir)\system32\Drivers\etc\hosts" "192.168.205.11 example.kube" }
```

Open a browser to [http://example.kube/](http://example.kube/).

#### Configure TLS

Create TLS Secret using [mkcert](https://github.com/FiloSottile/mkcert):

```bash
mkcert -install
mkcert example.kube
```

Then create the secret in the cluster via:

```bash
kubectl create secret tls examplecert --key example.kube-key.pem --cert example.kube.pem
```

Update the Ingress spec with TLS Secret:

```yaml
spec:
  tls:
    - hosts:
      - example.kube
      secretName: examplecert
```

Open a browser to [https://example.kube/](https://example.kube/).

See also NGINX Ingress Controller [TLS/HTTPS](https://kubernetes.github.io/ingress-nginx/user-guide/tls/).

### Bonus: forward all `.kube` host for local development

Install Dnsmasq (or [DNSAgent](https://github.com/stackia/DNSAgent) on Windows) and configure it to match any request which ends in `.kube` and send `192.168.205.11` in response.

```bash
# Example for dnsmasq.conf
address=/kube/192.168.205.11
```

## Use internal registry

Install Docker Registry using [Helm](https://hub.helm.sh/charts/stable/docker-registry):

- namespace: `kube-registry`
- service type: `NodePort`
- service node port: `30500`

```bash
helm install stable/docker-registry -n docker-registry --namespace kube-registry --set service.type=NodePort --set service.nodePort=30500
```

Install [Docker Registry User Interface](https://joxit.dev/docker-registry-ui/):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: registry-ui-deployment
  namespace: kube-registry
  labels:
    app: registry-ui
spec:
  replicas: 1
  selector:
    matchLabels:
      app: registry-ui
  template:
    metadata:
      labels:
        app: registry-ui
    spec:
      containers:
      - name: registry-ui
        image: joxit/docker-registry-ui
        ports:
        - containerPort: 80
        env:
        - name: DELETE_IMAGES
          value: "true"
        - name: URL
          value: "https://registry.example.kube"
---
apiVersion: v1
kind: Service
metadata:
  name: registry-ui-service
  namespace: kube-registry
spec:
  ports:
  - port: 9376
    protocol: TCP
    targetPort: 80
    name: web
  selector:
    app: registry-ui
```

Configure an ingress to access registry and ui:

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: registry-ingress
  namespace: kube-registry
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
    ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
    - hosts:
      - registry.example.kube
      secretName: examplecert
  rules:
  - host: registry.example.kube
    http:
      paths:
      - path: /v2
        backend:
          serviceName: docker-registry
          servicePort: registry
      - path: /
        backend:
          serviceName: registry-ui-service
          servicePort: web
```

Enable cluster's Docker daemon to pull from registry:

```bash
vagrant ssh k8s-head
# Execute this command within the cluster node
echo '{"insecure-registries" : ["192.168.205.10:30500","192.168.205.11:30500","192.168.205.12:30500"]}' | sudo tee /etc/docker/daemon.json
# ! -- REPEAT FOR k8s-node-1 / k8s-node-2 -- !
vagrant reload
```

Push your image to `registry.example.kube`:

```bash
docker push registry.example.kube/test/test-image
```

Use it within the cluster pulling images from the node port of the `docker-registry` Service:

```yaml
# Example for deployment
    spec:
      containers:
      - name: my-test
        image: 192.168.205.10:30500/test/test-image
```

## Useful Vagrant commands

```bash
#Create the cluster or start the cluster after a host reboot
vagrant up

#Execute provision in all the vagrant boxes
vagrant provision

#Execute provision in the Kubernetes node 1
vagrant provision k8s-node-1

#Open an ssh connection to the Kubernetes master
vagrant ssh k8s-head

#Open an ssh connection to the Kubernetes node 1
vagrant ssh k8s-node-1

#Open an ssh connection to the Kubernetes node 2
vagrant ssh k8s-node-2

#Stop all Vagrant machines (use vagrant up to start)
vagrant halt

#Destroy the cluster
vagrant destroy -f
```

## Use scoop to install the tools

Install [scoop](https://github.com/lukesampson/scoop#installation) and execute the following commands:

```powershell
# Must
scoop install git-with-openssh
scoop install kubectl
scoop install vagrant
# Extra buckets
scoop bucket add extras
scoop bucket add Ash258 'https://github.com/Ash258/scoop-Ash258.git'
scoop bucket add pot-pourri 'https://github.com/filippobuletto/pot-pourri'
scoop update
# Nice to have
scoop install kubebox
scoop install kubefwd
scoop install kubeval
scoop install octant
scoop install rakkess
scoop install stern
scoop install popeye
scoop install mkcert
```
