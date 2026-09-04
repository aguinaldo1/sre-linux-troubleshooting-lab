# INC-003 — Memory Pressure

## Summary

Controlled incident used to investigate memory pressure in a Linux environment, identify the process responsible for increased memory consumption, and validate memory recovery after mitigation.

## Symptom

The environment experienced a significant reduction in available memory during a controlled memory allocation workload.

## Environment

- Operating system: Linux running under WSL2
- Total memory: 3.8 GiB
- Total swap: 1.0 GiB
- Docker and Kubernetes workloads active

## Baseline

Before the incident, the environment showed:

- Memory used: 920 MiB
- Memory free: 1.6 GiB
- Memory available: 2.7 GiB
- Swap used: 20 MiB

The baseline did not indicate significant memory pressure.

The largest memory consumer observed was `kube-apiserver`, using approximately 258340 KiB RSS.

## Incident Generation

Memory consumption was intentionally increased using Python processes allocating memory through `bytearray`.

The first process allocated approximately 768 MiB.

A second process later allocated another 768 MiB, resulting in approximately 1.5 GiB of controlled memory allocation.

## Evidence Collected

### First Memory Allocation

After approximately 768 MiB was allocated:

- Memory used: 1.6 GiB
- Memory free: 826 MiB
- Memory available: 1.9 GiB
- Swap used: 20 MiB

Process-level investigation identified:

- Process: `python3`
- PID: `57557`
- Memory usage: 20.0%
- RSS: 794880 KiB

The RSS value was consistent with the controlled allocation performed by the Python process.

### Increased Memory Pressure

After introducing a second allocation of approximately 768 MiB:

- Memory used: 2.4 GiB
- Memory free: 121 MiB
- Memory available: 1.2 GiB
- Swap used: 27 MiB

Although free memory dropped significantly, approximately 1.2 GiB remained available.

Swap usage increased only slightly, from 20 MiB to 27 MiB.

## Diagnosis

The controlled workload produced increased memory pressure in the Linux environment.

The reduction in available memory was correlated with Python processes intentionally allocating approximately 1.5 GiB of RAM.

Process-level evidence identified the Python workload as a major memory consumer.

The incident did not reach complete memory exhaustion, severe swapping, or an Out Of Memory condition.

## Mitigation

The Python processes responsible for the controlled allocations were terminated, releasing the allocated memory.

## Validation

After mitigation, memory metrics were collected again.

Post-mitigation observations:

- Memory used: 840 MiB
- Memory free: 1.7 GiB
- Memory available: 2.7 GiB
- Swap used: 27 MiB

Available memory recovered from 1.2 GiB during the incident to 2.7 GiB after mitigation, matching the original baseline.

This indicates that RAM capacity recovered after the controlled workload was removed.

## Lessons Learned

- Low `free` memory alone does not prove memory exhaustion.
- `available` memory provides important context when investigating Linux memory usage.
- Linux may use otherwise idle memory for buffers and cache.
- Process RSS helps identify how much physical memory a process is using.
- Memory consumption should be correlated with process-level evidence before identifying a cause.
- Swap activity should be measured rather than assumed.
- Memory pressure does not necessarily mean an Out Of Memory condition occurred.
- Controlled experiments should avoid unnecessarily exhausting production-like environments.
- Mitigation must be followed by measurement to confirm resource recovery.
