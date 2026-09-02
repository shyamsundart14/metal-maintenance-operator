<!--
SPDX-FileCopyrightText: 2025 SAP SE or an SAP affiliate company and IronCore contributors
SPDX-License-Identifier: Apache-2.0
-->

# Replacing HPE OneView (firmware path) with the operator

**Status:** Discussion / design notes — **FUTURE WORK, not near-term**
**Date:** 2026-06-12

> **Not in the near-term plan.** The current implementation
> ([firmware-update-slice-connectx6.md](firmware-update-slice-connectx6.md)) updates
> HPE firmware via **`SimpleUpdate`**, the same mechanism as Dell and Lenovo — *not*
> the iLO Repository / Install Set path described here. This document is the route
> to *fuller* OneView parity (install sets, ordered baselines) and remains valid as
> a future enhancement; it does not describe how HPE firmware is updated today.
**Scope:** How `metal-maintenance-operator` can replace HPE OneView's *firmware
distribution* path for HPE ProLiant servers. BIOS **settings** (the other half of
a Server Profile) are explicitly out of scope here — they are a separate Redfish
API concern, not a firmware-distribution problem.

> Companion to [firmware-update-design.md](firmware-update-design.md) (see §8a) and
> [nic-discovery-findings.md](nic-discovery-findings.md). This file captures the
> OneView-specific reasoning in one place.

---

## 1. How OneView does firmware today

The operator-facing OneView workflow for firmware is:

1. **Upload the SPP ISO** (Service Pack for ProLiant) to OneView.
2. Create a **Server Profile Template (SPT)** that references the SPP (plus BIOS
   settings, etc.).
3. Spawn a **Server Profile** from the SPT and **associate it to a physical
   (HPE) server**. Applying the profile drives the firmware to baseline.

## 2. The key realization — OneView has no firmware engine of its own

OneView does **not** contain a proprietary firmware-flashing engine. Under the
hood it:

- **Ingests the SPP once** — cracks the ISO open, reads its manifest, and stores
  the individual firmware **components** in its own repository.
- **Per server, drives that server's iLO** — specifically the iLO's
  **Repository + Install Set + Update Task Queue** (native iLO 5 Redfish
  capabilities) — to apply the components computed against that server's
  inventory.

**So replacing OneView's firmware path = driving the same iLO mechanism
ourselves, removing the OneView middleman.** We are not reinventing a firmware
engine; iLO already is one.

## 3. The ISO-mount question — and why it is a non-problem

A natural concern: *"an ISO can only be mounted to one endpoint at a time — how
does OneView fan an SPP out to many servers?"*

The answer: **it doesn't mount the ISO to the servers at all.** Two distinct
mechanisms get loosely called "ISO":

| Mechanism | What it is | Mount contention? |
|---|---|---|
| **Virtual media boot** | iLO attaches a remote ISO as a virtual CD; host boots it (offline flashing) | **Yes** — one image per virtual-media slot per iLO |
| **Repository / component distribution** | SPP treated as a *container of components + manifest*; individual components uploaded/applied per iLO (online or staged) | **No** — nothing is mounted |

OneView's profile-driven firmware path uses the **second** mechanism. The SPP ISO
is extracted **once** (centrally), and per-server work is component upload +
Install Set — no per-server ISO mount. The "one endpoint at a time" constraint
only applies to the virtual-media-boot path, which this design does **not** use.

**Consequence for us:** there is no mount-contention problem to solve. We extract
the SPP once into components and drive each iLO's repository independently.

## 4. The iLO Repository mechanism (what we drive)

iLO 5's `UpdateService` extends base Redfish with HPE OEM resources:

- **iLO Repository** — persistent flash storage *on each iLO* holding firmware
  components (`.fwpkg`, `.bin`, …) plus their **required `.compsig` signature
  files**.
- **Update Task Queue** — an ordered queue of operations iLO processes one at a
  time. The agent that actually applies a task may be iLO, UEFI BIOS, or SUM/SUT.
- **Install Set** — a *named, ordered sequence* of `ApplyUpdate` tasks. `Invoke`
  appends the whole sequence to the queue. **This is the OneView SPT analog.**
- **Maintenance Window** — an ISO-8601 time window attached to an install set;
  iLO defers execution until then.

### Task / install-set item payload (HPE-documented)

```jsonc
{ "Name": "Update-460148",
  "Command": "ApplyUpdate",
  "Filename": "HPE_NIC_...fwpkg",      // as it appears in the iLO Repository
  "UpdatableBy": ["Uefi"],             // Bmc | Uefi | RuntimeAgent
  "Targets": ["/redfish/v1/UpdateService/FirmwareInventory/29/"] }
```

### `UpdatableBy` is the native staging knob

- **`Bmc`** — iLO performs the flash itself, often **without a host reboot**.
- **`Uefi`** — the UEFI BIOS performs the flash **on the next reboot** (staged).
- **`RuntimeAgent`** — an OS-resident agent (SUT) applies it.

Because most of the components in scope require a reboot to activate, they are
`Uefi`-applied — i.e. **iLO stages them and the activating reboot does the work.**
This maps directly onto the operator's existing **stage → manual reboot gate →
verify** model.

### Invoking an Install Set

```
POST /redfish/v1/updateservice/installsets/{id}/Actions/HpeComponentInstallSet.Invoke
```

Good practice: **clear the task queue before invoking** (iLO does not auto-clear,
to preserve prior task results).

## 5. OneView → operator mapping

| OneView concept | Operator equivalent |
|---|---|
| SPP ISO (uploaded once) | SPP **cracked once** into per-model component folders (§7) |
| Server Profile Template (SPT) | **Install Set** — the ordered component sequence |
| Apply Server Profile to a server | **`Invoke` the Install Set** on that host's iLO |
| Scheduled activation | **Maintenance Window**, or the operator's gated reboot |
| BIOS settings in the SPT | Out of scope here — separate Redfish APIs |

This is a *faithful* replacement of the firmware path, with two advantages over
OneView: it is **GitOps-native** (the desired baseline is a git-versioned,
PR-reviewed, ArgoCD-synced resource) and it has **native staging** via Install
Sets that fits the operator's single-reboot gate.

## 6. Where this fits the operator architecture — a pluggable strategy

`SimpleUpdate` is correct for Dell and Lenovo, but HPE is better served by the
iLO Repository path. The design therefore adopts a **pluggable `UpdateStrategy`**
behind the common controller (also recorded in design §8a):

```
FirmwareUpdate controller   (common: discover, stage, manual reboot gate, verify)
        │
        ▼  UpdateStrategy
        ├── SimpleUpdateStrategy   (Dell, Lenovo) — per-component SimpleUpdate + OnReset
        └── ILORepositoryStrategy  (HPE)          — iLO Repository + Install Set + Invoke
```

The common controller logic (rolling one-host-at-a-time, the manual reboot gate,
boot-from-disk, status/conditions) is unchanged; only the *apply + stage*
mechanics differ per vendor.

## 7. SPP repository layout — one folder per model + generation

Firmware components are **not** portable across generations even within a server
line — a DL360 Gen10 and a DL360 Gen11 differ in System ROM, iLO generation, and
supported cards, so their SPP components differ. The cracked SPP is therefore laid
out keyed by **exact model + generation**, each folder being that hardware's
component set (the source for its Install Set):

```
fw-repo/hpe/
  DL360Gen10/   DL360Gen11/
  DL380Gen11/
  DL560Gen10/   DL560Gen11/
  DL345Gen11/
  DL320Gen11/
  …
```

The controller resolves a host → its model + generation (from the argora
`kubernetes.metal.cloud.sap/type` label) → that folder → that model's Install Set.

Because the **iLO Repository is finite flash**, it cannot hold a whole SPP — we
upload only the components being applied for that model, which is exactly what the
per-model folders provide.

## 8. Open questions / verify against real hardware

These were reconstructed from HPE's published iLO 5 Redfish documentation;
WebFetch of the primary docs was unavailable during the discussion, so each
should be confirmed against a real iLO before implementation:

1. **Component upload mechanics** — the exact iLO Repository upload URI, the
   multipart format, and that the `.compsig` must accompany each component.
2. **gofish coverage** — `HpeComponentInstallSet` / `HpeComponentUpdateTask` are
   **HPE OEM extensions**; gofish likely has no typed support, so we POST raw JSON
   to the OEM action URIs (same discovered-action pattern as design §6).
3. **iLO Repository capacity** — confirm limits; informs how many components per
   model can be staged at once.
4. **Install Set vs. external repository** — distinguish the documented
   *upload-then-Invoke* Install Set / Task Queue path (what we target) from a pure
   point-at-external-repository flow.
5. **SPP manifest format** — what the "crack the SPP" step parses to derive the
   per-model component list and the dependency-ordered Install Set sequence
   (this is where the *vendor-supplied baseline* is consumed).
6. **Offline-only components** — confirm whether any in-scope component can *only*
   be updated via virtual-media boot (the one genuine mount-contention case);
   none expected, but it would force a serial virtual-media path for those.

## 9. Relationship to the broader "console replacement" idea

OneView, Dell OME, and Lenovo LXCA share three primitives: a **firmware
repository**, a **catalog/manifest** (component → correct firmware + version +
dependencies), and a **compliance/baseline engine** (drift → remediate).
Kubernetes already provides the compliance loop (reconciliation) and a GitOps
audit trail for free. A future `FirmwareCatalog` + `FirmwareBaseline` layer could
generalize this across vendors, with the **vendor supplying the baseline** (HPE
SPP manifest, Dell catalog, Lenovo UpdateXpress) so we consume tested
combinations rather than authoring them. That broader layer is **not** part of
the current design; this document covers the HPE/OneView firmware path only.

## 10. References

- iLO 5 Software/Firmware Update Service (HPE):
  <https://github.com/HewlettPackard/ilo-rest-api-docs/blob/master/source/includes/_ilo5_updateservicedoc.md>
- iLO 5 Redfish API reference: <https://hewlettpackard.github.io/ilo-rest-api-docs/ilo5/>
- HPE iLO update service (supplement): <https://servermanagementportal.ext.hpe.com/docs/redfishservices/ilos/supplementdocuments/updateservice>
- iLO firmware update blog series: <https://servermanagementportal.ext.hpe.com/docs/references_and_material/blogposts/firmware_updates/part2/firmware_update_part2.md>
- Companion design: [firmware-update-design.md](firmware-update-design.md) (§8a)
