# Kubernetes Logging using Elastic Cloud on Kubernetes (ECK)

>Elastic Cloud on Kubernetes (ECK) is based on kubernetes operator pattern, to extend the kubernetes capabilities for setup and management  of Elastic's products like Elasticsearch, Kibana, Beats, etc.

## Supported Kubernetes Versions

* Kubernetes 1.18-1.22
* OpenShift 3.11, 4.5-4.9
* Google Kubernetes Engine (GKE), Azure Kubernetes Service (AKS), and Amazon Elastic Kubernetes Service (EKS)

Check the [ECK Support Matrix](https://www.elastic.co/support/matrix#matrix_kubernetes) for more information


## Preparing the K8s clusters
1. [Create the K8s cluster](https://docs.digitalocean.com/products/kubernetes/quickstart/)
2. [Install DigitalOcean Command Line Interface (doctl)](https://github.com/digitalocean/doctl)
3. Create Personal access tokens in the "API" from the menu of the control panel for doctl to managed your DO account.
4. Type in `doctl auth init` command in your terminal and paste your access token.
5. Using automated certificate management for automated certificate renewal and multiple-cluster management. The doctl command can be found in your K8s cluster setting. (Something like `doctl kubernetes cluster kubeconfig save xxxxxxx`)  
6. Using `kubectl config get-contexts` command to check out your cluster connection.
7. We're good to go!

### (Optional) Installing Nginx Ingress Controller via DO's 1-Click App
Install from your DO's K8s control panel and follow the instructions to check the Ingress controller is running.

**Note: The NGINX Ingress Controller 1-Click App also includes a $10/month DigitalOcean Load Balancer.**


btw, found out that the LB health checks failed just after I create the Ingress Controller. 
After asking DO's support engineer, it's the "externalTrafficPolicy" configuration of the loadbalancer.

Refer: 
* [External Traffic Policies and Health Checks](https://docs.digitalocean.com/products/kubernetes/how-to/configure-load-balancers/)
* [A Deep Dive into Kubernetes External Traffic Policies](https://www.asykim.com/blog/deep-dive-into-kubernetes-external-traffic-policies)

## Deploy ECK in your Kubernetes cluster 

The manifests will create a namespace called elastic-system and deploy the operator in it.

* Install CRDs and the operator with its RBAC rules by YAML manifests
```
kubectl create -f https://download.elastic.co/downloads/eck/1.9.0/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/1.9.0/operator.yaml
```
* Install CRDs and the operator with its RBAC rules by Helm chart

```
helm repo add elastic https://helm.elastic.co
helm repo 
helm install elastic-operator elastic/eck-operator -n elastic-system --create-namespace
```


## To Remove ECK from your Kubernetes cluster
1. Delete all elastic application and resources
```
kubectl get namespaces --no-headers -o custom-columns=:metadata.name \
  | xargs -n1 kubectl delete elastic --all -n
```

2.Remove the ECK operator
```
kubectl delete -f https://download.elastic.co/downloads/eck/1.9.0/crds.yaml
kubectl delete -f https://download.elastic.co/downloads/eck/1.9.0/operator.yaml

```
