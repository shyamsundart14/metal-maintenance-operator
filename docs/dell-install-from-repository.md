<!--
SPDX-FileCopyrightText: 2025 SAP SE or an SAP affiliate company and IronCore contributors
SPDX-License-Identifier: Apache-2.0
-->

# The Dell `InstallFromRepository` firmware approach (PR #170)

**Status:** Analysis / design record
**Date:** 2026-08-17
**Subject:** [ironcore-dev/metal-maintenance-operator #170](https://github.com/ironcore-dev/metal-maintenance-operator/pull/170)
— *"Add `firmwareUpdate` controller and CRD for Dell"* (branch `dellfirmwareupgrade`).

This document records how PR #170 works, the design decisions embedded in it, and
how it compares to the per-component `SimpleUpdate` approach described in
[firmware-update-design.md](firmware-update-design.md). It exists to support design
review — it is not itself a proposal.

> **Inspiration.** The controller mirrors Dell's own reference script
> [`InstallFromRepositoryREDFISH.py`](https://github.com/dell/iDRAC-Redfish-Scripting/blob/master/Redfish%20Python/InstallFromRepositoryREDFISH.py)
> — i.e. it drives the Dell OEM Redfish action
> `DellSoftwareInstallationService.InstallFromRepository`.

---

## 1. What it is, in one line

A **Dell-only** `FirmwareUpdateDell` CRD + controller that points a server's iDRAC
at a **firmware catalog on a network share** and lets the **iDRAC** update **every
applicable component** (BIOS, iDRAC, NICs, PERC/RAID, drives, PSU, CPLD, …). The
controller does **no** per-component discovery or targeting — the BMC does all of
that against the catalog.

This is the **vendor-catalog model** (the same philosophy as Dell OME / HPE
OneView), realized for Dell via Redfish, rather than the per-component
`SimpleUpdate` model.

## 2. The CRD — `FirmwareUpdateDell`

Cluster-scoped, group `system.metal.ironcore.dev`, shortName `fwud`, one per
Server.

**Spec (key fields):**

- `repository` (`RepositorySpec`) — the network share hosting the catalog:
  - `shareType` — `NFS` | `CIFS` | `HTTP` | `HTTPS`
  - `address` — share host/IP (e.g. `downloads.dell.com`)
  - `shareName`, `catalogFile` (default `Catalog.xml`), `workgroup` (CIFS),
    `secretRef` (share creds), `ignoreCertWarning` (HTTPS)
- `applySameVersions` *(bool)* — re-apply packages already at the same version
- `applyDowngradeVersions` *(bool)* — allow downgrades
- `serverRef` (required, **immutable**) — the target Server
- `serverMaintenanceRef` / `serverMaintenancePolicy` — maintenance handshake
- `retryPolicy`

There is **no** per-component selector, **no** per-component version target, and
**no** firmware image URL. The catalog is the single input. `RebootNeeded` is
**not** a spec field — it is **hardcoded `true`** in the controller.

**Status (key fields):**

- `state` — `Pending` | `InProgress` | `Completed` | `Failed`
- `checkJob` — the dry-run job; `updateJob` — the main apply job
- `componentJobs[]` — the per-component iDRAC jobs *spawned by* the apply
- `baselineJobIDs[]` — job IDs present just *before* the apply, used to diff and
  discover which component jobs are new
- `passCount` — bounds the check→apply→recheck convergence loop
- `componentJobsSummary` — total/completed/inProgress/failed tallies (observability
  only)
- `failedAttempts`, `observedGeneration`, `conditions`

## 3. How the controller behaves (state machine)

```
Pending ─► dry-run check ─► InProgress ─► apply ─► track component jobs ─► Completed
             (nothing pending? → Completed directly)                          │
                                                                              ▼
                                                          re-check (drift / convergence)
```

1. **Dry-run check** (`processRepositoryCheck` → `issueRepositoryCheck` /
   `pollRepositoryCheck`): issues a repository check and calls
   `GetRepositoryUpdateList` — *"is anything out of date vs the catalog?"* If
   nothing is pending, it goes straight to `Completed`. This is the
   **idempotency / skip** mechanism — compliance is only knowable via this
   iDRAC-side dry-run.
2. **Apply** (`processInProgress`): requests/waits for `ServerMaintenance`, resets
   the BMC, then calls `InstallFromRepository` and polls the update job.
3. **Track component jobs** (`trackComponentJobs`): iDRAC spawns one job per
   component it updates. The controller finds them by **diffing the live job list
   against `baselineJobIDs`** (the jobs that existed before the apply), and waits
   for all of them to reach a terminal state.
4. **Convergence loop**: a completed pass hands back to `Completed`, whose dry-run
   **re-checks**. Dell frequently reveals *more* pending packages only after a
   prior one applies and reboots, so the controller re-enters `InProgress` and
   applies again. This repeats until a check finds nothing pending, **bounded by
   `MaxRepositoryPasses` (default 5)** — exceeding it marks the run `Failed`.
   `passCount` resets to 0 only when a check confirms convergence.
5. **Drift detection**: because `Completed` is re-checked on subsequent reconciles,
   a host that later falls out of catalog compliance is detected and re-updated.

## 4. Reboot handling

- `RebootNeeded` is **hardcoded `true`** — every run assumes a reboot.
- Reboots go through **maintenance-operator's own `ServerMaintenance`** (group
  `maintenance.metal.ironcore.dev`, distinct from metal-operator's built-in one) —
  the same Pending→InMaintenance→policy-approval handshake used by this repo's
  `BIOSVersion` controller.
- iDRAC itself reboots during `InstallFromRepository`, and the multi-pass loop can
  cause **several reboots** in one run (up to ~`MaxRepositoryPasses`).

## 5. Component coverage — "all components", but the BMC decides

The direct question this PR raises is *"does it cover all components or just
BIOS/BMC?"* — the answer is **all applicable components**, because the **iDRAC**
inventories the host and applies everything the catalog covers. There is no
component list in the CRD or controller; `componentJobs` are simply *observed*
after the fact. This is the defining trait of the catalog model: comprehensive
coverage, zero per-component control.

## 6. Delivery structure (two PRs)

The design is split across two repositories / PRs:

1. **metal-operator**: extend the existing `bmc.FirmwareUpdater` interface with the
   Dell OEM methods (`InstallFirmwareFromRepository`, `GetRepositoryUpdateList`,
   `ListJobs`, `GetJob`) + mock-server support.
2. **metal-maintenance-operator** (this PR): the `FirmwareUpdateDell` CRD +
   controller, consuming those methods.

A temporary `go.mod replace` bridges local development against the in-progress
metal-operator branch and **must be removed before the maintenance-operator PR is
merged**.

## 7. Comparison — `InstallFromRepository` vs. `SimpleUpdate`

This PR and [firmware-update-design.md](firmware-update-design.md) are **opposite
architectures** for the same goal:

| | PR #170 (`InstallFromRepository`) | `SimpleUpdate` design |
|---|---|---|
| Mechanism | Dell OEM catalog action | Redfish `SimpleUpdate` per component |
| **Who discovers** | **the iDRAC** (self-inventory vs catalog) | **our controller** (Redfish `FirmwareInventory`) |
| Components | **all** applicable | the ones we build (NIC/Storage/Drive/PSU/BIOS) |
| Vendors | **Dell only** | Dell + HPE + Lenovo (uniform) |
| Per-component control | **none** (whole catalog) | surgical (`modelSelector`, `Targets`, version-per-component) |
| Version pinning | none — "latest applicable in catalog" | explicit target version per component |
| Reboot | via `ServerMaintenance`, hardcoded | self-owned, boot-from-disk (§7a of the design) |
| New code to write | little (BMC does the work) | discovery + payload + guards |

**Trade in one sentence:** the catalog model buys *all-Dell-components with almost
no discovery code*, at the cost of *Dell-only, coarse (whole-catalog), no
per-component/per-version precision, and reliance on `ServerMaintenance`.*

## 8. Does the catalog model generalize to HPE and Lenovo?

Each vendor has a "update everything from a repository" capability, but the **API
shape differs** — there is no single portable call:

| Vendor | Equivalent | API shape | Fits the Dell pattern? |
|---|---|---|---|
| **Dell** | `DellSoftwareInstallationService.InstallFromRepository` | **one OEM action** (share + `Catalog.xml`) | baseline |
| **Lenovo XCC** | "Update From Repository" via **UXSP / Update Bundle** | push a bundle to `MultipartHttpPushUri` (`/mfwupdate`) → one Job with per-component steps; **or** (AMD) sync XCC to a CIFS/NFS repo of SUP/UXSP metadata + binaries | **Yes, cleanly** — same "hand the BMC a bundle, it self-inventories" philosophy |
| **HPE iLO** | **iLO Repository + Install Set + Task Queue** | **no single call** — upload each Smart Component **with its `.compsig` signature**, build an Install Set, apply via the `ApplyUpdate` task queue (agents: iLO/UEFI/SUM/SUT) | **No** — fundamentally different; *we* assemble the set, and handle `.compsig` |

**Implication:** a `FirmwareUpdateDell`-style controller generalizes cleanly to a
`FirmwareUpdateLenovo` (bundle / repo-sync), but **not** to HPE — iLO has no
install-from-catalog action, so HPE requires meaningfully more controller logic
(component upload, `.compsig` handling, install-set assembly). The Dell controller
is **not** a template HPE fits.

## 9. Questions worth raising in review

**Scope & strategy**
1. Dell-only via the catalog model — what is the plan for HPE and Lenovo? A
   parallel `FirmwareUpdateHPE`/`FirmwareUpdateLenovo`, and are we committing to
   **per-vendor CRDs**?
2. `InstallFromRepository` gives all components but no per-component or per-version
   control. Is *converge-to-catalog* the intended operating model, or do we also
   need surgical single-component / specific-version updates?
3. This lives in group `system` alongside `BIOSVersion`/`BMCVersion`. If both can
   update BIOS on a Dell host, which wins — can they conflict?

**Reboot & production safety**
4. `RebootNeeded` is hardcoded true and it enters `ServerMaintenance`. On a live
   vSphere/ESXi host, does that assert a **network-boot (PXE) override**? What
   guarantees the host boots back **from disk** into ESXi, not a network installer?
5. iDRAC self-reboots during the apply, and multi-pass can cause several reboots.
   Who **drains/evacuates** the ESXi host, and does the loop coordinate with that?
6. Worst-case reboot count in one run (up to ~`MaxRepositoryPasses`) — acceptable
   for a maintenance window?

**Catalog / operational model**
7. Where does the catalog come from — Dell's public `downloads.dell.com` or an
   internally-mirrored/curated one? Who keeps it current and **tested**?
8. No version pinning — you get "latest applicable in the catalog." How do we
   guarantee a **tested, known-good** fleet state (reproducibility) rather than
   "whatever the catalog says today"?

**Failure & convergence**
9. If one component fails but others succeed, is the whole `FirmwareUpdateDell`
   `Failed`, or partial? Can you retry just the failed component?
10. The loop keys off `GetRepositoryUpdateList` returning "nothing pending." Could a
    catalog entry that never applies cleanly (or targets absent hardware) loop to
    `Failed` at pass 5 even though the host is actually fine?
11. `baselineJobIDs` diffing to attribute spawned jobs — is it race-free if the
    iDRAC job queue already holds unrelated jobs?

**Strategic fit**
12. Each vendor's repository model uses a **proprietary catalog** (`Catalog.xml`,
    UXSP metadata, SPP + `.compsig`). Are we committing to three vendor-specific
    catalog integrations, or is there appetite for a **unified layer** (e.g. the
    OCI-baseline idea in [firmware-baseline-oci.md](firmware-baseline-oci.md)) that
    abstracts over them? The two directions do not obviously converge.

## 10. References

- Dell reference script — `InstallFromRepositoryREDFISH.py`:
  <https://github.com/dell/iDRAC-Redfish-Scripting/blob/master/Redfish%20Python/InstallFromRepositoryREDFISH.py>
- Lenovo XCC "Update From Repository" (UXSP): <https://pubs.lenovo.com/xcc3/updating_firmware_repository>
- Lenovo XCC AMD remote-repository sync: <https://pubs.lenovo.com/xcc-amd/updating_firmware_repository>
- HPE iLO 5 Update Service (repository / install sets):
  <https://github.com/HewlettPackard/ilo-rest-api-docs/blob/master/source/includes/_ilo5_updateservicedoc.md>
- HPE Smart Components & `.compsig`: <https://developer.hpe.com/blog/hpe-firmware-updates-part-1-file-types-and-smart-components/>
- Companion: [firmware-update-design.md](firmware-update-design.md) (the SimpleUpdate approach),
  [firmware-baseline-oci.md](firmware-baseline-oci.md) (the OCI-baseline idea).
