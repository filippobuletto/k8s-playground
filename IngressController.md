# Enable Ingress Controller

## Install load-balancer: MetalLB

As per [https://metallb.universe.tf/installation/](https://metallb.universe.tf/installation/):

```bash
kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.8.3/manifests/metallb.yaml
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

## Install the NGINX Ingress Controller

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

## Ingress example

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

### Configure TLS

Create TLS Secret using [mkcert](https://github.com/FiloSottile/mkcert):

```bash
mkcert -install
mkcert example.kube
```

that will generate 2 files in the current folder:

- example.kube-key.pem
- example.kube.pem

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
