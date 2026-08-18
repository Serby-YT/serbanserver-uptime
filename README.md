# serbanserver-uptime

External "is the server alive at all?" check for **serbanserver**, built as a
dead-man's switch.

## Why this exists

Uptime Kuma runs *on* serbanserver, so it catches a container dying but can never
report the box itself being gone — power cut, router dead, ISP down, drive failure.
This repo is the only check that lives outside the house.

## Why it is a heartbeat and not a URL probe

The obvious design — have GitHub probe `serban-photo.com` every few minutes — does
not work on this setup, and it is worth writing down so nobody rebuilds it.

Cloudflare's free **Bot Fight Mode** returns `403` to datacenter IPs, and a GitHub
runner is a datacenter IP. Verified on 2026-08-18: the runner received `403` on all
three public endpoints, while the same URLs returned `200/200/302` from a home
connection. Cloudflare's docs are explicit that Bot Fight Mode
["does not run on the Ruleset Engine"](https://developers.cloudflare.com/bots/get-started/bot-fight-mode/),
so **no WAF Skip rule can whitelist a prober**. The only inbound fixes are turning
off bot protection or paying for Super Bot Fight Mode.

**Do not "fix" the old design by treating 403 as healthy.** Cloudflare serves that
403 at its edge without ever contacting the origin, so the check would report green
with the server switched off at the wall — the exact failure it exists to catch.

So the direction is inverted. Outbound traffic never meets inbound bot protection:

```
serbanserver  --(every 10 min, git force-push)-->  heartbeat branch
GitHub Action --(every 15 min, reads timestamp)-->  alerts if stale
```

## How it works

- **Server:** `/usr/local/sbin/uptime-heartbeat.sh`, run by `uptime-heartbeat.timer`
  every 10 minutes. It builds a throwaway repo and force-pushes a Unix timestamp to
  the `heartbeat` branch. Stateless by design — no working copy to drift, and the
  branch stays at exactly one commit instead of ~144 a day.
- **Here:** `.github/workflows/external-uptime.yml` reads that timestamp every 15
  minutes and fails if it is older than 35 minutes (~3 missed beats, so a single
  blip does not page).

## Alerts

- **Always:** a failed run triggers GitHub's own email notification.
- **Optional:** set repo secret `NTFY_TOPIC` to also push to your phone via
  `ntfy.sh`. The self-hosted ntfy on serbanserver is Tailscale-only and cannot be
  reached from GitHub, which is why the public instance is used. Treat the topic
  name like a password — anyone who knows it can read the alerts.

## Coverage

| Layer | Catches | Blind to |
|---|---|---|
| Uptime Kuma (on the box) | Services dying, **and** public reachability — it probes through Cloudflare, so a dead tunnel shows up | The box itself being gone |
| This repo (outside) | Total loss: power, network, ISP, drive | Anything while the box is alive |

## Things not to break

- **Keep this repo public.** Scheduled Actions are unmetered on public repos.
- **The workflow is read-only** (`permissions: contents: read`). The server's own
  heartbeat pushes are what count as repository activity, which is what stops
  GitHub disabling the schedule after 60 days of inactivity.
- A push failure on the server side (GitHub down, SSH key revoked) looks identical
  to a dead server from here. That is the safe direction to be wrong in.
