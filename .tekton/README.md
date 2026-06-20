# Tekton Documentation

## Prerequisites

Before you begin, make sure you have the following in place:

- A local Kubernetes cluster like Kind or Minikube
- The **Tekton Pipelines** installed on your cluster
- The **Tekton CLI** installed, and the **Pipelines-as-Code CLI plugin** (`tkn-pac`) installed

## Installation

Use command below to install **Tekton Pipeline**

```sh
kubectl apply --filename https://infra.tekton.dev/tekton-releases/pipeline/latest/release.yaml
```

Install **Tekton CLI**

```sh
TKN_CLI_VERSION=0.43.2
curl -LO "https://github.com/tektoncd/cli/releases/download/v${TKN_CLI_VERSION}/tektoncd-cli-${TKN_CLI_VERSION}_Linux-64bit.deb"
sudo dpkg -i "./tektoncd-cli-${TKN_CLI_VERSION}_Linux-64bit.deb"

# For verifying
tkn version
```

Install **Tekton Pipeline-as-code** `tkn-pac`

```sh
TKN_PAC_VERSION=0.39.7
curl -LO "https://github.com/tektoncd/pipelines-as-code/releases/download/v${TKN_PAC_VERSION}/tkn-pac-${TKN_PAC_VERSION}_linux-x86_64.deb"
sudo dpkg -i "./tkn-pac-${TKN_PAC_VERSION}_linux-x86_64.deb"
```

Install **Gosmee**. This is a webhook relay server to relay Github Webhook to your internal Tekton Pipelines

```sh
GOSMEE_VERSION=0.31.1
curl -LO "https://github.com/chmouel/gosmee/releases/download/v${GOSMEE_VERSION}/gosmee-${GOSMEE_VERSION}_linux-x86_64.deb"
sudo dpkg -i "./gosmee-${GOSMEE_VERSION}_linux-x86_64.deb"
```

### Optional Dependencies

#### Tekton Dashboard

Install the dashboard

```sh
# By default, Tekton Dashboard is installed READONLY mode.
kubectl apply --filename https://infra.tekton.dev/tekton-releases/dashboard/latest/release.yaml

# If you want to install Tekton Dashboard with READ-WRITE mode, use command below.
kubectl apply --filename https://infra.tekton.dev/tekton-releases/dashboard/latest/release-full.yaml
```

To access dashboard, create an Ingress resource.

```sh
DASHBOARD_URL=dashboard.saritasa.test.com
DASHBOARD_PATH=/
kubectl apply -n tekton-pipelines -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dashboard-ingress
  namespace: tekton-pipelines
spec:
  rules:
  - host: $DASHBOARD_URL
    http:
      paths:
      - pathType: $DASHBOARD_PATH
        backend:
          service:
            name: tekton-dashboard
            port:
              number: 9097
EOF
```

## Implementation

### Enable Ingress Controller Addons (Optional)

***Note***: If the addon is enabled in your Minikube cluster, then you don't need to run this step.

You will need a route for the webhook to work successfully, you need to enable the ingress controller addon.

```sh
minikube addons enable ingress
```

### Create a public route for webhook

Run below command to create an ingress

```sh
DASHBOARD_URL=hook.saritasa.test.com
DASHBOARD_PATH=/
kubectl apply -n tekton-pipelines -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: pipeline-as-code-controller-ingress
  namespace: pipelines-as-code
spec:
  rules:
  - host: $DASHBOARD_URL
    http:
      paths:
      - pathType: $DASHBOARD_PATH
        backend:
          service:
            name: pipelines-as-code-controller
            port:
                number: 8080
EOF
```

### Setup Gosmee as Webhook Relay

Now, you need to set up a webhook relay server because you are hosting your CI system in a local K8s cluster. Go to this endpoint [https://hook.pipelinesascode.com/](https://hook.pipelinesascode.com/) to create a public endpoint so your Github Webhook can send its events to. This endpoint will create a unique endpoint once you accessing it.

Then, you configure `gosmee` client to relay events from [https://hook.pipelinesascode.com/](https://hook.pipelinesascode.com/) to your local CI system

```sh
gosmee client https://hook.pipelinesascode.com/<unique-id> http://hook.saritasa.test.com
```

***Note***: After setting relay server, you need to open new terminal to continue running the setup because `gosmee` keeps running in the foreground in the current terminal.

### Initialize a Github App

Initialize a Github App for the Tekton's Pipeline as a recommended way in the official document. The Github App already sets up a webhook between Tekton and your repo. You may be asked some questions to set up the app.

```sh
tkn pac bootstrap

```

### Create a Repository CR

Register current repo as Repository CR managed by Tekton.

```sh
tkn pac create repository

# You will be asked
? Enter the Git repository URL :  <Current repo URL>
? Please enter the namespace where the pipeline should run (default: default): <Namespace for containing pipeline runs>
```

### Create a Kubernetes secret to access Docker Registry

Use this command to create a K8S secret to hold content of the Docker's `config.json` file. This secret is then used for a task to push image to Docker registry.

```sh
kubectl create secret generic docker-config --from-file=.dockerconfigjson="$HOME/.docker/config.json" --type=kubernetes.io/dockerconfigjson -n $(k get repository -A -o jsonpath='{.items[0].metadata.namespace}')
```

***Note***: Considering there's only 1 namespace containing all Respository CRs
