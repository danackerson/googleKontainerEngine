# Launch your very own Kubernetes Cluster on GCP
This repo launches the cheapest possible (~$9/month http + IPv4 only) K8S cluster in the Google Cloud Platform in region us-east1 with standard disks and preemptible (killed w/i 24hrs), f1-micro instances.
By adding the appropriate ENV VARS to your CircleCI build, you can enable SSL & IPv6 (but each one of these doubles the cost of the cluster due to number of forwarding rules on the global load balancer).

# Prerequisites
- [Service Account in GCP](#create-a-service-account)
- CircleCI
- Docker Hub account (if you are managing your container images there)

## Create a Service Account
- Navigate to the IAM & admin" section of your project and select ["Service Accounts"](https://console.cloud.google.com/iam-admin/serviceaccounts)
- Create a new Account with the role "Kubernetes Engine Admin" (there are other 'K8S Admin roles, but they won't suffice)
- Ask for a new private key for download in JSON format

## Setup CircleCI env vars
The following env vars are required unless marked as *optional*.

*SSL_ENABLED* if not present, http:// only
*IPV6_ENABLED* if not present, IPV4 only

KUBERNETES_ENGINE_VERSION (e.g. [1.10.5-gke.0](https://cloud.google.com/kubernetes-engine/versioning-and-upgrades#versions_available_for_new_cluster_masters) )

PRIMARY_DOMAIN (e.g. ackerson.de)
**NOTE:** This deployment assumes you have a Google Cloud DNS Zone named `${PRIMARY_DOMAIN//.}` (e.g. `ackersonde`)

*CERTBOT_EMAIL* (e.g. dan@ackerson.de)
*CERTBOT_CSV_ALL_DOMAINS* (e.g. ackerson.de, www.ackerson.de)
 - if you specify SSL but omit this env var, $PRIMARY_DOMAIN is used to gen the cert

For setting up Docker Registry Credentials as a K8S Secret:
*DOCKER_USERNAME*
*DOCKER_PASSWD*
*DOCKER_EMAIL*

For launching & managing the cluster, follow this [CircleCI guide](https://circleci.com/docs/2.0/google-auth/) adding:
GCLOUD_SERVICE_KEY
GOOGLE_APPLICATION_CREDENTIALS
GOOGLE_CLUSTER_NAME
GOOGLE_COMPUTE_ZONE
GOOGLE_PROJECT_ID

# Pointing the Ingress to your services & applications
The deploy.yaml references a serviceName which will point to your application deployment.
In this case, it's my [homepage](https://github.com/danackerson/ackerson.de-go/blob/k8s/deploy.yaml#L49).
It is **NOT** required that you deploy this application before setting up your cluster and ingress. When you do deploy your app here, be patient - it may take up to 5 minutes for the load balancers to see the new service.

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

# Warnings
- If you deploy within a CircleCI config using a GitHub webhook, remember that every time you push this repo a cluster may be launched. Skip the deploy by just adding `[skip ci]` to your commit message.

- If you choose to enable IPv6, the address is "manually" added to the load balancer and doubles the number of rules: your monthly invoice (with SSL) can reach ~$36. These additions of rules on the K8S load balancer are NOT handled when you delete your cluster. TL/DR; After deleting your cluster, manually inspect the load balancer and ensure all rules are deleted or pay ~$25/month for NOTHING! You also won't be able to delete the reserved IPv6 address until you delete the rules.

- If you delete the cluster, remember that minimally an IPv4 static IP was reserved and will end up costing you ~$7/month for NOTHING - so delete it!
