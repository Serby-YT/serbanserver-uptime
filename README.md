# serbanserver-uptime

External uptime monitoring for **serban-photo.com** and **galleries.serban-photo.com**.

## Why this repo exists

Uptime Kuma runs *on* serbanserver, so it catches a container dying but can never
report the box itself being gone — power cut, router dead, ISP down, drive failure.
This repo runs the only check that lives outside the house.

It probes just the public endpoints every 15 minutes. Nextcloud, Immich, Jellyfin and
Umami are Tailscale-only and stay covered by Uptime Kuma.

## Alerts

- **Always:** a failed run triggers GitHub's own email notification.
- **Optional:** set a repo secret `NTFY_TOPIC` to also push to your phone via
  `ntfy.sh`. The self-hosted ntfy on serbanserver is Tailscale-only and cannot be
  reached from GitHub, which is why the public instance is used here.

## Things not to break

- **Keep this repo public.** Scheduled Actions are unmetered on public repos; on a
  private repo the 15-minute schedule (2880 runs/month) exceeds the 2000 free minutes.
  Change the cron to `*/30` if it ever goes private.
- **Workflow permissions must be "Read and write"** (Settings → Actions → General).
  GitHub disables scheduled workflows after 60 days of repository inactivity, so the
  workflow makes a weekly heartbeat commit to stay alive. If that push ever fails, the
  run logs a loud warning rather than failing the job — watch for it.
- **Detection latency is roughly 30 minutes.** GitHub's scheduler is best-effort and
  runs late under load. This is a total-loss detector, not a fine-grained monitor;
  Uptime Kuma remains the fast one for everything else.
