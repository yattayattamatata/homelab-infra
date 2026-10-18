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
- [x] Jellyfin, with settings copied from my home server and hardware transcoding
- [x] Kiwix, with the list of offline libraries kept in this repo
- [x] Copy selected media from my home server with a simple list
- [x] HTTPS and nice web addresses for each app (Traefik)
- [x] DNS record kept up to date by Ansible
- [x] Local DNS with light ad blocking that also works offline (Blocky)
- [ ] Travel router setup
- [ ] Dashboard page
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
inventory/   my machines and their settings (secrets are encrypted)
playbooks/   which roles each machine gets, plus one-off jobs like media sync
roles/       one folder per piece of the setup (storage, docker, traefik, blocky, jellyfin, kiwix)
docs/        notes and runbooks
```

## Common tasks

| Task | Command |
|---|---|
| Apply base setup | `ansible-playbook playbooks/base.yml` |
| Apply travel server setup | `ansible-playbook playbooks/travel.yml` |
| Apply just one part | `ansible-playbook playbooks/travel.yml --tags kiwix` |
| Edit media list | `vedit inventory/host_vars/travel/media.yml` |
| Edit secrets | `vedit inventory/host_vars/travel/vault.yml` |
| Preview media sync | `ansible-playbook playbooks/media-sync.yml --check` |
| Sync media | `ansible-playbook playbooks/media-sync.yml` |
| Turn ad blocking off | `ssh travel 'docker exec blocky /app/blocky blocking disable'` |
| Turn ad blocking on | `ssh travel 'docker exec blocky /app/blocky blocking enable'` |
| Check why a site is blocked | `ssh travel 'docker logs blocky 2>&1 \| grep <site>'` |

`vedit` is my shell alias for `EDITOR="codium --wait" ansible-vault edit`. It opens encrypted files in VSCodium and encrypts them again when I close the tab.

## How names work

Each app has its own address, like `jellyfin.travel.yattaya.xyz`, with a real HTTPS certificate.

- **On the travel network:** Blocky on the server answers with the server's local address, even with no internet.
- **At home:** my home router points the names at the server's local address.
- **Anywhere else:** Cloudflare points the names at the server's Tailscale address.

If something with DNS breaks, the first step is turning ad blocking off (see above).
