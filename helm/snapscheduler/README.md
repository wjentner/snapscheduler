# SnapScheduler

The SnapScheduler operator takes snapshots of Kubernetes CSI-based persistent
volumes according to user-supplied schedules.

## About this operator

The SnapScheduler operator takes snapshots of CSI-based PersistentVolumes
according to a configurable
[Cron-like](https://en.wikipedia.org/wiki/Cron#Overview) schedule. The schedules
include configurable retention policies for snapshots as well as selectors to
limit the volumes that are snapshotted. An example schedule could be:

> *Snapshot **all volumes** in a namespace **daily at midnight**, retaining the
> most recent **7** snapshots for each volume.*

Multiple schedules can be combined to provide more elaborate protection schemes.
For example, a given volume (or collection of volumes) could be protected with:

- 6 hourly snapshots
- 7 daily snapshots
- 4 weekly snapshots
- 12 monthly snapshots

### How it works

The operator watches for `SnapshotSchedule` CRs in each namespace. When the
current time matches the schedule's cronspec, the operator creates a
`VolumeSnapshot` object for each `PersistentVolumeClaim` in the namespace (or
subset thereof if a label selector is provided). The `VolumeSnapshot` objects
are named according to the template: `<pvcname>-<schedulename>-<timestamp>`.
After creating the new snapshots, the oldest snapshots are removed if necessary,
according to the retention policy of the schedule.

Please see the [full documentation](https://backube.github.io/snapscheduler/)
for more information.

## Requirements

- Kubernetes >= 1.17
- CSI-based storage driver that supports snapshots (i.e. has the
  `CREATE_DELETE_SNAPSHOT` capability)

## Installation

The snapscheduler operator is a "cluster-level" operator. A single instance will
watch `snapshotschedules` across all namespaces in the cluster. **Running more
than one instance of the scheduler at a time is not supported.**

```console
$ kubectl create ns backube-snapscheduler
namespace/backube-snapscheduler created

$ helm install --namespace backube-snapscheduler snapscheduler backube/snapscheduler
NAME: snapscheduler
LAST DEPLOYED: Mon Nov 25 17:38:26 2019
NAMESPACE: backube-snapscheduler
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Thank you for installing snapscheduler!

The snapscheduler operator is now installed in the backube-snapscheduler
namespace, and snapshotschedules should be enabled cluster-wide.

See https://backube.github.io/snapscheduler/usage.html to get started.

Schedules can be viewed via:
$ kubectl -n <mynampspace> get snapshotschedules
...
```

## Examples

The schedule for snapshotting is controlled by the
`snapshotschedules.snapscheduler.backube` Custom Resource. This is a namespaced
resource that applies only to the PersistentVolumeClaims in its namespace.

Below is a simple example that keeps 7 daily (taken at midnight) snapshots of
all PVCs in a given namespace:

```yaml
---
apiVersion: snapscheduler.backube/v1
kind: SnapshotSchedule
metadata:
  name: daily
spec:
  retention:
    maxCount: 7
  schedule: "0 0 * * *"
```

See the [usage
documentation](https://backube.github.io/snapscheduler/usage.html) for full
details, including how to:

- add label selectors to restrict which PVCs this schedule applies to
- set the VolumeSnapshotClass used by the schedule
- apply custom labels to the automatically created VolumeSnapshot objects

## Configuration

The following optional parameters in the chart can be configured, either by
using `--set` on the command line or via a `values.yaml` file. In the general
case, the defaults, shown below, should be sufficient.

- `replicaCount`: `1`
  - The number of replicas of the operator to run. Only one is active at a time
    via leader election.
- `image.repository`: `quay.io/backube/snapscheduler`
  - The location of the operator container image
- `image.tagOverride`: `""`
  - If set, it will override the operator container image tag. The default tag
    is set per chart version and can be viewed (as `appVersion`) via `helm show
    chart`.
- `image.pullPolicy`: `IfNotPresent`
  - Overrides the container image pull policy
- `imagePullSecrets`: none
  - May be set if pull secret(s) are needed to retrieve the operator image
- `rbacProxy.image.repository`: `quay.io/brancz/kube-rbac-proxy`
  - Specifies the container image used for the RBAC proxy
- `rbacProxy.image.tag`: `v0.11.0`
  - Specifies the tag for the RBAC proxy container image
- `rbacProxy.image.pullPolicy`: `IfNotPresent`
  - Specifies the RBAC proxy container image pull policy
- `rbacProxy.resources`: requests for 10m CPU and 100Mi memory; no limits
  - Allows overriding the resource requests/limits for the kube-rbac-proxy
    container of the operator pod.
- `serviceAccount.create`: `true`
  - Whether to create the ServiceAccount for the operator
- `serviceAccount.name`: none
  - Override the name of the operator's ServiceAccount
- `podSecurityContext`: none
  - Allows setting the security context for the operator pod
- `securityContext`: none
  - Allows setting the operator container's security context
- `resources`: requests for 10m CPU and 100Mi memory; no limits
  - Allows overriding the resource requests/limits for the manager
    container of the operator pod.
- `nodeSelector`: `kubernetes.io/arch: amd64`, `kubernetes.io/os: linux`
  - Allows applying a node selector to the operator pod
- `tolerations`: none
  - Allows applying tolerations to the operator pod
- `topologySpreadConstraints`: none
  - Topology spread constraints rely on node labels to identify the
    topology domain(s) that each Node is in.
- `affinity`: node-level anti-affinity
  - Allows setting the operator pod's affinity
