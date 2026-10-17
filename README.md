# homelab-infra

My homelab, set up with Ansible. I'm using this repo to learn infrastructure as code. The goal is to be able to wipe any machine and rebuild it from these files.

## Machines

**Travel server** (in use): a small mini PC running Debian that I take with me when I travel. It has my movies, shows, and audiobooks, plus offline copies of Wikipedia and other reference sites, so it works without internet. I can reach it from anywhere through Tailscale.

**Cluster** (planned for spring 2027): three Lenovo mini PCs that I'll set up as a Kubernetes cluster with k3s. I want to learn how to run apps that keep working when one machine goes down.

## Status

### Travel server
- [x] Base setup: updates, SSH security, time sync, Tailscale
- [x] Storage: separate drives for apps and media
- [x] Docker
- [x] Kiwix (offline Wikipedia and reference sites)
- [x] Copy selected media from my home server with a simple list
- [ ] Jellyfin: copy settings from my home server, turn on hardware transcoding
- [ ] HTTPS and nice web addresses for each app
- [ ] Local DNS and a dashboard page
- [ ] Rebuild from scratch to prove the repo has everything

### Cluster
- [ ] Debian on all three machines, reusing the travel server's base setup
- [ ] k3s installed with Ansible
- [ ] Storage, HTTPS, and deploying apps from Git
- [ ] Testing and deploying my own projects automatically
- [ ] Monitoring and backups
- [ ] Security hardening

## Layout

```
inventory/            my machines and their settings (secrets are encrypted)
playbooks/            the steps Ansible runs
playbooks/templates/  config files that get filled in for each machine
docs/                 notes and decisions
```

## Common tasks

| Task | Command |
|---|---|
| Apply base setup | `ansible-playbook playbooks/base.yml` |
| Apply travel server setup | `ansible-playbook playbooks/travel.yml` |
| Edit media list | `vedit inventory/host_vars/travel/media.yml` |
| Edit secrets | `vedit inventory/host_vars/travel/vault.yml` |
| Preview media sync | `ansible-playbook playbooks/media-sync.yml --check` |
| Sync media | `ansible-playbook playbooks/media-sync.yml` |

`vedit` is my shell alias for `EDITOR="codium --wait" ansible-vault edit`. It opens encrypted files in VSCodium and encrypts them again when I close the tab.
