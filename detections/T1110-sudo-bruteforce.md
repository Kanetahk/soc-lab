# T1110 - Brute Force (sudo authentication)

## MITRE ATT&CK
[T1110 - Brute Force](https://attack.mitre.org/techniques/T1110/)

*Read this in: [Português](./detections/T1110-sudo-bruteforce.pt-br.md)*

## Data source
- Index: `main`
- Sourcetype: `journald` (journalctl-identifier = sudo)

## Detection logic
Matches sudo's own summary line ("N incorrect password attempts"),
emitted once per `sudo` invocation with the final count already computed.

Severity is computed in the search itself: `high` at 5 or more
attempts, `medium` otherwise. The field travels to the pipeline
inside the webhook payload's `result`.

## SPL query
```spl
index=main "incorrect password attempts"
| rex field=_raw "(?<user>[\w-]+)\s*:\s*(?<attempts>\d+) incorrect password attempts"
| where attempts >= 3
| eval severity=if(attempts >= 5, "high", "medium")
```

## Alert configuration
- Schedule: cron `*/15 * * * *`, 15-minute search window
- Trigger: Number of Results > 0, trigger for each result
- Throttle: suppress for 30 minutes, based on `user`
- Actions: Add to Triggered Alerts, and Webhook to `http://127.0.0.1:8001/webhook`

## Detection iteration notes
Initial version matched individual PAM failure messages, but PAM
emits inconsistent formats across failure types, causing
under-counting. Revised to match sudo's own summary line instead,
more reliable, immune to PAM message format variance.

## Status
Tested end-to-end in production-like conditions
(not just manual search): triggered for real on 2026-08-25 00:00:02,
confirmed via both the Triggered Alerts UI and the internal scheduler
log (`index=_internal sourcetype=scheduler`), with `result_count=1`
and throttling (`suppressed=1`) correctly engaging on the following run.

The Webhook action was confirmed firing for real on 2026-09-09
(`fired=1`, `alert_actions="webhook"` in the scheduler log), with the
full payload received by the pipeline.

The `severity` field and the pipeline's branching on it were validated
with simulated payloads only. With sudo's default `passwd_tries` of 3,
a real alert is expected to come out as `medium` (see Known limitations).

## Known limitations
- Local host lockout (`pam_faillock`) already mitigates the attack;
  this detection provides visibility, not prevention.
- No SSH brute-force coverage yet, this host does not run sshd.
- The `high` severity (5 or more attempts) is not expected to be reached
  with sudo's default settings: the summary line counts attempts within
  a single `sudo` invocation, and `passwd_tries` defaults to 3 (no
  override is set in `/etc/sudoers` or `/etc/sudoers.d` on this host).
  The branching works end to end with simulated payloads, but a real
  alert currently comes out as `medium`.