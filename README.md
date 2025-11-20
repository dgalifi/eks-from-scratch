# eks-from-scratch


## Version
Currently tested with 
terraform v1.13.3 and kubernetes v1.34


## Run terraform apply

check infrastructure/locals.tf file and make adjustments to the values if needed

```
terraform apply
```

## Get kubeconfig updated

```
aws eks --region <region> update-kubeconfig --name <cluster_name>
```

test with 

```
kubectl get nodes
```

make sure you have admin access

```
kubectl auth can-i "*" "*"
```


## Horizontal Pod Autoscaler
the metrics-server.tf file installs a metrics server which is used by the HPA (horizontal pod autoscaler) to scale up or down the number of pods

The metrics server which scrapes metrics from kubelet and publishes them to the metrics.k8s.io kubernetes API.

This is deployed using helm (provider initialised on helm-provider.tf)

When using HPA you should never define the number of replicas on your deployment. Define a resources block instead with requests and limits (soft and hard limits) like:

```yaml
resources:
    requests:
        cpu: 100m
        memory: 256Mi
    limits:
        cpu: 100m
        memory: 256Mi
```

then create an HorizontalPodAutoscaler with metrics like:

```yaml
minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 80
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70
```

to check if the metrics server is up and running

```
kubectl get pods -n kube-system
```

check logs

```
kubectl logs -l app.kubernetes.io/instance=metrics-server -f -n kube-system
```

how to get metrics for pods and nodes

```
kubectl top pods -n kube-system
kubectl top nodes -n kube-system
```


## deploy quick app
move to the k8s folder and run

```
kubectl apply -f quick-app.yaml
```

and watch the pods and autoscaler with 

```
watch -t kubectl get pods

watch -t kubectl get hpa
```


## how to hit your service

get the service name 

```
kubeclt get svc
```

then run port-forward command on the port used by the service

```
kubectl port-forward svc/quick-app-service <port>
```

hit your service

```
curl http://localhost:10002
```

### Load Balancer Controller

aws-lbc.tf file contains terraform code to deploy the aws-load-balancer controller

run the following and make sure there's an aws-load-balancer-controller pod running 

```
kubectl get pods -n kube-system
```


The load balancer controller allows us to create AWS load balancers as service or ingress.


## deploy nlb-app
move to k8s folder and run

```
kubectl apply -f nlb-service.yaml
```

this will create a deployment for the app and a service of type LoadBalancer.

In AWS terms this is an NLB with a name like k8s-default-nlb... and a listener being added to it for the port 8080


Grab the DNS NLB name and hit

http://<nlb_dns>:8080

this should hit our nlb-service pod

port is 8080 because we configured it in the LoadBalancer service

```
spec:
  type: LoadBalancer
  ports:
    - port: 8080
      targetPort: http
```

delete the service and deployment if you don't need it anymore
```
kubectl delete -f nlb-service.yaml
```

## deploy alb-service

This will create an ingress through the controller.

To check the ingressclass of the controller

```
kubectl get ingressclass
```
Let's a plain ingree to expose our application 

```
kubectl apply -f alb-service.yaml
```

this will create and deployment for the app, a service and an ingress

In AWS terms this will be an ALB with a listener on port 80 and a rule based on host and path described by 
```
rules:
    - host: eks-from-scratch.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: alb-service
                port:
                  number: 8080
```

This will also create a target group for port 8080 linked to the alb-service app

To check the ingresses run

```
kubectl get ing
```

Grab the DNS ALB name and hit it with
```
curl -H "Host: eks-from-scratch.com" http://<alb_dns>
```

Notice how we need to change the host header to match the LB rule

If you want to change the listener's ports you can define them in the annotations

```
alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80}, {"HTTPS":443}]'
```

## Nginx ingress
nginx-controller.tf contains the terraform code to deploy the nginx controller using helm charts

check the service with 

```
kubectl get svc -n ingress
```

In AWS terms this will create a NLB which will point to the nginx ingress controller running in a pod

Check which ingressclass are available with:

```
kubectl get ingressclass
```

Deploy nginx-app with

```
kubectl apply -f nginx-app.yaml
```

get the ingress with 

```
kubectl get ing 
```

grab the LB DNS name and hit 


```
curl -H "Host: eks-from-scratch.com" http://<alb_dns>
```