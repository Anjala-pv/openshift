# OpenShift Day-2 Operations Lab

Hands-on Day-2 operations performed on a self-hosted Red Hat OpenShift cluster, documented as production-style runbooks.

## Lab environment

| Item | Details |
|---|---|
| Platform | Red Hat OpenShift Container Platform 4.22.6 (Kubernetes v1.35.5) |
| Topology | 3-node compact cluster (control-plane + worker roles) |
| OS | Red Hat Enterprise Linux CoreOS 9.8 |
| Infrastructure | Proxmox VE on Intel i9-12900H, 94 GB RAM, NVMe storage |
| Install method | UPI (User-Provisioned Infrastructure) |
| Supporting services | Bastion host with DNS (BIND), DHCP (Kea), HAProxy |
| Cluster network | 10.10.10.0/24, isolated behind the bastion |

## Runbooks

| # | Runbook | Status |
|---|---|---|
| 01 | [etcd backup and disaster recovery restore](01-etcd-backup-restore.md) | ✅ Completed |
| 02 | Application backup and restore with OADP | ⏳ Planned |
| 03 | Cluster upgrade with change plan | ⏳ Planned |
| 04 | GitOps with OpenShift GitOps (Argo CD) | ⏳ Planned |
| 05 | Identity Management (IdM) integration for OpenShift authentication | ⏳ Planned |

## Approach

Each runbook follows a production-style structure:

1. Pre-checks to confirm cluster health
2. A rollback plan before any disruptive change
3. Step-by-step procedure based on official Red Hat documentation
4. Verification with evidence
5. Outcome
