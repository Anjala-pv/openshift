# 01 — etcd Backup and Disaster Recovery Restore

**Date:** 9 November 2026
**Cluster:** OpenShift Container Platform 4.22.6, 3-node compact cluster
**Nodes:** master1 `10.10.10.11`, master2 `10.10.10.12`, master3 `10.10.10.13`

Commands run from the **bastion** unless stated otherwise.

## Objective

Take an etcd backup and validate it by performing a full control-plane restore, following the official OpenShift 4.22 procedure.

---

## 1. Pre-checks (bastion)

```bash
oc get nodes
oc get clusterversion
oc get clusteroperators
oc get csr | grep Pending
oc get pods -A | grep -Ev 'Running|Completed'
```

Time sync on all control-plane nodes:

```bash
for ip in 10.10.10.11 10.10.10.12 10.10.10.13; do
  echo "== $ip"; ssh core@$ip 'date; chronyc tracking | grep -E "Reference ID|System time|Leap status"'
done
```

**Result:** all nodes Ready, ClusterVersion 4.22.6 with no errors, all operators Available / not Progressing / not Degraded, no pending CSRs, clocks in sync.

---

## 2. etcd backup

**On master1, through a debug pod:**

```bash
oc debug node/master1
chroot /host
/usr/local/bin/cluster-backup.sh /home/core/assets/backup
ls -lh /home/core/assets/backup
exit
exit
```

**Copy the backup off the node (bastion):**

```bash
ssh core@10.10.10.11 'sudo chown -R core:core /home/core/assets/backup'
mkdir -p ~/etcd-backup-cluster1
scp core@10.10.10.11:/home/core/assets/backup/* ~/etcd-backup-cluster1/
ls -lh ~/etcd-backup-cluster1
```

**Backup files:**

```
snapshot_2026-11-09_111329.db                    90M   etcd database snapshot
static_kuberesources_2026-11-09_111329.tar.gz    81K   control-plane static pod resources
```

---

## 3. Restore preparation

**Validation marker** (created *after* the backup, so it should disappear after the restore):

```bash
oc new-project restore-test
```

**Rollback plan:** Proxmox snapshots of the three control-plane VMs before the restore.

**SSH sessions** to every control-plane node, opened in separate terminals (the API is unavailable during the restore):

```bash
ssh core@10.10.10.11    # master1 (recovery host)
ssh core@10.10.10.12    # master2
ssh core@10.10.10.13    # master3
```

The backup was taken from the same z-stream release (4.22.6) as the cluster, as required.

---

## 4. Restore

**Disable etcd on each control-plane node** (master2, master3, then master1):

```bash
sudo -E /usr/local/bin/disable-etcd.sh
```

**Restore from the backup on the recovery host (master1 only):**

```bash
sudo -E /usr/local/bin/cluster-restore.sh /home/core/assets/backup
```

Output (excerpt):

```
...stopping etcd-pod.yaml
Moving etcd data-dir /var/lib/etcd/member to /var/lib/etcd-backup
starting restore-etcd static pod
SNAPSHOT RESTORE COMPLETED
```

**Wait for the API to respond (bastion):**

```bash
oc get nodes
```

**Temporarily disable the etcd quorum guard (bastion):**

```bash
oc patch etcd/cluster --type=merge -p '{"spec": {"unsupportedConfigOverrides": {"useUnsupportedUnsafeNonHANonProductionUnstableEtcd": true}}}'
```

**Monitor recovery (bastion):**

```bash
oc adm wait-for-stable-cluster
watch -n 20 'oc get co etcd kube-apiserver; oc get pods -n openshift-etcd -l app=etcd'
```

The etcd and kube-apiserver operators rolled out new revisions to each master in turn. master2 and master3 rejoined etcd and synchronised from master1.

**Re-enable the quorum guard (bastion):**

```bash
oc patch etcd/cluster --type=merge -p '{"spec": {"unsupportedConfigOverrides": null}}'
```

**Troubleshooting only — if the etcd rollout stalls:**

```bash
oc patch etcd cluster -p='{"spec": {"forceRedeploymentReason": "recovery-'"$(date --rfc-3339=ns )"'"}}' --type=merge
```

(Not needed in this run.)

---

## 5. Verification (bastion)

```bash
oc get project restore-test
# Error from server (NotFound)  → cluster returned to its state at backup time

oc get nodes
oc get co | grep -v "True        False         False"
oc get pods -n openshift-etcd -l app=etcd
oc rsh -n openshift-etcd etcd-master1 etcdctl member list -w table
```

Final etcd membership:

```
+------------------+---------+---------+--------------------------+--------------------------+------------+
|        ID        | STATUS  |  NAME   |        PEER ADDRS        |       CLIENT ADDRS       | IS LEARNER |
+------------------+---------+---------+--------------------------+--------------------------+------------+
| 27c2e1255075c54a | started | master2 | https://10.10.10.12:2380 | https://10.10.10.12:2379 |      false |
| 59a08dc0aa15c77a | started | master1 | https://10.10.10.11:2380 | https://10.10.10.11:2379 |      false |
| aec2a3dbdc1a7ae1 | started | master3 | https://10.10.10.13:2380 | https://10.10.10.13:2379 |      false |
+------------------+---------+---------+--------------------------+--------------------------+------------+
```

---

## Result

- `restore-test` project removed, confirming the restore to the backup point
- All 3 nodes Ready, all cluster operators stable
- All 3 etcd pods `5/5 Running`, 3 started voting members (quorum restored)
- Quorum guard re-enabled
- Recovery time: about 30 minutes from the start of the restore to a stable cluster

## Reference

- [OpenShift 4.22 — Backing up and restoring etcd data](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/etcd/backing-up-and-restoring-etcd-data)
