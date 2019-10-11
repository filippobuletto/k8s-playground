# Use internal registry

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

Configure an ingress to access registry and UI:

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
