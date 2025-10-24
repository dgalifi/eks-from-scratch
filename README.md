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

### Load Balancer

aws-lbc.tf file contains terraform code to deploy the aws-load-balancer controller

run the following and make sure there's an aws-load-balancer-controller pod running 

```
kubectl get pods -n kube-system
```

## deploy nlb-service
move to k8s folder and run

```
kubectl apply -f nlb-service.yaml
```

this will create a deployment for the app and a service of type LoadBalancer
in AWS terms this is an NLB with a name like k8s-default-nlb... and a listener being added to it for the port 8080


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