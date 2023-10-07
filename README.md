# ArgoCD Self Managed


## How to install 
1. Create a file called *argocd-install/values-gke.yaml* based on *argocd-install/values-gke.example.yaml*.
2. Execute the following commands.
```sh
# If Minikube, start your cluster
minikube start

# If GKE, get the cluster credentials
export CLUSTER_NAME="YOUR_CLUSTER_NAME"
export CLUSTER_REGION="YOUR_CLUSTER_REGION"
export GCP_PROJECT_ID="YOUR_GCP_PROJECT_ID"
gcloud container clusters get-credentials $CLUSTER_NAME --region $CLUSTER_REGION --project $GCP_PROJECT_ID

# Go to argocd-install folder
cd argocd-install

# If GKE, put your cert into keys
mkdir keys

# Create Namespaces
kubectl create ns argocd
kubectl create namespace ingress-nginx

## Deploy Secrets
kubectl -n argocd create secret tls argocd-server-tls --cert keys/tls.crt --key keys/tls.key

## If Google SSO
kubectl -n argocd create secret generic  argocd-google-groups-json --from-file=keys/googleAuth.json


# If GKE, Install Ingress, if minikube, ignore.
helm install -n ingress-nginx argocd-ingress-nginx ./ingress-nginx --set "controller.extraArgs.enable-ssl-passthrough=" --set controller.admissionWebhooks.enabled=false

# Install Argo
helm install -n argocd argocd ./argo-cd -f values-<YOUR_FILE>.yaml 
```

## Post Installation
1. **If GKE, Get the ingress IP and add a DNS entry for that**. <br>
If minikube, expose through port forwarding:
```sh
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

2. Add the Git Repository by CLI or use the Web Interface 
```sh
UI user: admin
UI pass: kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

repo https: YOUR_HTTP_REPO_URL
username: YOUR_USERNAME
password: YOUR_PASS OR PERSONAL ACCESS TOKEN
```

3. Create the core apps
```sh
kubectl apply -f argocd-core-projects.yaml
kubectl apply -f argocd-core-applications.yaml
```

## How to Use
1. Define your **ApplicationSet** or **Application** in **argocd-apps** folder.
2. Define the **AppProject** for your applications with the right permissions in **argocd-appprojects** or edit a existing one with the cluster and namespaces allowed.
3. Create a repository to store your charts inside the DevOps/GitOps or other Group. Folder Structure Suggestion:

```
.
└── my-application/
    ├── chart/
    ├── configs/ (optional)
    ├── values-override.yaml
    └── values-<any>.yaml (optional)
```
4. Add your application repository in ArgoCD.

## Troubleshooting: 
1. **SSO doesn't work:** Restart the DEX Server Deployment
2. **Invalid certificate:** Try to register your SSL certificate at your cloud cert manager, ensure ArgoCD is deployed in insecure mode, and the nginx controller has ssl-passthrough enabled.
