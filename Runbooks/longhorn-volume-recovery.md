# Longhorn Disk Pressure And Replica Move Runbook

This runbook explains how to investigate Longhorn disk pressure, identify which
volumes consume space on each node, compare Longhorn usage with container image
cache, and safely move a single-replica volume to another node.

Example incident:

```text
Longhorn node disk schedulable = false
reason = DiskPressure
detached/faulted volumes still consume node disk space
a single-replica volume cannot attach, so the workload cannot start
```

A detached Longhorn volume is not deleted. It is only not attached to a workload.
Its replica data directory still exists under the Longhorn disk path and still
consumes disk space.

---

## 1. Collect Longhorn Volume Data

Collect Longhorn volumes, replicas, and PVCs:

```bash
kubectl get volumes.longhorn.io -n longhorn-system -o json > /tmp/lh-volumes.json
kubectl get replicas.longhorn.io -n longhorn-system -o json > /tmp/lh-replicas.json
kubectl get pvc -A -o json > /tmp/pvcs.json
```

---

## 2. Print A Node-Based Volume Report

Use this command when you want a readable report grouped by node:

```bash
jq -r '
  def gi2: (((. / 1024 / 1024 / 1024) * 100 | floor) / 100 | tostring + "Gi");
  ($vols[0].items
    | map({
        key: .metadata.name,
        ns: (.status.kubernetesStatus.namespace // "-"),
        pvc: (.status.kubernetesStatus.pvcName // "-"),
        workload: (.status.kubernetesStatus.workloadsStatus[0].podName // "-"),
        size: (.spec.size | tonumber),
        actual: ((.status.actualSize // "0") | tonumber),
        state: (.status.state // "-"),
        robustness: (.status.robustness // "-")
      })
    | INDEX(.key)
  ) as $vmap |
  [
    $reps[0].items[] |
    .spec.volumeName as $vol |
    ($vmap[$vol] // {}) as $v |
    {
      node: (.spec.nodeID // "-"),
      pvc: (($v.ns // "-") + "/" + ($v.pvc // "-")),
      scheduled: (($v.size // 0) | gi2),
      actual: (($v.actual // 0) | gi2),
      state: (($v.state // "-") + "/" + (.status.currentState // "-")),
      robustness: ($v.robustness // "-")
    }
  ] |
  group_by(.node)[] |
  "\(.[0].node)\nvolume/PVC\tscheduled\tactual\tstate",
  (.[] |
    [
      .pvc,
      .scheduled,
      .actual,
      (.state + "/" + .robustness)
    ] | @tsv
  ),
  ""
' \
  --slurpfile vols /tmp/lh-volumes.json \
  --slurpfile reps /tmp/lh-replicas.json \
  /tmp/lh-volumes.json | column -t -s $'\t'
```

Example shape:

```text
homelab-workers-pve2-dk699-g9fc8
volume/PVC                                  scheduled   actual    state
production/postgres-storage-user-db-0       2Gi         0.14Gi    attached/running/healthy
observability/prometheus-data               2Gi         0.33Gi    attached/running/healthy
jenkins/jenkins-docker-cache-frontend-pvc   5Gi         0.59Gi    detached/stopped/unknown
sonarqube/sonarqube-postgresql-pvc          2Gi         0.28Gi    attached/running/healthy
sonarqube/sonarqube-data-pvc                5Gi         0.86Gi    attached/running/healthy
jenkins/jenkins-venv-cache-pvc              2Gi         0.14Gi    detached/stopped/unknown
```

The workload/pod name shown in the Longhorn UI can look like a branch-specific
Jenkins pod. That does not mean the volume belongs to that branch. It usually
means "this was the last workload that used the volume." Use the PVC name to
understand what the volume really is.

---

## 3. Compare Scheduled Size, Actual Size, And Node Disk

Longhorn scheduled and actual size per node:

```bash
jq -r '
  def gi2: (((. / 1024 / 1024 / 1024) * 100 | floor) / 100 | tostring + "Gi");
  ($vols[0].items
    | map({
        key: .metadata.name,
        size: (.spec.size | tonumber),
        actual: ((.status.actualSize // "0") | tonumber)
      })
    | INDEX(.key)
  ) as $vmap |
  [
    $reps[0].items[] |
    .spec.nodeID as $node |
    .spec.volumeName as $vol |
    ($vmap[$vol] // {}) as $v |
    {
      node: $node,
      size: ($v.size // 0),
      actual: ($v.actual // 0)
    }
  ] |
  "node\tlonghorn scheduled\tlonghorn actual",
  (
    group_by(.node)[] |
    [
      .[0].node,
      (map(.size) | add | gi2),
      (map(.actual) | add | gi2)
    ] | @tsv
  )
' \
  --slurpfile vols /tmp/lh-volumes.json \
  --slurpfile reps /tmp/lh-replicas.json \
  /tmp/lh-volumes.json | column -t -s $'\t'
```

Longhorn disk schedulable status:

```bash
kubectl get nodes.longhorn.io -n longhorn-system -o json | jq -r '
  def gi2: (((. / 1024 / 1024 / 1024) * 100 | floor) / 100 | tostring + "Gi");
  "node\tdisk path\tscheduled\tavailable\tmaximum\tschedulable\treason",
  (
    .items[] |
    .metadata.name as $node |
    .status.diskStatus |
    to_entries[]? |
    .value as $d |
    [
      $node,
      ($d.diskPath // "-"),
      (($d.storageScheduled // 0) | gi2),
      (($d.storageAvailable // 0) | gi2),
      (($d.storageMaximum // 0) | gi2),
      (($d.conditions[]? | select(.type == "Schedulable") | .status) // "-"),
      (($d.conditions[]? | select(.type == "Schedulable") | .reason) // "-")
    ] | @tsv
  )
' | column -t -s $'\t'
```

Important distinction:

```text
Kubernetes node DiskPressure != Longhorn disk schedulable state
```

Longhorn uses its own `storage-minimal-available-percentage` setting to decide
whether it can schedule new replicas on a disk.

Check the setting:

```bash
kubectl get settings.longhorn.io -n longhorn-system \
  storage-minimal-available-percentage -o yaml
```

---

## 4. Compare Longhorn Usage With Container Image Cache

Kubelet node filesystem and image filesystem usage:

```bash
{
  printf "node\tnode fs used\tnode fs capacity\timage fs used\timage fs capacity\n"
  for node in $(kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'); do
    kubectl get --raw /api/v1/nodes/${node}/proxy/stats/summary | jq -r --arg node "$node" '
      def gi2: (((. / 1024 / 1024 / 1024) * 100 | floor) / 100 | tostring + "Gi");
      [
        $node,
        (.node.fs.usedBytes | gi2),
        (.node.fs.capacityBytes | gi2),
        (.node.runtime.imageFs.usedBytes | gi2),
        (.node.runtime.imageFs.capacityBytes | gi2)
      ] | @tsv
    '
  done
} | column -t -s $'\t'
```

Large images per node:

```bash
kubectl get nodes -o json | jq -r '
  [
    .items[] |
    .metadata.name as $node |
    .status.images[]? |
    {
      node: $node,
      sizeMi: (.sizeBytes / 1024 / 1024 | floor),
      names: (.names | join(","))
    }
  ] |
  group_by(.node)[] |
  sort_by(.sizeMi) | reverse |
  "\(.[0].node)\nsize\timage",
  (.[:30][] | [(.sizeMi | tostring + "Mi"), .names] | @tsv),
  ""
' | column -t -s $'\t'
```

Large images that exist on only one node:

```bash
kubectl get nodes -o json | jq -r '
  .items as $nodes |
  [
    $nodes[] |
    {
      node: .metadata.name,
      images: (.status.images // [])
    }
  ] as $rows |
  [
    $rows[] as $r |
    $r.images[]? |
    {
      name: (.names[0] // "<none>"),
      size: .sizeBytes,
      node: $r.node
    }
  ] |
  [
    group_by(.name)[] |
    {
      name: .[0].name,
      sizeMi: ((map(.size) | max) / 1024 / 1024 | floor),
      nodes: (map(.node) | unique)
    } |
    select((.nodes | length) == 1 and .sizeMi >= 100) |
    {
      node: (.nodes[0]),
      sizeMi: .sizeMi,
      name: .name
    }
  ] |
  group_by(.node)[] |
  sort_by(.sizeMi) | reverse |
  "\(.[0].node)\nsize\timage",
  (.[] | [(.sizeMi | tostring + "Mi"), .name] | @tsv),
  ""
' | column -t -s $'\t'
```

Comparison rule of thumb:

```text
Longhorn actual size + image cache ~= part of node filesystem usage
```

Do not expect an exact match. Node filesystem usage also includes the OS,
kubelet logs, container writable layers, containerd metadata, Longhorn engine
metadata, and other runtime files.

### 4.1 Inspect Containerd Usage On The Affected Node

When Longhorn reports `DiskPressure`, inspect the node filesystem directly.
Use the node IP and SSH account available for the cluster:

```bash
NODE_IP="192.168.0.154"
LONGHORN_NODE="homelab-workers-pve2-dk699-g9fc8"

ssh root@"$NODE_IP" 'hostname; df -hT / /mnt/longhorn-storage'
ssh root@"$NODE_IP" \
  'du -xhd1 /var/lib/containerd 2>/dev/null | sort -h'
ssh root@"$NODE_IP" \
  'du -sh /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs \
          /var/lib/containerd/io.containerd.content.v1.content 2>/dev/null'
```

The incident values were obtained from these commands:

```text
df -h /                                                   -> 40G total, 32G used, 6.4G available
du -sh /var/lib/containerd                                -> about 26G
du -sh .../io.containerd.snapshotter.v1.overlayfs          -> about 19G
du -sh .../io.containerd.content.v1.content                -> about 7G
```

The two containerd subdirectories are parts of the 26 GiB total. Do not add
the 19 GiB and 7 GiB values to the 26 GiB value again. The overlayfs directory
contains unpacked image layers and container filesystem snapshots. The content
directory contains content blobs and image layer data.

Inspect the runtime objects and images before cleanup:

```bash
ssh root@"$NODE_IP" 'crictl ps -a'
ssh root@"$NODE_IP" 'crictl images'
ssh root@"$NODE_IP" 'ctr -n k8s.io images ls'
```

### 4.2 Remove Stale Runtime Objects Without Touching PVC Data

Kubernetes/containerd garbage collection is threshold- and policy-driven; it
does not guarantee that every exited container is removed immediately. Reboots,
restarts, retained runtime records, and a small node disk can leave exited
container snapshots behind until garbage collection runs successfully.

List and remove only exited containers:

```bash
ssh root@"$NODE_IP" '
  ids=$(crictl ps -a --state Exited -q)
  echo "Exited containers: $(printf "%s\\n" "$ids" | awk "NF" | wc -l)"
  if [ -n "$ids" ]; then
    crictl rm $ids
  fi
'
```

This removes stopped container runtime objects and their writable snapshot
references. It does not delete Kubernetes Deployments, StatefulSets, PVCs,
Longhorn volumes, or Longhorn replica directories. Running containers are not
selected by this command.

After exited containers are removed, optionally remove image data that is no
longer referenced by running containers:

```bash
ssh root@"$NODE_IP" 'crictl rmi --prune'
```

`crictl rmi --prune` can time out when containerd is under heavy disk pressure.
That is not a reason to delete files manually from `/var/lib/containerd` or
`/mnt/longhorn-storage`. Check the result and retry after exited containers
have been removed. Images needed by running pods remain in use; images removed
by the prune may be pulled again when a workload starts.

Confirm that the cleanup created enough space:

```bash
ssh root@"$NODE_IP" 'df -h /'
ssh root@"$NODE_IP" \
  'du -sh /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs \
          /var/lib/containerd/io.containerd.content.v1.content 2>/dev/null'
kubectl get nodes.longhorn.io -n longhorn-system "$LONGHORN_NODE" -o json | jq -r '
  .status.diskStatus[] |
  .conditions[] |
  select(.type == "Schedulable") |
  [.status, .reason, .message] | @tsv
'
```

In the incident, removing 34 exited containers reduced the node filesystem
from about 32 GiB used to about 14 GiB used. Containerd overlayfs usage fell
from about 19 GiB to about 6.1 GiB, and content usage fell from about 7 GiB to
about 2.2 GiB. The Longhorn disk then became schedulable again. The
`crictl rmi --prune` command was also attempted, but containerd timed out for
most image removals; one unused image was removed. The large space recovery
came from removing the 34 exited containers and their stale snapshot
references, not from deleting Longhorn data.

### 4.3 Reclaim Deleted Files With Longhorn Filesystem Trim

Deleting files inside a mounted PVC makes the space reusable inside that
filesystem, but it does not immediately reduce the Longhorn volume's `Actual
Size`. Longhorn is a block storage system and needs an UNMAP/TRIM request to
learn which blocks are no longer used.

At the Linux storage layer, `sudo fstrim -av` and Longhorn filesystem trim use
the same operation: `fstrim` tells the filesystem to issue discard/UNMAP for
free extents. They are not different compaction algorithms. Their scope and
orchestration are different. `fstrim -av` trims every eligible filesystem
visible in the node mount namespace, including host filesystems. Longhorn finds
the mount point for the selected attached volume and runs trim for that volume,
with recurring-job scheduling and status managed by Longhorn. Use the Longhorn
operation for normal PVC maintenance and reserve node-wide `fstrim -av` for
intentional host maintenance.

This repository defines a conservative recurring job:

```text
manifest: 3-Longhorn/longhorn-filesystem-trim.yaml
schedule: every day at 03:30
concurrency: 1
scope: the existing Prometheus PVC and future longhorn-storageclass volumes
```

Install the recurring job:

```bash
kubectl apply -f 3-Longhorn/longhorn-filesystem-trim.yaml
```

The Prometheus PVC was created before the StorageClass selector existed, so it
opts in explicitly through these labels:

```yaml
metadata:
  labels:
    recurring-job.longhorn.io/source: enabled
    recurring-job.longhorn.io/filesystem-trim-daily: enabled
```

The source label tells Longhorn to synchronize recurring-job labels from the
PVC to its Longhorn Volume. The second label assigns the named recurring job.

The `longhorn-storageclass` manifest also contains this parameter:

```yaml
parameters:
  recurringJobSelector: >-
    [{"name":"filesystem-trim-daily","isGroup":false}]
```

Longhorn applies the selector while provisioning a volume. It therefore assigns
the trim job automatically to future volumes created from this StorageClass,
but does not modify volumes that already exist. Add the PVC labels above when
an existing volume must opt in. StorageClass parameters are immutable, so
changing this selector on a live cluster requires deleting and immediately
recreating only the StorageClass object; that operation does not delete its
existing PVs or PVCs.

Only attached and mounted volumes can be trimmed. Verify the assignment and job
history with:

```bash
kubectl get recurringjob.longhorn.io -n longhorn-system filesystem-trim-daily
kubectl get pvc -n observability prometheus-data --show-labels
kubectl get jobs -n longhorn-system
```

Filesystem trim performs storage I/O. Running it every 30 minutes across many
volumes creates unnecessary work, so start with a daily schedule and
`concurrency: 1`. A successful trim may still reclaim little space when valid
Longhorn snapshots retain the old blocks. Review snapshot retention separately
before enabling automatic snapshot removal during trim.

### 4.4 Do Not Schedule Blind `crictl` Cleanup

The exited-container cleanup in Section 4.2 is an incident recovery action, not
a Longhorn recurring job. Longhorn does not own containerd runtime objects.

A Kubernetes CronJob would need privileged access to every node's containerd
socket and host filesystem. A host cron or systemd timer avoids that pod access
but creates configuration drift and can remove stopped-container evidence while
an incident is being investigated. Kubelet and containerd should normally
garbage-collect exited containers and unused images themselves.

Kubelet checks unused containers every minute and unused images every five
minutes. A check does not mean that every exited container is deleted. Container
GC applies a minimum age, a per-Pod/container retention limit, and an optional
global dead-container limit. The default command-line values retain up to one
old instance per container and do not impose a global total limit. Containers
from deleted Pods become eligible after the minimum age, and kubelet only
garbage-collects containers that it manages. It is therefore normal to see
dozens of exited entries after many workloads restart, even while GC is working.

The current cluster does not pass custom container-GC flags to kubelet. Its
separate image-GC policy starts when image filesystem usage reaches 85 percent
and removes unused images until usage falls to 80 percent; unused images must be
at least two minutes old. These image thresholds do not force all exited
container records to be removed.

If stale runtime objects repeatedly accumulate, investigate kubelet image and
container garbage-collection settings and containerd health first. Keep the
explicit `crictl rm` and `crictl rmi --prune` commands as controlled recovery
steps unless repeated incidents prove that an additional node maintenance
timer is necessary.

---

## 5. Move A Faulted/Detached Single-Replica Volume

Use this procedure only for a single-replica volume whose data must be kept.

Example variables:

```bash
VOLUME="pvc-1fa6cfa6-2e21-4d44-ad1d-8748ef0a544e"
NAMESPACE="dependency-track"
STATEFULSET="dtrack-dependency-track-api-server"
SOURCE_NODE="homelab-workers-pve1-85hh8-br9hw"
TARGET_NODE="homelab-workers-pve2-dk699-g9fc8"
ORIGINAL_REPLICAS="1"
```

### 5.0 Pause GitOps Reconciliation When Argo CD Self-Heal Is Enabled

Scaling a workload to zero can be immediately reverted by Argo CD when the
application has automated sync with `selfHeal: true`. Pause the relevant
parent and child applications before scaling down. Pause the root application
first so it does not recreate the child application policies:

```bash
kubectl patch application root-app-helm -n argocd --type=merge \
  -p '{"spec":{"syncPolicy":{"automated":null}}}'

kubectl patch application helm-production -n argocd --type=merge \
  -p '{"spec":{"syncPolicy":{"automated":null}}}'
kubectl patch application helm-production-user-service -n argocd --type=merge \
  -p '{"spec":{"syncPolicy":{"automated":null}}}'
```

Use the equivalent staging application names when the affected workload is in
staging. This is a temporary cluster change; it does not modify the GitOps
repository.

### 5.1 Stop The Workload

Stop the workload that writes to the volume:

```bash
kubectl scale sts -n "$NAMESPACE" "$STATEFULSET" --replicas=0
```

Confirm the pod stopped:

```bash
kubectl get pods -n "$NAMESPACE" -o wide
```

### 5.2 Make The Longhorn Disk Schedulable

Salvage or rebuild may fail with:

```text
disk ... is unschedulable for replica ...
```

Prefer freeing node disk space first. Use the containerd inspection and
cleanup procedure in Section 4.1 and Section 4.2, then check the Longhorn disk
condition again. In the incident, this was sufficient; the threshold did not
need to be changed.

Check the current minimal available percentage:

```bash
kubectl get setting.longhorn.io -n longhorn-system \
  storage-minimal-available-percentage -o jsonpath='{.value}{"\n"}'
```

Temporarily lower it, for example from `25` to `20`:

```bash
kubectl patch setting.longhorn.io -n longhorn-system \
  storage-minimal-available-percentage \
  --type=merge \
  -p '{"value":"20"}'
```

Restore the old value after the move.

### 5.3 Salvage The Volume

Longhorn UI:

```text
Volume -> <VOLUME> -> Salvage
```

Check the result:

```bash
kubectl get volumes.longhorn.io -n longhorn-system "$VOLUME" -o json | jq -r '
  [
    .metadata.name,
    .status.state,
    .status.robustness,
    .status.currentNodeID,
    (.spec.numberOfReplicas | tostring)
  ] | @tsv
'

kubectl get replicas.longhorn.io -n longhorn-system \
  -l longhornvolume="$VOLUME" -o wide
```

The Longhorn API accepts the replica names explicitly. The UI is preferred,
but the following command is useful when the UI action is unavailable:

```bash
MANAGER_POD=$(kubectl get pod -n longhorn-system \
  -l app=longhorn-manager -o jsonpath='{.items[0].metadata.name}')
MANAGER_IP=$(kubectl get pod -n longhorn-system \
  -l app=longhorn-manager -o jsonpath='{.items[0].status.podIP}')
REPLICA_NAME=$(kubectl get replicas.longhorn.io -n longhorn-system \
  -l longhornvolume="$VOLUME" -o jsonpath='{.items[0].metadata.name}')

kubectl exec -n longhorn-system "$MANAGER_POD" -c longhorn-manager -- \
  curl -sS -X POST \
  -H 'Content-Type: application/json' \
  -d "{\"names\":[\"$REPLICA_NAME\"]}" \
  "http://${MANAGER_IP}:9500/v1/volumes/${VOLUME}?action=salvage" | jq
```

Salvage is valid only when the volume is detached and faulted, the selected
replica belongs to that volume, and its disk is schedulable. A response such
as `invalid robustness state: unknown` means Longhorn is still reconciling the
volume; wait and inspect the volume/replica status again.

### 5.4 Manually Attach The Volume

Longhorn UI:

```text
Volume -> <VOLUME> -> Attach
Node -> TARGET_NODE
```

Why attach first?

```text
attach -> starts the Longhorn engine
       -> opens the existing replica
       -> allows Longhorn to rebuild/sync a new replica
```

A detached volume is passive. The Longhorn engine must run before Longhorn can
rebuild a new replica from the existing data.

The equivalent Longhorn API attach request is:

```bash
ATTACHMENT_ID="longhorn-runbook-$(date +%s)"

kubectl exec -n longhorn-system "$MANAGER_POD" -c longhorn-manager -- \
  curl -sS -X POST \
  -H 'Content-Type: application/json' \
  -d "{\"hostId\":\"$TARGET_NODE\",\"disableFrontend\":false,\"attachedBy\":\"runbook\",\"attacherType\":\"longhorn-api\",\"attachmentID\":\"$ATTACHMENT_ID\"}" \
  "http://${MANAGER_IP}:9500/v1/volumes/${VOLUME}?action=attach" | jq
```

Use the Longhorn UI when possible. Do not create a manual attachment while a
workload is still using the volume. Remove the manual attachment after the
move, so CSI can attach the volume normally.

### 5.5 Force The New Replica To The Target Node

Temporarily disable scheduling on nodes that should not receive the new replica:

```text
Node -> SOURCE_NODE -> Edit Disk -> Allow Scheduling = false
Node -> any other unwanted node -> Edit Disk -> Allow Scheduling = false
Node -> TARGET_NODE -> Allow Scheduling = true
```

This does not delete existing data. It only controls where Longhorn can schedule
new replicas.

### 5.6 Temporarily Increase Replica Count To 2

Longhorn UI:

```text
Volume -> <VOLUME> -> Edit -> Number Of Replicas = 2
```

Alternative kubectl command:

```bash
kubectl patch volumes.longhorn.io -n longhorn-system "$VOLUME" \
  --type=merge \
  -p '{"spec":{"numberOfReplicas":2}}'
```

Watch the rebuild:

```bash
watch -n 2 "kubectl get replicas.longhorn.io -n longhorn-system -l longhornvolume=$VOLUME -o wide"
```

Watch volume health:

```bash
watch -n 2 "kubectl get volumes.longhorn.io -n longhorn-system $VOLUME"
```

Target state:

```text
SOURCE_NODE replica running
TARGET_NODE replica running
volume robustness healthy
```

### 5.7 Remove The Old Replica

Do not remove the old replica until the target-node replica is healthy.

Longhorn UI:

```text
Volume -> <VOLUME> -> Replicas
SOURCE_NODE replica -> Delete/Evict
```

After this step only the target-node replica should remain.

### 5.8 Restore Replica Count

Restore the original replica count. In this setup it is usually `1`:

```bash
kubectl patch volumes.longhorn.io -n longhorn-system "$VOLUME" \
  --type=merge \
  -p "{\"spec\":{\"numberOfReplicas\":${ORIGINAL_REPLICAS}}}"
```

Do not set the replica count to `0`. The goal is to leave one healthy replica on
the new node.

### 5.9 Detach The Manual Attachment

Longhorn UI:

```text
Volume -> <VOLUME> -> Detach
```

Why detach after the move?

```text
detach -> releases the manual attachment
       -> lets Kubernetes/CSI attach the volume normally when the workload starts
```

### 5.10 Start The Workload

```bash
kubectl scale sts -n "$NAMESPACE" "$STATEFULSET" --replicas=1
```

Check the workload and volume:

```bash
kubectl get pods -n "$NAMESPACE" -o wide
kubectl get pvc -n "$NAMESPACE"
kubectl get volumes.longhorn.io -n longhorn-system "$VOLUME"
kubectl get replicas.longhorn.io -n longhorn-system -l longhornvolume="$VOLUME" -o wide
```

### 5.11 Restore Temporary Settings

Longhorn UI:

```text
Node -> SOURCE_NODE -> Edit Disk -> Allow Scheduling = true
Node -> other nodes -> Edit Disk -> Allow Scheduling = true
```

Restore the minimal available percentage:

```bash
kubectl patch setting.longhorn.io -n longhorn-system \
  storage-minimal-available-percentage \
  --type=merge \
  -p '{"value":"25"}'
```

If Argo CD reconciliation was paused in Section 5.0, restore it after the
workload and volume are healthy. Restore the root application first, followed
by the parent and child applications:

```bash
kubectl patch application root-app-helm -n argocd --type=merge \
  -p '{"spec":{"syncPolicy":{"automated":{"prune":true,"selfHeal":true}}}}'

kubectl patch application helm-production -n argocd --type=merge \
  -p '{"spec":{"syncPolicy":{"automated":{"prune":true,"selfHeal":true}}}}'
kubectl patch application helm-production-user-service -n argocd --type=merge \
  -p '{"spec":{"syncPolicy":{"automated":{"prune":true,"selfHeal":true}}}}'
```

Confirm that the relevant applications are `Synced` and `Healthy`:

```bash
kubectl get applications.argoproj.io -n argocd
```

---

## 6. Quick Decision Tree

```text
Volume detached but healthy:
  Attach -> replica count 2 -> rebuild -> delete old replica -> restore replica count

Volume detached/faulted and single-replica:
  scale workload to 0 -> make disk schedulable -> salvage -> attach -> rebuild -> delete old replica

Salvage says disk is unschedulable:
  free disk space or temporarily lower storage-minimal-available-percentage

Longhorn UI workload name looks like a Jenkins branch:
  treat it as last-used workload information; identify the volume by PVC name
```
