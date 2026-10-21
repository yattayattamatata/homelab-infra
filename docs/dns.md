# DNS runbook

My notes on how names like `jellyfin.travel.yattaya.xyz` work, how I test them, and what I do when something breaks. I keep this usable offline since that's when I'll need it most.

**If the internet seems broken on the travel network, the first thing I try is turning ad blocking off (below).** If that fixes it, the blocklist was the problem, not the network.

## How it works

My devices on the travel network ask the Beryl router for names. The Beryl asks Blocky on the travel server first, and only goes to Quad9 (`9.9.9.9`) if Blocky doesn't answer. Blocky answers the `*.travel` names itself with `10.2.1.15`, so they work without internet. It answers ad sites with `0.0.0.0`, and it looks up everything else online over encrypted DNS.

| Where I am | Who answers `*.travel.yattaya.xyz` | Answer |
|---|---|---|
| Travel network (Beryl) | Blocky | `10.2.1.15` |
| Home network | UniFi DNS record | `10.2.1.15` |
| Anywhere else (phone data, hotel Wi-Fi without the Beryl) | Cloudflare | Tailscale IP (Tailscale has to be on) |

The UniFi record at home always says `10.2.1.15`, even when the travel server isn't home. Nothing at home uses it, so that's fine.

## Where everything is set up

| Piece | What it does | Where I set it up |
|---|---|---|
| Blocky | Local names, ad blocking, encrypted lookups | `roles/blocky/templates/config.yml.j2` |
| Beryl router | Hands out IPs, sends DNS to Blocky first and Quad9 as backup | Router admin page, by hand (steps below) |
| Cloudflare record | `*.travel` → Tailscale IP so I can reach it from anywhere | `base.yml`, updated every run |
| UniFi record | `*.travel` → `10.2.1.15` at home | UniFi Network app, by hand |
| Traefik certificate | HTTPS for all the `*.travel` names, renews by itself | `roles/traefik/templates/traefik.yml.j2` |

## Turning ad blocking off and on

```bash
ssh travel 'docker exec blocky /app/blocky blocking disable'   # off
ssh travel 'docker exec blocky /app/blocky blocking enable'    # on
```

The Blocky tile on Homepage shows if it's on or off ("Ad Blocking: true/false").

## Finding out why a site didn't work

Blocky logs every lookup and why it answered the way it did:

```bash
ssh travel 'docker logs blocky 2>&1 | grep <part-of-site-name> | tail -5'
```

| `response_type` | What it means |
|---|---|
| `CUSTOMDNS` | One of my local names, Blocky answered it |
| `RESOLVED` | Looked up online |
| `CACHED` | Blocky already knew it from earlier |
| `BLOCKED` | On the blocklist. This is the one to look for when a site breaks. |

If a site someone needs is getting blocked, I add it to an allowlist in the Blocky config and run `ansible-playbook playbooks/travel.yml --tags blocky`.

## Testing

From my laptop on the travel network. I go top to bottom, and the first one that fails tells me where the problem is.

```bash
dig jellyfin.travel.yattaya.xyz @10.2.1.1 +short    # through the router  → 10.2.1.15
dig jellyfin.travel.yattaya.xyz @10.2.1.15 +short   # straight to Blocky → 10.2.1.15
dig debian.org @10.2.1.15 +short                    # Blocky looking up online → real IPs
```

From the server itself (works even if the network around it is broken):

```bash
ssh travel 'dig jellyfin.travel.yattaya.xyz @127.0.0.1 +short'
```

At home, testing from my Mac doesn't tell me much. My home network grabs all DNS and answers it itself, no matter which server I ask. I test from the server instead.

## Common problems

**Everything works except one site.** Check the Blocky log for `BLOCKED`. Turn blocking off to be sure.

**No internet on the travel network at all.**
1. Is the Beryl connected to the internet? Check its admin page at `http://10.2.1.1`.
2. Is Blocky running? `ssh travel 'docker ps | grep blocky'`
3. If Blocky is down, the Beryl should switch to Quad9 by itself. If it doesn't, check that Strict order and Ignore resolve file are still on in LuCI (Beryl setup below).

**The `*.travel` names don't load on the travel network, but the internet works.**
- Check that DNS rebinding protection is still off (Beryl setup below). If it's on, the Beryl throws away Blocky's `10.2.1.15` answers.
- Try `dig ... @10.2.1.15`. If that works but `@10.2.1.1` doesn't, the problem is the router.

**The hotel Wi-Fi login page doesn't show up.** (Haven't tried this yet.) Hotels block everything until you log in, including Blocky's lookups. Try logging in from the Beryl's admin page (repeater settings). If that doesn't work, take `10.2.1.15` out of the Beryl's Manual DNS, log in, then put it back.

**Certificate expired or a browser warning.** Traefik renews about 30 days before it expires, but it needs internet for that. Get online and check `ssh travel 'docker logs traefik 2>&1 | grep -iE "certificate|ERR" | tail -5'`.

**Can't reach it by name remotely after a rebuild.** The Tailscale IP probably changed. Run `ansible-playbook playbooks/base.yml` to update Cloudflare.

**Apps can’t reach the internet, but the server can.** Containers can’t use Tailscale’s DNS (100.100.100.100). Docker is set to use Blocky first, then Quad9 (daemon.json in the docker role). Test with ssh travel 'docker exec homepage nslookup debian.org'

## Beryl setup (GL-MT1300)

Firmware 4.3.25. GL.iNet doesn't update this model anymore. All of this is set by hand, so I keep this list matching the router. Everything stays after a reboot.

**GL admin page (`http://10.2.1.1`)**
- Wi-Fi: same name on both bands, WPA2/WPA3 mixed, guest network off
- Network → LAN: router is `10.2.1.1`, DHCP gives out addresses from `.100` up
- Clients: travel server reserved at `10.2.1.15`
- Network → DNS: Manual DNS, `10.2.1.15` first, then `9.9.9.9`
- Network → DNS: DNS rebinding attack protection **off**. If it's on, the Beryl throws away Blocky's `10.2.1.15` answers.
- System → Admin Password: this is the root password for LuCI and SSH. It's not the same as the web login.

**LuCI (System → Advanced Settings), Network → DHCP and DNS**
- Resolv and Hosts Files: Ignore resolve file **on**. The Beryl only uses my DNS servers above, not the hotel's.
- General: Strict order **on**. The Beryl tries Blocky first, and Quad9 only if Blocky doesn't answer.

**Getting the Beryl online:** repeater mode (Internet → Repeater) on the hotel Wi-Fi, or a cable into the WAN port. If the hotel network is also `10.2.1.x`, the Beryl won't work on it, so I use my phone's hotspot instead.

**Testing the Beryl at home:** I connect it to my IoT Wi-Fi in repeater mode. IoT is a different subnet, so it doesn't clash. My home network still grabs DNS, so with Blocky stopped, UniFi answers instead of Quad9 and `*.travel` still says `10.2.1.15`. Anywhere else, it would come back with the Tailscale IP.

**Offline test:** Internet → Repeater → disconnect. The `*.travel` names, Homepage, Jellyfin, and Kiwix all still worked. Tested Oct 19.

**Skipping Blocky completely:** take `10.2.1.15` out of Manual DNS. Everything goes straight to Quad9. No ad blocking, and the `*.travel` names stop working on the travel network, but the internet works.
