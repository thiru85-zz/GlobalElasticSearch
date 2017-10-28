# Globally Federated and AutoScalable ElasticSearch Cluster with Cloud Endpoints
Yes, it is a mouthful.
                                                             
## Purpose: Commands and order needed for building a federated autoscalable
##          elasticsearch cluster on Google Container Engine on GCP
## Date: 28/10/2017   

### Set auth options for gcloud before cluster auth

```
gcloud config set container/use_client_certificate True
```

### Create clusters in 3 regions globally	
```
gcloud container clusters create elasticsearchclusterasia01 --zone asia-southeast1-a --scopes cloud-platform --num-nodes 2 --enable-autoscaling --min-nodes=2 --max-nodes=7 & gcloud container clusters create elasticsearchclustereurope01 --zone europe-west2-a --scopes cloud-platform --num-nodes 2 --enable-autoscaling --min-nodes=2 --max-nodes=7 & gcloud container clusters create elasticsearchclusterus01 --zone us-central1-a --scopes cloud-platform --num-nodes 2 --enable-autoscaling --min-nodes=2 --max-nodes=7
```
### Create managed DNS Zone
```
gcloud dns managed-zones create elasticfed --description “ElasticSearch Kubernetes Federation Zone” --dns-name elasticfed.gcpdemo.xyz
```

### Add Container Clusters to kubectl
```
gcloud container clusters get-credentials elasticsearchclusterasia01 --zone asia-southeast1-a --project peak-responder-174202
gcloud container clusters get-credentials elasticsearchclustereurope01 --zone europe-west2-a  --project peak-responder-174202
gcloud container clusters get-credentials elasticsearchclusterus01 --zone us-central1-a --project peak-responder-174202
```

### Set contexts using RFC1192 standards
#### AsiaContext
```
kubectl config set-context asiacontext --cluster gke_peak-responder-174202_asia-southeast1-a_elasticsearchclusterasia01 --user gke_peak-responder-174202_asia-southeast1-a_elasticsearchclusterasia01
kubectl config delete-context gke_peak-responder-174202_asia-southeast1-a_elasticsearchclusterasia01
```
#### USContext
```
kubectl config set-context uscontext --cluster gke_peak-responder-174202_us-central1-a_elasticsearchclusterus01 --user gke_peak-responder-174202_us-central1-a_elasticsearchclusterus01
kubectl config delete-context gke_peak-responder-174202_us-central1-a_elasticsearchclusterus01
```
#### EuropeContext
```
kubectl config set-context europecontext --cluster gke_peak-responder-174202_europe-west2-a_elasticsearchclustereurope01 --user gke_peak-responder-174202_europe-west2-a_elasticsearchclustereurope01
kubectl config delete-context gke_peak-responder-174202_europe-west2-a_elasticsearchclustereurope01
```

### Set Active Context for kubectl
```
kubectl config use-context uscontext
```
### Create the Federation Control Plane for kubernetes (this creates the Federation with the 3 clusters created previously)
```
kubefed init federated-elasticsearch --host-cluster-context=uscontext --dns-zone-name="elasticfed.gcpdemo.xyz." --dns-provider="google-clouddns"
```
### Join container clusters to Federation Control Plane
```
kubefed --context=federated-elasticsearch join elasticsearchclusterus01 --cluster-context=uscontext --host-cluster-context=uscontext
kubefed --context=federated-elasticsearch join elasticsearchclustereurope01 --cluster-context=europecontext --host-cluster-context=uscontext
kubefed --context=federated-elasticsearch join elasticsearchclusterasia01 --cluster-context=asiacontext --host-cluster-context=uscontext
```
### Create NameSpace and apply DNS ConfigMap
```
kubectl --context=federated-elasticsearch create ns default
kubectl --context=federated-elasticsearch create ns kube-system
kubectl --context=federated-elasticsearch apply -f kubedns-config.yml
```
### Create PersistentVolumeClaims
```
kubectl --context=europecontext create -f elasticsearch-PersistentVolumeClaim.yaml
kubectl --context=asiacontext create -f elasticsearch-PersistentVolumeClaim.yaml
kubectl --context=uscontext create -f elasticsearch-PersistentVolumeClaim.yaml
```
### Create Services (Discovery and HTTP services)
```
kubectl --context=federated-elasticsearch apply -f elasticsearch-discovery-service.yaml
kubectl --context=federated-elasticsearch apply -f elasticsearch-service.yaml
```
### Create Deployments in Every region 
This needs to be done on a per-context basis because the initcontainer needs Privileged Access, which is not currently available in the Kubernetes Federation API
#### EuropeContext Deployment Creation
```
kubectl --context=europecontext create -f elasticsearch-master.yaml
kubectl --context=europecontext create -f elasticsearch-client.yaml
kubectl --context=europecontext create -f elasticsearch-data.yaml
```
#### AsiaContext Deployment Creation
```
kubectl --context=asiacontext create -f elasticsearch-master.yaml
kubectl --context=asiacontext create -f elasticsearch-client.yaml
kubectl --context=asiacontext create -f elasticsearch-data.yaml
```
#### USContext Deployment Creation
```
kubectl --context=uscontext create -f elasticsearch-master.yaml
kubectl --context=uscontext create -f elasticsearch-client.yaml
kubectl --context=uscontext create -f elasticsearch-data.yaml
```

### AutoScale Creation on a per-context basis as well.
#### EuropeContext AutoScaler
```
kubectl --context=europecontext create -f elasticsearch-autoscaler-master.yaml
kubectl --context=europecontext create -f elasticsearch-autoscaler-client.yaml
kubectl --context=europecontext create -f elasticsearch-autoscaler-data.yaml
```
#### AsiaContext AutoScaler
```
kubectl --context=asiacontext create -f elasticsearch-autoscaler-master.yaml
kubectl --context=asiacontext create -f elasticsearch-autoscaler-client.yaml
kubectl --context=asiacontext create -f elasticsearch-autoscaler-data.yaml
```
#### USContext AutoScaler
```
kubectl --context=uscontext create -f elasticsearch-autoscaler-master.yaml
kubectl --context=uscontext create -f elasticsearch-autoscaler-client.yaml
kubectl --context=uscontext create -f elasticsearch-autoscaler-data.yaml
```

### Create Ingress

The final piece of the puzzle, the Ingress, which is the magic that load-balances the traffic to the appropriate region.

```
kubectl --context=federated-elasticsearch create -f elasticsearch-ingress
```

### Delete everything

kubectl config delete-context

gcloud container clusters delete elasticsearchclusterasia01 --zone asia-southeast1-a
gcloud container clusters delete elasticsearchclustereurope01 --zone europe-west2-a
gcloud container clusters delete elasticsearchclusterus01 --zone us-central1-a


gcloud compute firewall-rules create federated-ingress-firewall-rule --source-ranges 130.211.0.0/22 --allow tcp:30036 --network default
