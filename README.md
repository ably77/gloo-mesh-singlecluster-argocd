# Deploy a Single Cluster Gloo Mesh with Argo CD

## Introduction
GitOps is becoming increasingly popular approach to manage Kubernetes components. It works by using Git as a single source of truth for declarative infrastructure and applications, allowing your application definitions, configurations, and environments to be declarative and version controlled. This helps to make these workflows automated, auditable, and easy to understand.

## Purpose of this Tutorial
The main goal of this tutorial is to showcase how Gloo Mesh components can seamlessly integrate into a GitOps workflow, with Argo CD being our tool of choice. We'll guide you through the installation of Argo CD, Gloo Platform, Istio, and finally, we'll explore the Gloo Mesh Dashboard.

## High Level Architecture
![High Level Architecture](.images/single-cluster-arch1.png)

## Prerequisites
This tutorial assumes a single Kubernetes cluster for demonstration. Instructions have been validated on k3d, as well as in EKS and GKE. Please note that the setup and installation of Kubernetes are beyond the scope of this guide. Ensure that your cluster contexts are named `gloo` by running:
```bash
kubectl config get-contexts
```
```
CURRENT   NAME   CLUSTER    AUTHINFO         NAMESPACE
*         gloo   k3d-gloo   admin@k3d-gloo   
```

#### Renaming Cluster Context
If your local clusters have a different context name, you will want to have it match the expected context name(s). In this example, we are setting the context name as `gloo`.

```bash
export MY_CLUSTER_CONTEXT=gloo
export MY_CLUSTER_NAME=k3d-gloo
```

```bash
kubectl config rename-context <k3d-your_cluster_name> "${MY_CLUSTER_CONTEXT}"
```

### Installing Argo CD	
Let's start by deploying Argo CD to our `gloo` cluster context

Create Argo CD namespace
```bash
kubectl create namespace argocd --context "${MY_CLUSTER_CONTEXT}"
```

Deploy Argo CD 2.8.0 using the [non-HA YAML manifests](https://github.com/argoproj/argo-cd/releases)
<!-- Using https://github.com/solo-io/gitops-library/tree/main/argocd/deploy/default -->
```bash
until kubectl apply -k https://github.com/solo-io/gitops-library.git/argocd/deploy/default/ --context "${MY_CLUSTER_CONTEXT}" > /dev/null 2>&1; do sleep 2; done
```

Check deployment status:
```bash
kubectl --context ${MY_CLUSTER_CONTEXT} -n argocd rollout status deploy/argocd-applicationset-controller
kubectl --context ${MY_CLUSTER_CONTEXT} -n argocd rollout status deploy/argocd-dex-server
kubectl --context ${MY_CLUSTER_CONTEXT} -n argocd rollout status deploy/argocd-notifications-controller
kubectl --context ${MY_CLUSTER_CONTEXT} -n argocd rollout status deploy/argocd-redis
kubectl --context ${MY_CLUSTER_CONTEXT} -n argocd rollout status deploy/argocd-repo-server
kubectl --context ${MY_CLUSTER_CONTEXT} -n argocd rollout status deploy/argocd-server
```

Check to see Argo CD status.
```bash
kubectl get pods -n argocd --context "${MY_CLUSTER_CONTEXT}"
```

Output should look similar to below
```bash
NAME                                                READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                     1/1     Running   0          31s
argocd-applicationset-controller-765944f45d-569kn   1/1     Running   0          33s
argocd-dex-server-7977459848-swnm6                  1/1     Running   0          33s
argocd-notifications-controller-6587c9d9-6t4hg      1/1     Running   0          32s
argocd-redis-b5d6bf5f5-52mk5                        1/1     Running   0          32s
argocd-repo-server-7bfc968f69-hrqt6                 1/1     Running   0          32s
argocd-server-77f84bfbb8-lgdlr                      2/2     Running   0          32s
```

We can also change the password to: `admin / solo.io`:
```bash
# bcrypt(password)=$2a$10$79yaoOg9dL5MO8pn8hGqtO4xQDejSEVNWAGQR268JHLdrCw6UCYmy
# password: solo.io
kubectl --context "${MY_CLUSTER_CONTEXT}" -n argocd patch secret argocd-secret \
  -p '{"stringData": {
    "admin.password": "$2a$10$79yaoOg9dL5MO8pn8hGqtO4xQDejSEVNWAGQR268JHLdrCw6UCYmy",
    "admin.passwordMtime": "'$(date +%FT%T%Z)'"
  }}'
```

#### Navigating to Argo CD UI
At this point, we should be able to access our Argo CD server using port-forward at http://localhost:9999
```
kubectl port-forward svc/argocd-server -n argocd 9999:443 --context "${MY_CLUSTER_CONTEXT}"
```

## Provide Gloo Mesh Enterprise License Key variable
Gloo Mesh Enterprise requires a Trial License Key:
```bash
GLOO_MESH_LICENSE_KEY=<input_license_key_here>
```

## Installing Gloo Mesh
Gloo Mesh can be installed and configured easily using Helm + Argo CD. To install Gloo Mesh Enterprise

First we will deploy the Gloo Platform CRD helm chart using an Argo Application
```bash
export GLOO_MESH_VERSION=2.4.7;

kubectl apply --context "${MY_CLUSTER_CONTEXT}" -f- <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gloo-platform-crds
  namespace: argocd
spec:
  destination:
    namespace: gloo-mesh
    server: https://kubernetes.default.svc
  project: default
  source:
    chart: gloo-platform-crds
    repoURL: https://storage.googleapis.com/gloo-platform/helm-charts
    targetRevision: "${GLOO_MESH_VERSION}"
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true 
    retry:
      limit: 2
      backoff:
        duration: 5s
        maxDuration: 3m0s
        factor: 2
EOF
```

Then deploy the Gloo Platform helm chart
```bash
kubectl apply --context "${MY_CLUSTER_CONTEXT}" -f- <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gloo-platform-helm
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  destination:
    server: https://kubernetes.default.svc
    namespace: gloo-mesh
  project: default
  source:
    chart: gloo-platform
    helm:
      skipCrds: true
      values: |
        licensing:
          licenseKey: ${GLOO_MESH_LICENSE_KEY}
        common:
          cluster: "${MY_CLUSTER_NAME}"
        glooMgmtServer:
          enabled: true
          serviceType: ClusterIP
          registerCluster: true
          createGlobalWorkspace: true
          ports:
            healthcheck: 8091
        prometheus:
          enabled: true
        redis:
          deployment:
            enabled: true
        telemetryGateway:
          enabled: true
          service:
            type: ClusterIP
        telemetryCollector:
          enabled: true
          config:
            exporters:
              otlp:
                endpoint: gloo-telemetry-gateway.gloo-mesh:4317
        glooUi:
          enabled: true
          serviceType: ClusterIP
        glooAgent:
          enabled: true
          relay:
            serverAddress: gloo-mesh-mgmt-server:9900
    repoURL: https://storage.googleapis.com/gloo-platform/helm-charts
    targetRevision: "${GLOO_MESH_VERSION}"
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
  # ignore the self-signed certs that are being generated automatically    
  ignoreDifferences:
  - group: v1
    kind: Secret    
EOF
```

You can check to see that the Gloo Mesh Management Plane is deployed
```bash
kubectl get pods -n gloo-mesh  --context "${MY_CLUSTER_CONTEXT}"
```

Output should look similar to below:
```bash
NAME                                      READY   STATUS    RESTARTS   AGE
gloo-mesh-agent-7b79dd5c9-hfw9p           1/1     Running   0          62s
gloo-mesh-mgmt-server-7f6968d9b9-67l7j    1/1     Running   0          62s
gloo-mesh-redis-84f57d46bc-dwqx9          1/1     Running   0          62s
gloo-mesh-ui-56879cb8b-52vpx              3/3     Running   0          61s
gloo-telemetry-collector-agent-7fgb5      1/1     Running   0          62s
gloo-telemetry-collector-agent-txwbm      1/1     Running   0          62s
gloo-telemetry-gateway-5445d7d6b5-5k9sd   1/1     Running   0          62s
prometheus-server-565cb79f89-jc5ns        2/2     Running   0          62s
```

### install `meshctl`
```bash
curl -sL https://run.solo.io/meshctl/install | GLOO_MESH_VERSION="v${GLOO_MESH_VERSION}" sh - ;
export PATH=$HOME/.gloo-mesh/bin:$PATH
```

You can check status using the `meshctl` CLI
```bash
meshctl check --kubecontext "${MY_CLUSTER_CONTEXT}"
```

Output should look similar to below:
```bash
🟢 License status

 INFO  gloo-mesh enterprise license expiration is 04 Jun 24 16:41 EDT
 INFO  No GraphQL license module found for any product

🟢 CRD version check


🟢 Gloo Platform deployment status

Namespace | Name                           | Ready | Status
gloo-mesh | gloo-mesh-agent                | 1/1   | Healthy
gloo-mesh | gloo-mesh-mgmt-server          | 1/1   | Healthy
gloo-mesh | gloo-mesh-redis                | 1/1   | Healthy
gloo-mesh | gloo-mesh-ui                   | 1/1   | Healthy
gloo-mesh | gloo-telemetry-gateway         | 1/1   | Healthy
gloo-mesh | prometheus-server              | 1/1   | Healthy
gloo-mesh | gloo-telemetry-collector-agent | 2/2   | Healthy

🟢 Mgmt server connectivity to workload agents

Cluster                         | Registered | Connected Pod
demo-gloo-platform-cluster-arka | true       | gloo-mesh/gloo-mesh-mgmt-server-7f6968d9b9-67l7j

Connected Pod                                    | Clusters
gloo-mesh/gloo-mesh-mgmt-server-7f6968d9b9-67l7j | 1
```

At this point, we should be able to access our Gloo Platform UI using port-forward at http://localhost:8090
```bash
kubectl port-forward svc/gloo-mesh-ui -n gloo-mesh 8090:8090 --context "${MY_CLUSTER_CONTEXT}"
```

## Installing Istio
Here we will use Argo CD to demonstrate how to deploy and manage Istio using helm.

First, deploy the `istio-base` helm chart.

```bash
export ISTIO_VERSION=1.19.6
export ISTIO_REVISION=1-19-6
```

```bash
kubectl apply --context "${MY_CLUSTER_CONTEXT}" -f- <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: istio-base
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
  annotations:
    argocd.argoproj.io/sync-wave: "-3"
spec:
  destination:
    server: https://kubernetes.default.svc
    namespace: istio-system
  project: default
  source:
    chart: base
    repoURL: https://istio-release.storage.googleapis.com/charts
    targetRevision: "${ISTIO_VERSION}"
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
EOF
```

Now, lets deploy the Istio control plane:

Get the Hub value from the [Solo support page for Istio Solo images](https://support.solo.io/hc/en-us/articles/4414409064596). The value is present within the `Solo.io Istio Versioning Repo key` section
```bash
export HUB=
```

```bash
kubectl apply --context "${MY_CLUSTER_CONTEXT}" -f- <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: istiod
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  destination:
    server: https://kubernetes.default.svc
    namespace: istio-system
  project: default
  source:
    chart: istiod
    repoURL: https://istio-release.storage.googleapis.com/charts
    targetRevision: "${ISTIO_VERSION}"
    helm:
      values: |
        revision: "${ISTIO_REVISION}"
        global:
          meshID: mesh1
          multiCluster:
            clusterName: gloo
          network: network1
          hub: ${HUB}
          tag: "${ISTIO_VERSION}-solo"
        meshConfig:
          trustDomain: "${MY_CLUSTER_NAME}.local"
          accessLogFile: /dev/stdout
          accessLogEncoding: JSON
          enableAutoMtls: true
          defaultConfig:
            # Wait for the istio-proxy to start before starting application pods
            holdApplicationUntilProxyStarts: true
            envoyAccessLogService:
              address: gloo-mesh-agent.gloo-mesh:9977
            proxyMetadata:
              ISTIO_META_DNS_CAPTURE: "true"
              ISTIO_META_DNS_AUTO_ALLOCATE: "true"
          outboundTrafficPolicy:
            mode: ALLOW_ANY
          rootNamespace: istio-system
        pilot:
          env:
            PILOT_ENABLE_K8S_SELECT_WORKLOAD_ENTRIES: "false"
            PILOT_SKIP_VALIDATE_TRUST_DOMAIN: "true"
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    #automated: {}
  ignoreDifferences:
  - group: '*'
    kind: '*'
    managedFieldsManagers:
    - argocd-application-controller
EOF
```

Now we can deploy the Istio Ingressgateway
```bash
kubectl apply --context "${MY_CLUSTER_CONTEXT}" -f- <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: istio-ingressgateway
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
spec:
  destination:
    server: https://kubernetes.default.svc
    namespace: istio-gateways
  project: default
  source:
    chart: gateway
    repoURL: https://istio-release.storage.googleapis.com/charts
    targetRevision: "${ISTIO_VERSION}"
    helm:
      values: |
        # Name allows overriding the release name. Generally this should not be set
        name: "istio-ingressgateway-${ISTIO_REVISION}"
        # revision declares which revision this gateway is a part of
        revision: "${ISTIO_REVISION}"
        
        replicaCount: 1
        
        service:
          # Type of service. Set to "None" to disable the service entirely
          type: LoadBalancer
          ports:
          - name: http2
            port: 80
            protocol: TCP
            targetPort: 80
          - name: https
            port: 443
            protocol: TCP
            targetPort: 443
          annotations:
            # AWS NLB Annotation
            service.beta.kubernetes.io/aws-load-balancer-type: "nlb-ip"
          loadBalancerIP: ""
          loadBalancerSourceRanges: []
          externalTrafficPolicy: ""
        
        # Pod environment variables
        env: {}

        annotations:
          proxy.istio.io/config: '{ "holdApplicationUntilProxyStarts": true }'
        
        # Labels to apply to all resources
        labels:
          istio.io/rev: ${ISTIO_REVISION}
          istio: ingressgateway
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
EOF
```

You can check to see that istiod and the istio ingressgateways have been deployed
```bash
kubectl get pods -n istio-system --context "${MY_CLUSTER_CONTEXT}" && \
kubectl get pods -n istio-gateways --context "${MY_CLUSTER_CONTEXT}"
```

Output should look similar to below:
```bash
NAME                             READY   STATUS    RESTARTS   AGE
istiod-1-19-6-86499c5945-bbsfl   1/1     Running   0          38m
NAME                                           READY   STATUS    RESTARTS   AGE
istio-ingressgateway-1-19-6-6575484979-5fbn7   1/1     Running   0          36m
```

### Visualize in Gloo Mesh Dashboard
Access Gloo Mesh Dashboard at `http://localhost:8090`:
```bash
kubectl port-forward -n gloo-mesh svc/gloo-mesh-ui 8090 --context "${MY_CLUSTER_CONTEXT}"
```

At this point, you should have ArgoCD, Gloo Mesh, and Istio installed on the cluster!
![Gloo Mesh UI](.images/gmui2.png)

## Lets have some fun
To streamline deployment, utilize the Argo app-of-apps pattern to manage and orchestrate the entire setup in one `Application`!

To avoid storing the license key in Git, we can enhance security by utilizing a licenseSecretKeyRef in the Helm values, necessitating the creation of the license key as a secret.

## Provide Gloo Mesh Enterprise License Key variable
In case you havent configured it already, Gloo Mesh Enterprise requires a Trial License Key:
```bash
GLOO_MESH_LICENSE_KEY=<input_license_key_here>
```

The license is stored as a base64 encoded secret, set your the following variable based on your OS:

For Linux:
```bash
# Linux
    BASE64_LICENSE_KEY=$(echo -n "${GLOO_MESH_LICENSE_KEY}" | base64 -w 0)
```

For Mac:
```bash
# Mac OSX
    BASE64_LICENSE_KEY=$(echo -n "${GLOO_MESH_LICENSE_KEY}" | base64 | tr -d '[:space:]')
```

Create the license secret:
```bash
# Create namespace for Gloo Mesh
kubectl create ns gloo-mesh --context "${MY_CLUSTER_CONTEXT}"

# Apply license keys as a Kubernetes secret
kubectl apply --context "${MY_CLUSTER_CONTEXT}" -f - <<EOF
apiVersion: v1
data:
  gloo-gateway-license-key: ${BASE64_LICENSE_KEY}
  gloo-mesh-license-key: ${BASE64_LICENSE_KEY}
  gloo-network-license-key: ${BASE64_LICENSE_KEY}
  gloo-trial-license-key: ${BASE64_LICENSE_KEY}
kind: Secret
metadata:
  name: gloo-license
  namespace: gloo-mesh
type: Opaque
EOF
```

Now we can deploy an Argo app-of-app which will configure the entire setup that we walked through in the previous section
```bash
kubectl apply --context "${MY_CLUSTER_CONTEXT}" -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: easybutton
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/ably77/gloo-mesh-singlecluster-argocd/
    path: easybutton/app-of-app
    targetRevision: HEAD
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m0s
EOF
```

You can check to see that Gloo Mesh, Istiod and the Istio ingressgateways have been deployed
```bash
kubectl get pods -n gloo-mesh --context "${MY_CLUSTER_CONTEXT}" && \
kubectl get pods -n istio-system --context "${MY_CLUSTER_CONTEXT}" && \
kubectl get pods -n istio-gateways --context "${MY_CLUSTER_CONTEXT}"
```

### Visualize in Gloo Mesh Dashboard
Access Gloo Mesh Dashboard at `http://localhost:8090`:
```bash
kubectl port-forward -n gloo-mesh svc/gloo-mesh-ui 8090 --context "${MY_CLUSTER_CONTEXT}"
```
