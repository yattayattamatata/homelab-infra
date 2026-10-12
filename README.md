# homelab-infra

Ansible playbooks and notes for my 3-node k3s cluster.

The hardware is three Lenovo M920q Tiny PCs (i5-9500T, 16 GB RAM, NVMe for the OS, SATA SSD for storage) running Debian on their own VLAN. All three nodes run as k3s servers with embedded etcd, so the cluster stays up if any one node goes down.

The goal is to be able to wipe the nodes and rebuild everything from this repo. Apps running on the cluster will live in a separate GitOps repo once I get to that point.

## Status

Waiting on hardware. Network is being set up.

## Plan

- [ ] Debian on all nodes, base config with Ansible
- [ ] k3s HA cluster, installed with Ansible
- [ ] Longhorn, MetalLB, cert-manager, ArgoCD
- [ ] CI/CD for my own projects
- [ ] Monitoring and backups
- [ ] Hardening

## Layout

```
inventory/   hosts and variables
playbooks/   what gets run
roles/       reusable pieces
docs/        decisions and notes
```

No secrets are committed here. Anything sensitive is encrypted or gitignored.