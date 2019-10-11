# Starting A Sample Service / Deployment

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
