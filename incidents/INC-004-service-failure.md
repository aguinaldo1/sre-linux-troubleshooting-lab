# INC-004 — Service Failure

## Summary

A controlled service failure was simulated in the Java application managed by systemd.

The incident was caused by an invalid Java main class configured in the `ExecStart` directive.

The failure prevented the application from starting and caused systemd to repeatedly attempt automatic restarts.

---

## Symptom

The application became unavailable after a service configuration change.

The systemd service was no longer able to remain in a healthy running state.

---

## Baseline

Before the incident, the service was healthy:

```text
Active: active (running)
