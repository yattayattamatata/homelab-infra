# DNS

How names like `jellyfin.travel.yattaya.xyz` get answered, how to test each piece, and what to do when something breaks. Written to be usable offline.

**First thing to try when "the internet is broken" on the travel network:** turn ad blocking off (below). If that fixes it, the problem is the blocklist, not the network.

## How it fits together

Devices on the travel network ask the Beryl router for names. The router asks Blocky on the travel server first, and only uses Quad9 (`9.9.9.9`) if Blocky doesn't answer. Blocky answers `*.travel.yattaya.xyz` itself with `10.2.1.15` (works offline), answers blocked ad domains with `0.0.0.0`, and looks up everything else over encrypted DNS (Quad9 / Cloudflare).

| Where I am | Who answers `*.travel.yattaya.xyz` | Answer |
|---|---|---|
| Travel network (Beryl) | Blocky | `10.2.1.15` |
| Home network | UniFi DNS record | `10.2.1.15` |
| Anywhere else (phone data, hotel Wi-Fi without the Beryl) | Cloudflare | Tailscale IP (needs Tailscale on) |

Home caveat: the UniFi record always says `10.2.1.15`, even when the travel server is away. Nothing at home depends on it, so that's fine.

## Pieces and where they're configured

| Piece | What it does | Configured in |
|---|---|---|
| Blocky | Local names, ad blocking, encrypted lookups | `roles/blocky/templates/config.yml.j2` |
| Beryl router | Hands out IPs, sends DNS to Blocky first, public DNS as backup | Router admin page (by hand, steps below) |
| Cloudflare record | `*.travel` → Tailscale IP for remote access | `base.yml`, updated every run |
| UniFi record | `*.travel` → `10.2.1.15` at home | UniFi Network app (by hand) |
| Traefik certificate | HTTPS for every `*.travel` name, renewed automatically | `roles/traefik/templates/traefik.yml.j2` |

## Ad blocking off switch

```bash
ssh travel 'docker exec blocky /app/blocky blocking disable'   # off
ssh travel 'docker exec blocky /app/blocky blocking enable'    # on
```

The Homepage dashboard shows the current state on the Blocky tile ("Ad Blocking: true/false").

## Checking why something didn't work

Every lookup is logged with the reason for its answer:

```bash
ssh travel 'docker logs blocky 2>&1 | grep <part-of-site-name> | tail -5'
```

| `response_type` | Meaning |
|---|---|
| `CUSTOMDNS` | Local name, answered by Blocky itself |
| `RESOLVED` | Looked up online |
| `CACHED` | Answered from memory |
| `BLOCKED` | On the blocklist. If a site breaks, this is the line to look for. |

If a site someone needs is blocked, add it to an allowlist in the Blocky config and run `ansible-playbook playbooks/travel.yml --tags blocky`.

## Testing each layer

Run these from a laptop on the travel network. Each one skips the layers above it, so the first one that fails is where the problem is.

```bash
dig jellyfin.travel.yattaya.xyz @10.2.1.1 +short    # through the router  → 10.2.1.15
dig jellyfin.travel.yattaya.xyz @10.2.1.15 +short   # straight to Blocky → 10.2.1.15
dig debian.org @10.2.1.15 +short                    # Blocky looking up online → real IPs
```

From the server itself (works even when the network around it is broken):

```bash
ssh travel 'dig jellyfin.travel.yattaya.xyz @127.0.0.1 +short'
```

At home, testing from my Mac doesn't work. The home network intercepts all DNS and answers it itself, no matter which server I ask. Test from the server instead.

## Common problems

**Everything works except one site.** Check the Blocky log for `BLOCKED`. Turn blocking off to confirm.

**No internet on the travel network at all.**
1. Is the Beryl connected upstream? Check its admin page at `http://10.2.1.1`.
2. Is Blocky running? `ssh travel 'docker ps | grep blocky'`
3. If Blocky is down, the router should fall back to Quad9 by itself. If it doesn't, check that "strict order" is on (Beryl setup below).

**`*.travel` names don't load on the travel network, but the internet works.**
- Check the router's rebind protection allows `travel.yattaya.xyz` (Beryl setup below). Without it, the router throws away Blocky's `10.2.1.15` answers.
- Test with `dig ... @10.2.1.15`. If that works but `@10.2.1.1` doesn't, it's the router.

**Hotel Wi-Fi login page doesn't show up.** (Untested.) Before logging in, hotels block everything except their login page, including Blocky's encrypted lookups. Log in through the Beryl's admin page (repeater settings). If that fails, temporarily remove `10.2.1.15` from the router's DNS forwarding, log in, then put it back.

**Certificate expired / browser warning.** Traefik renews about 30 days before expiry, but it needs internet to do it. Connect to the internet and check `ssh travel 'docker logs traefik 2>&1 | grep -iE "certificate|ERR" | tail -5'`.

**Remote access by name stopped working after a rebuild.** The Tailscale IP probably changed. Run `ansible-playbook playbooks/base.yml` to update Cloudflare.

## Beryl setup (GL-MT1300)

Done by hand in the router's advanced settings (LuCI). Keep this list in sync with the router.

**Network**
- LAN subnet: `10.2.1.0/24`, router at `10.2.1.1`
- DHCP reservation: travel server → `10.2.1.15`

**Network → DHCP and DNS**
- DNS forwardings: `10.2.1.15`, then `9.9.9.9`
- Strict order: **on** (try Blocky first, Quad9 only if Blocky doesn't answer)
- Rebind protection: on, with `travel.yattaya.xyz` in the allowed domains

**Testing the Beryl at home:** connect its internet side to my phone's hotspot, not the home network. Home is also `10.2.1.x`, and the router can't have the same subnet on both sides.

**Bypassing Blocky completely:** remove `10.2.1.15` from DNS forwardings. Everything then goes to Quad9 directly. No ad blocking, and `*.travel` names stop working on the travel network, but the internet works.
