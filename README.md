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
- Create a cluster (prepared for Ingress):
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
```
- Install kubectl:
  - https://v1-32.docs.kubernetes.io/docs/tasks/tools/install-kubectl-linux/
- Configure Ingress controller:
  - https://kind.sigs.k8s.io/docs/user/ingress/
```
kubectl apply -f https://kind.sigs.k8s.io/examples/ingress/deploy-ingress-nginx.yaml

kubectl get pods -n ingress-nginx
```
- Deploy the app on kubernetes:
```
kubectl apply -f k8s/app/deploy.yaml
kubectl apply -f k8s/app/service.yaml
kubectl apply -f k8s/app/ingress.yaml
```
- Modify `\etc\hosts` file:
```
127.0.0.1   python-app.test.com
```

# Install Kubernetes dashboard
- Modify `\etc\hosts` file:
```
127.0.0.1   dashboard.test.com
```
- Install kubernetes dashboard:
  - https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/
```
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/

helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard

kubectl apply -f k8s/dashboard/service-account.yaml
kubectl apply -f k8s/dashboard/ingress.yaml

kubectl -n kube-system create token admin-user

## kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443 ---> (Chrome) https://localhost:8443
```
- Kubernetes dashboard can be accessed at:
  - https://dashboard.test.com/

# Use Helm to deploy the application
- Install Helm:
  - https://helm.sh/docs/intro/install/
- Create Helm chart:
```
helm create python-app
```
- Delete previously installed service:
```
kubectl delete -f k8s/app/deploy.yaml
kubectl delete -f k8s/app/service.yaml
kubectl delete -f k8s/app/ingress.yaml
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
  - Install ArgoCD CLI
  - Update app in ArgoCD using CLI

# Install Backstage
- Follow instructions at:
  - https://backstage.io/docs/getting-started/
- Get Node Docker image:
```
docker pull node:20-bookworm-slim
```
- Run Node in a Docker container:
```
docker run --rm -p 3000:3000 -ti -p 7007:7007 -v /home/jorge/Projects/Udemy/PlatformEngineering/python-app/backstage:/app -w /app node:20-bookworm-slim bash
```
- Create Backstage app in the container:
```
npx @backstage/create-app@latest
```
- Execute:
```
cd backstage && yarn start
```
- Configure frontend host on `app-config.yaml`:
```
app:
  title: Scaffolded Backstage App
  baseUrl: http://localhost:3000
  listen:
    host: 0.0.0.0
```
- Create OAuth application in Github:
  - https://github.com/settings/developers
- Configure Backstage authentication with Github Authentication Provider on `app-config.local.yaml`:
  - https://backstage.io/docs/auth/github/provider
- Export required environment variables to Docker container:
```
docker run --rm -e AUTH_GITHUB_CLIENT_ID={AUTH_GITHUB_CLIENT_ID} -e AUTH_GITHUB_CLIENT_SECRET={AUTH_GITHUB_CLIENT_SECRET} -p 3000:3000 -ti -p 7007:7007 -v /home/jorge/Projects/Udemy/PlatformEngineering/python-app/backstage:/app -w /app node:20-bookworm-slim bash
```
- Install the provider package:
```
yarn --cwd packages/backend add @backstage/plugin-auth-backend-module-github-provider
```
- Add to `index.ts`:
```
backend.add(import('@backstage/plugin-auth-backend'));
backend.add(import('@backstage/plugin-auth-backend-module-github-provider'));
```
- Do required changes on frontend:
  - https://backstage.io/docs/auth/#sign-in-configuration
- Create user catalog in `catalog\entities\users.yaml`.

# Backstage Software Catalog
- Create `catalog-info.yaml` file in `python-app`.
- Register new component and use Github URL:
  - https://github.com/jrbarrio/python-app/blob/main/catalog-info.yaml

# Backstage Tech Docs
- Install MkDocs in Backstage.
  - https://backstage.io/docs/features/techdocs/getting-started/
- Configure MkDocs to run in local at `app-config.local.yaml`:
```
techdocs:
  builder: 'local'
  publisher:
    type: 'local'
  generator:
    runIn: local
```
- Install MkDocs in Backstage app container:
```
apt-get update && \
  apt-get install -y python3 python3-pip python3-venv && \
  rm -rf /var/lib/apt/lists/*

VIRTUAL_ENV=/opt/venv
python3 -m venv $VIRTUAL_ENV
PATH="$VIRTUAL_ENV/bin:$PATH"

pip3 install mkdocs-techdocs-core
```
- Create application documentation in Markdown. Start on `docs\index.md`.
- Render the documentation as HTML using MkDocs:
  - https://example-mkdocs-basic.readthedocs.io/en/latest/
  - Configuration must be in the `mkdocs.yaml` file.
- Documentation available at:
  - http://localhost:3000/docs/default/Component/python-app

# Backstage Software Templates
- Documentation:
  - https://backstage.io/docs/features/software-templates/
  - https://backstage.io/docs/features/software-templates/writing-templates
- Install Publish Github action:
  - https://backstage.io/docs/features/software-templates/builtin-actions#installing-action-modules
- In `app-config.local.yaml` create an integrations section with a Github Token.
```
docker run --rm -e AUTH_GITHUB_CLIENT_ID={AUTH_GITHUB_CLIENT_ID} -e AUTH_GITHUB_CLIENT_SECRET={AUTH_GITHUB_CLIENT_SECRET} -e GITHUB_TOKEN={GITHUB_TOKEN} -p 3000:3000 -ti -p 7007:7007 -v /home/jorge/Projects/Udemy/PlatformEngineering/python-app/backstage:/app -w /app node:20-bookworm-slim bash
```
- Create a new Github repo for Backstage software templates:
  - https://github.com/jrbarrio/backstage-software-templates
- Add a new rule in `app-config.local.yaml`:
```
catalog:
  rules:
    - ...
    - type: url
      target: https://github.com/jrbarrio/backstage-software-templates/blob/main/python-app/template.yaml
      rules:
        - allow: [Template]
```
- If you're running Backstage with Node 20 or later, you'll need to pass the flag --no-node-snapshot to Node in order to use the templates feature:
```
docker run --rm -e AUTH_GITHUB_CLIENT_ID={AUTH_GITHUB_CLIENT_ID} -e AUTH_GITHUB_CLIENT_SECRET={AUTH_GITHUB_CLIENT_SECRET} -e GITHUB_TOKEN={GITHUB_TOKEN} -e NODE_OPTIONS="${NODE_OPTIONS:-} --no-node-snapshot" -p 3000:3000 -ti -p 7007:7007 -v /home/jorge/Projects/Udemy/PlatformEngineering/python-app/backstage:/app -w /app node:20-bookworm-slim bash
```
# Github Organizations
- Create a Github Organization in your personal account:
  - https://github.com/jorgeroldanbarrio
- Copy the previously used Action secrets into the organization configuration:
  - https://github.com/organizations/jorgeroldanbarrio/settings/secrets/actions
- Configure the Backstage template to create the app in the Github Organization:
```
    - id: publish
      name: Publish
      action: publish:github
      input:
        description: This is ${{ parameters.component_id }}
        repoUrl: github.com?owner=jorgeroldanbarrio&repo=${{parameters.component_id}}
        protectDefaultBranch: false
        repoVisibility: public
```
- Reconfigure existing Github Actions Runners to run on the organization's repo:
```
INSTALLATION_NAME="arc-runner-set"
NAMESPACE="arc-runners"
GITHUB_CONFIG_URL="https://github.com/jorgeroldanbarrio"
GITHUB_PAT="<PAT>"
helm upgrade "${INSTALLATION_NAME}" \
    --namespace "${NAMESPACE}" \
    --create-namespace \
    --set githubConfigUrl="${GITHUB_CONFIG_URL}" \
    --set githubConfigSecret.github_token="${GITHUB_PAT}" \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```
- Watch the runner appear at https://github.com/organizations/jorgeroldanbarrio/settings/actions/runners.
- Go to Actions General settings and set "Read and write permissions" on "Workflow permissions" section.

# Build custome Backstage image

```
docker build -t backstage_custom .
```