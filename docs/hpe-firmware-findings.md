<!--
SPDX-FileCopyrightText: 2025 SAP SE or an SAP affiliate company and IronCore contributors
SPDX-License-Identifier: Apache-2.0
-->

# HPE firmware update — research findings and evidence

**Status:** Findings / evidence log (background for the design)
**Date:** 2026-09-03

This document records **how** the HPE firmware-update approach was arrived at — the
extracted SPP ISO contents, the manifest schema, the scope analysis, and the
live-iLO verification. The **approach that will actually be built** lives in
[hpe-spp-manifest.md](hpe-spp-manifest.md); this file is the supporting evidence
behind it (kept separate so the design doc stays lean).

Companions: [hpe-ilo-flashfwpkg.md](hpe-ilo-flashfwpkg.md) (the per-component iLO
apply flow), [lenovo-redfish-updatefromrepository.md](lenovo-redfish-updatefromrepository.md),
[dell-install-from-repository.md](dell-install-from-repository.md).

---

## 1. What was examined

- **Extracted SPP ISO:** `Gen10 SPP 2026.07.00.00` (`bp008651.xml`, bundle version
  `2026.07.00.00`), mounted read-only; ~9.2 GB.
- **Live iLO 6** on a **DL560 Gen11** — `GET /redfish/v1/UpdateService` and its
  sub-collections (`$expand=.`), plus `FirmwareInventory` and `ComponentRepository`.

Together these confirm the manifest format (from the ISO) and the apply mechanism +
join key (from the device).

## 2. SPP on-disk layout

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

**Finding:** an SPP ships a bundle manifest in **both XML and JSON**. The JSON
(`manifest.json` / `metadata.json`) is a direct structural analog to Lenovo's
`<bundle>_index.json` — a flat component catalog a controller can parse with no XML.
The controller should prefer the JSON.

## 3. The JSON manifest schema

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

### 3.4 The XML bundle manifest (`bp008651.xml`) — SUM parity, informational

`/cpq_bundle` carries the bundle identity the SUM path uses: `version.value`
(`2026.07.00.00`), `divisions`, `operating_systems` (SLES/RHEL/Windows/**ESXi 8.0/9.0**/…),
and `products` — **31 ROM families** (`U32 U45 A43 A47 U37 U40 U41 H08 …`). These ROM
families are HPE's coarse "which systems" key. The controller does not need the XML if
it uses the JSON manifest.

## 4. Scope boundary — what a Redfish-only controller can and cannot do

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
subset) over Redfish. It does not replace SUM for OS drivers — and it does not need
to, because the maintenance-operator's job is firmware.

## 5. Payload & signature note

Modern firmware payloads are **`*.pldm.fwpkg`** (PLDM, `FWPKG-v2`), each paired with a
**`.json`** sidecar (e.g. `16_35_8008-MCX512F-ACH_Ax_Bx.pldm.fwpkg` + `…pldm.json`),
not a `.compsig`. The ISO still contains 750 `.compsig` files (used by older component
formats), but for FWPKG-v2 the signature handling differs from the `.fwpkg`+`.compsig`
model described in [hpe-ilo-flashfwpkg.md](hpe-ilo-flashfwpkg.md) §3.1. The exact
upload/signature requirement per iLO generation (iLO 5 vs iLO 6/Gen11) is a live-device
detail (§9 of the design doc).

## 6. Does iLO ingest an SPP directly? — No (device-confirmed)

The ISO's `restful_api/` directory holds `rest-classes-bios-U*.zip` — **BIOS attribute
registries / Redfish class definitions per ROM family**, *not* an "iLO ingests the SPP"
action. The manifest-orchestration tooling on the ISO is **SUM** (`launch_sum.sh`).

**Live-iLO confirmation:** `UpdateService.Actions` has only the standard
`#UpdateService.SimpleUpdate` (HTTP/HTTPS). The HPE OEM actions are `AddFromUri`,
`BundleUpdateForceStop`, `DeleteInstallSets`, `DeleteMaintenanceWindows`,
`DeleteUnlockedComponents`, `DeleteUpdateTaskQueueItems`, `RemoveLanguagePack`,
`SetDefaultLanguage`, `StartFirmwareIntegrityCheck`. **None ingest an SPP or a
manifest.** So the manifest→component-set resolution is the controller's job (SUM does
it in HPE's own tooling).

## 7. Live-iLO verification (device-confirmed)

Against the live iLO 6 (DL560 Gen11):

### 7.1 `AddFromUri` — iLO can PULL components

`#HpeiLOUpdateServiceExt.AddFromUri` lets iLO **fetch a component from a URI** into the
`ComponentRepository`, instead of multipart-pushing each file to `/cgi-bin/uploadFile`.
So staging can be "host the `.fwpkg`s on HTTP(S), tell iLO to pull each" — close to the
Dell/Lenovo repo ergonomics for upload.

### 7.2 Capabilities (`Oem.Hpe.Capabilities`)

`UpdateFWPKG: true`, `PLDMFirmwareUpdate: true`, **`StageBundleUpdateSupport: true`**
(stage now / apply later), `BundleDowngradeSupport: true` + `DowngradePolicy:
"AllowDowngrade"`, `BundleUpdateForceStopSupport: true`, `HostPoweroffSupport: true`,
`OfflineRuntimeBundleUpdate: "ProductionMode"`. `ComponentRepository` has a **~1 GB
budget** (`TotalSizeBytes` ~1.07 GB) — the controller must manage repo space.

### 7.3 Collection paths — read from `@odata.id`

On this device the collections resolve to the **non-`Oem/Hpe`** paths:
`/redfish/v1/UpdateService/InstallSets`, `/…/ComponentRepository`, `/…/UpdateTaskQueue`,
`/…/MaintenanceWindows`, `/…/BundleUpdateReport`. A `GET` on `…/Oem/Hpe/InstallSets`
returned **404**; the real link is `Oem.Hpe.InstallSets.@odata.id` in the UpdateService
root. **Always follow the `@odata.id` from the root**, and read the invoke target from
the set's own `Actions["#HpeComponentInstallSet.Invoke"]["target"]`.

### 7.4 The join key is `Target` (GUID) — confirmed end-to-end ★

Every `FirmwareInventory` member carries `Oem.Hpe.Targets[]` (+ `DeviceClass`,
`DeviceContext`, `DeviceInstance`); every `ComponentRepository` component carries
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

### 7.5 Two realities the live data added to the diff

1. **`Updateable: true|false` per inventory member is the OOB-flashability authority.**
   On this box, CPU microcode (S3M/PUcode), TPM, the Broadcom NIC and embedded video
   report `Updateable: false` — even though an SPP ships those components. The
   controller must **filter to `Updateable: true`** from the inventory — a sharper gate
   than the manifest's `UpdatableBy` alone.
2. **Many devices share one `Target` → one payload updates N instances.** The 4 UBM
   backplanes and the paired ConnectX-6 NICs share a single `Target`. The diff must
   **deduplicate by `Target`** so the Install Set lists each component once.

### 7.6 `Activates` is the apply-time reboot signal

Each `ComponentRepository` component carries `Activates` ∈ `Immediately` /
`AfterDeviceReset` / `AfterReboot` / `AfterHardPowerCycle`. This is the
authoritative, device-side reboot signal at apply time (the manifest's `ResetRequired`
is the offline predictor).

### 7.7 Generation matters

The Gen10 SPP's iLO/BIOS `Target`s did **not** match the Gen11 DL560 (option cards like
the ConnectX-6/NS204i did). So the controller must resolve the **matching-generation
SPP** for the host (per the `firmware-baseline-oci.md` per-model resolution). Applied to
this box (repo vs inventory), nothing was newer than installed — the correct
"already-converged / no-op" dry-run result.

## 8. References

- SPP examined: `Gen10 SPP 2026.07.00.00` (`manifest/manifest.json`,
  `manifest/metadata.json`, `packages/bp008651.xml`).
- Live iLO 6 (DL560 Gen11): `GET /redfish/v1/UpdateService` + sub-collections.
- `SPPCmdlets` (parses the SPP XML manifest): <https://github.com/szeidat/SPPCmdlets>
- HPE OneView firmware baseline / SPT:
  <https://www.hpe.com/psnow/resources/ebooks/a00115977en_us_v4/s_profiles-about-firmware.html>
- HPE Server Management Portal — firmware updates via Redfish, part 3:
  <https://servermanagementportal.ext.hpe.com/docs/references_and_material/blogposts/firmware_updates/part3/firmware_update_part3>
- Design doc built on these findings: [hpe-spp-manifest.md](hpe-spp-manifest.md).
