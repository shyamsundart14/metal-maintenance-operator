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
4. **Stage** each applicable `.fwpkg` into the iLO `ComponentRepository` (§3, step A),
   respecting the **~1 GB repo budget** (§4 — stream per component, or wave-batch).
5. **Assemble + invoke** an Install Set (§3, steps B–C); iLO applies the ordered set
   and reboots per the control steps. (Or, under Strategy A, flash per component as it
   is staged — §4.3.)
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

## 4. Repository size limit & the many-component strategy

The iLO `ComponentRepository` is **small and bounded** — the staging shelf, not a
bulk store. This directly constrains how many components can be in flight, so the
controller must handle it explicitly.

### 4.1 The limit (read it from the device — do not hard-code)

`GET /redfish/v1/UpdateService/ComponentRepository` reports the budget under
`Oem.Hpe`:

```jsonc
"ComponentCount": 8,
"TotalSizeBytes": 1073168384,   // ~1.0 GB total on the live iLO 6 (Gen11) checked
"FreeSizeBytes":  972914688     // free right now
```

- The ceiling is **~1 GB on the iLO 6 examined**, but it **varies by iLO generation /
  firmware** — the controller must read `TotalSizeBytes`/`FreeSizeBytes` live, never
  assume 1 GB.
- **`FreeSizeBytes`, not total, is what matters:** components already in the repo can
  be `Locked: true` (part of a persistent Install Set or the recovery set) and cannot
  be evicted — on the live box all 8 were locked. Always size against *free* space.
- Individual components **> 32 MiB** upload in chunks (multipart `ComponentFileName` +
  `Section`); `AddFromUri` handles large files by having iLO pull them.

### 4.2 Why this is usually a non-issue — diff first

**Stage the diff, not the SPP.** After the version comparison (§2) the set is only the
components actually out-of-date on *this* server — typically a handful (a few hundred
MB), not the full ~126-`.fwpkg` SPP. For most updates the diff fits comfortably in
1 GB and none of the below triggers. The strategies matter only for a very-behind
server whose applicable set exceeds free space.

### 4.3 Strategy A (default) — stream per component: pull → flash → free

Sidesteps the limit almost entirely. Instead of staging the whole set then invoking,
have iLO fetch-flash-discard **one component at a time**:

```jsonc
POST …/Actions/Oem/Hpe/HpeiLOUpdateServiceExt.AddFromUri
{
  "ImageURI":         "https://<repo>/component.fwpkg",
  "UpdateTarget":     true,     // flash now
  "UpdateRepository": false     // do NOT retain in the repo afterwards
}
```

The repo only ever holds one component, so the size ceiling is irrelevant. The
controller sequences the components itself (it already computed the order) and inserts
reboots per `ResetRequired`/`Activates`. **Trade-off:** you give up the single atomic
Install Set and its maintenance-window scheduling — the controller owns the
sequencing. This is the recommended default because it removes size as a constraint.

### 4.4 Strategy B — size-bounded Install Set waves

Keep the atomic Install Set model, but if the applicable set exceeds free space, split
it into waves that each fit:

```text
applicable set (e.g. 2.5 GB)
  wave 1: stage components up to ~free-space → InstallSet → Invoke → wait → reclaim
  wave 2: next batch → InstallSet → Invoke → wait → reclaim
  wave 3: remainder
```

Order waves so reboot-requiring components cluster (from `ResetRequired`/`Order`), to
minimise reboots. Use this when the atomic-set or maintenance-window semantics are
wanted for a batch.

### 4.5 Space reclamation (both strategies need it)

Before staging (each component in A, each wave in B): `GET ComponentRepository`, and if
`FreeSizeBytes` is insufficient, evict unlocked components:

- **`…/Actions/Oem/Hpe/HpeiLOUpdateServiceExt.DeleteUnlockedComponents`** — clears
  staged components not locked into an Install Set / recovery set.
- **`…/HpeiLOUpdateServiceExt.DeleteUpdateTaskQueueItems`** — clears the queue.

Then re-read `FreeSizeBytes` and stage only what fits. Locked components cannot be
reclaimed, so the controller plans against the free remainder.

## 5. Reusable vs new

- **Reusable as-is:** the `ServerMaintenance` reboot-gating and the CRD state-machine
  shape (`Pending → InProgress → Completed/Failed`, dry-run → gate → apply → track) —
  the same skeleton as the `FirmwareUpdateLenovo` scaffold.
- **Reusable logic:** the manifest-diff (proven to have the needed fields).
- **New, HPE-specific:** parse `metadata.json`; `AddFromUri`/upload to
  `ComponentRepository`; **manage the ~1 GB repo budget (§4)**; assemble + invoke the
  Install Set; FWPKG-v2 signature handling.

## 6. Open items (live-device details)

The apply path is fully specified above. What still needs a live run to finalise:

- The **per-task status shape** in `UpdateTaskQueue` during execution (`TaskState`
  values; whether a failed `ApplyUpdate` aborts the rest of the sequence) —
  `BundleUpdateReport/Current` and `/Completed` are the result views to poll.
- Confirm `AddFromUri` **`UpdateRepository:true, UpdateTarget:false`** stages without
  flashing, and **`UpdateTarget:true, UpdateRepository:false`** flashes without
  retaining (Strategy A, §4.3), on the target iLO generation.
- **Repo budget per iLO generation** — `TotalSizeBytes` on iLO 5 vs iLO 6 (read live);
  confirm `DeleteUnlockedComponents` reclaims as expected and how locked/recovery-set
  components reduce usable space (§4.5).
- **Maintenance Window** creation and how its id binds into the `Invoke` payload.
- The exact `AddFromUri`/upload **signature** requirement for FWPKG-v2 per iLO
  generation (iLO 5 vs iLO 6/Gen11).
- Empirical **ordering rules** iLO enforces vs. what the controller must impose in the
  `Sequence` (e.g. iLO/BMC before UEFI).

## 7. References

- Evidence and schema behind this approach: [hpe-firmware-findings.md](hpe-firmware-findings.md).
- HPE Server Management Portal — firmware updates via Redfish, part 3 (Install Sets,
  `AddFromUri`, Invoke): <https://servermanagementportal.ext.hpe.com/docs/references_and_material/blogposts/firmware_updates/part3/firmware_update_part3>
- Companions: [lenovo-redfish-updatefromrepository.md](lenovo-redfish-updatefromrepository.md),
  [dell-install-from-repository.md](dell-install-from-repository.md),
  [hpe-ilo-flashfwpkg.md](hpe-ilo-flashfwpkg.md).
