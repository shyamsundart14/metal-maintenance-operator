<!--
SPDX-FileCopyrightText: 2025 SAP SE or an SAP affiliate company and IronCore contributors
SPDX-License-Identifier: Apache-2.0
-->

# HPE SPP manifest — the bundle catalog, examined from a real ISO

**Status:** Findings / design record — **verified by extracting a real SPP ISO**
**Date:** 2026-09-03
**Bundle examined:** `Gen10 SPP 2026.07.00.00` (`bp008651.xml`, bundle version
`2026.07.00.00`), mounted read-only; ~9.2 GB ISO.

Companion to [hpe-ilo-flashfwpkg.md](hpe-ilo-flashfwpkg.md) (the iLO apply flow),
[lenovo-redfish-updatefromrepository.md](lenovo-redfish-updatefromrepository.md)
(Lenovo `UpdateFromRepository` + its `_index.json` manifest), and
[dell-install-from-repository.md](dell-install-from-repository.md).

This document records the **actual SPP manifest schema** so a `FirmwareUpdateHPE`
controller can consume the SPP bundle the way OneView's Server Profile Template
does (same source of truth), and answers "does HPE have a manifest like Lenovo?"
(yes) at the same fidelity we reached for Lenovo's `_index.json`.

---

## 1. Headline

An extracted SPP contains a **bundle manifest in BOTH XML and JSON**. The JSON form
(`manifest/manifest.json`, `manifest/metadata.json`) is a **direct structural analog
to Lenovo's `<bundle>_index.json`** — a flat component catalog a controller can parse
with no XML. Each component carries the join keys needed to match host hardware,
resolve its payload, and predict its reboot behaviour — the same capabilities the
Lenovo design relied on.

**Key consequence:** taking "the SPP route" (as OneView's SPT does) is not only
viable, it is *cleaner than expected* — the controller can read JSON, and the
manifest is self-sufficient for an offline dry-run.

**Scope boundary (also from the manifest, see §4):** only the subset of components
marked `UpdatableBy: Bmc`/`Uefi` is flashable **out-of-band via iLO/Redfish**. The
majority (`RuntimeAgent` — OS drivers/software) needs an in-OS agent (iSUT) and is
**out of scope** for a Redfish-only, workload-agnostic controller. That is fine:
firmware is what we are after.

## 2. SPP on-disk layout (Gen10 SPP 2026.07)

```
<SPP root>/
  packages/
    bp008651.xml            ← XML bundle manifest (/cpq_bundle) — the SUM/legacy path
    cp*.xml                 ← per-component XML descriptors
    *.fwpkg  (126)          ← firmware payloads (PLDM/FWPKG-v2)
    *.compsig (750)         ← detached signatures (older path; see §5)
    *.rpm (269) *.exe (229) *.zip (138) *.deb (85)   ← OS drivers/software payloads
  manifest/
    index.xml               ← lists the XML manifest parts
    manifest.json  (803 components)     ← JSON component catalog  ★
    metadata.json  (Bundles/Components/Prerequisites)  ← JSON, cleanest for a controller ★
    pkg_details.json                    ← per-package detail
    meta.xml device.xml category.xml os.xml prerequisite_meta.xml …  ← XML parts
  restful_api/              ← rest-classes-bios-U*.zip (BIOS attribute registries; NOT an SPP-ingest API — see §6)
  hp/swpackages/  launch_sum.sh  launch_sum.bat  pxe/  usb/  efi/  boot/
```

Two manifest worlds coexist: **XML** (`bp*.xml` + `meta.xml`, consumed by SUM) and
**JSON** (`manifest.json` / `metadata.json`). A controller should prefer the JSON.

## 3. The JSON manifest schema (the Lenovo `_index.json` analog)

### 3.1 `metadata.json` — the cleanest entry point

```jsonc
{
  "SchemaVersion": "1.0",
  "Bundles":       [ /* 1 — the bundle identity */ ],
  "Components":    [ /* 802 — the component catalog */ ],
  "Prerequisites": [ /* 38 — ordering/dependency rules */ ]
}
```

`manifest.json` carries the same 803 components as a flat `{"manifest": [ … ]}`
array; `metadata.json` additionally separates `Bundles` and `Prerequisites`.

### 3.2 A component entry (real, from `manifest.json`)

```jsonc
{
  "DeviceClass":   "79f0c163-0c13-4662-9dea-09235fef90cb",  // component-class GUID
  "PackageFormat": "FWPKG-v2",                               // FWPKG-v2 | FWPKG | Windows | Linux | null
  "Type":          "Firmware",                               // Firmware | ComboFirmware | Software
  "UpdatableBy":   ["Bmc"],                                  // Bmc | Uefi | RuntimeAgent (+combos)
  "MinimumActiveVersion": "…",
  "Devices": { "Device": [ {
      "DeviceName": "HPE Broadcom NetXtreme-E adapters",
      "Target":  "a6b1a447-382a-5a4f-14e4-16d815900212",     // ← device match key (note 14e4 = Broadcom PCI vendor)
      "Version": "237.1.148000",                             // ← target version
      "FirmwareImages": [ {
          "FileName":          "bcm237.1.148000.Optimized.HPb",  // ← the payload
          "Order":             1,
          "ResetRequired":     true,     // ← per-component REBOOT signal
          "ServerPowerOff":    false,
          "SysPowerON":        true,
          "DelayAfterInstallSec": 0,
          "InstallDurationSec":   360,   // ← for timeout/predicted-duration
          "PLDMImage":         true,
          "DirectFlashOk":     true,
          "UefiFlashable":     true,
          "Type":              "Firmware"
      } ]
  } ] },
  "package": { /* categories, descriptions, divisions, documentation, on-disk filename(s) */ }
}
```

### 3.3 Field mapping to Lenovo `_index.json`

| HPE `manifest.json` | Lenovo `_index.json` | Role |
|---|---|---|
| `Devices.Device[].Target` (GUID, embeds PCI vendor) | `Oem.Inventory[].SoftwareId` / `AgentlessId` | **the join key** host-inventory ↔ catalog |
| `Devices.Device[].Version` | `Oem.Inventory[].Version` | target version (diff vs installed) |
| `FirmwareImages[].FileName` | `Updates[].Payload` | the payload to apply |
| `FirmwareImages[].ResetRequired` / `ServerPowerOff` | `ActivationMethod` (`System reboot`/…) | **reboot prediction** |
| `UpdatableBy` (`Bmc`/`Uefi`/`RuntimeAgent`) | `UpdateType` (`OOB`/`BMU`/`Core`) | out-of-band vs in-band applicability |
| `DeviceClass` (GUID) | component class | grouping |
| `Prerequisites` / `MinimumActiveVersion` | `Prerequisites` / `MinimumSupportedVersion` | ordering/floor |
| `bp*.xml` `divisions` / `products` (ROM families) | `ApplicableMachineTypes` | which systems the bundle serves |

The correspondence is close enough that **the Lenovo dry-run logic ports directly**:
diff `manifest.json` `Target`→`Version` against `/redfish/v1/UpdateService/FirmwareInventory`,
and read `ResetRequired` up front to predict the reboot.

### 3.4 The XML bundle manifest (`bp008651.xml`) — for reference / SUM parity

`/cpq_bundle` carries the bundle identity the SUM path uses: `version.value`
(`2026.07.00.00`), `divisions`, `operating_systems` (SLES/RHEL/Windows/**ESXi 8.0/9.0**/…),
and `products` — **31 ROM families** (`U32 U45 A43 A47 U37 U40 U41 H08 …`). These ROM
families are HPE's coarse "which systems" key (the SPP-level analog of Lenovo's
machine types). The controller does not need the XML if it uses the JSON manifest.

## 4. The scope boundary — what a Redfish-only controller can and cannot do

Distribution across the 803 components of this SPP:

- **`UpdatableBy`:** `RuntimeAgent` ×614 · `Bmc` ×95 · `Uefi+RuntimeAgent` ×63 ·
  `Bmc+Uefi` ×22 · `Uefi` ×9
- **`Type`:** `Firmware` ×436 · `ComboFirmware` ×149 · `Software` ×218
- **`PackageFormat`:** `null` ×597 · `FWPKG-v2` ×122 · `FWPKG` ×4 · `Windows` ×40 · `Linux` ×40

Reading this:

- **Out-of-band flashable via iLO/Redfish** = the `Bmc` and `Uefi` components:
  roughly **~140** (`Bmc` 95 + `Bmc+Uefi` 22 + `Uefi` 9, plus the `Uefi` portion of
  `Uefi+RuntimeAgent`). These are the `.fwpkg`/FWPKG-v2 firmware images.
- **NOT out-of-band** = the **614 `RuntimeAgent`** components — OS drivers/software
  (`.rpm`/`.deb`/`.exe`). Applying them needs **iSUT running inside the host OS**,
  which breaks the workload-agnostic contract (ESXi/KVM/bare-metal). **Out of scope.**

**Net:** a `FirmwareUpdateHPE` covers the **firmware** of an SPP (the `Bmc`/`Uefi`
subset) over Redfish. It does **not** replace SUM for OS drivers — and it does not
need to, because the maintenance-operator's job is firmware.

## 5. Payload & signature — a correction to the flashfwpkg note

Modern firmware payloads are **`*.pldm.fwpkg`** (PLDM, `FWPKG-v2`), and each pairs
with a **`.json`** sidecar (e.g. `16_35_8008-MCX512F-ACH_Ax_Bx.pldm.fwpkg` +
`…pldm.json`), not a `.compsig`. The ISO still contains 750 `.compsig` files (used by
older component formats), but for FWPKG-v2 the signature handling differs from the
`.fwpkg`+`.compsig` model described in
[hpe-ilo-flashfwpkg.md](hpe-ilo-flashfwpkg.md) §3.1. The exact upload/signature
requirement per iLO generation (iLO 5 vs iLO 6/Gen11) should be confirmed on a live
device before implementing the upload step.

## 6. Does iLO ingest an SPP directly? (still: no evidence)

The ISO's `restful_api/` directory holds `rest-classes-bios-U*.zip` — these are
**BIOS attribute registries / Redfish class definitions per ROM family**, *not* an
"iLO ingests the SPP" action. Combined with the fact that the manifest-orchestration
tooling on the ISO is **SUM** (`launch_sum.sh`), this **supports the working
assumption** that there is **no Redfish action that ingests a whole SPP** — the
manifest→component-set resolution is done by SUM (offline/iSUT) or, in our design, by
the controller.

**Authoritative confirmation still requires a live iLO** (`GET /redfish/v1/UpdateService`
+ its JsonSchemas), per the same "trust the device over docs" discipline that
corrected the Lenovo analysis twice. The ISO strongly implies "no direct ingest"; a
live device settles it.

## 7. The design, one step further — Option A ("controller-as-SUM"), now concrete

Because SUM is closed-source (no SDK) and iLO does not ingest the SPP, the
Redfish-native path is: **the controller plays the slice of SUM's role we need**,
reading the JSON manifest and driving iLO. Concretely:

1. **Stage the SPP** at a path/repo the controller can read (the same SPP OneView
   uses as its SPT baseline).
2. **Parse `manifest/metadata.json`** → the `Components[]` with
   `Target`/`Version`/`UpdatableBy`/`ResetRequired`/`FileName`.
3. **Filter** to `UpdatableBy ∈ {Bmc, Uefi}` (out-of-band firmware) — skip
   `RuntimeAgent`.
4. **Diff** against `/redfish/v1/UpdateService/FirmwareInventory` by `Target` → the
   set to apply (this is the offline dry-run, the `GetRepoUpdateDetail` analog).
5. **Predict reboot** from `ResetRequired`/`ServerPowerOff` across the set → gate a
   **`ServerMaintenance`** (`OwnerApproval`/`Enforced`) — the same vendor-neutral
   reboot orchestration as Lenovo/Dell (see
   [lenovo-redfish-updatefromrepository.md](lenovo-redfish-updatefromrepository.md) §5a).
6. **Upload** each applicable `.fwpkg` to iLO `ComponentRepository`, **build an
   Install Set**, and apply (the
   [hpe-ilo-flashfwpkg.md](hpe-ilo-flashfwpkg.md) `UpdateTaskQueue`/`InstallSets`
   flow). Optionally schedule via a maintenance window.

### What is reusable vs new

- **Reusable as-is:** `ServerMaintenance` reboot-gating; the CRD state machine shape
  (`Pending → InProgress → Completed/Failed`, dry-run → gate → apply → track) — same
  skeleton as the `FirmwareUpdateLenovo` scaffold.
- **Reusable logic:** the manifest-diff (now proven to have the needed fields).
- **New, HPE-specific:** parse `metadata.json`; upload to `ComponentRepository`; build
  the Install Set; the FWPKG-v2 signature handling (§5).

## 8. Open items to confirm on a live iLO

- `GET /redfish/v1/UpdateService` + JsonSchemas: confirm **no** SPP/manifest-ingest
  OEM action (§6), and the exact `ComponentRepository` upload contract for FWPKG-v2.
- Install Set + Maintenance Window response/status shape for atomic multi-component
  apply.
- How `UpdatableBy: ["Uefi"]` deferral surfaces (applied at next boot) in the task
  queue.

## 9. References

- SPP examined: `Gen10 SPP 2026.07.00.00` (ISO contents: `manifest/manifest.json`,
  `manifest/metadata.json`, `packages/bp008651.xml`).
- HPE `ilorest` apply flow: see [hpe-ilo-flashfwpkg.md](hpe-ilo-flashfwpkg.md).
- `SPPCmdlets` (parses the SPP XML manifest):
  <https://github.com/szeidat/SPPCmdlets>
- HPE OneView firmware baseline / SPT:
  <https://www.hpe.com/psnow/resources/ebooks/a00115977en_us_v4/s_profiles-about-firmware.html>
- Companions: [lenovo-redfish-updatefromrepository.md](lenovo-redfish-updatefromrepository.md),
  [dell-install-from-repository.md](dell-install-from-repository.md),
  [hpe-ilo-flashfwpkg.md](hpe-ilo-flashfwpkg.md).
