# RHCOS 9 to 10 osImageStream Upgrade Test Cases

## Overview

E2e coverage for MCO behavior when a `MachineConfigPool` moves from `rhel-9` to
`rhel-10` via `osImageStream`:

- **UC6A (guard):** runc via `ContainerRuntimeConfig` → blocked with `RenderDegraded`
- **UC5/UC14 (happy path):** crun default runtime → upgrade succeeds

A separate e2e in the same suite covers the **osImageURL** guard path (OCPNODE-4518);
see below — that is **not** UC6A.

## Test Environment

- OpenShift 5.0–5.3 (dual stream cluster)
- Skip on MicroShift, Hypershift, SNO

## Test Implementation

Automated in [`test/extended/node/runc_upgrade_cases.go`](runc_upgrade_cases.go)

- **Suite:** `[Suite:openshift/disruptive-longrunning][sig-node][Serial][Disruptive][OCPFeatureGate:OSStreams] runc RHCOS 10 upgrade guard`
- **Lifecycle:** `ote.Informing()`

### UC6A — runc guard blocks upgrade

- **Case:** `blocks RHCOS 9 to 10 osImageStream upgrade when ContainerRuntimeConfig sets runc default runtime`

Run:

```bash
cd origin && make WHAT=cmd/openshift-tests
./openshift-tests run-test \
  "[Suite:openshift/disruptive-longrunning][sig-node][Serial][Disruptive][OCPFeatureGate:OSStreams] runc RHCOS 10 upgrade guard blocks RHCOS 9 to 10 osImageStream upgrade when ContainerRuntimeConfig sets runc default runtime"
```

### UC5/UC14 — crun happy path

- **Case:** `allows RHCOS 9 to 10 osImageStream upgrade when default runtime is crun`

Run:

```bash
cd origin && make WHAT=cmd/openshift-tests
./openshift-tests run-test \
  "[Suite:openshift/disruptive-longrunning][sig-node][Serial][Disruptive][OCPFeatureGate:OSStreams] runc RHCOS 10 upgrade guard allows RHCOS 9 to 10 osImageStream upgrade when default runtime is crun"
```

### osImageURL guard (OCPNODE-4518) — separate from UC6A

- **Case:** `blocks RHCOS 9 to 10 upgrade when MachineConfig osImageURL targets RHEL 10 and ContainerRuntimeConfig sets runc default runtime`
- MCP must **not** set `spec.osImageStream` — if both `osImageURL` and `osImageStream` are set, render fails with `cannot override MachineConfig osImageURL and set MachineConfigPool spec.osImageStream.name simultaneously` (configuration conflict, **not** the runc guard)
- Expected guard `RenderDegraded` message contains `targets a RHEL 10 OS image where runc is not available`
- On clusters whose `OSImageStream` default is `rhel-10`, the test pins RHCOS 9 via a temporary `osImageURL` MC, then removes it before applying the RHEL 10 URL (only one user `osImageURL` MC at guard time)
- On clusters whose default is `rhel-9`, the pool inherits RHCOS 9 without a baseline MC
- Requires MCO with osImageURL stream-class inspection (OCPNODE-4518 / MCO #6238)

Run:

```bash
cd origin && make WHAT=cmd/openshift-tests
./openshift-tests run-test \
  "[Suite:openshift/disruptive-longrunning][sig-node][Serial][Disruptive][OCPFeatureGate:OSStreams] runc RHCOS 10 upgrade guard blocks RHCOS 9 to 10 upgrade when MachineConfig osImageURL targets RHEL 10 and ContainerRuntimeConfig sets runc default runtime"
```

Suggested CI: `periodic-ci-openshift-release-main-nightly-5.0-e2e-aws-disruptive-longrunning-techpreview-1of2` (with MCO payload)

## Notes

- `RenderDegraded` is the authoritative guard signal; `Degraded` may appear shortly after.
- CO/CVO `Degraded=True` can take ~30 minutes on a stuck pool; the test asserts
  `Upgradeable=False` within 5 minutes and recovers before delayed Degraded propagation.
- Typical runtime: UC6A ~15–30 minutes (optional RHCOS 10 recovery on 5.0 clusters); UC5/UC14 ~15–30 minutes (includes node reboot onto RHCOS 10).
- UC6A MCP is pinned to `rhel-9` at creation so it does not inherit cluster default `rhel-10`.
- osImageURL guard MCP omits `spec.osImageStream` entirely (do not pin `rhel-9` on the pool — that conflicts with `osImageURL`).
- Runtime is configured via **ContainerRuntimeConfig** (supported path), not a hand-crafted MC drop-in.

## References

- [openshift/enhancements#2032](https://github.com/openshift/enhancements/blob/master/enhancements/machine-config/block-runc-on-rhcos10-upgrade.md)
- [OCPSTRAT-3154](https://issues.redhat.com/browse/OCPSTRAT-3154) — runc deprecation warning (separate)
- [OCPNODE-4518](https://issues.redhat.com/browse/OCPNODE-4518) — osImageURL guard (separate from UC6A stream path)
