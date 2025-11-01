# Develop Python app
- Create a fresh Python environment:
```
pyenv virtualenv 3.10.18 backstage-python-app
echo "backstage-python-app" > .python-version
pip install -r requirements.txt
````
- Develop simple Flask app
- Run the app locally:
```
python src/app.py
````
Browse to:
- http://127.0.0.1:5000/api/v1/details
- http://127.0.0.1:5000/api/v1/healthz

# Run app in Docker and push it to Docker registry
- Install Docker following instructions at https://docs.docker.com/engine/install/ubuntu/.
- Create Dockerfile
- Create Docker image:
```
docker build -t python-app:latest .
```
- Create Docker container:
```
docker run -p 8080:5000 python-app:latest
```
- Register at Docker Hub: 
  - https://hub.docker.com/repositories/jrbarrio
- Create repository in Docker Hub (python-app): 
  - https://hub.docker.com/repository/docker/jrbarrio/python-app/general
- Create a Personal Access Token (python-app): 
  - https://app.docker.com/accounts/jrbarrio/settings/personal-access-tokens
- Upload image at Docker Hub registry:
```
docker login -u jrbarrio -p <DOCKER_HUB_LOCAL_PAT>
docker tag python-app:latest jrbarrio/python-app:latest
docker push jrbarrio/python-app:latest
```

# Deploy the app to Kubernetes
- Install Kind:
  - https://kind.sigs.k8s.io/docs/user/quick-start/#installing-from-release-binaries
- Create a cluster:
```
kind create cluster
```
- Install kubectl:
  - https://v1-32.docs.kubernetes.io/docs/tasks/tools/install-kubectl-linux/
- Install kubernetes dashboard:
  - https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/
```
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/

helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard

kubectl apply -f k8s/service-account.yaml

kubectl -n kube-system create token admin_user

kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443
```
- Kubernetes dashboard can be accessed at:
  - https://localhost:8443/
- Configure Ingress controller:
  - https://kind.sigs.k8s.io/docs/user/ingress/
```
cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
EOF

kubectl apply -f https://kind.sigs.k8s.io/examples/ingress/deploy-ingress-nginx.yaml

kubectl get pods -n ingress-nginx
```
- Deploy the app on kubernetes:
```
kubectl apply -f k8s/deploy.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/ingress.yaml
```
- Modify `\etc\hosts` file:
```
127.0.0.1   python-app.test.com
```

# Use Helm to deploy the application
- Install Helm:
  - https://helm.sh/docs/intro/install/
- Create Helm chart:
```
helm create python-app
```
- Delete previously installed service:
```
kubectl delete -f k8s/deploy.yaml
kubectl delete -f k8s/service.yaml
kubectl delete -f k8s/ingress.yaml
```
- Install the chart:
```
helm upgrade --install python-app -n python-app charts/python-app --create-namespace
```
- Access the app:
  - http://python-app.test.com/api/v1/details

# Use ArgoCD to help with the deployment of the app
- Install ArgoCD on the cluster:
  - https://github.com/argoproj/argo-helm/tree/main/charts/argo-cd
```
helm repo add argo https://argoproj.github.io/argo-helm

helm upgrade --install argocd argo/argo-cd -n argocd --create-namespace -f charts/argocd/values-argo.yaml

kubectl get pods -n argocd

kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```
- Modify `\etc\hosts` file:
```
127.0.0.1   argocd.test.com
```
- Browse to ArgoCD:
  - https://argocd.test.com
- Link ArgoCD to Github repo:
  - https://argocd.test.com/settings/repos
- Create application in ArgoCD:
  - https://argocd.test.com/applications

# Create a pipeline that automatizes Docker image building and publication (Github Actions)
- Allow all actions and reusable workflows on the Github repo:
  - https://github.com/jrbarrio/python-app/settings/actions
- Create Github Actions secret to store Docker Hub username and password:
  - https://github.com/jrbarrio/python-app/settings/secrets/actions
- Create Gitbub Actions CI job on a new workflow executing the following actions:
  - Shorten commit id
  - Login to Docker Hub (secret required)
  - Build and push Docker image

# Extend the pipeline to automatize the app deployment after successful building (Github Actions)
- Create a Github Personal Access Token (PAT):
  - https://github.com/settings/tokens
- Install Actions Runner Controller for Github Actions in Kubernetes:
  - https://docs.github.com/en/actions/tutorials/use-actions-runner-controller/quickstart
```
NAMESPACE="arc-systems"
helm install arc \
    --namespace "${NAMESPACE}" \
    --create-namespace \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller

INSTALLATION_NAME="arc-runner-set"
NAMESPACE="arc-runners"
GITHUB_CONFIG_URL="https://github.com/jrbarrio/python-app"
GITHUB_PAT="<PAT>"
helm install "${INSTALLATION_NAME}" \
    --namespace "${NAMESPACE}" \
    --create-namespace \
    --set githubConfigUrl="${GITHUB_CONFIG_URL}" \
    --set githubConfigSecret.github_token="${GITHUB_PAT}" \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```
- See self-hosted runners at:
  - https://github.com/jrbarrio/python-app/actions/runners?tab=self-hosted
- Create Gitbub Actions CD job on a new workflow executing the following actions:
  - Install Python
  - Modify values file
  - Install ArgoCD
  - Update app in ArgoCD