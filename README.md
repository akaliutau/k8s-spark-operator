About
======

This is a step-by-step guide to create an ephemeral Spark cluster on Kubernetes via Spark Operator for R + D purposes

This particular demo runs once a simple Spark app (which calculates pi number and prints output to STDOUT)

Pre-requisites
==============

1. Install the latest versions of kubectl and helm:

https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

https://helm.sh/docs/intro/install/

2. Install kubectl plugin called "gke-gcloud-auth-plugin"

https://cloud.google.com/blog/products/containers-kubernetes/kubectl-auth-changes-in-gke

```shell
sudo apt-get install google-cloud-sdk-gke-gcloud-auth-plugin
```

Creating Kubernetes cluster at GKE
==================================

```shell
gcloud config set project dev-k8s-playground
gcloud services enable container
gcloud container clusters create crowdstrike-k8s-cluster \
    --zone us-central1-a \
    --num-nodes 3 \
    --machine-type n1-standard-2
```
NB: using a too small machine for master node may result in infinite `Pending` status

Connect `kubectl` to cluster

```shell
export USE_GKE_GCLOUD_AUTH_PLUGIN=True
gcloud container clusters get-credentials crowdstrike-k8s-cluster --zone us-central1-a --project dev-k8s-playground
kubectl version
helm version
```

Installing Spark Operator
==========================

https://github.com/GoogleCloudPlatform/spark-on-k8s-operator

```shell
helm repo add spark-operator https://googlecloudplatform.github.io/spark-on-k8s-operator
helm install spo-release spark-operator/spark-operator --namespace spark-ns --create-namespace --set webhook.enable=true
```
NB: SA `spo-release-spark` will be auto-created, the exact name can be found via `kubectl get serviceaccount -n spark-ns`

Writing spec for Spark Job
===========================

https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/blob/master/docs/user-guide.md

Creating Spark Job from Kubernetes resource (SparkApplication)
==============================================================
```shell
kubectl apply -f spark-pi.yaml
kubectl get pods -n spark-ns
kubectl logs spark-pi-driver -n spark-ns
```

Other details:

```shell
kubectl get sparkapplications spark-pi -o=yaml -n spark-ns
kubectl describe sparkapplication spark-pi -n spark-ns
```

Observability
==============

To see Spark UI you will probably need to increase the number of slices to make app running longer (spec.arguments 30 -> 10000)

```shell
kubectl port-forward spark-pi-driver -n spark-ns 4040:4040
```

Go to localhost:4040 to see Spark UI

Clean up
========

Stop and delete all resources 
```shell
gcloud container clusters delete crowdstrike-k8s-cluster --zone us-central1-a
```

Noticed issues
==============

The spark-operator in the following image does not work:

`ghcr.io/googlecloudplatform/spark-operator:v1beta2-1.3.8-3.1.1`

Using OSS apache image:

`apache/spark:v3.1.3`


Using Spark History server for better observability
====================================================

Preliminarily steps to prepare GCS bucket and SA:

```shell
gsutil mb -c nearline gs://spark-history-server-logs-test
export ACCOUNT_NAME=sparkonk8s
export GCP_PROJECT_ID=dev-k8s-playground
gcloud iam service-accounts create ${ACCOUNT_NAME} --display-name "${ACCOUNT_NAME}"
gcloud iam service-accounts keys create "${ACCOUNT_NAME}.json" --iam-account "${ACCOUNT_NAME}@${GCP_PROJECT_ID}.iam.gserviceaccount.com"
gcloud projects add-iam-policy-binding ${GCP_PROJECT_ID} --member "serviceAccount:${ACCOUNT_NAME}@${GCP_PROJECT_ID}.iam.gserviceaccount.com" --role roles/storage.admin
gsutil iam ch serviceAccount:${ACCOUNT_NAME}@${GCP_PROJECT_ID}.iam.gserviceaccount.com:objectAdmin gs://spark-history-server-logs-test
```

Then create a secret using the JSON key file:
```shell
kubectl -n spark-ns create secret generic history-secrets --from-file=sparkonk8s.json
```

Creating a test job:

```shell
kubectl apply -f spark-pi-history-final.yaml
kubectl get pods -n spark-ns
kubectl logs spark-pi-driver -n spark-ns
```
All logs will be stored at GCS and available via Spark History Server UI

```shell
kubectl apply -f spark-pi-history-final.yaml
kubectl port-forward <spark-history-server-pod> -n spark-ns 18080:18080
```

Use cases and potential solutions
==================================

 

References
===========

1. https://github.com/GoogleCloudPlatform/spark-on-k8s-operator

2. https://doc.lucidworks.com/how-to/807/configure-the-spark-history-server 

