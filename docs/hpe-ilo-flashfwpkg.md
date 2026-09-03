<!--
SPDX-FileCopyrightText: 2025 SAP SE or an SAP affiliate company and IronCore contributors
SPDX-License-Identifier: Apache-2.0
-->

# HPE iLO firmware update — the `flashfwpkg` / iLO Repository model (source-verified)

**Status:** Findings / design record — **verified against HPE's `ilorest` source**
(`python-redfish-utility`), not yet against a live iLO.
**Date:** 2026-09-03

Companion to [dell-install-from-repository.md](dell-install-from-repository.md)
(Dell `InstallFromRepository` / PR #170) and
[lenovo-redfish-updatefromrepository.md](lenovo-redfish-updatefromrepository.md)
(Lenovo `UpdateFromRepository`). This document records how HPE ProLiant servers
apply firmware over Redfish via iLO, so the third vendor can be evaluated to the
same depth as Dell and Lenovo.

---

## 1. Headline — and a correction to the earlier "HPE is the odd one out" framing

Earlier notes placed HPE at the far end of a spectrum as the vendor with **no
single repository action** — needing per-component upload plus install-set
assembly. That is true for a **full SPP-scale** update, but it **overstates the
per-component case**. HPE's `ilorest flashfwpkg <file.fwpkg>` is, for a single
component, a **two-call Redfish flow** where **iLO itself does device discovery**:

1. **Upload** the `.fwpkg` (+ its `.compsig`) into the **iLO Repository**.
2. **POST** an `ApplyUpdate` task to the **UpdateTaskQueue**; iLO discovers the
   applicable device(s), applies, and reboots if required.

The client "just names the fwpkg." Device discovery was **removed from the client**
on purpose — the `flashfwpkg` changelog notes *"device discovery checks ... have
been removed, as this is now taken care of by iLO."* So the per-component HPE flow
is closer to Dell/Lenovo than previously stated: the BMC still does the matching.

The real difference from Dell/Lenovo is **payload delivery**: HPE requires the
client to **push each `.fwpkg` into the iLO Repository** (iLO does not pull from a
remote repo the way iDRAC/XCC do). It is per-`.fwpkg`, not a giant bundle.

## 2. Two tools, two roles (don't confuse them)

| Tool | Role | Talks to iLO? |
|---|---|---|
| **`fwget`** (`HewlettPackard/fwget`) | Download firmware from HPE SDR; read the `fwrepo.json` catalog | **No** — pure client-side downloader |
| **`fwlist`** (same repo) | Read `FirmwareInventory` over Redfish | Read-only |
| **`ilorest flashfwpkg`** (`python-redfish-utility`) | **Upload + apply** a `.fwpkg` via iLO | **Yes — this is the updater** |

`fwget`/`fwlist` cover *discovery + download + inventory* (the catalog half). They
contain **no apply code**. The apply half lives in `ilorest`'s
`iLO_REPOSITORY_COMMANDS`. This doc is about the apply half.

## 3. The `flashfwpkg` flow (from `FwpkgCommand.py` / `UploadComponentCommand.py`)

### 3.1 Step 1 — upload to the iLO Repository (`uploadcomp`)

- The `.fwpkg` is parsed; its payload image(s) and the matching **`.compsig`**
  signature are located (or generated). `.compsig` is mandatory — iLO verifies it.
- The component is uploaded (multipart) and lands in the iLO **ComponentRepository**:
  `GET /redfish/v1/UpdateService/ComponentRepository/?$expand=.` lists what is
  present. The controller then **waits** for the UpdateService to return to idle:
  `GET /redfish/v1/UpdateService` → `Oem.Hpe.State` (poll until not `UPDATING`).

### 3.2 Step 2 — create an `ApplyUpdate` task (`UpdateTaskQueue`)

The apply is a **single POST** to the task queue
(`/redfish/v1/UpdateService/UpdateTaskQueue/`):

```jsonc
POST /redfish/v1/UpdateService/UpdateTaskQueue/
{
  "Name":        "Update-<random>",
  "Command":     "ApplyUpdate",
  "Filename":    "<basename of the fwpkg component>",
  "UpdatableBy": ["Bmc"],          // or ["Uefi"] for System-ROM-class components
  "TPMOverride": false,            // --tpmover
  "Targets":     [                 // OPTIONAL — omit to let iLO discover targets
    "/redfish/v1/UpdateService/FirmwareInventory/29/",
    "/redfish/v1/UpdateService/FirmwareInventory/30/"
  ]
}
```

- **`Targets` is optional.** Omit it and iLO matches the uploaded component against
  applicable devices itself (the "just name the fwpkg" behaviour). Supply it
  (`--targets 29,30`, the FirmwareInventory member IDs) only to *narrow* the update.
- **`UpdatableBy`** comes from the fwpkg metadata and decides the apply path:
  `Bmc` (iLO flashes out-of-band) vs `Uefi` (applied at next boot by UEFI) vs
  combinations. This is HPE's analog of Lenovo's `ActivationMethod` / `UpdateType`.
- iLO processes the queue and **reboots when the component requires it** (e.g.
  System ROM). As with Dell/Lenovo, **there is no client "apply-time" reboot flag on
  this task** — reboot is decided by the component/`UpdatableBy`.

### 3.3 Component types (why some updates behave differently)

`get_comp_type` classifies each fwpkg (A/B/C/BC/D) from its metadata
(`UpdatableBy`, `Devices`, PLDM image flags, `per_device_install_time_seconds`).
Type drives upload handling and wait timeouts (default upload 30 min, flash 1 h).
A controller does not need to replicate the classification to *apply* — but the
per-device install-time and `UpdatableBy` are useful for predicting reboot/timeout.

## 4. Redfish endpoints a controller would use

| Purpose | Endpoint | Method |
|---|---|---|
| Login | `/redfish/v1/SessionService/Sessions` | `POST {UserName,Password}` → `X-Auth-Token` |
| Installed inventory | `/redfish/v1/UpdateService/FirmwareInventory/?$expand=.` | `GET` |
| Repository contents | `/redfish/v1/UpdateService/ComponentRepository/?$expand=.` | `GET` |
| UpdateService state | `/redfish/v1/UpdateService` → `Oem.Hpe.State` | `GET` (poll) |
| Upload component | iLO upload endpoint (multipart, `.fwpkg` + `.compsig`) | `POST` |
| **Apply** | `/redfish/v1/UpdateService/UpdateTaskQueue/` | **`POST`** (payload in §3.2) |
| Install Set (multi-component, atomic) | `/redfish/v1/UpdateService/InstallSets/` | `POST` (SPP-scale only) |
| Maintenance window (scheduled apply) | `/redfish/v1/UpdateService/MaintenanceWindows/` | `POST` (optional) |

Session auth, `FirmwareInventory`, and generic Redfish GET/POST are already handled
by metal-operator's BMC client — so most of this is standard Redfish the controller
gets for free; the HPE-specific parts are the **upload** and the **UpdateTaskQueue
POST**.

## 5. Single fwpkg vs. Install Set

- **Single `.fwpkg`** (`flashfwpkg`): upload → one `ApplyUpdate` task. The common
  case. Close to the Dell/Lenovo single-action model.
- **Install Set** (`makeinstallset` + `installset`): bundle **many** components to
  apply **atomically**, optionally within a **maintenance window**. This is the
  SPP-scale path and the only place the "assemble an install set" complexity
  applies. A `FirmwareUpdateHPE` can start with single-fwpkg and add install-set
  support later.

## 6. Three-vendor comparison (revised)

| | Dell | Lenovo | **HPE (per fwpkg)** | HPE (SPP set) |
|---|---|---|---|---|
| Trigger | 1 OEM action (share + `Catalog.xml`) | 1 OEM action (`RepoURI`) | **upload fwpkg + POST 1 ApplyUpdate task** | upload N + Install Set |
| Who discovers targets | iDRAC | XCC | **iLO** (client discovery removed) | iLO |
| Payload delivery | BMC pulls from share | BMC pulls from repo | **client pushes each fwpkg to iLO** | client pushes all |
| Reboot | BMC decides | XCC decides (`ActivationMethod`) | iLO decides (`UpdatableBy`) | iLO decides |
| Client apply-time flag | none | none | none | maintenance window (optional) |

The delivery column is the only structural difference: **HPE = push, Dell/Lenovo =
pull.** Discovery and reboot are BMC-driven on all three.

## 7. Mapping onto a `FirmwareUpdateHPE` controller

- **Reusable as-is:** the `ServerMaintenance` reboot-gating (§5a of the Lenovo doc)
  is vendor-neutral — request `ServerMaintenance` (`OwnerApproval`/`Enforced`), wait
  for the Server to be parked, then apply. HPE reboots a host like any other vendor,
  so ESXi/KVM/bare-metal drain is identical.
- **Reusable shape:** the CRD state machine (`Pending → InProgress →
  Completed/Failed`, dry-run → gate → apply → track) — the same skeleton as the
  `FirmwareUpdateLenovo` scaffold.
- **New, HPE-specific (the actual work):**
  1. **Stage/upload** each `.fwpkg` (+`.compsig`) to the iLO Repository and poll
     `Oem.Hpe.State`. (The "push" step Dell/Lenovo don't need.)
  2. **POST** the `ApplyUpdate` task to `UpdateTaskQueue` (payload §3.2), optionally
     with `Targets`.
  3. **Track** the task to completion (and optionally build an Install Set for
     multi-component atomic apply).
- **Optional catalog half:** reading HPE SDR `fwrepo.json` (à la `fwget`) to resolve
  which `.fwpkg` a host needs is a controller-side convenience; it is **not**
  required for apply, since iLO does target discovery once a component is uploaded.

## 8. Open items to verify on a live iLO

Same discipline as the Lenovo write-up (trust the device over docs):

- The exact **multipart upload** endpoint/headers iLO expects for `.fwpkg` +
  `.compsig` (the `ilorest` source uses an internal blobstore/`/cgi-bin/uploadFile`
  path in some modes; the pure-Redfish upload path must be confirmed per iLO
  generation — iLO 5 vs iLO 6/Gen11+).
- The **UpdateTaskQueue** task lifecycle/status fields to track to completion, and
  how `UpdatableBy: ["Uefi"]` deferral surfaces (applied at next boot).
- Whether the target iLO generation exposes any newer bulk helper on `UpdateService`
  (query `GET /redfish/v1/UpdateService` + its JsonSchemas on a real device).
- Install Set + maintenance-window schema, if multi-component atomic apply is in
  scope.

## 9. References

- HPE `ilorest` `FwpkgCommand.py` (the `flashfwpkg` apply flow):
  <https://github.com/HewlettPackard/python-redfish-utility/blob/master/ilorest/extensions/iLO_REPOSITORY_COMMANDS/FwpkgCommand.py>
- HPE `ilorest` `UploadComponentCommand.py` (the `.fwpkg`/`.compsig` upload):
  <https://github.com/HewlettPackard/python-redfish-utility/blob/master/ilorest/extensions/iLO_REPOSITORY_COMMANDS/UploadComponentCommand.py>
- HPE `fwget`/`fwlist` (SDR download + Redfish inventory):
  <https://github.com/HewlettPackard/fwget/blob/master/fwget.py>
- iLOrest user guide — iLO Repository commands:
  <https://redfish.redoc.ly/docs/redfishclients/ilorest-userguide/ilorepositorycommands/>
- HPE Software Delivery Repository (SDR): <https://downloads.linux.hpe.com/SDR/project/fwpp/>
- Companions: [dell-install-from-repository.md](dell-install-from-repository.md),
  [lenovo-redfish-updatefromrepository.md](lenovo-redfish-updatefromrepository.md).
