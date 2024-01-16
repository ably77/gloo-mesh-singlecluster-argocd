# Add a cluster

## Purpose of this Tutorial
Continuing with the setup from previous example, lets explore how to onboard an additional workload cluster using Argo CD + GitOps setup.

## Prerequisites
This tutorial utilizes an additional Kubernetes cluster for demonstration. Instructions have been validated on k3d, as well as in EKS and GKE. Please note that the setup and installation of Kubernetes are beyond the scope of this guide. Ensure that your cluster contexts are named `cluster1` and `cluster1` by running:
```
% kubectl config get-contexts
CURRENT   NAME       CLUSTER        AUTHINFO             NAMESPACE
          cluster1   k3d-cluster1   admin@k3d-cluster1   
*         mgmt       k3d-mgmt       admin@k3d-mgmt       
```

#### Renaming Cluster Context
If your local clusters have a different context name, you will want to have it match the expected context name(s)
```
kubectl config rename-context <k3d-your_cluster_name> cluster1
```

### Note on Argo CD Setup
Argo CD supports several deployment strategies and architectures for configuring multi cluster setups, primarly
- Standalone - An Argo instance is installed and co-located with the cluster it’s managing. Each cluster has its own Argo instance.
- Hub and Spoke - A single Argo instance is used to connect and deploy to many Kubernetes instances
- Control Plane - Argo instances can be deployed in a mix of standalone and hub and spoke but a control-plane is added for managing and rolling information into a centralized place

Here are a following reference links to deep dive on the various Argo CD deployment architectures:

- [Akuity - How many do you need? - Argo CD Architectures Explained](https://akuity.io/blog/argo-cd-architectures-explained/)
- [CodeFresh - A Comprehensive Overview of Argo CD Architectures – 2023](https://codefresh.io/blog/a-comprehensive-overview-of-argo-cd-architectures-2023/)



This guide uses the standalone architecture, deploying a dedicated Argo CD instance on this new workload cluster with a per-cluster blast radius.

### Installing Argo CD	
Let's start by deploying Argo CD to our `cluster1` cluster context

Create Argo CD namespace
```
kubectl create namespace argocd --context cluster1
```

Deploy Argo CD 2.8.0 using the [non-HA YAML manifests](https://github.com/argoproj/argo-cd/releases)
```
until kubectl apply -k https://github.com/solo-io/gitops-library.git/argocd/deploy/default/ --context cluster1 > /dev/null 2>&1; do sleep 2; done
```

Check to see Argo CD status
```
kubectl get pods -n argocd --context cluster1
```

Output should look similar to below
```
% kubectl get pods -n argocd --context cluster1
NAME                                  READY   STATUS    RESTARTS   AGE
argocd-redis-74d8c6db65-lj5qz         1/1     Running   0          5m48s
argocd-dex-server-5896d988bb-ksk5j    1/1     Running   0          5m48s
argocd-application-controller-0       1/1     Running   0          5m48s
argocd-repo-server-6fd99dbbb5-xr8ld   1/1     Running   0          5m48s
argocd-server-7dd7894bd7-t92rr        1/1     Running   0          5m48s
```

We can also change the password to: `admin / solo.io`:
```
# bcrypt(password)=$2a$10$79yaoOg9dL5MO8pn8hGqtO4xQDejSEVNWAGQR268JHLdrCw6UCYmy
# password: solo.io
kubectl --context cluster1 -n argocd patch secret argocd-secret \
  -p '{"stringData": {
    "admin.password": "$2a$10$79yaoOg9dL5MO8pn8hGqtO4xQDejSEVNWAGQR268JHLdrCw6UCYmy",
    "admin.passwordMtime": "'$(date +%FT%T%Z)'"
  }}'
```

#### Navigating to Argo CD UI
At this point, we should be able to access our Argo CD server using port-forward at http://localhost:9999
```
kubectl port-forward svc/argocd-server -n argocd 9999:443 --context cluster1
```

## Deploy and Register Gloo Mesh Agent

First we will deploy the Gloo Platform CRD helm chart using an Argo Application
```
kubectl apply --context cluster1 -f- <<EOF
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
    targetRevision: 2.5.0
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

### Set Gloo Mesh Mgmt Variables
```
mgmt_context="gloo"

MGMT_SERVER_ADDRESS=$(kubectl --context ${mgmt_context} -n gloo-mesh get svc gloo-mesh-mgmt-server -o jsonpath='{.status.loadBalancer.ingress[0].*}')

METRICS_GATEWAY_ADDRESS=$(kubectl --context ${mgmt_context} -n gloo-mesh get svc gloo-telemetry-gateway -o jsonpath='{.status.loadBalancer.ingress[0].*}')
```

## a few manual steps (for now)

Gloo Mesh requires a `KubernetesCluster` object that references the cluster where you will be registering the agent. We will configure this manually for now, but typically this would be pushed to the management cluster repo (i.e. `gloo`)
```
kubectl apply --context gloo -f- <<EOF
apiVersion: admin.gloo.solo.io/v2
kind: KubernetesCluster
metadata:
  name: cluster1
  namespace: gloo-mesh
spec:
  clusterDomain: cluster.local
EOF
```

Copy the self-signed certs generated from the mgmt server to complete agent registration. In a production-ready environment these secrets can be generated by cert-manager, vault, or other secrets management solution however in this case our deployment went with the automatically generated self-signed certificates that Gloo Mesh provides by default.
```
kubectl get secret relay-root-tls-secret -n gloo-mesh --context gloo -o jsonpath='{.data.ca\.crt}' | base64 -d > ca.crt
kubectl create secret generic relay-root-tls-secret -n gloo-mesh --context cluster1 --from-file ca.crt=ca.crt
rm ca.crt

kubectl get secret relay-identity-token-secret -n gloo-mesh --context gloo -o jsonpath='{.data.token}' | base64 -d > token
kubectl create secret generic relay-identity-token-secret -n gloo-mesh --context cluster1 --from-file token=token
rm token
```

Now we can deploy our Gloo Mesh Agent
```
kubectl apply --context cluster1 -f- <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gloo-agent
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
        common:
            adminNamespace: "gloo-mesh"
            cluster: cluster1
        global: {}
        glooAgent:
            enabled: true
            relay:
                serverAddress: "${MGMT_SERVER_ADDRESS}:9900"
                authority: gloo-mesh-mgmt-server.gloo-mesh              
    repoURL: https://storage.googleapis.com/gloo-platform/helm-charts
    targetRevision: 2.5.0
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
```

You can check the cluster(s) have been registered correctly using the following commands:
```
meshctl --kubecontext gloo check
```

Output should look similar to below:
```
% meshctl --kubecontext gloo check

🟢 License status

 INFO  gloo enterprise license expiration is 29 Jun 24 10:05 PDT
 INFO  gloo-mesh enterprise license expiration is 29 Jun 24 10:05 PDT
 INFO  gloo-core enterprise license expiration is 29 Jun 24 10:05 PDT
 INFO  gloo-mesh-gateway enterprise license expiration is 29 Jun 24 10:05 PDT
 INFO  gloo-network enterprise license expiration is 29 Jun 24 10:05 PDT
 INFO  gloo-gateway enterprise license expiration is 29 Jun 24 10:05 PDT
 INFO  gloo-trial enterprise license expiration is 29 Jun 24 10:05 PDT
 INFO  Valid GraphQL license module found

🟢 CRD version check


🟢 Gloo Platform deployment status

Namespace | Name                           | Ready | Status 
gloo-mesh | gloo-mesh-redis                | 1/1   | Healthy
gloo-mesh | gloo-telemetry-gateway         | 1/1   | Healthy
gloo-mesh | gloo-mesh-mgmt-server          | 1/1   | Healthy
gloo-mesh | gloo-mesh-agent                | 1/1   | Healthy
gloo-mesh | gloo-mesh-ui                   | 1/1   | Healthy
gloo-mesh | prometheus-server              | 1/1   | Healthy
gloo-mesh | svclb-gloo-telemetry-gateway   | 1/1   | Healthy
gloo-mesh | gloo-telemetry-collector-agent | 1/1   | Healthy
gloo-mesh | svclb-gloo-mesh-mgmt-server    | 1/1   | Healthy

🟢 Mgmt server connectivity to workload agents

Cluster  | Registered | Connected Pod                                   
cluster1 | true       | gloo-mesh/gloo-mesh-mgmt-server-5d68667ff8-rg9sr
gloo     | true       | gloo-mesh/gloo-mesh-mgmt-server-5d68667ff8-rg9sr

Connected Pod                                    | Clusters
gloo-mesh/gloo-mesh-mgmt-server-5d68667ff8-rg9sr | 2    
```

### Visualize in Gloo Mesh Dashboard
Access Gloo Mesh Dashboard at `http://localhost:8090`:
```
kubectl port-forward -n gloo-mesh svc/gloo-mesh-ui 8090 --context gloo
```

You should see that an additional workload cluster named `cluster1` has been registered with Gloo Mesh

## Installing Istio
Here we will use Argo CD to demonstrate how to deploy and manage Istio using helm.

First deploy the Istio base 1.19.3 helm chart to `cluster1`
```
kubectl apply --context cluster1 -f- <<EOF
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
    targetRevision: 1.19.3
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
EOF
```

Now, lets deploy the Istio control plane:
```
kubectl apply --context cluster1 -f- <<EOF
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
    targetRevision: 1.19.3
    helm:
      values: |
        revision: 1-19
        global:
          meshID: mesh1
          multiCluster:
            clusterName: cluster1
          network: network1
          hub: us-docker.pkg.dev/gloo-mesh/istio-workshops
          tag: 1.19.3-solo
        meshConfig:
          trustDomain: cluster1
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
```
kubectl apply --context cluster1 -f- <<EOF
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
    targetRevision: 1.19.3
    helm:
      values: |
        # Name allows overriding the release name. Generally this should not be set
        name: "istio-ingressgateway-1-19"
        # revision declares which revision this gateway is a part of
        revision: "1-19"
        
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
            service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
          loadBalancerIP: ""
          loadBalancerSourceRanges: []
          externalTrafficPolicy: ""
        
        # Pod environment variables
        env: {}

        annotations:
          proxy.istio.io/config: '{ "holdApplicationUntilProxyStarts": true }'
        
        # Labels to apply to all resources
        labels:
          istio.io/rev: 1-19
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
```
kubectl get pods -n istio-system --context cluster1 && kubectl get pods -n istio-gateways --context cluster1
```

Output should look similar to below:
```
% kubectl get pods -n istio-system --context cluster1 && kubectl get pods -n istio-gateways --context cluster1
NAME                          READY   STATUS    RESTARTS   AGE
istiod-1-19-b6ff5fbf7-92bx9   1/1     Running   0          60s
NAME                                         READY   STATUS    RESTARTS   AGE
istio-ingressgateway-1-19-844fc985cd-hfbm5   1/1     Running   0          45s
```

Now you should see that the new workload cluster has been registered with Istio 1.19.3 deployed