# gke-examples

## Expose a web service via GKE ingress controller

### Deployment

Create three echo-server pods
``` 
kubectl apply -f ./workload/pod.yaml
```

Create a service and and ingress controller
```
kubectl apply -f ./workload/service.yaml
```

Wait for at least 15 minutes to let the ingress controller start.

### Testing

15 minutes after the deployment of the ingress curl the external loadbalancer
```
curl $(kubectl get ing web-ingress -o json | jq -r .status.loadBalancer.ingress[].ip):80
```

Output should be something like
```
Request served by webapplication-76bb8c4cf-j5n4z

HTTP/1.1 GET /

Host: 35.190.43.11
Via: 1.1 google
X-Forwarded-For: YOURIP, 35.190.40.208
X-Forwarded-Proto: http
Connection: Keep-Alive
User-Agent: curl/7.54.0
Accept: */*
X-Cloud-Trace-Context: ee30cde53122e5652b3edd00046e5eac/10202682875756082998
```

### Debugging

If this fails try to get a response from the Kubernetes service
Forward the service port to your local machine
```
kubectl port-forward svc/web-service 31337:8081
```
Curl the forwarded port
```
curl localhost:31337
```
Output should be something like
```
Request served by webapplication-76bb8c4cf-j5n4z

HTTP/1.1 GET /

Host: localhost:31337
Accept: */*
User-Agent: curl/7.54.0
```

If this succeeds wait for five additional minutes and curl the external loadbalancer again.
If it fails curl one of the pods. Do not forget to stop the port-forwarding.

Forward the pods port to your local machine
```
kubectl port-forward $(kubectl get pods -o json | jq -r '.items[] | select(.spec.containers[].name == "echo") | .metadata.name ' | head -n 1) 31336:8080
```
Curl the forwarded port
```
curl localhost:31336
```
Output should be something like this
```
Request served by webapplication-76bb8c4cf-6znld

HTTP/1.1 GET /

Host: localhost:31336
Accept: */*
User-Agent: curl/7.54.0
```

Do not forget to stop the port-forwarding.


### Clear up

Delete all created resources
```
kubectl delete ingress web-ingress && kubectl delete service web-service && kubectl delete deployment webapplication
```
Output must be
```
ingress.extensions "web-ingress" deleted
service "web-service" deleted
deployment.apps "webapplication" deleted
```
If this is not true you have to clear up manually.
