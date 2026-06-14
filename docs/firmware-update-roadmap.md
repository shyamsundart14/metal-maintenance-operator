<!--
SPDX-FileCopyrightText: 2025 SAP SE or an SAP affiliate company and IronCore contributors
SPDX-License-Identifier: Apache-2.0
-->

# Firmware update — overall design & rollout roadmap

**Status:** Roadmap
**Date:** 2026-06-14

This is the top-level plan for extending `metal-maintenance-operator` with
firmware updates. It sequences the work as a series of **incremental slices** that
each add one component type (or NIC family) to the **same** `FirmwareUpdate` CR
and the **same** uniform update mechanism, rather than building everything at once.

**Companion documents:**

- [firmware-update-design.md](firmware-update-design.md) — master design (CRD, controller, discovery, guards).
- [firmware-update-slice-connectx6.md](firmware-update-slice-connectx6.md) — detailed plan for Phase 1.
- [nic-discovery-findings.md](nic-discovery-findings.md) — empirical NIC-naming evidence (live BMC scan).
- [oneview-replacement.md](oneview-replacement.md) — HPE OneView / iLO-Repository path (**future work**, not used here).

---

## Guiding principles (apply to every phase)

These were settled across the design discussion and hold for all phases below:

- **One CRD, per building block.** A single `FirmwareUpdate` CR per building block
  (vSphere cluster); no `*Set` / per-component CRDs. GitOps: Ops edits the CR,
  ArgoCD syncs, the controller remediates.
- **One uniform mechanism: Redfish `SimpleUpdate`** for **all** vendors
  (Dell / HPE / Lenovo). We do **not** build console-replacement engines
  (OneView Install Sets, LXCA policies, OME catalogs) — those remain future work.
- **Discovery, not addressing.** Ops never types a Redfish URI. They give a short
  token (e.g. `cx6dx`); the controller resolves it against `FirmwareInventory.Name`
  and collects the matching entries' `@odata.id` into the `SimpleUpdate` `Targets`.
- **Staged, gated reboot.** Components are staged with `OnReset` apply-time; the
  reboot is governed per-CR by `rebootPolicy` (mirroring metal-operator's
  `ServerMaintenancePolicy`): **`OwnerApproval`** (default) waits for the
  `metal.ironcore.dev/maintenance-approved` label on the host's `Server` — set by a
  human or an automated drain-orchestrator; **`Enforced`** reboots automatically
  once staged. The controller then owns the
  reboot and always boots **from disk** (never PXE; we do not use
  metal-operator's `Maintenance` state).
- **One host at a time** (rolling) across the building block.
- **Safety guards everywhere:** Dell `Current/Installed/Previous` dedup, the
  `Updateable: false` gate, the over-match guard (2+ distinct models → block) and
  under-match guard (0 matches on a NIC-bearing host → block, never silent skip).
- **BIOS is always upgraded first** once BIOS is in scope (Phase 4 onward) — and
  only if it is not already at the desired version.
- **Validate against real data, fail safe on the unknown.** Tokens/selectors are
  validated against captured fixtures; current data is **QA-only**, so a production
  probe precedes production rollout.

---

## Phase order at a glance

| Phase | Adds | Mechanism | Notes |
|---|---|---|---|
| **1** | **NIC — ConnectX-6 Dx** | SimpleUpdate | The foundational slice; builds the whole control loop |
| **2** | **NIC — ConnectX-5 (EN/Ex), then Broadcom** | SimpleUpdate | Token-map + fixture work; no new control flow |
| **3** | **NIC — Intel** | SimpleUpdate | Completes NIC coverage |
| **4** | **+ BIOS** | SimpleUpdate | **BIOS always first**; introduces multi-component ordering |
| **5** | **+ StorageController** | SimpleUpdate | First non-NIC, non-BIOS component |
| **6** | **+ HardDrive** | SimpleUpdate | After the controller it sits behind |
| **7** | **+ PSU** | SimpleUpdate | Completes the five component types |

Each phase is **additive to the same CRD** — later phases extend `spec.components`
and the selector token map; they do not replace earlier work.

---

## Phase 1 — NIC, ConnectX-6 Dx (foundation)

**Goal:** the first end-to-end firmware update, on real hardware, across Dell,
HPE, and Lenovo, for ConnectX-6 Dx NICs only. Detailed in
[firmware-update-slice-connectx6.md](firmware-update-slice-connectx6.md).

This phase builds the **entire reusable machinery** — everything after it is
mostly content (tokens + fixtures), not new control flow:

- The `FirmwareUpdate` CRD (lean: one NIC component).
- Discovery library: `cx6dx → "connectx-6 dx"` vs `FirmwareInventory.Name`, dedup,
  guards, `Updateable` gate.
- The `SimpleUpdate` + `OnReset` update engine + Task polling.
- The controller loop: rolling one-host-at-a-time, the manual reboot gate,
  boot-from-disk, status/conditions.

**Prerequisite spike** (retire the make-or-break unknowns by hand first): confirm
each vendor stages on `OnReset`, and that HPE accepts an individual component via
`SimpleUpdate`. **Definition of done:** the full loop verified on a real host of
all three vendors (atomic — all three must pass).

---

## Phase 2 — NIC, ConnectX-5 (EN/Ex), then Broadcom

**Goal:** broaden NIC coverage to the rest of the Mellanox line and Broadcom.

Mostly **token-map + fixture** additions onto Phase 1's machinery:

- **ConnectX-5:** add `cx5en → "connectx-5 en"` and `cx5ex → "connectx-5 ex"`.
  Note `EN`/`Ex` are *orthogonal axes* (protocol vs PCIe tier), not a clean speed
  split — variant-specific tokens keep them separated (see design §5).
- **Broadcom:** tokens resolve to the **chip number** (`bcm57508 → "57508"`,
  `bcm5720 → "5720"`), which is portable across `BCM57508` (Dell) and
  `Broadcom 57508` (Lenovo). **Watch-out:** some Dell 25G cards have chip-less
  *marketing* names (`Broadcom Adv. Dual 25Gb Ethernet`) — those need a
  name-based token, and the **under-match guard** ensures they are never silently
  skipped.

No new controller logic; the per-NIC update path is identical to Phase 1.

---

## Phase 3 — NIC, Intel

**Goal:** complete NIC coverage.

- Token resolves to the **chip model**: `i350 → "i350"`. **Never use `intel`** as a
  token — it would match HPE's non-NIC `Intelligent Provisioning` /
  `Intelligent Platform Abstraction Data` firmware entries.
- Intel's adapter `Model` may be an opaque OEM SKU (Lenovo `J31979`) while the
  *firmware Name* still says `I350` — another confirmation that matching
  `FirmwareInventory.Name` is correct.

At the end of Phase 3, **all NIC families** in the fleet are covered by the same
mechanism and CR.

---

## Phase 4 — Add BIOS (BIOS always first)

**Goal:** add BIOS to the same `FirmwareUpdate` CR. This is the first phase with
**multiple component types**, so it introduces ordering.

- **BIOS is always upgraded first**, and **only if it is not already at the desired
  version** (idempotent — skip if current == target).
- BIOS needs **no model selector** (empty `Targets`; the BMC infers the component
  from the image).
- BIOS failure on a host **aborts that host** (skip remaining components, mark
  Failed); other hosts continue. (Non-BIOS component failures are recorded but do
  not abort — see design §2b.)
- Ordering rule introduced here and reused by Phases 5–7:
  **BIOS → StorageController → HardDrive → NIC → PSU**, enforced by the controller
  regardless of CR list order.

This phase turns the single-component loop from Phase 1 into the ordered,
multi-component, stage-all-then-one-gated-reboot loop described in design §2.

---

## Phase 5 — Add StorageController

**Goal:** first non-NIC, non-BIOS component.

- Selector resolves against `FirmwareInventory.Name` (token convention per the
  vendor's controller naming — e.g. PERC/RAID model).
- Ordered **after BIOS, before HardDrive** — a storage controller sits between the
  host and its drives, so it is updated before the drives that depend on it.
- A StorageController staging failure does **not** abort the host (only BIOS does);
  it is recorded with a `PartialStaging` condition so it is visible at the reboot
  gate.

---

## Phase 6 — Add HardDrive

**Goal:** drive firmware, after the controller it sits behind.

- Ordered **after StorageController** (drives land into the controller firmware
  they will run under).
- Watch-out: drives can be numerous and slow; per-component sequencing plus
  one-host-at-a-time must keep cluster storage redundancy safe throughout (design
  Risk: HardDrive blast radius).

---

## Phase 7 — Add PSU

**Goal:** complete the five component types.

- Ordered **last** — the most physical-layer component; compute/IO firmware is
  settled first.
- Completes the canonical order `BIOS → StorageController → HardDrive → NIC → PSU`.

---

## What stays constant across all phases

- The **CRD shape** — each phase only adds component entries / tokens.
- The **control loop** — discover → stage (`OnReset`) → gate → reboot (boot-from-disk)
  → verify, one host at a time.
- The **safety guards** — dedup, `Updateable`, over/under-match.
- **`SimpleUpdate` as the sole mechanism** — no console-replacement engines.

## Deferred / future work (explicitly not on this roadmap)

- HPE **iLO Repository / Install Set** path (fuller OneView parity) — §8a,
  oneview-replacement.md.
- **`FirmwareCatalog` / `FirmwareBaseline`** compliance layer (OME/LXCA-style
  baselines and fleet drift dashboards).
- **Production discovery** — all current data is QA-only; run `hack/nicprobe`
  against a production sample before production rollout (design Risk #10).
