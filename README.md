# promtail-to-alloy

> Migration recipes and ready-to-adapt config templates for moving a logging fleet from **Grafana Promtail** to **Grafana Alloy**.

[Promtail reached end-of-life in 2026](https://grafana.com/docs/loki/latest/send-data/promtail/) and Grafana now points everyone to **Alloy** as the supported agent. This repo collects the component mappings, working `.alloy` templates, and the **operational gotchas you only hit in production** — drawn from migrating a real ~57-host fleet (Debian + Alpine, systemd journal + flat files).

It is config templates and notes, not a tool — copy what you need, swap the placeholders, ship.

> 📖 **The story behind it** — how this played out across the real fleet, including the two *silent* failures (a missing Unix group, and a binary 4× heavier than its predecessor): [**Promtail est mort, vive Alloy**](https://blog.pixelium.win/promtail-est-mort-vive-alloy) (FR).

---

## Why migrate

| | Promtail | Alloy |
|---|---|---|
| Status | EOL (2026) | Actively developed |
| Config | YAML, Promtail-only | Alloy syntax (HCL-like), shared with metrics/traces |
| Scope | Logs only | Logs **+** metrics + traces + OTel in one agent |
| Footprint | ~60 MB RSS | ~190 MB RSS (plan for it — see gotchas) |

If you already run Grafana Agent or want one agent for logs *and* metrics, Alloy consolidates the stack.

## Component mapping

| Promtail concept | Alloy component |
|---|---|
| `scrape_configs: journal:` | [`loki.source.journal`](https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.journal/) |
| `scrape_configs:` static files | [`local.file_match`](https://grafana.com/docs/alloy/latest/reference/components/local/local.file_match/) + [`loki.source.file`](https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.file/) |
| `docker_sd_configs` | [`discovery.docker`](https://grafana.com/docs/alloy/latest/reference/components/discovery/discovery.docker/) + [`loki.source.docker`](https://grafana.com/docs/alloy/latest/reference/components/loki/loki.source.docker/) |
| `relabel_configs` | [`loki.relabel`](https://grafana.com/docs/alloy/latest/reference/components/loki/loki.relabel/) / [`discovery.relabel`](https://grafana.com/docs/alloy/latest/reference/components/discovery/discovery.relabel/) |
| `clients: - url:` | [`loki.write`](https://grafana.com/docs/alloy/latest/reference/components/loki/loki.write/) |

The key mental shift: in Promtail `relabel_configs` lives *inside* the scrape config; in Alloy it is a **separate component** (`loki.relabel`) whose `.rules` export you wire into the source's `relabel_rules` argument. See [`templates/systemd-journal.alloy`](templates/systemd-journal.alloy) for the wiring.

## Templates

| File | Replaces |
|---|---|
| [`templates/systemd-journal.alloy`](templates/systemd-journal.alloy) | Promtail `journal:` scrape with `__journal__systemd_unit` → `unit` relabel |
| [`templates/file-logs.alloy`](templates/file-logs.alloy) | Promtail static file scrape (`/var/log/messages`, `*.log`) |
| [`templates/docker.alloy`](templates/docker.alloy) | Promtail `docker_sd_configs` |

Replace `LOKI_HOST` with your Loki push endpoint, and `/tmp/positions.yaml` with your Promtail `positions.filename` (see gotcha 5). Each template is self-contained (includes its own `loki.write`). Templates load on Alloy v1.21.3.

## Gotchas (the part the docs don't warn you about)

### 1. Usage reporting can DoS your DNS resolver 🔴
Alloy's anonymous usage reporting is **on by default** and contacts `stats.grafana.org`. If that name fails to resolve — air-gapped network, or a DNS blocklist (Hagezi/OISD already list it) returning `NXDOMAIN` — Alloy retries roughly **every 3 seconds per agent with no backoff**. On a fleet that's a DNS query storm (we measured ~900k queries/day across 57 hosts).

**Fix:** disable reporting fleet-wide.
- Debian/systemd: `CUSTOM_ARGS="--disable-reporting"` in `/etc/default/alloy`
- Alpine/OpenRC: `command_args="--disable-reporting"` in `/etc/conf.d/alloy`

> Reported upstream: [grafana/alloy#6474](https://github.com/grafana/alloy/issues/6474) (related: [#2524](https://github.com/grafana/alloy/issues/2524)).
> ✅ **Fixed upstream in Alloy v1.18.0** (2026-07-20): [#6478](https://github.com/grafana/alloy/pull/6478) backs off when usage reporting keeps failing, so the query storm stops on its own. On **v1.17.x and older, the flag above is still the fix** — and disabling reporting outright remains the right call on an air-gapped or blocklisted network.

### 2. The `alloy` user needs the right groups, or you get **zero logs, silently**
- **systemd journal**: `alloy` must be in the `systemd-journal` (and usually `adm`) group, otherwise it reads nothing and logs no error.
- **Flat files on Alpine**: `/var/log/messages` is `root:wheel 0640` — `alloy` must be in `wheel`, or the read fails silently.

### 3. Footprint is ~3× Promtail
The Alloy binary is ~400 MB on disk and RSS sits around ~190 MB (vs ~60 MB for Promtail). On small LXC/containers, **resize before rolling out** — a 1 GB container is tight.

### 4. Alloy adds labels Promtail didn't
Alloy attaches `service_name` and `detected_level` automatically. These are additive (existing dashboards keep working), but worth knowing when you see new labels appear.

### 5. Carry Promtail's read positions over, or Loki gets duplicates 🔴
Alloy keeps its own positions file and does not read Promtail's. Started without one, `loki.source.file` re-reads every matched file **from the beginning**, and `loki.source.journal` replays up to `max_age` of journal: everything Promtail already shipped lands in Loki a second time.

Alloy can convert Promtail's file once, on its first start:
- **files**: `legacy_positions_file = "<Promtail positions.filename>"` in `loki.source.file`. `alloy convert` sets it for you; a hand-written config does not.
- **journal**: a `legacy_position { file = "…", name = "<Promtail job_name>" }` block in `loki.source.journal`. `alloy convert` does **not** generate this one.

Where Promtail kept it: `positions.filename` in its config. The built-in default is `/var/log/positions.yaml`; the sample config many packages ship uses `/tmp/positions.yaml`, which a reboot may have wiped.

The conversion fails quietly in two ways, then Alloy re-reads everything:
- a wrong path or a missing file is a single `level=info` line ("will not convert the legacy positions file");
- a wrong journal `name` converts nothing, **and Alloy still logs `successfully converted legacy positions file`** (seen on v1.21.3).

So don't trust the log line alone. On the first start, read the positions file Alloy wrote under its `--storage.path`: `loki.source.file.<label>/positions.yml` must list your files, and `loki.source.journal.<label>/positions.yml` must hold a `cursor-…` entry. An empty `positions: {}` means the conversion missed. The opposite case is loud: if the file exists but `alloy` cannot read it, the component refuses to start (since v1.19.0, [#6750](https://github.com/grafana/alloy/issues/6750)), so give `alloy` read access first (see gotcha 2).

The conversion only runs while Alloy has no positions file of its own. If Alloy already ran once on the host, it is too late for that host.

### 6. Cut over safely
Stop Promtail **before** Alloy's first start, so the positions Alloy converts are final:
1. Install Alloy with the `legacy_*` settings from gotcha 5, but don't start it yet.
2. Stop Promtail. Its positions file is now frozen.
3. Start Alloy. Check the positions files it wrote (gotcha 5), then check that the host appears in Loki (`label/host/values?since=10m`, below).
4. Only then remove Promtail.

If Alloy misbehaves, stop it and start Promtail again: its positions are still valid, so nothing is lost or sent twice. Running both agents side by side ships every new line twice for as long as the overlap lasts.

## Validate the migration

```bash
# Confirm a host is shipping logs to Loki in the last 10 minutes
curl -G "http://LOKI_HOST:3100/loki/api/v1/label/host/values" \
  --data-urlencode "since=10m"
```

## License

MIT

---

*Born from a real homelab migration — more infrastructure & observability write-ups at [blog.pixelium.win](https://blog.pixelium.win) · [pixelium.win](https://pixelium.win).*
