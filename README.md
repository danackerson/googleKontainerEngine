# Launching your very own Kubernetes Cluster on GCP
This repo launches the cheapest possible (~$6/month) K8S cluster in the Google Cloud Platform in region us-east1 with slow, standard disks and preemptible (killed w/i 24hrs) f1-micro instances.

# Prerequisites
- [Service Account in GCP](#create-a-service-account)
- CircleCI
- Docker Hub account (if you are managing your container images there pull from)

## Create a Service Account
- Navigate to the IAM & admin" section of your project and select ["Service Accounts"](https://console.cloud.google.com/iam-admin/serviceaccounts)
- Create a new Account with the role "Kubernetes Engine Admin" (there are other 'K8S Admin roles, but they won't suffice)
- Ask for a new private key for download in JSON format

## Setup CircleCI env vars
For setting up Docker Registry Credentials as a K8S Secret (skip if you use gcr.io):
```
DOCKER_EMAIL
DOCKER_PASSWD
DOCKER_USERNAME
```

For launching & managing the cluster, follow this [CircleCI guide](https://circleci.com/docs/2.0/google-auth/) adding:
```
GCLOUD_SERVICE_KEY
GOOGLE_APPLICATION_CREDENTIALS
GOOGLE_CLUSTER_NAME
GOOGLE_COMPUTE_ZONE
GOOGLE_PROJECT_ID
```

# Accessing the K8S cluster locally including the Dashboard
```
gcloud auth login
gcloud config set project $GOOGLE_PROJECT_ID
gcloud --quiet container clusters --region $GOOGLE_COMPUTE_ZONE get-credentials $GOOGLE_CLUSTER_NAME
gcloud config config-helper --format=json | jq .credential.access_token
kubectx (ensuring the current context is your new cluster)
kubectl proxy
```
Now navigate to http://127.0.0.1:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/login selecting the 'Token' option and paste the output of the `.credential.access_token` parsing above.

Sweet!

# Warnings (if you use CircleCI)
As the deployment is within the CircleCI config, every time you push this repo a cluster may be launched. Skip the deploy by just adding `[skip ci]` to your commit message.
