# gatus-vendor-status

A self-hosted [Gatus](https://github.com/TwiN/gatus) instance that aggregates
third-party vendor status pages into one dashboard with Slack alerts. Replaces
a paid status-page aggregator for personal use at the cost of a container.

## Quick start

```bash
cp .env.example .env      # add your Slack webhook, or delete the alerting block
docker compose up -d
open http://localhost:8080
```

## How it works

Most SaaS vendors run Atlassian Statuspage, which publishes a JSON summary at
`<status-host>/api/v2/status.json`:

```json
{ "status": { "indicator": "none", "description": "All Systems Operational" } }
```

Gatus fetches that on an interval and evaluates conditions against the response
body, so instead of asking "is status.stripe.com reachable" you ask "does
Stripe's own status page say Stripe is fine."

Indicator values are `none`, `minor`, `major`, and `critical`. Some pages also
use `maintenance`. The shipped config alerts only on `major` and `critical` so
scheduled maintenance stays quiet; switch the conditions to
`[BODY].status.indicator == none` if you want to hear about everything.

## Adding a vendor

Find the vendor's status host, confirm the JSON endpoint responds, then append
four lines to `config/config.yaml`:

```bash
curl -s https://status.stripe.com/api/v2/status.json | jq .status
```

```yaml
  - name: stripe
    group: payments
    url: "https://status.stripe.com/api/v2/status.json"
    interval: 5m
    conditions: *statuspage-conditions
    alerts: *statuspage-alerts
```

If `curl` returns 404, the vendor is not on Atlassian Statuspage. Check for an
RSS or Atom feed, or fall back to a plain reachability check on the status page
itself, which is what the AWS entry in the config does.

## Things worth knowing

- **Config hot-reloads.** Gatus watches the config *directory* and reloads on
  change, which is why the compose file mounts `./config` rather than the file
  directly. Bind-mounting the file alone silently breaks reload
  ([TwiN/gatus#151](https://github.com/TwiN/gatus/issues/151)). An invalid
  config will exit the process by default.
- **No shell in the container.** The image is built `FROM scratch`, so
  `docker exec` and container healthchecks do not work. Use `docker logs gatus`
  to debug.
- **History lives in `data/data.db`** via SQLite and survives restarts. It is
  gitignored.
- **Polling only.** You learn about an outage when the vendor publishes it.
  There is no crowdsourced early-warning signal here.
- **Pin the image tag.** The compose file uses `:latest` to avoid a stale pin;
  swap in a specific version from the
  [releases page](https://github.com/TwiN/gatus/releases) once you're settled.

## Reference

- [Gatus README](https://github.com/TwiN/gatus) — full condition syntax and the
  40+ alerting providers (Discord, ntfy, PagerDuty, Teams, Telegram, webhook).
