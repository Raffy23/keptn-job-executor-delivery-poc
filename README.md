# Job Executor Continuous Delivery PoC

**Goal**: Create a simple but powerful Proof of Concept using Keptn Control-Plane and Job-Executor-Service to

* Deploy a service (using `helm upgrade`)
* Run performance tests (using `locust`) against this service


**Please note**: This is not a comprehensive tutorial, it is **not** covering every case regarding Keptn installation, Kubernetes, prometheus, nor job-executor-service.


## Pre Requisite

**You need access to a Kubernetes cluster with at least 4 vCPUs and 8 GB memory**

I recommend using Google Kubernetes Engine, but local setups with K3s/K3d should also work fine.


**Install Keptn 0.13.4 control-plane only**

```bash
curl -sL https://get.keptn.sh | KEPTN_VERSION=0.13.4 bash
keptn install --endpoint-service-type=LoadBalancer
```

**Install job-executor-service (remote-control-plane)**

Minimum version: a66fbe89b97a45873435d5b3cbbf97c9ee25e55e

```bash
KEPTN_API_PROTOCOL=http
KEPTN_API_HOST=<INSERT-YOUR-HOSTNAME-HERE> # e.g., 1.2.3.4.nip.io
 KEPTN_API_TOKEN=<INSERT-YOUR-KEPTN-API-TOKEN-HERE>

TASK_SUBSCRIPTION=sh.keptn.event.je-deployment.triggered,sh.keptn.event.je-test.triggered

helm upgrade --install --create-namespace -n <NAMESPACE> \
  job-executor-service https://github.com/keptn-contrib/job-executor-service/releases/download/<VERSION>/job-executor-service-<VERSION>.tgz \
 --set remoteControlPlane.topicSubscription=${TASK_SUBSCRIPTION},remoteControlPlane.api.protocol=${KEPTN_API_PROTOCOL},remoteControlPlane.api.hostname=${KEPTN_API_HOST},remoteControlPlane.api.token=${KEPTN_API_TOKEN} --wait
```

(*Note*: Replace `<VERSION>` with the version you want to install; If the version is a commit hash, please clone the repository, checkout the specified commit and build it from source)

**Install Prometheus Monitoring**

```bash
kubectl create namespace monitoring
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/prometheus --namespace monitoring --wait
```


**Install Keptns prometheus-service**
```bash
helm install -n keptn prometheus-service https://github.com/keptn-contrib/prometheus-service/releases/download/0.7.2/prometheus-service-0.7.2.tgz --wait
kubectl apply -f https://raw.githubusercontent.com/keptn-contrib/prometheus-service/release-0.7.2/deploy/role.yaml -n monitoring
```

## Project Setup

**Download or clone this repo**

```bash
git clone https://github.com/christian-kreuzberger-dtx/keptn-job-executor-delivery-poc.git
cd keptn-job-executor-delivery-poc
```

**Create Project and Service**

See [shipyard.yaml](shipyard.yaml) for details

```bash
PROJECT=podtato-head
keptn create project $PROJECT --shipyard=./shipyard.yaml
keptn create service helloservice --project=$PROJECT
```

**Add helm charts**

```bash
keptn add-resource --project=$PROJECT --service=helloservice --all-stages --resource=./helm/helloservice.tgz --resourceUri=charts/helloservice.tgz
```

**Add locust test files**

```bash
keptn add-resource --project=$PROJECT --service=helloservice --stage=qa --resource=./locust/basic.py
keptn add-resource --project=$PROJECT --service=helloservice --stage=qa --resource=./locust/locust.conf
```

**Add job-executor config**

```bash
keptn add-resource --project=$PROJECT --service=helloservice --all-stages --resource=job-executor-config.yaml --resourceUri=job/config.yaml
```

**Add SLO/SLI config files**

```bash
keptn add-resource --project=$PROJECT --service=helloservice --stage=qa --resource=prometheus/sli.yaml --resourceUri=prometheus/sli.yaml
keptn add-resource --project=$PROJECT --service=helloservice --stage=qa --resource=slo.yaml --resourceUri=slo.yaml
```

**Configure Prometheus Monitoring**

```bash
keptn configure monitoring prometheus --project=$PROJECT --service=helloservice
```

**Create namespaces**

Create the necessary namespaces, or modify the [job configuration](/job-executor-config.yaml) that the helm command contains `--create-namespaces`.
But be aware that this does only work if the service account has cluster wide access for namespaces. 
```bash
kubectl create namespace podtato-head-production
kubectl create namespace podtato-head-qa
```

**Configure the serviceAccount for helm**

By default the job-executor-service does not grant access to the kubernetes api and therefore we have to define a service account for helm:

```bash
kubectl apply -f job-executor/helm-serviceAccount.yaml
kubectl apply -f job-executor/helm-clusterRole.yaml
```

The permissions for the service account can be configured in two different ways:
* *ClusterRole*: Allows the helm service to make modifications to the whole cluster and should not be used for production setups. You may also want to modify the [job configuration](/job-executor-config.yaml) to include the `--create-namespace` flag if you haven't create the namespaces.
* *Role*: The appropiate namespaces with the role bindings have to be created before the job-executor-service can execute the jobs, otherwise helm is not able to install the services correctly. This approach also prevents the `--create-namespace` helm parameter from working correctly.

For this PoC we use the *Role* binding approach, but be aware that it may not work for other services:

```bash
kubectl apply -f job-executor/helm-roleBinding.yaml
```


**Trigger delivery**

```bash
IMAGE="ghcr.io/podtato-head/podtatoserver"
VERSION=v0.1.1
SLOW_VERSION=v0.1.2

keptn trigger delivery --project=$PROJECT --service=helloservice --image=$IMAGE --tag=$VERSION --labels=version=$VERSION
```


**Trigger delivery of slow version**

```bash
keptn trigger delivery --project=$PROJECT --service=helloservice --image=$IMAGE --tag=$SLOW_VERSION --labels=version=$SLOW_VERSION,slow=true
```

## Cleanup

Once you're done, I recommend cleaning up:

**Delete project and resulting Kubernetes namespaces**

```bash
keptn delete project $PROJECT
kubectl delete namespace $PROJECT-qa $PROJECT-production
```

**Uninstall prometheus-service**
```bash
helm uninstall -n keptn prometheus-service
```

**Uninstall prometheus**
```bash
helm uninstall prometheus --namespace monitoring
```

**Remove role and role bindings**

```bash
kubectl delete -f job-executor/helm-clusterRoleBinding.yaml
kubectl delete -f job-executor/helm-roleBinding.yaml
kubectl delete -f job-executor/helm-clusterRole.yaml
```

**Remove service account**
```bash
kubectl delete -f job-executor/helm-serviceAccount.yaml
```

**Uninstall job-executor-service**

```bash
helm uninstall -n keptn job-executor-service
```
