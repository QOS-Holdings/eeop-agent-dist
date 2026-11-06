# eeop-agent-dist

Public distribution of the EEOP Windows agent **test** artifacts, so a dedicated internal test host can fetch
them over plain HTTPS with no credential of any kind and verify them against published hashes.

This repository holds artifacts only. The source lives in the private `eeop-platform` repository, and nothing
here is a product release: these builds exist to prove one controlled enrollment, and then one controlled
heartbeat validation, against the Development environment. **Not for a customer or production system.**

## Why anything is public

The test host must download the agent without holding a GitHub token, an Azure credential or any other
reusable secret — a machine that is about to hold a device private key should carry as few credentials as
possible. Publishing the artifact and pinning its hash achieves that: the download needs no authority, and
integrity comes from the hash rather than from the transport or the hosting account.

Nothing published here contains a secret. The agent binaries hold no token, no key, no credential, no
connection string and no environment-specific configuration; the platform address is supplied on the host at
`configure` time and stored there. The only sensitive material in the enrollment flow — the enrollment token
and the device private key — never leaves the operator's session and the device respectively.

## Contents

| File | Purpose |
| --- | --- |
| `EEOP-Gate4-Prepare.ps1` | Prepares the canonical Windows Server 2025 host for the Gate 4 enrollment, refusing any other host, and stops before enrolling |
| `EEOP-Gate4-Enroll.ps1` | Enrolls that prepared host, then proves the enrolled key with one signed request |
| `EEOP-Gate5-Heartbeat.ps1` | Runs the real heartbeat worker on that enrolled host: first beats, restart, silence past the offline threshold, recovery |
| `eeop-agent-win-x64.zip` | The self-contained `win-x64` agent, published as a release asset |
| `SHA256SUMS.txt` | The pinned hashes the scripts and the operator verify against |

`eeop-agent-win-x64.zip` is attached to the release rather than committed, because a 34 MB binary in Git
history is a cost that never goes away.

All three scripts pin **the same** package hash and the same three binary hashes, and the platform repository
fails CI if they ever disagree — preparation once installed the canonical package while enrollment still
expected the superseded one, which made a correctly prepared host unenrollable at the moment it mattered.

There is **one canonical package**, `fleet2225-a0d1cbf`, and every script uses it: it carries everything the
superseded `fleet2021-3ae41e1` carried — enrollment and signed authentication (Gate 4), the heartbeat worker
(Gate 5), the core, software, Windows servicing, identity and Windows service inventories, performance and
availability telemetry, signed agent-health reports, effective-configuration retrieval and on-demand
collection — and, new to it, agent self-upgrade (the agent learns which release it should run, obtains it and
proves it before touching anything), the installer/enrollment-key/tray flow, device classification sensing
(chassis types and CPUID hypervisor facts), and pending-updates collection behind its own consent gate.
On-demand poll and
execution cycles are published as their own `collection-requests` Agent Health module, so a failing poll cannot
hide behind a successful health self-report or the other way round. It runs as a Windows service installed
under Program Files, so the host that enrolls is byte-for-byte the host that heartbeats.

The earlier `fleet2021-3ae41e1`, `health15-e3d3137`, `mon1112-cfc9c32`, `gatec-d328f14`, `gateb1-ff7a8cd`, `ws3b-524a915`,
`gate5-ede0a4d` and `gate4-fcea97b` packages are superseded — the first predates configuration apply-and-report
and on-demand collection, the second signed agent-health publication, the third the performance and
availability collectors, the fourth the Windows service inventory collector, the fifth the software, servicing
and identity collectors, the sixth the service and the Program Files installation, the seventh the heartbeat
worker as well — and their hashes stay recorded in `SHA256SUMS.txt`, with their releases still published, so the
earlier evidence remains verifiable, not so they can be run again.

`health15-e3d3137` is also what the enrolled hosts still run: publishing a package here, and pinning it, says
what a new preparation, enrollment or fresh installation gets — it upgrades nothing that is already enrolled.
An already-enrolled host is upgraded only under its own separate authorization, and no such upgrade or field
acceptance is claimed by this publication.

The platform must know the `collection-requests` Agent Health module before a host installed from this package
emits it, so server support for it is deployed before this agent is installed anywhere it reports to.

## Canonical validation host

The canonical host is **`EEOP-SRV2025-01`, Windows Server 2025, standalone (not domain-joined) for the first
validation**, since Windows Server is the primary target. Prerequisites: x64, local administrator, Windows
PowerShell 5.1 (present on both Full and Core), outbound 443 to the ingestion host and to `github.com`,
`raw.githubusercontent.com` and `objects.githubusercontent.com`, a synchronised clock, and no pre-existing
`C:\ProgramData\EEOP`. No .NET runtime or SDK is installed and no Azure credential is present on the host.

The preparation script **asserts** those it can, before it downloads or extracts anything: elevation, the
hostname (`-ExpectedHostname` to name a different canonical host, deliberately explicit), an `AMD64`
architecture, a `Windows Server 2025` caption with a server product type — no build number is matched, since
servicing moves it — and a clock within **±120s** of the platform's own, read from the `Date` header of
`/health/live`. That bound is tighter than the ±300s device authentication allows, so drift is found before a
single-use enrollment token is spent on it. Nothing adjusts the clock or the Windows Time service.

The first Gate 4 validation ran on a Windows 11 host, which was deliberately reformatted afterwards. That
evidence stands — enrollment, DPAPI persistence and a signed request were verified on both sides before the
machine was rebuilt — but the host and its private key no longer exist, so nothing can be re-run against it,
and its device id appears nowhere in these scripts.

## Verifying

Everything is pinned twice: the caller pins the script's hash before running it, and the script pins the
package's and the binaries' hashes before extracting or executing anything. A mismatch stops the run; the
script never proceeds on an unverified artifact.

```powershell
(Get-FileHash .\EEOP-Gate4-Prepare.ps1    -Algorithm SHA256).Hash.ToLower()
(Get-FileHash .\EEOP-Gate4-Enroll.ps1     -Algorithm SHA256).Hash.ToLower()
(Get-FileHash .\EEOP-Gate5-Heartbeat.ps1  -Algorithm SHA256).Hash.ToLower()
(Get-FileHash .\eeop-agent-win-x64.zip    -Algorithm SHA256).Hash.ToLower()
```

Verify `eeop-agent.dll` as well as `eeop-agent.exe`: the `.exe` is the generic .NET apphost launcher and
carries none of this project's code, so its hash alone would not detect a substituted payload.

## What the scripts do not do

No script provisions a token, installs a Windows service, collects inventory, or requires a .NET SDK, a .NET
runtime or the Azure CLI. Each writes one sanitized results file and uploads nothing.

The preparation script does not enroll. The enrollment script does enroll, but it cannot fetch a token: the
token is typed at a masked prompt, held only for the duration of one call, and never written to a file, an
argument or the results.

The heartbeat script is the mirror image of that: it **requires** `-ExpectedDeviceId` — there is no default,
and it refuses to start with none, with a malformed one or with an empty one — it refuses to run unless the
host is **already** enrolled as exactly that device, and it never calls `enroll`. The identity check happens
before the agent is started, so a mismatched host runs nothing at all. It runs the agent's real `run` host — the same runtime the
Windows Service hosts — kills it, waits out the platform's offline threshold and starts it again, and leaves
the host with no agent running and no service installed. It cannot see a device's `online` or `offline` state,
because that is the platform's judgment and not the agent's; what it records is the UTC windows the platform's
own history is then read against.

Its results file carries identifiers, states, UTC timestamps, sequence numbers, hash verdicts and counts. The
agent's per-phase log stays on the host, and the script scans it for key, signature, header and token markers
before reporting either way.
