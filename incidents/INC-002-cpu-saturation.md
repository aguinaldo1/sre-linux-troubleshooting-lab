# INC-002 — CPU Saturation

## Summary

Controlled incident used to investigate CPU saturation in a Linux environment and identify the processes responsible for excessive CPU consumption.

## Symptom

The environment experienced CPU resource exhaustion during a controlled CPU-bound workload.

## Environment

- Operating system: Linux running under WSL2
- Logical CPUs: 4

## Baseline

Before the incident, the environment showed:

- CPU idle: 82.8%
- CPU user: 2.2%
- CPU system: 5.4%
- Load average: 0.59, 0.76, 1.02

The baseline did not indicate CPU saturation.

## Incident Generation

CPU load was intentionally generated using multiple `yes` processes with their output redirected to `/dev/null`.

Four CPU-bound processes were executed to create contention across the four logical CPUs available to the environment.

## Evidence Collected

During the incident, CPU metrics showed:

- CPU idle: 0.0%
- CPU user: 13.5%
- CPU system: 51.0%
- CPU nice: 35.4%
- Load average: 5.12, 4.63, 3.05

The following `yes` processes were observed as major CPU consumers:

| PID | CPU Usage |
|---|---:|
| 49852 | 94.7% |
| 50611 | 90.8% |
| 50610 | 90.1% |
| 50609 | 88.2% |

## Diagnosis

CPU saturation was identified during the controlled incident.

The environment had four logical CPUs and reached 0.0% CPU idle while the one-minute load average increased to 5.12.

Four `yes` processes were simultaneously running and each consumed approximately 88% to 95% CPU.

Because these processes were intentionally introduced as CPU-bound workloads, the evidence directly correlates their execution with the CPU saturation observed during the incident.

## Mitigation

The controlled workload was terminated using:

`pkill yes`

The absence of remaining `yes` processes was used to confirm removal of the workload.

## Validation

After mitigation, CPU metrics were collected again.

Post-mitigation observations:

- CPU idle: 80.0%
- CPU user: 1.5%
- CPU system: 3.1%
- Load average: 0.99, 0.99, 2.42

CPU idle recovered from 0.0% during the incident to 80.0% after mitigation.

The one-minute load average also decreased from 5.12 to 0.99.

These observations indicate that CPU capacity recovered after the controlled workload was removed.

## Lessons Learned

- High CPU consumption should be confirmed with metrics rather than assumed from application slowness.
- A single CPU-bound process does not necessarily saturate a multi-core environment.
- CPU idle provides useful evidence of available processing capacity.
- Load average should be interpreted relative to the number of logical CPUs.
- Process-level analysis helps identify the workloads responsible for CPU consumption.
- Mitigation must be followed by measurement to confirm recovery.
- Baseline, incident, and post-mitigation measurements provide stronger evidence than isolated metrics.
