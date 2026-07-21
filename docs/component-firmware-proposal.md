<!--
SPDX-FileCopyrightText: 2025 SAP SE or an SAP affiliate company and IronCore contributors
SPDX-License-Identifier: Apache-2.0
-->

# Proposal: extend firmware maintenance to NIC, StorageController, HardDrive and PSU

**Status:** Proposal / discussion starter
**Date:** 2026-06-14
**Audience:** `metal-operator` maintainers, the `MaintenancePlan` feature authors, and
the author of the `feature/nic-firmware-update` branch.

**Companion documents:**
[firmware-update-design.md](firmware-update-design.md) ·
[nic-discovery-findings.md](nic-discovery-findings.md) ·
[firmware-update-slice-connectx6.md](firmware-update-slice-connectx6.md)

---

## 1. The gap

`metal-operator` today has firmware/settings CRDs for **BMC and BIOS only**:
`BMCVersion`, `BMCSettings`, `BIOSVersion`, `BIOSSettings`.

The `MaintenancePlan` / `MaintenancePlanRun` feature (branch `maintenance-plan` of
`metal-maintenance-operator`) is a **fleet-scale orchestrator** on top of those: it
groups servers by BMC, creates one `MaintenancePlanRun` per BMC, and executes an
ordered list of stages. Each stage simply **creates a metal-operator child CR and
polls its `.Status.State`**:

```
MaintenancePlan  ──(one run per BMC)──►  MaintenancePlanRun
                                             │  per stage: Create + poll
                                             ▼
              metal-operator: BMCVersion | BMCSettings | BIOSVersion | BIOSSettings
                                             │
                                             ▼  the actual work
                          Redfish SimpleUpdate, ServerMaintenance, Task polling
```

`MaintenancePlanRun` performs **no** firmware work itself — a repo-wide scan of the
feature finds no `SimpleUpdate`, no reboot logic, no Jobs/exec/scripts. Its
`StageKind` enum is exactly the set of metal-operator CRDs that exist.

**Consequence:** the orchestrator cannot cover **NIC, StorageController, HardDrive
or PSU**, because there are no CRDs for them to create. Extending coverage is
therefore *not* primarily a MaintenancePlan change.

## 2. The ask, in dependency order

Adding the four component types requires **two changes, in this order**:

| # | Where | Change | Effort |
|---|---|---|---|
| **1** | **`metal-operator`** | New CRDs + controllers: `NICVersion` (then `StorageControllerVersion`, `DriveVersion`, `PSUVersion`) that perform the Redfish work | **Substantial** — this is where all the real complexity lives |
| **2** | `metal-maintenance-operator` (MaintenancePlan) | Add the kinds to `StageKind` + `StageTemplate`, plus the matching `buildObject` / `checkCRStatus` cases | **Small** — mechanical, follows the existing BIOS/BMC pattern |

Step 2 is impossible without step 1: the orchestrator has nothing to create or poll
until the CRD exists. **The substantive request is to `metal-operator`.**

Encouragingly, step 1 is already in flight: the `feature/nic-firmware-update` branch
prototypes a `NICVersion` CRD + controller.

## 3. What a `NICVersion` controller must solve (evidence-backed)

These requirements come from a **live read-only Redfish probe of 15 Dell / HPE /
Lenovo servers** (see [nic-discovery-findings.md](nic-discovery-findings.md)); they
are empirical, not theoretical.

### 3.1 Discovery — match `FirmwareInventory.Name`, not `NetworkAdapter.Model`

`Chassis.NetworkAdapters().Model` is **not** a usable cross-vendor key. The *same*
Mellanox ConnectX-6 Dx reports as:

| Vendor | `NetworkAdapter.Model` | `FirmwareInventory.Name` |
|---|---|---|
| Dell | `ConnectX-6 DX 2-port 100GbE QSFP56 PCIe Adapter` | `Mellanox ConnectX-6 Dx Dual Port 100 GbE QSFP56 Adapter` |
| HPE | `P25962-001 / Mellanox MCX623106AS-CDAT …` | `ConnectX-6 Dx 100GE 2P NIC` |
| Lenovo | `CX623106AN-CDAT` | `Firmware:DEVICE-Mellanox ConnectX-6 Dx 100GbE QSFP56 2-port …` |

The `Model` field is free-form, OEM-rebranded, sometimes empty, and each vendor puts
the identifier in a different field. **`FirmwareInventory.Name` consistently embeds
the silicon family** (`ConnectX-6 Dx` appears in all three), so a case-insensitive
substring match works there — and the matched entry's `@odata.id` is exactly what
`SimpleUpdate` wants in `Targets`.

> Note: case-insensitivity is **mandatory** — Dell writes `DX`, HPE/Lenovo write `Dx`.

### 3.2 Selector granularity must include the variant

A family token is unsafe. ConnectX-6 ships as **Lx** (≤50GbE) and **Dx** (≤100GbE) —
different cards, different firmware. The fleet already carries **both**
`ConnectX-5 EN` and `ConnectX-5 Ex`. A `cx6`-level token would match across variants.

Suggested approach: Ops supplies a short **variant-specific token**, resolved in code
to the distinguishing substring:

```go
"cx6dx" → "connectx-6 dx"      "cx6lx" → "connectx-6 lx"
"cx5en" → "connectx-5 en"      "cx5ex" → "connectx-5 ex"
"bcm57508" → "57508"           "i350"  → "i350"
```

Token conventions differ by silicon vendor: **Mellanox** = family+variant,
**Broadcom** = bare chip number (portable across Dell `BCM57508` and Lenovo
`Broadcom 57508`), **Intel** = chip model. Do **not** use `intel` as a token — it
matches HPE's non-NIC `Intelligent Provisioning` firmware entries.

### 3.3 Three safety gates the probe proved necessary

1. **`Updateable` gate.** `SoftwareInventory.Updateable: false` means reporting-only
   firmware. Observed live: an **HPE DL560 Gen11 reports its ConnectX-6 NICs as
   `Updateable: false`**. Such entries must be excluded from `Targets` and surfaced,
   never sent to `SimpleUpdate`.
2. **Dell rollback-slot dedup.** Dell exposes each component **three times** —
   `Current-` / `Installed-` / `Previous-` — at two different versions. Discovery must
   collapse to the active entry and ignore `Previous-`, or it counts one card three
   times at conflicting versions. (HPE/Lenovo use single IDs and are unaffected.)
3. **Over-match / under-match guards.** If a token matches **2+ distinct model names**
   on a host → block, require a more specific token. If it matches **zero** entries on
   a NIC-bearing host → surface it, never silently skip. The under-match case is real:
   some Dell 25G cards are named `Broadcom Adv. Dual 25Gb Ethernet` with **no chip
   number**, so a chip-number token finds nothing.

> **Do not key on OPN/part-number suffixes** (e.g. `-CDAT`). Verified against NVIDIA's
> catalog: the suffix encodes speed + PCIe + bracket (`C`=100GbE, `A`=25GbE, `G`=50GbE,
> `V`=200GbE). `CDAT ⟹ Dx` is true, but `Dx ⟹ CDAT` is **false** (Dx also ships as
> `VDAT`/200GbE), so it silently under-matches — and it lives on the unreliable
> `Model` field.

### 3.4 Payload construction

Per matched card, one `SimpleUpdate`:

```jsonc
{
  "ImageURI": "https://fw-repo/.../cx6dx-<version>.bin",
  "TransferProtocol": "HTTPS",
  "@Redfish.OperationApplyTime": "OnReset",     // vendor-dependent; see §4
  "Targets": [ "<@odata.id of the matched FirmwareInventory entry>" ]
}
```

`Targets` accepts an array; issuing **one call per card** (rather than one batched
call per host) gives per-card task tracking and failure isolation.

## 4. Open design question worth aligning on: reboot semantics

This is the one point where our design **deliberately diverges** from the existing
`BIOSVersion`/`BMCVersion` pattern, and it is worth an explicit decision upstream
rather than an inherited default.

`BIOSVersion`/`BMCVersion` drive reboots through **`ServerMaintenance`**. While a
`Server` sits in `ServerState == Maintenance`, `metal-operator`'s `ServerReconciler`
**re-asserts a persistent network-boot (PXE) override on every reconcile**
(`SetBootOverride(systemURI, true)`), clearing it on maintenance exit. Their own code
comment explains the intent: keep a machine being provisioned from falling through to
the production OS, including if a vendor firmware task self-reboots.

For a **live vSphere/ESXi host** that intent is inverted — the host *is* running
production and must return to it. A reboot under that override would land in a network
installer.

Our design therefore **owns the reboot** and forces **boot-from-disk** before every
power-on, without entering the `Maintenance` state. If `NICVersion` follows the
existing pattern unchanged, it inherits the PXE behaviour.

**Suggested discussion points:**
- Should component-version CRDs offer a boot-source policy (e.g. *boot-from-disk* vs
  *network-boot*) rather than a single hardcoded behaviour?
- Reboot activation is also policy-driven today: `ServerMaintenancePolicy` supports
  `Enforced` (automatic) and `OwnerApproval` (waits for the
  `metal.ironcore.dev/maintenance-approved` label). That model is a good fit and worth
  reusing for the new component kinds.
- `OnReset` staging support needs per-vendor verification: `metal-operator` currently
  hardcodes Dell to `Immediate`, so "stage now, activate on one reboot" is unverified
  for Dell.

## 5. Proposed collaboration

| Who | What |
|---|---|
| **`metal-operator` maintainers** | Accept `NICVersion` (then Storage/Drive/PSU) CRDs + controllers. Decide the reboot/boot-source question in §4. |
| **`feature/nic-firmware-update` author** | Already prototyping `NICVersion`. The findings in §3 are offered as review input — notably the missing `Updateable` check and Dell rollback-slot dedup. |
| **`MaintenancePlan` authors** | Once the CRDs land, extend `StageKind` + `StageTemplate` and add the `buildObject` / `checkCRStatus` cases — small and mechanical. |

We are happy to contribute the discovery logic, the multi-vendor Redfish fixtures
captured by the probe, and the design documents referenced above.

## 6. Sketch: `StageKind` extension (step 2)

For illustration — the MaintenancePlan change once the CRDs exist:

```go
// +kubebuilder:validation:Enum=BMCSettings;BMCVersion;BIOSSettings;BIOSVersion;NICVersion;StorageControllerVersion;DriveVersion;PSUVersion
type StageKind string

type StageTemplate struct {
    // … existing four …
    NICVersion *metalv1alpha1.NICVersionTemplate `json:"nicVersion,omitempty"`
    // … etc.
}
```

Plus matching cases in `buildServerObject` and `checkCRStatus` (terminal states
`Completed` / `Failed`, mirroring `BIOSVersion`). NIC/Storage/Drive/PSU are
**Server-scoped**, so they fan out one child CR per server, like the BIOS stages.

## 7. Caveats on the evidence

- The probe covered **QA hosts only**; production may carry additional NIC variants,
  firmware-name formats, or BMC firmware levels. Selector tokens should be validated
  against captured fixtures per fleet rather than assumed universal.
- `Supermicro` has no firmware-upgrade path in `metal-operator` today (the base
  implementation returns "not supported") — a known gap if the fleet includes it.
