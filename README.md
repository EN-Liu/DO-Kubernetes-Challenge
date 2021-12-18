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

## The architcture
The architecture is divide into three part:
* Filebeat: Collect the logs produced from pod
* Elasticsearch: Store the logs
* Kibana: Log statistics  visualization
![](https://i.imgur.com/89OBhIJ.png)


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
>With Kibana, you can:
>* Search and observe your data. 
>* Analyze your data. Search for hidden insights, visualize what you’ve found in charts, gauges, maps, graphs, and more, and combine them in a dashboard.
>* Manage, monitor, and secure the Elastic Stack. Manage your data, monitor the health of your Elastic Stack cluster, and control which users have access to which features.

* Create Kibana as Deployment from `/manifests/setup/kibana.yaml`
```
kubectl apply -f ./manifests/setup/kibana.yaml
```
* Open a seperate terminal and port-forward your Kibana service to your localhost
```
kubectl port-forward service/quickstart-kb-http 5601
```
* Get your credentials from the secret(Same as Elasticsearch's password)
```
kubectl get secret quickstart-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode; echo
```
* Open https://localhost:5601 in your browser and login as username elastic and the password you get from the secret
```
https://localhost:5601
```
## Deploy Filebeats
>Filebeat is a lightweight shipper for forwarding and centralizing log data. Installed as an agent on your servers, Filebeat monitors the log files or locations that you specify, collects log events, and forwards them either to Elasticsearch or Logstash for indexing.

* Create Filebeats as Daemonset from `/manifests/setup/filebeats.yaml`
```
kubectl apply -f ./manifests/setup/filebeats.yaml
```
* Filebeats will run as agent on every node to collect logs

## Deploy a busybox applicaton to produce some logs

* Using kubectl run to create a busybox pod that produce "hello world" log
```
kubectl run counter --image=busybox --dry-run=client -o yaml -- /bin/sh, -c, 'i=0; while true; do echo "hello world: $i: $(date)"; i=$((i+1)); sleep 3; done
```

## Check the logs in Kibana
* Open Kibana in the browser and login with default 
    * username: elastic
    * password: check the elastic secret using command:
        *  `kubectl get secret quickstart-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode; echo`

 ![](https://i.imgur.com/OdXcx7V.png)

* Select Kibana and select "Discover"

 ![](https://i.imgur.com/VYzxeia.png)

* Type in "hello world" in the search bar for full text search
 ![](https://i.imgur.com/zodKAUH.png)
* Now you have monitored your first log!

## What next?
* Configure a certificate for the ECK deployment
* replace filebeat with fluentd or add logstash to parse logs
* Many else to try out!

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
