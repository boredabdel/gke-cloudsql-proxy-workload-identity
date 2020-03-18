# CloudSQL proxy on GKE with Workload identity example ###

### This repo is a sample of how to use Workload identity with GKE CloudSQL proxy, you will learn how to deploy a Workload Identity enabled cluster, deploy a CloudSQL proxy that uses a KSA (kubernetes Service Account) to authenticate to a the CloudSQL API via a proxy and Workload Identity ###

 ### NB: the commands below have been performed on a Linux system, MacOS should work the same way, i havent's tried this from a Windows computer, if you are a Windows user, maybe you can try using the Cloud Shell ###

## You will need gcloud and kubectl to be installed ##

### Start by exporting your project ID as an environment variable
```
export PROJECT_ID=xxxxx
```

### Authenticate on Set you default compute region and zone
```
gcloud config compute/region europe-west4
```
```
gcloud config compute/zone europe-west4-a
```

### Create a Cluster with 3 nodes and workload identity enabled
```
gcloud beta container --project "abdelfettahlab" clusters create "wp-cluster" --zone "europe-west4-a" --no-enable-basic-auth --release-channel "regular" --machine-type "n1-standard-1" --image-type "COS" --disk-type "pd-standard" --disk-size "100" --metadata disable-legacy-endpoints=true --scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" --num-nodes "4" --enable-stackdriver-kubernetes --enable-private-nodes --master-ipv4-cidr "172.16.0.0/28" --enable-ip-alias --network "projects/abdelfettahlab/global/networks/default" --subnetwork "projects/abdelfettahlab/regions/europe-west4/subnetworks/default" --default-max-pods-per-node "110" --no-enable-master-authorized-networks --addons HorizontalPodAutoscaling,HttpLoadBalancing --enable-autoupgrade --enable-autorepair --identity-namespace "abdelfettahlab.svc.id.goog"

```

## Create a MYSQL CloudSQL Instance, this command will create a MYSQL 5.7 instance with 1vCPU and 3.75GB of memory in europe-west4-a. Replace PASSWORD with a secure password, this is the root password for the DB.
```
gcloud beta sql instances create wordpress --no-assign-ip --database-version MYSQL_5_7 --memory 3840MiB --cpu 1  --network default --zone europe-west4-a --root-password PASSWORD
```

### Grab the GKE Cluster credentials
```
gcloud container clusters get-credentials wp-cluster
```

### Create a GSA (Google Service Account)
```
cloud iam service-accounts create cloudsql-proxy-sa
```

### Grant the GSA the required CloudSQL permissions
```
gcloud projects add-iam-policy-binding ${PROJECT_ID} --member "serviceAccount:cloudsql-proxy-sa@$PROJECT_ID.iam.gserviceaccount.com" --role roles/cloudsql.admin
```

### Create a namespace in the cluster ###
```
kubectl create namespace wordpress
```

### Create a KSA (Kubernetes Service Account) that will be used by the proxy in the wordpress namespace ###
```
kubectl create serviceaccount wordpress-sa -n wordpress
```

### Allow the Kubernetes service account to use the Google Service Account ###
```
gcloud iam service-accounts add-iam-policy-binding \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:$PROJECT_ID.svc.id.goog[wordpress/wordpress-sa]" \
  cloudsql-proxy-sa@$PROJECT_ID.iam.gserviceaccount.com
```

### Add the iam.gke.io/gcp-service-account=gsa-name@project-id annotation to the Kubernetes service account ###

```
kubectl annotate serviceaccount \
  --namespace wordpress \
  wordpress-sa \
  iam.gke.io/gcp-service-account=cloudsql-proxy-sa@$PROJECT_ID.iam.gserviceaccount.com
```

### Now you are ready to deploy wordpress, start by creating a secret containing the db username and password (Replace PASSWORD with the password used when creating the CloudSQL instance ###
```
kubectl create secret generic cloudsql-db-credentials --from-literal=username=root --from-literal=password=PASSWORD -n wordpress
```

### Get you instance connection name ###
```
gcloud sql instances describe wordpress | grep connectionName
```

### Replace the <INSTANCE_CONNECTION_NAME> in the wordpress.yaml file with the output of the previous command, then deploy wordpress ###
```
kubectl apply -f wordpress.yaml
```

### Verify you pod is running ###
```
kubectl get pods -n wordpress
```

### When the wordpress pod is ready you can connect to it using kubectl port-forward, replace POD_NAME with the output of the previous command ###
```
kubectl port-forward POD_NAME -n wordpress 8082:80
```

### Open http://localhost:8082 on your computer, wordpress should be up ###