# SOC Lab

**Bottom line:** a personal SOC lab that takes a real `sudo`
brute-force attempt from the raw `journald` log all the way to
a T1110 alert firing in Splunk, and from there into an
automation pipeline through a webhook. The detection was tested
with the alert firing for real, not only with a manual search.
`docker compose up` brings up Splunk; the Universal Forwarder
and the alert itself are set up by hand (see Setup).

*Read this in: [Português](./locales/pt-br/README.md)*

## Quick start

The whole lab, in order. Each `celery` and `uvicorn` command goes in its own terminal, started from the repo root.

**First time only**, in the [splunk-alert-response-pipeline](https://github.com/Kanetahk/splunk-alert-response-pipeline)
repo (see its [Setup](https://github.com/Kanetahk/splunk-alert-response-pipeline#setup)):

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
sudo systemctl enable --now valkey
```

The `.env` must be filled in before the first run.

**Every time:**

```bash
# 1. soc-lab/ = Splunk
docker compose up -d

# 2. splunk-alert-response-pipeline/ = Celery worker
source .venv/bin/activate
cd src
celery -A tasks worker --loglevel=info

# 3. splunk-alert-response-pipeline/ = webhook
source .venv/bin/activate
cd src
uvicorn api:app --host 0.0.0.0 --port 8001
```

## Built with

Splunk Enterprise 9.3 · Docker Compose · systemd-journald · auditd · Splunk Universal Forwarder

## Architecture

Splunk Enterprise runs **containerized** (Docker Compose), acting as
Indexer and Search Head. The
**Universal Forwarder runs natively on the host**, not in a container,
it needs direct access to the host's own `systemd-journald` to collect
authentication events, which a container can't see without extra
mounts. The Forwarder ships events to the containerized Indexer over
port 9997.

```
Host (systemd-journald, sudo events)
   │
   ▼
Universal Forwarder (native)
   │  port 9997
   ▼
Splunk Indexer / Search Head (Docker container)
   │  scheduled search: T1110, every 15 minutes
   ▼
Alert fires (3 or more incorrect sudo password attempts)
   │  Webhook action >> http://127.0.0.1:8001/webhook
   ▼
splunk-alert-response-pipeline (FastAPI >> Celery >> DB >> email)
```

## Project structure

```
soc-lab/
├── detections/
│   ├── T1110-sudo-bruteforce.md
│   └── T1110-sudo-bruteforce.pt-br.md>
├── locales/
│   └── pt-br
│       └── README.md
├── .gitignore
├── docker-compose.yml
└── README.md 
```

## Detections

[T1110 - Brute Force (sudo authentication)](./detections/T1110-sudo-bruteforce.md)
— scheduled every 15 minutes, fires when a single `sudo` invocation ends
with 3 or more incorrect password attempts. Its Webhook alert action is
consumed by the companion repo
[splunk-alert-response-pipeline](https://github.com/Kanetahk/splunk-alert-response-pipeline),
which handles enrichment, persistence, and notification. The SPL query,
alert configuration, and detection notes are in the linked file.

## Setup

```bash
docker compose up -d
```

Environment variables required (`.env`, never committed):
```
SPLUNK_PASSWORD=...
SPLUNK_FWD_PASSWORD=...
```

Splunk Web available at `http://127.0.0.1:8000` (`admin` / `SPLUNK_PASSWORD`).

The Universal Forwarder is installed separately, directly on the host
(not part of this repo's containers), download from
[splunk.com](https://www.splunk.com/en_us/download/universal-forwarder.html),
configure the journald modular input, and point it at
`127.0.0.1:9997`.

## Usage

Confirm ingestion is working:
```spl
index=main sourcetype=journald
```

Confirm a specific detection's logic, see the SPL query and full alert
configuration in [detections/T1110-sudo-bruteforce.md](./detections/T1110-sudo-bruteforce.md).

## Log ingestion

Configured a native Splunk Universal Forwarder on the host, using the
journald modular input to collect `sudo` authentication events and
forward them to the containerized Indexer via port 9997.

**Note:** sudo's COMMAND field captures full command-line arguments,
which can include sensitive data (e.g., passwords passed inline). In a
production SIEM, this would require field masking/sensitive data
filtering before indexing.

## Known limitations

- **Hardware-pinned Splunk version.** This host (Intel i3-2328M, Sandy
  Bridge mobile) does not have AES-NI enabled, Intel disabled this
  instruction set on some mobile i3 SKUs of that generation. Splunk
  versions from 9.4/10.x onward require AES-NI/AVX in the KVStore
  (MongoDB) binary, with no way to work around it via environment
  variable. The image is pinned to `splunk/splunk:9.3`, the last
  version line still compatible with this hardware.
- **Container runs in `network_mode: host`.** This removes Docker's
  network isolation for the Splunk container, a deliberate trade,
  made after bridge networking (container >> host webhook) proved
  unreliable in combination with firewalld/nftables on this specific
  host. Acceptable for a single-user lab; not a pattern to carry into
  a shared or production environment.
- **No TLS on Splunk Web or the Forwarder-to-Indexer link.** Traffic
  between the Forwarder and Indexer, and access to Splunk Web itself,
  are unencrypted, fine for `127.0.0.1`-only lab traffic, not for any
  deployment reachable over an untrusted network.
- **Single-node, no HA.** One Indexer, one Search Head, same
  container, sufficient to prove the detection pipeline, not sized
  or architected for production log volume or availability
  requirements.

The alert is not created by `docker compose`. Create it in Splunk Web
from the SPL query and alert configuration in [detections/T1110-sudo-bruteforce.md](./detections/T1110-sudo-bruteforce.md).

Splunk's data and configuration (alerts included) live in two named
volumes, `splunk-data` and `splunk-etc`. `docker compose down` keeps
them; `docker compose down -v` deletes them, along with the alert.

`auditd` also runs on the host, watching reads of `/etc/shadow` and
writes to `/var/spool/cron` and `/etc/systemd/system`. A second
Forwarder input (`monitor:///var/log/audit/audit.log`,
sourcetype=linux_audit`) ships those events to the same Indexer,
and the Forwarder user reads the file through `log_group = splunkread`
in `auditd.conf`. No detection is built on this data yet. The raw
rules also match legitimate processes (`unix_chkpwd`, `systemd-userwork`),
so a real alert will need to filter by process.

## Related projects

- [splunk-alert-response-pipeline](https://github.com/Kanetahk/splunk-alert-response-pipeline)
  the automation pipeline these detections feed into.
- [siem-detection-simulator](https://github.com/Kanetahk/siem-detection-simulator)
  synthetic log generator to test these detections without a live attack