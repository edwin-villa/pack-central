# Metrics Server

[Metrics Server](https://github.com/kubernetes-sigs/metrics-server) is a scalable, efficient source of container resource metrics for Kubernetes built-in autoscaling pipelines. It collects resource usage (CPU and memory) from the `kubelet` on each node and exposes it through the Kubernetes [Metrics API](https://github.com/kubernetes/metrics), which is consumed by core components such as the Horizontal Pod Autoscaler (`kubectl autoscale`), the Vertical Pod Autoscaler, and `kubectl top`.

This pack packages the official Metrics Server Helm chart, version `3.13.1`, which deploys Metrics Server `v0.8.1`.


## Prerequisites

- A Kubernetes cluster running v1.29 or later.
- Network connectivity between the API server and the `kubelet` on every node (the API server must be able to reach the Metrics Server pod, and Metrics Server must be able to reach each node's `kubelet` on the configured port).
- No previous, non-Palette-managed installation of Metrics Server on the target cluster. Some managed Kubernetes distributions (for example EKS, AKS and GKE) ship their own Metrics Server add-on; disable it before deploying this pack to avoid a conflict over the `v1beta1.metrics.k8s.io` API service.


## Parameters

The pack exposes the upstream Helm chart values under the `charts.metrics-server` key. The defaults work for most clusters; the parameters below are the ones most likely to need changing.

| **Parameter** | **Description** | **Type** | **Default Value** | **Required** |
|---|---|---|---|---|
| `charts.metrics-server.image.repository` | Metrics Server image repository. | String | `registry.k8s.io/metrics-server/metrics-server` | No |
| `charts.metrics-server.image.tag` | Image tag. Empty means the chart's `appVersion` (`v0.8.1`). | String | `""` | No |
| `charts.metrics-server.replicas` | Number of Metrics Server replicas. | Int | `1` | No |
| `charts.metrics-server.defaultArgs` | Default command-line arguments passed to Metrics Server. | List | See values | No |
| `charts.metrics-server.args` | Additional command-line arguments appended to `defaultArgs`. | List | `[]` | No |
| `charts.metrics-server.apiService.insecureSkipTLSVerify` | Skip TLS verification when the API server calls the Metrics Server API service. Disable and set `apiService.caBundle` for a hardened setup. | Bool | `true` | No |
| `charts.metrics-server.tls.type` | TLS certificate method: `metrics-server`, `helm`, `cert-manager`, or `existingSecret`. | String | `"metrics-server"` | No |
| `charts.metrics-server.hostNetwork.enabled` | Run Metrics Server in the host network namespace. Required on some overlay networks (for example Weave on EKS) where the API server cannot otherwise reach the pod. | Bool | `false` | No |
| `charts.metrics-server.resources` | CPU/memory requests and limits for the Metrics Server container. | Object | `requests: {cpu: 100m, memory: 200Mi}` | No |
| `charts.metrics-server.podDisruptionBudget.enabled` | Create a PodDisruptionBudget for the Metrics Server deployment. | Bool | `false` | No |
| `charts.metrics-server.service.type` | Service type for the Metrics Server Service. | String | `ClusterIP` | No |
| `charts.metrics-server.serviceMonitor.enabled` | Create a Prometheus `ServiceMonitor` for Metrics Server. Requires the Prometheus Operator CRDs to be present. | Bool | `false` | No |
| `charts.metrics-server.nodeSelector` | Node selector for the Metrics Server pod. | Object | `{}` | No |
| `charts.metrics-server.tolerations` | Tolerations for the Metrics Server pod. | List | `[]` | No |

> [!CAUTION]
> If your cloud provider or Kubernetes distribution already ships a Metrics Server add-on (for example the EKS or AKS built-in add-on), remove it before installing this pack. Two Metrics Server instances competing for the same `v1beta1.metrics.k8s.io` API service will conflict.


## Upgrade

This is the initial release of this pack, so there are no prior pack versions to migrate from. When upgrading to a future version of this pack, review the [Metrics Server releases](https://github.com/kubernetes-sigs/metrics-server/releases) and the chart's [CHANGELOG](https://github.com/kubernetes-sigs/metrics-server/blob/master/charts/metrics-server/CHANGELOG.md) for breaking changes between chart versions, since Helm does not automatically reconcile changes to cluster-scoped resources such as `ClusterRole`, `ClusterRoleBinding`, or the `APIService`.


## Usage

Add this pack as a layer in an [add-on cluster profile](https://docs.spectrocloud.com/profiles/cluster-profiles/create-cluster-profiles/create-addon-profile/). The default values work for most clusters without any changes.

If your cluster uses an overlay network that prevents the API server from reaching the Metrics Server pod directly (a common case on EKS/AKS/GKE), enable host networking:

```yaml
charts:
  metrics-server:
    hostNetwork:
      enabled: true
```

```yaml
charts:
  metrics-server:
    apiService:
      create: false
    serviceAccount:
      create: true
```

Once the cluster reconciles, confirm Metrics Server is running and serving metrics:

```shell
kubectl get pods -n kube-system -l app.kubernetes.io/name=metrics-server
kubectl top nodes
kubectl top pods -A
```

To use Metrics Server with the Horizontal Pod Autoscaler, create an `HorizontalPodAutoscaler` resource that targets your workload:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: example-app
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: example-app
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```


## References

- [Metrics Server on GitHub](https://github.com/kubernetes-sigs/metrics-server)
- [Metrics Server Helm chart](https://github.com/kubernetes-sigs/metrics-server/tree/master/charts/metrics-server)
- [Kubernetes Metrics API](https://github.com/kubernetes/metrics)
- [Horizontal Pod Autoscaler documentation](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Metrics Server FAQ](https://github.com/kubernetes-sigs/metrics-server/blob/master/FAQ.md)
- [Metrics Server known issues](https://github.com/kubernetes-sigs/metrics-server/blob/master/KNOWN_ISSUES.md)
