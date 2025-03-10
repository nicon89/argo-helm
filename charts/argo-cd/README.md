# Argo CD Chart

A Helm chart for ArgoCD, a declarative, GitOps continuous delivery tool for Kubernetes.

Source code can be found [here](https://argoproj.github.io/argo-cd/)

## Additional Information

This is a **community maintained** chart. This chart installs [argo-cd](https://argoproj.github.io/argo-cd/), a declarative, GitOps continuous delivery tool for Kubernetes.

The default installation is intended to be similar to the provided ArgoCD [releases](https://github.com/argoproj/argo-cd/releases).

This chart currently installs the non-HA version of ArgoCD.

### Synchronizing Changes from Original Repository

In the original [ArgoCD repository](https://github.com/argoproj/argo-cd/) an [`manifests/install.yaml`](https://github.com/argoproj/argo-cd/blob/master/manifests/install.yaml) is generated using `kustomize`. It's the basis for the installation as [described in the docs](https://argo-cd.readthedocs.io/en/stable/getting_started/#1-install-argo-cd).

When installing ArgoCD using this helm chart the user should have a similar experience and configuration rolled out. Hence, it makes sense to try to achieve a similar output of rendered `.yaml` resources when calling `helm template` using the default settings in `values.yaml`.

To update the templates and default settings in `values.yaml` it may come in handy to look up the diff of the `manifests/install.yaml` between two versions accordingly. This can either be done directly via github and look for `manifests/install.yaml`:

https://github.com/argoproj/argo-cd/compare/v1.8.7...v2.0.0#files_bucket

Or you clone the repository and do a local `git-diff`:

```bash
git clone https://github.com/argoproj/argo-cd.git
cd argo-cd
git diff v1.8.7 v2.0.0 -- manifests/install.yaml
```

Changes in the `CustomResourceDefinition` resources shall be fixed easily by copying 1:1 from the [`manifests/crds` folder](https://github.com/argoproj/argo-cd/tree/master/manifests/crds) into this [`charts/argo-cd/crds` folder](https://github.com/argoproj/argo-helm/tree/master/charts/argo-cd/crds).

## Upgrading

### 3.13.0

This release removes the flag `--staticassets` from argocd server as it has been dropped upstream. If this flag needs to be enabled e.g for older releases of ArgoCD, it can be passed via the `server.extraArgs` field 

### 3.10.2

ArgoCD has recently deprecated the flag `--staticassets` and from chart version `3.10.2` has been disabled by default
It can be re-enabled by setting `server.staticAssets.enabled` to true

### 3.8.1

This bugfix version potentially introduces a rename (and recreation) of one or more ServiceAccounts. It _only happens_ when you use one of these customization:

```yaml
# Case 1) - only happens when you do not specify a custom name (repoServer.serviceAccount.name)
repoServer:
  serviceAccount:
    create: true

# Case 2)
controller:
  serviceAccount:
    name: "" # or <nil>

# Case 3)
dex:
  serviceAccount:
    name: "" # or <nil>

# Case 4)
server:
  serviceAccount:
    name: "" # or <nil>
```

Please check if you are affected by one of these cases **before you upgrade**, especially when you use **cloud IAM roles for service accounts.** (eg. IRSA on AWS or Workload Identity for GKE)

### 3.2.* 

With this minor version we introduced the evaluation for the ingress manifest (depending on the capabilities version), See [Pull Request](https://github.com/argoproj/argo-helm/pull/637).
[Issue 703](https://github.com/argoproj/argo-helm/issues/703) reported that the capabilities evaluation is **not handled correctly when deploying the chart via an ArgoCD instance**,
especially deploying on clusters running a cluster version prior to `1.19` (which misses  `Ingress` on apiVersion `networking.k8s.io/v1`).

If you are running a cluster version prior to `1.19` you can avoid this issue by directly installing chart version `3.6.0` and setting `kubeVersionOverride` like:

```yaml
kubeVersionOverride: "1.18.0"
``` 

Then you should no longer encounter this issue.


### 3.0.0 and above

Helm apiVersion switched to `v2`. Requires Helm `3.0.0` or above to install. [Read More](https://helm.sh/blog/migrate-from-helm-v2-to-helm-v3/) on how to migrate your release from Helm 2 to Helm 3.

### 2.14.7 and above

The `matchLabels` key in the ArgoCD Application Controller is no longer hard-coded. Note that labels are immutable so caution should be exercised when making changes to this resource.

### 2.10.x to 2.11.0

The application controller is now available as a `StatefulSet` when the `controller.enableStatefulSet` flag is set to true. Depending on your Helm deployment this may be a downtime or breaking change if enabled when using HA and will become the default in 3.x.

### 1.8.7 to 2.x.x

`controller.extraArgs`, `repoServer.extraArgs` and `server.extraArgs`  are now arrays of strings instead of a map

What was
```yaml
server:
  extraArgs:
    insecure: ""
```

is now

```yaml
server:
  extraArgs:
  - --insecure
```

## Prerequisites

- Kubernetes 1.7+
- Helm v3.0.0+

## Installing the Chart

To install the chart with the release name `my-release`:

```console
$ helm repo add argo https://argoproj.github.io/argo-helm
"argo" has been added to your repositories

$ helm install --name my-release argo/argo-cd
NAME: my-release
...
```

## Chart Values

| Parameter | Description | Default |
|-----|------|---------|
| global.image.imagePullPolicy | If defined, a imagePullPolicy applied to all ArgoCD deployments. | `"IfNotPresent"` |
| global.image.repository | If defined, a repository applied to all ArgoCD deployments. | `"argoproj/argocd"` |
| global.image.tag | If defined, a tag applied to all ArgoCD deployments. | `"v2.0.5"` |
| global.securityContext | Toggle and define securityContext | See [values.yaml](values.yaml) |
| global.imagePullSecrets | If defined, uses a Secret to pull an image from a private Docker registry or repository. | `[]` |
| global.hostAliases | Mapping between IP and hostnames that will be injected as entries in the pod's hosts files | `[]` |
| global.networkPolicy.create | Create NetworkPolicy objects for all components | `false` |
| global.networkPolicy.defaultDenyIngress | Default deny all ingress traffic | `false` |
| kubeVersionOverride | Override the Kubernetes version, which is used to evaluate certain manifests | `""` |
| nameOverride | Provide a name in place of `argocd` | `"argocd"` |
| fullnameOverride | String to fully override `"argo-cd.fullname"` | `""` |
| apiVersionOverrides.certmanager | String to override apiVersion of certmanager resources rendered by this helm chart | `""` |
| apiVersionOverrides.ingress | String to override apiVersion of ingresses rendered by this helm chart | `""` |
| createAggregateRoles | Create clusterroles that extend aggregated roles to use argo-cd crds | `false` |
| configs.clusterCredentials | Provide one or multiple [external cluster credentials](https://argoproj.github.io/argo-cd/operator-manual/declarative-setup/#clusters) | `[]` (See [values.yaml](values.yaml)) |
| configs.gpgKeysAnnotations | GnuPG key ring annotations | `{}` |
| configs.gpgKeys | [GnuPG](https://argoproj.github.io/argo-cd/user-guide/gpg-verification/) keys to add to the key ring | `{}` (See [values.yaml](values.yaml)) |
| configs.knownHostsAnnotations | Known Hosts configmap annotations | `{}` |
| configs.knownHosts.data.ssh_known_hosts | Known Hosts | See [values.yaml](values.yaml) |
| configs.secret.annotations | Annotations for argocd-secret | `{}` |
| configs.secret.argocdServerAdminPassword | Bcrypt hashed admin password | `null` |
| configs.secret.argocdServerAdminPasswordMtime | Admin password modification time | `date "2006-01-02T15:04:05Z" now` if configs.secret.argocdServerAdminPassword is set |
| configs.secret.bitbucketSecret | BitBucket incoming webhook secret | `""` |
| configs.secret.createSecret | Create the argocd-secret. | `true` |
| configs.secret.githubSecret | GitHub incoming webhook secret | `""` |
| configs.secret.gitlabSecret | GitLab incoming webhook secret | `""` |
| configs.tlsCertsAnnotations | TLS certificate configmap annotations | `{}` |
| configs.tlsCerts.data."argocd.example.com" | TLS certificate | See [values.yaml](values.yaml) |
| configs.secret.extra | add additional secrets to be added to argocd-secret | `{}` |
| configs.styles | Define custom CSS styles for your argo instance ([Read More](https://argo-cd.readthedocs.io/en/stable/operator-manual/custom-styles/)). This Settings will automatically mount the provided css and reference it in the argo configuration. | `""` (See [values.yaml](values.yaml)) |
| openshift.enabled | enables using arbitrary uid for argo repo server | `false` |

## ArgoCD Controller

| Parameter | Description | Default |
|-----|---------|-------------|
| controller.affinity | [Assign custom affinity rules to the deployment](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/) | `{}` |
| controller.args.operationProcessors | define the controller `--operation-processors` | `"10"` |
| controller.args.appResyncPeriod | define the controller `--app-resync` | `"180"` |
| controller.args.selfHealTimeout | define the controller `--self-heal-timeout-seconds` | `"5"` |
| controller.args.statusProcessors | define the controller `--status-processors` | `"20"` |
| controller.clusterAdminAccess.enabled | Enable RBAC for local cluster deployments. | `true` |
| controller.containerPort | Controller listening port. | `8082` |
| controller.extraArgs | Additional arguments for the controller. A list of flags | `[]` |
| controller.enableStatefulSet | Enable deploying the controller as a StatefulSet instead of a Deployment. Used for HA installations. | `false` |
| controller.env | Environment variables for the controller. | `[]` |
| controller.envFrom | `envFrom` to pass to the controller. | `[]` (See [values.yaml](values.yaml)) |
| controller.image.repository | Repository to use for the controller | `global.image.repository` |
| controller.image.imagePullPolicy | Image pull policy for the controller | `global.image.imagePullPolicy` |
| controller.image.tag | Tag to use for the controller | `global.image.tag` |
| controller.livenessProbe.failureThreshold | [Kubernetes probe configuration](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) | `3` |
| controller.livenessProbe.initialDelaySeconds | [Kubernetes probe configuration](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) | `10` |
| controller.livenessProbe.periodSeconds | [Kubernetes probe configuration](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) | `10` |
| controller.livenessProbe.successThreshold | [Kubernetes probe configuration](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) | `1` |
| controller.livenessProbe.timeoutSeconds | [Kubernetes probe configuration](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) | `1` |
| controller.logLevel | Controller log level | `"info"` |
| controller.metrics.enabled | Deploy metrics service | `false` |
| controller.metrics.service.annotations | Metrics service annotations | `{}` |
| controller.metrics.service.labels | Metrics service labels | `{}` |
| controller.metrics.service.servicePort | Metrics service port | `8082` |
| controller.metrics.serviceMonitor.enabled | Enable a prometheus ServiceMonitor. | `false` |
| controller.metrics.serviceMonitor.selector | Prometheus ServiceMonitor selector. | `{}` |
| controller.name | Controller name string. | `"application-controller"` |
| controller.nodeSelector | [Node selector](https://kubernetes.io/docs/user-guide/node-selection/) | `{}` |
| controller.podAnnotations | Annotations for the controller pods | `{}` |
| controller.podLabels | Labels for the controller pods | `{}` |
| controller.priorityClassName | Priority class for the controller pods | `""` |
| controller.readinessProbe.failureThreshold | [Kubernetes probe configuration](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) | `3` |
| controller.readinessProbe.initialDelaySeconds | [Kubernetes probe configuration](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) | `10` |
| controller.readinessProbe.periodSeconds | [Kubernetes probe configuration](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) | `10` |
| controller.readinessProbe.successThreshold | [Kubernetes probe configuration](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) | `1` |
| controller.readinessProbe.timeoutSeconds | [Kubernetes probe configuration](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) | `1` |
| controller.replicas | The number of controller pods to run | `1` |\
| controller.resources | Resource limits and requests for the controller pods. | `{}` |
| controller.service.annotations | Controller service annotations. | `{}` |
| controller.service.labels | Controller service labels. | `{}` |
| controller.service.port | Controller service port. | `8082` |
| controller.serviceAccount.annotations | Controller service account annotations | `{}` |
| controller.serviceAccount.automountServiceAccountToken | Automount API credentials for the Service Account | `true` |
| controller.serviceAccount.create | Create a service account for the controller | `true` |
| controller.serviceAccount.name | Service account name. | `"argocd-application-controller"` |
| controller.tolerations | [Tolerations for use with node taints](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/) | `[]` |
| controller.volumeMounts | Controller volume mounts | `[]` |
| controller.volumes | Controller volumes | `[]` |

## Argo Repo Server

| Property | Description | Default |
|-----|---------|-------------|
| repoServer.affinity | [Assign custom affinity rules to the deployment](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/) | `{}` |
| repoServer.autoscaling.enabled | Enable Horizontal Pod Autoscaler ([HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)) for the repo server | `false` |
| repoServer.autoscaling.minReplicas | Minimum number of replicas for the repo server [HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) | `1` |
| repoServer.autoscaling.maxReplicas | Maximum number of replicas for the repo server [HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) | `5` |
| repoServer.autoscaling.targetCPUUtilizationPercentage | Average CPU utilization percentage for the repo server [HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) | `50` |
| repoServer.autoscaling.targetMemoryUtilizationPercentage | Average memory utilization percentage for the repo server [HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) | `50` |
| repoServer.containerPort | Repo server port | `8081` |
| repoServer.extraArgs | Additional arguments for the repo server. A  list of flags. | `[]` |
| repoServer.env | Environment variables for the repo server. | `[]` |
| repoServer.envFrom | `envFrom` to pass to the repo server. | `[]` (See [values.yaml](values.yaml)) |
| repoServer.image.repository | Repository to use for the repo server | `global.image.repository` |
| repoServer.image.imagePullPolicy | Image pull policy for the repo server | `global.image.imagePullPolicy` |
| repoServer.image.tag | Tag to use for the repo server | `global.image.tag` |
| repoServer.livenessProbe.failureThreshold | [Kubernetes probe configuration](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) | `3` |
| repoServer.livenessProbe.initialDelaySeconds | [Kubernetes probe configuration](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) | `10` |
| repoServer.livenessProbe.periodSeconds | [Kubernetes probe configuration](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) | `10` |
| repoServer.livenessProbe.successThreshold | [Kubernetes probe configuration](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) | `1` |
| repoServer.livenessProbe.timeoutSeconds | [Kubernetes probe configuration](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) | `1` |
| repoServer.logLevel | Log level | `"info"` |
| repoServer.metrics.enabled | Deploy metrics service | `false` |
| repoServer.metrics.service.annotations | Metrics service annotations | `{}` |
| repoServer.metrics.service.labels | Metrics service labels | `{}` |
| repoServer.metrics.service.servicePort | Metrics service port | `8082` |
| repoServer.metrics.serviceMonitor.enabled | Enable a prometheus ServiceMonitor. | `false` |
| repoServer.metrics.serviceMonitor.selector | Prometheus ServiceMonitor selector. | `{}` |
| repoServer.name | Repo server name | `"repo-server"` |
| repoServer.nodeSelector | [Node selector](https://kubernetes.io/docs/user-guide/node-selection/) | `{}` |
| repoServer.podAnnotations | Annotations for the repo server pods | `{}` |
| repoServer.podLabels | Labels for the repo server pods | `{}` |
| repoServer.priorityClassName | Priority class for the repo server | `""` |
| repoServer.readinessProbe.failureThreshold | [Kubernetes probe configuration](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) | `3` |
| repoServer.readinessProbe.initialDelaySeconds | [Kubernetes probe configuration](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) | `10` |
| repoServer.readinessProbe.periodSeconds | [Kubernetes probe configuration](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) | `10` |
| repoServer.readinessProbe.successThreshold | [Kubernetes probe configuration](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) | `1` |
| repoServer.readinessProbe.timeoutSeconds | [Kubernetes probe configuration](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) | `1` |
| repoServer.replicas | The number of repo server pods to run | `1` |
| repoServer.resources | Resource limits and requests for the repo server pods. | `{}` |
| repoServer.service.annotations | Repo server service annotations. | `{}` |
| repoServer.service.automountServiceAccountToken | Automount API credentials for the Service Account | `true` |
| repoServer.service.labels | Repo server service labels. | `{}` |
| repoServer.service.port | Repo server service port. | `8081` |
| repoServer.serviceAccount.annotations | Repo server service account annotations | `{}` |
| repoServer.serviceAccount.create | Create repo server service account | `false` |
| repoServer.serviceAccount.name | Repo server service account name | `"argocd-repo-server"` |
| repoServer.tolerations | [Tolerations for use with node taints](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/) | `[]` |
| repoServer.volumeMounts | Repo server volume mounts | `[]` |
| repoServer.volumes | Repo server volumes | `[]` |

## Argo Server

| Parameter | Description | Default |
|-----|---------|-------------|
| server.affinity | [Assign custom affinity rules to the deployment](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/) | `{}` |
| server.autoscaling.enabled | Enable Horizontal Pod Autoscaler ([HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)) for the server | `false` |
| server.autoscaling.minReplicas | Minimum number of replicas for the server [HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) | `1` |
| server.autoscaling.maxReplicas | Maximum number of replicas for the server [HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) | `5` |
| server.autoscaling.targetCPUUtilizationPercentage | Average CPU utilization percentage for the server [HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) | `50` |
| server.autoscaling.targetMemoryUtilizationPercentage | Average memory utilization percentage for the server [HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) | `50` |
| server.GKEbackendConfig.enabled | Enable BackendConfig custom resource for Google Kubernetes Engine. | `false` |
| server.GKEbackendConfig.spec | [BackendConfigSpec](https://cloud.google.com/kubernetes-engine/docs/concepts/backendconfig#backendconfigspec_v1beta1_cloudgooglecom) | `{}` |
| server.certificate.additionalHosts | Certificate manager additional hosts | `[]` |
| server.certificate.domain | Certificate manager domain | `"argocd.example.com"` |
| server.certificate.enabled | Enables a certificate manager certificate. | `false` |
| server.certificate.issuer | Certificate manager issuer | `{}` |
| server.clusterAdminAccess.enabled | Enable RBAC for local cluster deployments. | `true` |
| server.configAnnotations | ArgoCD configuration configmap annotations | `{}` |
| server.config | [General Argo CD configuration](https://argoproj.github.io/argo-cd/operator-manual/declarative-setup/#repositories) | See [values.yaml](values.yaml) |
| server.containerPort | Server container port. | `8080` |
| server.extraArgs | Additional arguments for the server. A list of flags. | `[]` |
| server.staticAssets.enabled | Disable deprecated flag --staticassets | `false` |
| server.env | Environment variables for the server. | `[]` |
| server.envFrom | `envFrom` to pass to the server. | `[]` (See [values.yaml](values.yaml)) |
| server.image.repository | Repository to use for the server | `global.image.repository` |
| server.image.imagePullPolicy | Image pull policy for the server | `global.image.imagePullPolicy` |
| server.image.tag | Tag to use for the server | `global.image.tag` |
| server.ingress.annotations | Additional ingress annotations | `{}` |
| server.ingress.enabled | Enable an ingress resource for the server | `false` |
| server.ingress.hosts | List of ingress hosts | `[]` |
| server.ingress.labels | Additional ingress labels. | `{}` |
| server.ingress.ingressClassName | Defines which ingress controller will implement the resource | `""` |
| server.ingress.tls | Ingress TLS configuration. | `[]` |
| server.ingress.https | Uses `server.service.servicePortHttps` instead `server.service.servicePortHttp` | `false` |
| server.ingressGrpc.annotations | Additional ingress annotations for dedicated [gRPC-ingress] | `{}` |
| server.ingressGrpc.enabled | Enable an ingress resource for the server for dedicated [gRPC-ingress] | `false` |
| server.ingressGrpc.hosts | List of ingress hosts for dedicated [gRPC-ingress] | `[]` |
| server.ingressGrpc.labels | Additional ingress labels for dedicated [gRPC-ingress] | `{}` |
| server.ingressGrpc.ingressClassName | Defines which ingress controller will implement the resource [gRPC-ingress] | `""` |
| server.ingressGrpc.tls | Ingress TLS configuration for dedicated [gRPC-ingress] | `[]` |
| server.ingressGrpc.isAWSALB | Setup up GRPC ingress to work with an AWS ALB | `false` |
| server.ingressGrpc.awsALB.serviceType | Service type for the AWS ALB GRPC service | `NodePort` |
| server.route.enabled | Enable a OpenShift route for the server | `false` |
| server.route.hostname | Hostname of OpenShift route | `""` |
| server.lifecycle | PostStart and PreStop hooks configuration | `{}` |
| server.livenessProbe.failureThreshold | [Kubernetes probe configuration](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) | `3` |
| server.livenessProbe.initialDelaySeconds | [Kubernetes probe configuration](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) | `10` |
| server.livenessProbe.periodSeconds | [Kubernetes probe configuration](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) | `10` |
| server.livenessProbe.successThreshold | [Kubernetes probe configuration](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) | `1` |
| server.livenessProbe.timeoutSeconds | [Kubernetes probe configuration](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) | `1` |
| server.logLevel | Log level | `"info"` |
| server.metrics.enabled | Deploy metrics service | `false` |
| server.metrics.service.annotations | Metrics service annotations | `{}` |
| server.metrics.service.labels | Metrics service labels | `{}` |
| server.metrics.service.servicePort | Metrics service port | `8082` |
| server.metrics.serviceMonitor.enabled | Enable a prometheus ServiceMonitor. | `false` |
| server.metrics.serviceMonitor.selector | Prometheus ServiceMonitor selector. | `{}` |
| server.name | Argo CD server name | `"server"` |
| server.nodeSelector | [Node selector](https://kubernetes.io/docs/user-guide/node-selection/) | `{}` |
| server.podAnnotations | Annotations for the server pods | `{}` |
| server.podLabels | Labels for the server pods | `{}` |
| server.priorityClassName | Priority class for the server | `""` |
| server.rbacConfigAnnotations | RBAC configmap annotations | `{}` |
| server.rbacConfig | [Argo CD RBAC policy](https://argoproj.github.io/argo-cd/operator-manual/rbac/) | `{}` |
| server.readinessProbe.failureThreshold | [Kubernetes probe configuration](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) | `3` |
| server.readinessProbe.initialDelaySeconds | [Kubernetes probe configuration](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) | `10` |
| server.readinessProbe.periodSeconds | [Kubernetes probe configuration](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) | `10` |
| server.readinessProbe.successThreshold | [Kubernetes probe configuration](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) | `1` |
| server.readinessProbe.timeoutSeconds | [Kubernetes probe configuration](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) | `1` |
| server.replicas | The number of server pods to run | `1` |
| server.resources | Resource limits and requests for the server | `{}` |
| server.service.annotations | Server service annotations | `{}` |
| server.service.labels | Server service labels | `{}` |
| server.service.servicePortHttp | Server service http port | `80` |
| server.service.servicePortHttps | Server service https port | `443` |
| server.service.servicePortHttpName | Server service http port name, can be used to route traffic via istio | `http` |
| server.service.servicePortHttpsName | Server service https port name, can be used to route traffic via istio | `https` |
| server.service.nodePortHttp | Server service http port for NodePort service type| `30080` |
| server.service.servicePortHttps | Server service http port for NodePort service type | `30443` |
| server.service.loadBalancerSourceRanges | Source IP ranges to allow access to service from. | `[]` |
| server.service.externalIPs | Server service external IPs. | `[]` |
| server.service.type | Server service type | `"ClusterIP"` |
| server.serviceAccount.annotations | Server service account annotations | `{}` |
| server.serviceAccount.automountServiceAccountToken | Automount API credentials for the Service Account | `true` |
| server.serviceAccount.create | Create server service account | `true` |
| server.serviceAccount.name | Server service account name | `"argocd-server"` |
| server.tolerations | [Tolerations for use with node taints](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/) | `[]` |
| server.volumeMounts | Server volume mounts | `[]` |
| server.volumes | Server volumes | `[]` |

## Dex

| Property | Description | Default |
|-----|---------|-------------|
| dex.affinity | [Assign custom affinity rules to the deployment](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/) | `{}` |
| dex.containerPortGrpc | GRPC container port | `5557` |
| dex.containerPortHttp | HTTP container port | `5556` |
| dex.enabled | Enable dex | `true` |
| dex.image.imagePullPolicy | Dex imagePullPolicy | `"IfNotPresent"` |
| dex.image.repository | Dex image repository | `"quay.io/dexidp/dex"` |
| dex.image.tag | Dex image tag | `"v2.14.0"` |
| dex.initImage.repository | Argo CD init image repository. | `global.image.repository` |
| dex.initImage.imagePullPolicy | Argo CD init image imagePullPolicy | `global.image.imagePullPolicy` |
| dex.initImage.tag | Argo CD init image tag | `global.image.tag` |
| dex.metrics.enabled | Deploy metrics service | `false` |
| dex.metrics.service.annotations | Metrics service annotations | `{}` |
| dex.metrics.service.labels | Metrics service labels | `{}` |
| dex.metrics.serviceMonitor.enabled | Enable a prometheus ServiceMonitor. | `false` |
| dex.metrics.serviceMonitor.selector | Prometheus ServiceMonitor selector. | `{}` |
| dex.name | Dex name | `"dex-server"` |
| dex.env | Environment variables for the Dex server. | `[]` |
| dex.envFrom | `envFrom` to pass to the Dex server. | `[]` (See [values.yaml](values.yaml)) |
| dex.nodeSelector | [Node selector](https://kubernetes.io/docs/user-guide/node-selection/) | `{}` |
| dex.podAnnotations | Annotations for the Dex server pods | `{}` |
| dex.podLabels | Labels for the Dex server pods | `{}` |
| dex.priorityClassName | Priority class for dex | `""` |
| dex.resources | Resource limits and requests for dex | `{}` |
| dex.serviceAccount.automountServiceAccountToken | Automount API credentials for the Service Account | `true` |
| dex.serviceAccount.create | Create dex service account | `true` |
| dex.serviceAccount.name | Dex service account name | `"argocd-dex-server"` |
| dex.servicePortGrpc | Server GRPC port | `5557` |
| dex.servicePortGrpcName | Server GRPC port name | `grpc` |
| dex.servicePortHttp | Server HTTP port | `5556` |
| dex.servicePortHttpName | Server GRPC port name | `http` |
| dex.tolerations | [Tolerations for use with node taints](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/) | `[]` |
| dex.volumeMounts | Dex volume mounts | `"/shared"` |
| dex.volumes | Dex volumes | `{}` |

## Redis

When Redis is completely disabled from the chart (`redis.enabled=false`) and
an external Redis instance wants to be used or
when Redis HA subcart is enabled (`redis.enabled=true and redis-ha.enabled=true`)
but HA proxy is disabled `redis-ha.haproxy.enabled=false` Redis flags need to be specified
through `xxx.extraArgs`

| Parameter | Description | Default |
|-----|---------|-------------|
| redis.affinity | [Assign custom affinity rules to the deployment](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/) | `{}` |
| redis.containerPort | Redis container port | `6379` |
| redis.enabled | Enable redis | `true` |
| redis.image.imagePullPolicy | Redis imagePullPolicy | `"IfNotPresent"` |
| redis.image.repository | Redis repository | `"redis"` |
| redis.image.tag | Redis tag | `"6.2.1-alpine"` |
| redis.extraArgs | Additional arguments for the `redis-server`. A list of flags. | `[]` |
| redis.name | Redis name | `"redis"` |
| redis.env | Environment variables for the Redis server. | `[]` |
| redis.envFrom | `envFrom` to pass to the Redis server. | `[]` (See [values.yaml](values.yaml)) |
| redis.nodeSelector | [Node selector](https://kubernetes.io/docs/user-guide/node-selection/) | `{}` |
| redis.podAnnotations | Annotations for the Redis server pods | `{}` |
| redis.podLabels | Labels for the Redis server pods | `{}` |
| redis.priorityClassName | Priority class for redis | `""` |
| redis.resources | Resource limits and requests for redis | `{}` |
| redis.securityContext | Redis Pod Security Context | See [values.yaml](values.yaml) |
| redis.servicePort | Redis service port | `6379` |
| redis.tolerations | [Tolerations for use with node taints](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/) | `[]` |
| redis-ha | Configures [Redis HA subchart](https://github.com/DandyDeveloper/charts/tree/master/charts/redis-ha) The properties below have been changed from the subchart defaults | |
| redis-ha.enabled | Enables the Redis HA subchart and disables the custom Redis single node deployment| `false` |
| redis-ha.exporter.enabled | If `true`, the prometheus exporter sidecar is enabled | `true` |
| redis-ha.persistentVolume.enabled | Configures persistency on Redis nodes | `false`
| redis-ha.redis.masterGroupName | Redis convention for naming the cluster group: must match `^[\\w-\\.]+$` and can be templated | `argocd`
| redis-ha.redis.config | Any valid redis config options in this section will be applied to each server (see `redis-ha` chart) | `` |
| redis-ha.redis.config.save | Will save the DB if both the given number of seconds and the given number of write operations against the DB occurred. `""`  is disabled | `""` |
| redis-ha.haproxy.enabled | Enabled HAProxy LoadBalancing/Proxy | `true` |
| redis-ha.haproxy.metrics.enabled | HAProxy enable prometheus metric scraping | `true` |
| redis-ha.image.tag | Redis tag | `"6.2.1-alpine"` |

[gRPC-ingress]: https://argoproj.github.io/argo-cd/operator-manual/ingress/


### Using AWS ALB Ingress Controller With GRPC
If you are using an AWS ALB Ingress controller, you will need to set `server.ingressGrpc.isAWSALB` to `true`. This will create a second service with the annotation `alb.ingress.kubernetes.io/backend-protocol-version: HTTP2` and modify the server ingress to add a condition annotation to route GRPC traffic to the new service. 

Example:
```yaml
server:
  ingress:
    enabled: true
    annotations: 
      alb.ingress.kubernetes.io/backend-protocol: HTTPS
      alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
      alb.ingress.kubernetes.io/scheme: internal
      alb.ingress.kubernetes.io/target-type: ip
  ingressGrpc:
    enabled: true
    isAWSALB: true
    awsALB:
      serviceType: ClusterIP
      
```
