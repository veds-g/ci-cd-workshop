# ci-cd-workshop
anomaly detection using numaflow

## Installation

The following steps are to install the anomaly detection pipeline in your Kubernetes cluster and run it to show how it works.

### Prerequisites

- [Docker Engine](https://docs.docker.com/desktop/install/windows-install/)
- [Chocolatey](https://chocolatey.org/install)
```bash
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
```
- [`kubectl`](https://kubernetes.io/docs/tasks/tools/#kubectl)
```bash
choco install kubernetes-cli
```
- [`k3d`](https://k3d.io/v5.6.0/#install-script)
```bash
choco install k3d
```
- [`argocd`](https://argo-cd.readthedocs.io/en/stable/cli_installation/)
```bash
choco install argocd-cli
```
- [`helm`](https://helm.sh/docs/intro/install/)
```bash
choco install kubernetes-helm
```
- `git`
```bash
choco install git
```

### Installation Steps

1. Creating a local Kubernetes cluster using k3d

```bash
k3d cluster create ci-cd-workshop-cluster --api-port 6550 -p "8081:80@loadbalancer" --agents 2
```

2. Installing Prometheus Operator using helm

```bash
helm install --wait --timeout 15m --repo https://prometheus-community.github.io/helm-charts kube-prometheus-stack kube-prometheus-stack --values promConfig.txt
```

3. Create ArgoCD namespace and install ArgoCD to the cluster

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

4. Retrieve login credentials for argoCD

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"
```
use https://www.base64decode.org/ to decode the password

5. Setup ArgoCD UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8085:443
```

Open in the browser "https://localhost:8085/", login with username="admin" and password from last step.\
Update the password in "user-info" tab and re-login

6. Create an application using ArgoCD UI

In the 'Applications' Tab, Click on 'NEW APP'.\
Now Click on 'EDIT AS YAML' in top right and paste the following config

```bash
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: workshop-metrics-app
spec:
  destination:
    name: ''
    namespace: default
    server: 'https://kubernetes.default.svc'
  source:
    path: ./metrics-app/manifests/
    repoURL: 'https://github.com/veds-g/ci-cd-workshop'
    targetRevision: HEAD
  sources: []
  project: default
```

Click 'Save' and then click 'Create'.\
Now on the 'workshop-metrics-app' App in 'Applications' Tab, Click on 'SYNC'.\
Pods creation can be verified in terminal with

```bash
kubectl get pods
```

7. Emit metrics from the Flask application

```bash
kubectl port-forward svc/flask-service 5001
```

Open in the browser "http://localhost:5001/",
On hitting the `/url3` we should see message 'Hello World',
Try hitting the `/url1` and `/url2` routes to generate metrics for respective routes.

## Anomaly Detection Flow
![alt text](Flow-Light.png)

8. Install Kafka locally

```bash
kubectl apply -f https://raw.githubusercontent.com/veds-g/ci-cd-workshop/master/anomaly-pl/manifests/minimal-kafka.yaml
```

9. Deploying an application to write metrics from Prometheus to Kafka

```bash
kubectl apply -f https://raw.githubusercontent.com/veds-g/ci-cd-workshop/master/prom-kafka-writer/manifests/config.yaml
kubectl apply -f https://raw.githubusercontent.com/veds-g/ci-cd-workshop/master/prom-kafka-writer/manifests/deployment.yaml
kubectl apply -f https://raw.githubusercontent.com/veds-g/ci-cd-workshop/master/prom-kafka-writer/manifests/service.yaml
```

10. Install Numaflow

```bash
kubectl create ns numaflow-system
kubectl apply -n numaflow-system -f https://raw.githubusercontent.com/numaproj/numaflow/stable/config/install.yaml
kubectl apply -f https://raw.githubusercontent.com/numaproj/numaflow/stable/examples/0-isbsvc-jetstream.yaml
```

11. Create the anomaly detection pipeline using Numaflow

```bash
kubectl apply -f https://raw.githubusercontent.com/veds-g/ci-cd-workshop/master/anomaly-pl/manifests/pipeline.yaml
```

12. View the pipeline

```bash
kubectl port-forward svc/numaflow-server 8443 -n numaflow-system
```

Open the browser "https://localhost:8443/", then go to Numaflow UI, select `default` namespace, and click the `anomaly-detection-pl` pipeline.


## Argo Rollouts

### Installing the Argo Rollouts Plugin 
#### Windows
1. Navigate to https://github.com/argoproj/argo-rollouts/releases and download the executable corresponding to your operating system
2. Extract the file to a directory in your system and rename it to kubectl-argo-rollouts.exe (Windows)
3. Add the path of the directory to your PATH
4. Close all existing cmds and open a new one
5. Run kubectl-argo-rollouts version to validate your installation

#### MacOS/Linux
```
brew install argoproj/tap/kubectl-argo-rollouts
```

Alternatively, if this command doesn't work, you can run the following commands to install the kubectl argo rollouts plugin

```
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x ./kubectl-argo-rollouts-linux-amd64
sudo mv ./kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
```

PS: If you're using mac, replace linux with darwin when running the above commands

### Installing Argo Rollouts in your cluster

1. Create a new namespace named argo-rollouts and install the Argo Rollout CRDs (Custom Resource Defintions):
    ```bash
    kubectl create namespace argo-rollouts
    kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
    ```

### Creating the Rollout and its accompanying components

1. Create the Argo Rollout:
    ```bash
   kubectl apply -f https://raw.githubusercontent.com/veds-g/ci-cd-workshop/master/argo-rollouts/manifests/rollouts_demo.yaml -n argo-rollouts
    ```
   
2. Create the canary and stable services:
    ```bash
   kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-rollouts/master/docs/getting-started/nginx/services.yaml -n argo-rollouts
    ```

### Installing Ingress

1. Create a deployment and service for nginx
    ```bash
    kubectl create deployment nginx --image=nginx -n argo-rollouts
    kubectl create service clusterip nginx --tcp=80:80 -n argo-rollouts
    ```

2. Create the ingress resource
    ```
   kubectl apply -f https://raw.githubusercontent.com/veds-g/ci-cd-workshop/master/argo-rollouts/manifests/rollouts_ingress.yaml -n argo-rollouts
   ```

3. Open http://localhost:8081/ in your browser to access


### Rolling out 

1. Run the command to get the link to the argo-rollouts dashboard
    ```
   kubectl-argo-rollouts dashboard
   ```
   
2. Navigate to the rollout named rollouts-demo (as we created earlier). We can see all the steps that we had earlier defined.

3. Change the image name from argoproj/rollouts-demo:**blue** to argoproj/rollouts-demo:**green**

    ![](resources/rollouts-dashboard.png)
    We can see the steps getting executed in the left panel

    ![](resources/rollouts-application.png)
    We can also see the traffic moving from blue to green

    ![](resources/rollouts-cli-plugin.png)
    We can also see the status of the rollout in the terminal

4. Now click promote in the top right corner of the rollouts dashboard

5. We can now see the rest of the steps getting executed

## Commands for docker image build and push to dockerhub

1. Login to docker desktop using dockerhub account credentials.

2. To access dockerhub from terminal run

```bash
docker login -u <username>
```

3. To build docker image\
Navigate to the directory path which contains the Dockerfile in the repo and run

```bash
docker build -t <username>/<image_name>:<tag>
```

Example 'docker build -t trathi/metrics-app-image:1.0.0'

4. To publish image to dockerhub run

```bash
docker push <username>/<image_name>:<tag>
```
