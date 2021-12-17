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

## Deploy ECK CRD & operator in your Kubernetes cluster 

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
## Deploy an Elasticsearch Cluster
* Create an Elasticsearch cluster from `/manifests/setup/elasticsearch.yaml` as a statefulset
```
kubectl apply -f ./manifests/setup/elasticsearch.yaml
```

* Check the Elasticsearch Cluster
```
kubectl get elasticsearch
```
```
#You should see something like this:
NAME          HEALTH    NODES     VERSION   PHASE         AGE
quickstart    green     1         7.16.1     Ready         1m
```

* Access to Elasticsearch cluster
```
#Check your clusterIP service that automatically created with your cluster
kubectl get service quickstart-es-http

#You should get a service like this
NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
quickstart-es-http   ClusterIP   10.15.251.145   <none>        9200/TCP   34m
```
* Get your credentials from the secret
```
PASSWORD=$(kubectl get secret quickstart-es-elastic-user -o go-template='{{.data.elastic | base64decode}}')
echo $PASSWORD
```
* Open a seperate terminal and port-forward your service to your localhost
```
kubectl port-forward service/quickstart-es-http 9200
```
* Request access from your localhost (default username is "elastic")
```
curl -u "elastic:$PASSWORD" -k "https://localhost:9200"
```
* If you got response like this, you have successfully deploy your first elasticsearch cluster!! :metal:
```
{
  "name" : "quickstart-es-default-0",
  "cluster_name" : "quickstart",
  "cluster_uuid" : "XqWg0xIiRmmEBg4NMhnYPg",
  "version" : {...},
  "tagline" : "You Know, for Search"
}
```
## Deploy Kibana
* Create Kibana from `/manifests/setup/kibana.yaml`
```
kubectl apply -f ./manifests/setup/kibana.yaml
```
```
kubectl port-forward service/quickstart-kb-http 5601
```
```
kubectl get secret quickstart-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode; echo
```
```
https://localhost:5601
```
## Deploy Filebeats
* Create Filebeats from `/manifests/setup/filebeats.yaml`
```
kubectl apply -f ./manifests/setup/filebeats.yaml
```

## Deploy a busybox applicaton to produce some logs


## To Remove ECK from your Kubernetes cluster

1. Delete all elastic application and resources
```
kubectl get namespaces --no-headers -o custom-columns=:metadata.name \
  | xargs -n1 kubectl delete elastic --all -n
```

2. Remove the ECK operator
```
kubectl delete -f https://download.elastic.co/downloads/eck/1.9.0/crds.yaml
kubectl delete -f https://download.elastic.co/downloads/eck/1.9.0/operator.yaml
```
