# k8s-ingress

**Table of Contents**

- [Expose a web service via GKE ingress controller](#ingress)
    - [Intention](#intention)
    - [Requirements](#requirements)
    - [Deployment](#deployment)
        - [Deploy pods](#deploy-pods)
        - [Create service](#create-service)
        - [Create ingress](#create-ingress)
    - [Testing](#testing)
    - [Debugging](#debugging)
    - [Clear up](#clear-up)


## Expose a web service via GKE ingress controller

### Intention 

I struggled to understand how Kubernetes pods, services and the default GKE HTTP ingress controller interact with each other.
The howtos I found worked, but I did not know why. My own attempts to deploy a webservice and make it reachable from the internet failed and I were not able to debug them.

Thats why I documented the information I needed to understand this setup in the yaml files and described the steps to reproduce it.

### Requirements

You need to have a basic understanding about Kubernetes pods and services.
You need to know how to [create a GKE cluster in Google Cloud Platform.](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-a-cluster)
You need to have an up and running public GKE test cluster.

### Deployment

#### Deploy pods

Create three echo-server pods
``` 
kubectl apply -f ./deployment.yaml
```

Verify that all three pods are running
```
kubectl get pods
```

Output will be something like
```
NAME                             READY   STATUS    RESTARTS   AGE
webapplication-76bb8c4cf-2dxft   1/1     Running   0          94s
webapplication-76bb8c4cf-c5l7w   1/1     Running   0          94s
webapplication-76bb8c4cf-p6ltq   1/1     Running   0          94s
```

#### Create service

Create a service
```
kubectl apply -f ./service.yaml
```

Verify that the service is available
```
kubectl get services
```

Output will be something like
```
NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
kubernetes    ClusterIP   10.215.0.1     <none>        443/TCP          9d
web-service   NodePort    10.215.4.115   <none>        8081:30748/TCP   8s
```

#### Create ingress

Create an ingress controller
```
kubectl apply -f ./ingress.yaml
```

Verify that the service is available
```
kubectl get ingress
```

Output will be something like
```
NAME          HOSTS   ADDRESS   PORTS   AGE
web-ingress   *                 80      11s
```

Wait for up to 15 minutes to let the ingress controller start.
Verify that the ingress is available
```
kubectl get ingress
```

Output will be something like
```
NAME          HOSTS   ADDRESS         PORTS   AGE
web-ingress   *       35.190.37.125   80      5m20s
```

### Testing

After the deployment of the ingress curl the external loadbalancer
```
curl $(kubectl get ing web-ingress -o json | jq -r .status.loadBalancer.ingress[].ip):80
```

Output should be something like
```
Request served by webapplication-76bb8c4cf-c5l7w

HTTP/1.1 GET /

Host: 35.190.37.125
Accept: */*
X-Cloud-Trace-Context: 106cfbcc6a00d4c0f9f030db665591ef/14716276750467048669
Via: 1.1 google
X-Forwarded-For: YOURIP, 35.190.37.125
X-Forwarded-Proto: http
Connection: Keep-Alive
User-Agent: curl/7.54.0
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
If it fails curl one of the pods. Do not forget to stop the former port-forwarding before.

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
