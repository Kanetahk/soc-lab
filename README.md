# SOC Lab

Personal SOC Laboratory (Security Operations Center): Splunk, threat
detection, and response automation.

*Read this in: [Português](./README.pt-br.md)*

## Built with

Splunk Enterprise 9.3 · Docker Compose · systemd-journald · Splunk Universal Forwarder

## Architecture

Splunk Enterprise runs **containerized** (Docker Compose), acting as
Indexer and Search Head. The **Universal Forwarder runs natively on the
host**, not in a container — it needs direct access to the host's own
`systemd-journald` to collect authentication events, which a container
can't see without extra mounts. The Forwarder ships events to the
containerized Indexer over port 9997.

```
Host (systemd-journald)
   │
   ▼
Universal Forwarder (native)
   │  port 9997
   ▼
Splunk Indexer/Search Head (Docker container)
```

## Project structure

```
├── docker-compose.yml
├── detections/
│   └── T1110-sudo-bruteforce.md
└── .env
```

## Detections

[T1110 - Brute Force (sudo authentication)](./detections/T1110-sudo-bruteforce.md)
— triggers a Webhook alert action, consumed by the companion repo
[soar-lab](https://github.com/Kanetahk/soar-lab), which handles
enrichment, persistence, and notification.

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
(not part of this repo's containers) — download from
[splunk.com](https://www.splunk.com/en_us/download/universal-forwarder.html),
configure the journald modular input, and point it at
`127.0.0.1:9997`.

## Usage

Confirm ingestion is working:
```spl
index=main sourcetype=journald
```

Confirm a specific detection's logic — see the SPL query and full alert
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
  Bridge mobile) does not have AES-NI enabled — Intel disabled this
  instruction set on some mobile i3 SKUs of that generation. Splunk
  versions from 9.4/10.x onward require AES-NI/AVX in the KVStore
  (MongoDB) binary, with no way to work around it via environment
  variable. The image is pinned to `splunk/splunk:9.3`, the last
  version line still compatible with this hardware.
- **Container runs in `network_mode: host`.** This removes Docker's
  network isolation for the Splunk container — a deliberate trade,
  made after bridge networking (container → host webhook) proved
  unreliable in combination with firewalld/nftables on this specific
  host. Acceptable for a single-user lab; not a pattern to carry into
  a shared or production environment.
- **No TLS on Splunk Web or the Forwarder-to-Indexer link.** Traffic
  between the Forwarder and Indexer, and access to Splunk Web itself,
  are unencrypted — fine for `127.0.0.1`-only lab traffic, not for any
  deployment reachable over an untrusted network.
- **Single-node, no HA.** One Indexer, one Search Head, same
  container — sufficient to prove the detection pipeline, not sized
  or architected for production log volume or availability
  requirements.