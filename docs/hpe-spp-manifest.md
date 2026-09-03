<!--
SPDX-FileCopyrightText: 2025 SAP SE or an SAP affiliate company and IronCore contributors
SPDX-License-Identifier: Apache-2.0
-->

# HPE firmware update — the approach (`FirmwareUpdateHPE`)

**Status:** Design record — the approach to be built
**Date:** 2026-09-03

This document describes the **approach a `FirmwareUpdateHPE` controller will take**:
consume an HPE SPP the way OneView's Server Profile Template does, diff it against a
server's live firmware inventory, and drive iLO's batch-apply (Install Set) over
Redfish — reusing the shared `ServerMaintenance` reboot gate.

The **evidence behind every claim here** (extracted SPP schema, distributions, and
live-iLO verification) is in [hpe-firmware-findings.md](hpe-firmware-findings.md).

Companions: [lenovo-redfish-updatefromrepository.md](lenovo-redfish-updatefromrepository.md)
(Lenovo `UpdateFromRepository`), [dell-install-from-repository.md](dell-install-from-repository.md)
(Dell `InstallFromRepository`), [hpe-ilo-flashfwpkg.md](hpe-ilo-flashfwpkg.md)
(per-component iLO apply, background).

---

## 1. Summary

- HPE has **no single "ingest the SPP" Redfish action** (device-confirmed). The
  manifest→component-set resolution is the **controller's** job; iLO provides the
  **batch apply** (Install Set).
- The SPP ships a **JSON manifest** (`manifest/metadata.json`) — a flat component
  catalog with a stable **`Target` GUID** join key, versions, payload filenames, and
  reboot signals.
- Scope is the **firmware** subset (`UpdatableBy` `Bmc`/`Uefi`, further gated by the
  inventory's `Updateable:true`). OS-driver/`RuntimeAgent` components need in-OS iSUT
  and are **out of scope** — firmware is the operator's job.
- **Division of labour:** controller = *select + order + upload + assemble the set*;
  iLO = *apply the set atomically, in order, with reboots*. (Opposite balance to
  Lenovo, where XCC does both from one `RepoURI`.)

## 2. The flow

For a target server whose model/generation selects the matching SPP:

1. **Parse `manifest/metadata.json`** → `Components[]` with
   `Target`/`Version`/`UpdatableBy`/`ResetRequired`/`FileName`.
2. **Dry-run diff** against `/redfish/v1/UpdateService/FirmwareInventory` — join by the
   **`Target` GUID**, keep components where the manifest version is newer, filter to
   the inventory's **`Updateable:true`**, and **dedup by `Target`** (one payload can
   cover N identical devices). This computes the applicable set with no writes.
3. **Predict reboot** from `ResetRequired`/`ServerPowerOff` (offline) and the repo
   component's `Activates` (device) → gate a **`ServerMaintenance`**
   (`OwnerApproval`/`Enforced`) — the vendor-neutral reboot orchestration shared with
   Lenovo/Dell (see [lenovo-redfish-updatefromrepository.md](lenovo-redfish-updatefromrepository.md) §5a).
4. **Stage** each applicable `.fwpkg` into the iLO `ComponentRepository` (§3, step A).
5. **Assemble + invoke** an Install Set (§3, steps B–C); iLO applies the ordered set
   and reboots per the control steps.
6. **Track** `UpdateTaskQueue` to convergence; re-run the dry-run to confirm.

### Dry-run diff algorithm

```text
for each FirmwareInventory member M:
    if not M.Updateable: continue                       # iLO says not OOB-flashable
    for T in M.Oem.Hpe.Targets:
        C = manifest component where T in C.Devices.Device[].Target
        if C and version_gt(C.Version, M.Version):       # newer available
            add C to update_set  (dedup by Target)
# update_set → stage each C payload → build InstallSet Sequence
#            → order by prerequisites/ResetRequired → Invoke → poll UpdateTaskQueue
```

## 3. The iLO batch-apply mechanism (Install Set)

Endpoints (follow the `@odata.id` from the UpdateService root — on real iLO the
collections are at these non-`Oem/Hpe` paths; `…/Oem/Hpe/InstallSets` 404s):

| Purpose | Endpoint | Method |
|---|---|---|
| Component repository | `/redfish/v1/UpdateService/ComponentRepository?$expand=.` | `GET` |
| Add a component by URI (iLO pulls) | `…/Actions/Oem/Hpe/HpeiLOUpdateServiceExt.AddFromUri` | `POST` |
| List / create install sets | `/redfish/v1/UpdateService/InstallSets` | `GET` / `POST` |
| Invoke a set | `…/InstallSets/{id}/Actions/HpeComponentInstallSet.Invoke` | `POST` |
| Monitor the apply | `/redfish/v1/UpdateService/UpdateTaskQueue` | `GET` |

**Step A — stage each component** (before it can be referenced). Prefer `AddFromUri`
(iLO pulls; host the `.fwpkg` on HTTP(S)):

```jsonc
POST …/Actions/Oem/Hpe/HpeiLOUpdateServiceExt.AddFromUri
{
  "ImageURI":         "https://<repo>/22_49_1014-MCX623106AS-CDA_Ax.pldm.fwpkg",
  "CompSigURI":       "https://<repo>/…compsig",   // optional; omit for embedded-signature FWPKG-v2
  "UpdateRepository": true,      // keep in the iLO Repository so an Install Set can reference it
  "UpdateTarget":     false,     // false = STAGE ONLY (don't flash now); true = flash immediately
  "TPMOverride":      false
}
```

`UpdateRepository:true, UpdateTarget:false` = **stage without applying** — the
behaviour the controller wants (assemble the whole set, then invoke). Nothing flashes
until `Invoke`, so the reboot stays gated. (Fallback: multipart push to
`/cgi-bin/uploadFile`; >32 MiB uploads in chunks via `ComponentFileName` + `Section`.)

**Step B — create the Install Set.** An ordered `Sequence`; each step is an
`ApplyUpdate` (referencing an uploaded component by `Filename`) or a control step:

```jsonc
POST /redfish/v1/UpdateService/InstallSets
{
  "Name":        "maintenance-operator-<server>-<spp-version>",
  "Description": "SPP 2026.07 firmware baseline",
  "IsRecovery":  false,
  "Sequence": [
    { "Name": "connectx6", "UpdatableBy": ["Bmc"], "Command": "ApplyUpdate",
      "Filename": "22_49_1014-MCX623106AS-CDA_Ax.pldm.fwpkg" },
    { "Name": "sysrom",    "UpdatableBy": ["Bmc"], "Command": "ApplyUpdate",
      "Filename": "U59_2.22_06_19_2024.signed.flash" },
    { "Name": "settle",    "Command": "Wait", "WaitTimeSeconds": 60 },
    { "Name": "reboot",    "Command": "ResetServer" }
  ]
}
```

- **`Command`** ∈ **`ApplyUpdate`** (needs `Filename`) | **`ResetServer`** |
  **`ResetBmc`** | **`Wait`** (needs `WaitTimeSeconds`).
- The controller encodes **ordering and reboot/settle points itself** in the
  `Sequence`, from the manifest's `Order`/`Prerequisites`/`ResetRequired` and the repo
  component's `Activates`. (This sequencing intelligence is what XCC does internally on
  Lenovo.)
- Each `ApplyUpdate` `Filename` must match a component already in `ComponentRepository`.

**Step C — invoke:**

```jsonc
POST …/InstallSets/{id}/Actions/HpeComponentInstallSet.Invoke
{
  "ClearTaskQueue":    true,             // clear the UpdateTaskQueue before enqueuing this set
  "MaintenanceWindow": "<window-id>"     // optional — defer execution to a scheduled window
}
```

iLO enqueues the sequence into `UpdateTaskQueue` and executes it (immediately, or at
the maintenance window), rebooting per the control steps. Track via the task queue /
TaskService `TaskState`.

## 4. Reusable vs new

- **Reusable as-is:** the `ServerMaintenance` reboot-gating and the CRD state-machine
  shape (`Pending → InProgress → Completed/Failed`, dry-run → gate → apply → track) —
  the same skeleton as the `FirmwareUpdateLenovo` scaffold.
- **Reusable logic:** the manifest-diff (proven to have the needed fields).
- **New, HPE-specific:** parse `metadata.json`; `AddFromUri`/upload to
  `ComponentRepository`; assemble + invoke the Install Set; FWPKG-v2 signature
  handling.

## 5. Open items (live-device details)

The apply path is fully specified above. What still needs a live run to finalise:

- The **per-task status shape** in `UpdateTaskQueue` during execution (`TaskState`
  values; whether a failed `ApplyUpdate` aborts the rest of the sequence) —
  `BundleUpdateReport/Current` and `/Completed` are the result views to poll.
- Confirm `AddFromUri` **`UpdateRepository:true, UpdateTarget:false`** stages without
  flashing on the target iLO generation.
- **Maintenance Window** creation and how its id binds into the `Invoke` payload.
- The exact `AddFromUri`/upload **signature** requirement for FWPKG-v2 per iLO
  generation (iLO 5 vs iLO 6/Gen11).
- Empirical **ordering rules** iLO enforces vs. what the controller must impose in the
  `Sequence` (e.g. iLO/BMC before UEFI).

## 6. References

- Evidence and schema behind this approach: [hpe-firmware-findings.md](hpe-firmware-findings.md).
- HPE Server Management Portal — firmware updates via Redfish, part 3 (Install Sets,
  `AddFromUri`, Invoke): <https://servermanagementportal.ext.hpe.com/docs/references_and_material/blogposts/firmware_updates/part3/firmware_update_part3>
- Companions: [lenovo-redfish-updatefromrepository.md](lenovo-redfish-updatefromrepository.md),
  [dell-install-from-repository.md](dell-install-from-repository.md),
  [hpe-ilo-flashfwpkg.md](hpe-ilo-flashfwpkg.md).
