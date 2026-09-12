# T1110 - Brute Force (sudo authentication)

## MITRE ATT&CK
[T1110 - Brute Force](https://attack.mitre.org/techniques/T1110/)

## Data source
- Index: `main`
- Sourcetype: `journald` (journalctl-identifier = sudo)

## Detection logic
Matches `sudo's` own summary line ("N incorrect password attempts"), emitted once per session with the final count already computed.

## SPL Query
index=main "incorrect password attempts"
| rex field=_raw "(?<user>[\w-]+)\s*:\s*(?<attempts>\d+) incorrect password attempts"
| where attempts >= 3

## Configuração do Alert
- Schedule: cron */15 * * * *, 15-minute search window
- Trigger: Number of Results > 0
- Throttle: suppress for 30 minutes, based on `user`
- Action: Add to Triggered Alerts

## Detection iteration notes
Initial version matched individual PAM failure messages, but PAM emits inconsistent formats across failure types, causing under-counting. Revised to match sudo's own summary line instead, more reliable, immune to PAM message format variance.

## Status
Tested end-to-end in production-like conditions (not just manual search): triggered for real on 2026-08-25 00:00:02, confirmed via both the Triggered Alerts UI and the internal scheduler log (`index=_internal sourcetype=scheduler`), with result_count=1 and throttling (`suppressed=1`) correctly engaging on the following run.

# Known limitations
- Local host lockout (pam_faillock) already mitigates the attack; this detection provides visibility, not prevention.
- No SSH brute-force coverage yet, this host does not run sshd.