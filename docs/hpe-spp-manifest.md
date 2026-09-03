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

## 6. Does iLO ingest an SPP directly? (No — confirmed on a live iLO, see §8.1)

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
   Install Set**, and **invoke it** (§7a). Optionally schedule via a maintenance
   window.

### 7a. The InstallSet batch mechanism — who does what

**Question: "will iLO upgrade all the components from the manifest, or must we do
something?" Answer: we build the set; iLO applies it.** iLO has **no** concept of the
SPP manifest — it never reads `manifest.json`. What iLO *does* provide is a
first-class **batch-apply** primitive: the **HPE Install Set**. The controller
assembles the set from the manifest diff, and iLO then applies the whole set —
ordered, with reboots, atomically.

Source-verified from `ilorest` (`MakeInstallSetCommand.py` / `InstallSetCommand.py`)
and confirmed against the iLO OEM `UpdateService`:

| Purpose | Endpoint | Method |
|---|---|---|
| Component repository (what's uploaded) | `/redfish/v1/UpdateService/ComponentRepository/?$expand=.` | `GET` |
| List / create install sets | `/redfish/v1/UpdateService/Oem/Hpe/InstallSets/` | `GET` / `POST` |
| Invoke a specific set | `…/Oem/Hpe/InstallSets/{id}/Actions/HpeComponentInstallSet.Invoke` | `POST` |
| Monitor the apply | `/redfish/v1/UpdateService/UpdateTaskQueue/` | `GET` |

(iLO also exposes the collection at the shorter alias
`/redfish/v1/UpdateService/InstallSets/`, which `ilorest` uses; the `Oem/Hpe/` form
is canonical. Read the `Invoke` target from the set's own
`Actions["#HpeComponentInstallSet.Invoke"]["target"]` rather than hard-coding it.)

**The Install Set body** (`POST` to the collection) — an ordered `Sequence` of steps,
each an `ApplyUpdate` (or a control step). This is the artifact the controller builds
from the manifest diff:

```jsonc
POST /redfish/v1/UpdateService/Oem/Hpe/InstallSets/
{
  "Name":        "maintenance-operator-<server>-<spp-version>",
  "Description": "SPP 2026.07 firmware baseline",
  "IsRecovery":  false,
  "Sequence": [
    { "Name": "nic-bcm",  "Command": "ApplyUpdate", "Filename": "bcm237.1.148000.Optimized.HPb", "WaitTimeSeconds": 0 },
    { "Name": "raid",     "Command": "ApplyUpdate", "Filename": "<component>.fwpkg" },
    { "Name": "sysrom",   "Command": "ApplyUpdate", "Filename": "<systemrom>.fwpkg" },
    { "Name": "reboot",   "Command": "ResetServer" }        // control step; also ResetBmc, Wait
  ]
}
```

- Each `Sequence` step's `Filename` must match a component **already uploaded** to
  `ComponentRepository` (step 6 upload precedes set creation).
- `Command` ∈ `ApplyUpdate` | `ResetServer` | `ResetBmc` | `Wait` (+ `WaitTimeSeconds`).
  So the controller encodes ordering and the reboot points **itself**, driven by the
  manifest's `Order` and `ResetRequired` fields (§3.2) — this is the "sequencing
  intelligence" that on Lenovo lives inside XCC.
- After creation, **`POST … Actions/HpeComponentInstallSet.Invoke`** kicks off the
  apply; iLO then executes the sequence and reboots per the control steps. Track via
  `UpdateTaskQueue`.

**So the division is:** controller = *select + order + upload + assemble the set*;
iLO = *apply the set atomically, in order, with reboots*. iLO takes care of the
**apply**, not the **selection** — the opposite balance to Lenovo, where XCC does
both from a single `RepoURI`.

### What is reusable vs new

- **Reusable as-is:** `ServerMaintenance` reboot-gating; the CRD state machine shape
  (`Pending → InProgress → Completed/Failed`, dry-run → gate → apply → track) — same
  skeleton as the `FirmwareUpdateLenovo` scaffold.
- **Reusable logic:** the manifest-diff (now proven to have the needed fields).
- **New, HPE-specific:** parse `metadata.json`; upload to `ComponentRepository`;
  assemble + invoke the Install Set (§7a); the FWPKG-v2 signature handling (§5).

## 8. Live-iLO verification (device-confirmed)

Verified against a **live iLO 6 on a DL560 Gen11** (`GET /redfish/v1/UpdateService`
and its sub-collections, `$expand=.`). This settles most of the earlier open items.

### 8.1 No SPP-ingest action — confirmed

`UpdateService.Actions` has only the standard `#UpdateService.SimpleUpdate`
(HTTP/HTTPS). The HPE OEM actions are `AddFromUri`, `BundleUpdateForceStop`,
`DeleteInstallSets`, `DeleteMaintenanceWindows`, `DeleteUnlockedComponents`,
`DeleteUpdateTaskQueueItems`, `RemoveLanguagePack`, `SetDefaultLanguage`,
`StartFirmwareIntegrityCheck`. **None ingest an SPP or a manifest** — confirming §6.
The manifest diff + Install-Set assembly is the controller's job.

### 8.2 `AddFromUri` — iLO can PULL components (softens "push-only")

`#HpeiLOUpdateServiceExt.AddFromUri`
(`…/Actions/Oem/Hpe/HpeiLOUpdateServiceExt.AddFromUri`) lets iLO **fetch a component
from a URI** into the `ComponentRepository`, instead of the controller multipart-
pushing each file to `/cgi-bin/uploadFile`. So the staging step can be "host the
`.fwpkg`s on HTTP(S), tell iLO to pull each" — much closer to the Dell/Lenovo
repo ergonomics for upload. (`SimpleUpdate` also allows HTTP/HTTPS.)

### 8.3 Capabilities (from `Oem.Hpe.Capabilities`)

`UpdateFWPKG: true`, `PLDMFirmwareUpdate: true`, **`StageBundleUpdateSupport: true`**
(stage now / apply later — useful for reboot-gating), `BundleDowngradeSupport: true`
+ `DowngradePolicy: "AllowDowngrade"`, `BundleUpdateForceStopSupport: true`,
`HostPoweroffSupport: true`, `OfflineRuntimeBundleUpdate: "ProductionMode"`.
`ComponentRepository` has a **~1 GB budget** (`TotalSizeBytes` ~1.07 GB), so the
controller must manage repo space (delete unlocked components when full).

### 8.4 Collection paths — read from `@odata.id`, don't hard-code

On this device the collections resolve to the **non-`Oem/Hpe`** paths:
`/redfish/v1/UpdateService/InstallSets`, `/…/ComponentRepository`,
`/…/UpdateTaskQueue`, `/…/MaintenanceWindows`, `/…/BundleUpdateReport`. A `GET` on
`…/Oem/Hpe/InstallSets` returned **404**; the real link is published as
`Oem.Hpe.InstallSets.@odata.id` in the UpdateService root. **Always follow the
`@odata.id` from the root**, and read the Invoke target from the set's own
`Actions["#HpeComponentInstallSet.Invoke"]["target"]`.

### 8.5 The join key is `Target` (GUID) — confirmed end-to-end ★

Every `FirmwareInventory` member carries `Oem.Hpe.Targets[]` (+ `DeviceClass`,
`DeviceContext`, `DeviceInstance`), and every `ComponentRepository` component carries
`Targets[]` + `Version` + `Filename`. The **same `Target` GUID** appears in the SPP
`manifest.json` (`Devices.Device[].Target`), the live inventory, and the live repo —
verified for a ConnectX-6 NIC across all three:

```
manifest.json   Target a6b1a447-382a-5a4f-15b3-101d15b30042  ver 22.49.1014  file 22_49_1014-MCX623106AS-CDA_Ax.pldm.signed  UpdatableBy[Bmc] ResetRequired
FirmwareInvent. /24,/25  same Target                          ver 22.49.1014
ComponentRepo   /440f0711 same Target                         ver 22.49.1014  file 22_49_1014-MCX623106AS-CDA_Ax.pldm.fwpkg
```

The GUID's middle bytes embed the PCI IDs (`15b3` = NVIDIA/Mellanox, `101d` = device),
the same identity scheme as Lenovo's `AgentlessId`. **The diff joins on this GUID.**

### 8.6 Two realities the live data added to the diff

1. **`Updateable: true|false` per inventory member is the OOB-flashability authority.**
   On this box, CPU microcode (S3M/PUcode), TPM, the Broadcom NIC and embedded video
   report `Updateable: false` — even though an SPP ships those components. So the
   controller must **filter to `Updateable: true`** from the inventory, which is a
   sharper gate than the manifest's `UpdatableBy` alone.
2. **Many devices share one `Target` → one payload updates N instances.** The 4 UBM
   backplanes (inv /31–/34) and the paired ConnectX-6 NICs share a single `Target`.
   The diff must **deduplicate by `Target`** so the Install Set lists each component
   once.

### 8.7 The device-confirmed diff algorithm

```text
for each FirmwareInventory member M:
    if not M.Updateable: continue                       # iLO says not OOB-flashable
    for T in M.Oem.Hpe.Targets:
        C = manifest component where T in C.Devices.Device[].Target
        if C and version_gt(C.Version, M.Version):       # newer available
            add C to update_set  (dedup by Target)
# update_set → (AddFromUri | push) each C payload → build InstallSet Sequence
#            → order by prerequisites/ResetRequired → Invoke → poll UpdateTaskQueue
```

Applied to this box (repo vs inventory), nothing was newer than installed — the
correct "already converged / no-op" dry-run result. **Generation matters:** the Gen10
SPP's iLO/BIOS `Target`s did not match this Gen11 server (option cards like the
ConnectX-6/NS204i did), so the controller must resolve the **matching-generation
SPP** for the host (the `firmware-baseline-oci.md` per-model resolution).

## 9. Remaining open items (live-device details)

- The exact `ComponentRepository` **upload/`AddFromUri`** contract for FWPKG-v2
  (headers; whether the `.json` sidecar / a `.compsig` is required) — the apply path
  itself (§7a) is settled.
- The `InstallSet` **status/result** shape during `Invoke` (per-step success/failure
  in `UpdateTaskQueue`), and whether a failed step aborts the sequence
  (`BundleUpdateReport/Current` and `/Completed` collections exist for this).
- **Maintenance Window** binding to an Install Set for a scheduled apply
  (`/redfish/v1/UpdateService/MaintenanceWindows`).
- How `UpdatableBy: ["Uefi"]` deferral / `Activates` (`Immediately` /
  `AfterReboot` / `AfterDeviceReset` / `AfterHardPowerCycle`) maps to the reboot the
  controller must gate — the `Activates` field (seen per repo component) is the
  authoritative reboot signal at apply time.

## 10. References

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
