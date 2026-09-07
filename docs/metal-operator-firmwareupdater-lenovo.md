<!--
SPDX-FileCopyrightText: 2025 SAP SE or an SAP affiliate company and IronCore contributors
SPDX-License-Identifier: Apache-2.0
-->

# `FirmwareUpdaterLenovo` — the metal-operator client + mock (modelled on Dell PR #1142)

**Status:** Design / implementation note
**Date:** 2026-09-08

The Lenovo firmware-update capability, like Dell's, spans **two repositories**:

| Layer | Repo | Dell (done) | Lenovo (this note) |
|---|---|---|---|
| **BMC client** (talks to the BMC over Redfish) + **mock** | `metal-operator` | **PR #1142** (`bmc/redfish_dell.go` + mock) | **to build** |
| **Controller** (CRD + reconcile loop) | `metal-maintenance-operator` | PR #170 (`FirmwareUpdateDell`) | our scaffold (Lenovo PR #205) |

Our Lenovo controller scaffold ([firmwareupdatelenovo](../internal/controller/system/))
stubs the vendor calls behind a `lenovoRepositoryUpdater` interface. **This note
specifies the metal-operator side** — a `bmc.FirmwareUpdaterLenovo` interface +
`LenovoRedfishBMC` implementation + mock handlers — following the exact shape merged for
Dell in [metal-operator#1142](https://github.com/ironcore-dev/metal-operator/pull/1142),
and reconciles our scaffold's stub to it.

Companion: [lenovo-redfish-updatefromrepository.md](lenovo-redfish-updatefromrepository.md)
(the device-verified mechanism), [redfish-mock-vendor-handlers.md](redfish-mock-vendor-handlers.md)
(the mock handlers).

---

## 1. The Dell template (from PR #1142)

Dell added, in one PR:

- `bmc.FirmwareUpdaterDell` — a **vendor-only interface, NOT part of the BMC union**.
  Callers discover support by a type assertion:
  `updater, ok := bmcClient.(bmc.FirmwareUpdaterDell)`.
- `DellRedfishBMC` implements it (`var _ FirmwareUpdaterDell = (*DellRedfishBMC)(nil)`).
- `RepositoryUpdateParameters`, `DellJob` (+ `IsCompleted/IsFailed/IsTerminal`).
- Mock handlers + data (`Oem/Dell/Jobs/repository/…`).

```go
// bmc/redfish_dell.go  (PR #1142)
type FirmwareUpdaterDell interface {
    InstallFirmwareFromRepository(ctx context.Context, systemURI string, parameters *RepositoryUpdateParameters) (jobID string, isFatal bool, err error)
    GetRepositoryUpdateList(ctx context.Context, systemURI string) (hasPendingPackages bool, packageListXML string, err error)
    ListJobs(ctx context.Context, UUID string) ([]string, error)
    GetJob(ctx context.Context, UUID string, jobID string) (*DellJob, error)
}
```

Key conventions to copy:

- **`isFatal bool`** on the install call — "any error after the POST is fatal: the
  request may have reached the BMC and started executing, so retrying risks a second
  install." A non-fatal error is safe to retry.
- **State is a raw string**, not a gofish enum (vendor states have no schema equivalent);
  the vendor `*Job`/`*Task` type owns `IsCompleted/IsFailed/IsTerminal`.
- The interface is deliberately **outside** the BMC union — discovered via type assertion.

## 2. The proposed `FirmwareUpdaterLenovo`

Lenovo's OEM actions (device-verified — see
[lenovo-redfish-updatefromrepository.md](lenovo-redfish-updatefromrepository.md)) map
cleanly onto the Dell shape. `LenovoRedfishBMC` already exists in
`bmc/redfish_lenovo.go` (with BIOS/BMC methods and **reusable task helpers**
`lenovoExtractTaskMonitorURI` / `lenovoParseTaskDetails`), so this is an *extension*, not
a new struct.

```go
// bmc/redfish_lenovo.go  (proposed, modelled on FirmwareUpdaterDell)

// FirmwareUpdaterLenovo is a Lenovo-only interface for OEM repository-based firmware
// updates (XCC LenovoUpdateService). Not part of the BMC union; discover via:
//   updater, ok := bmcClient.(bmc.FirmwareUpdaterLenovo)
type FirmwareUpdaterLenovo interface {
    // UpdateFromRepository POSTs LenovoUpdateService.UpdateFromRepository. Returns the
    // Redfish Task id. isFatal follows the Dell rule (any post-POST error is fatal).
    UpdateFromRepository(ctx context.Context, systemURI string, parameters *LenovoRepositoryUpdateParameters) (taskID string, isFatal bool, err error)
    // GetRepoUpdateDetail POSTs LenovoUpdateService.GetRepoUpdateDetail (the read-only
    // dry-run). Returns whether components are pending + the raw detail payload.
    GetRepoUpdateDetail(ctx context.Context, systemURI string) (hasPending bool, detail string, err error)
    // GetTask returns the current state of a Redfish Task (the apply/dry-run monitor).
    GetTask(ctx context.Context, systemURI, taskID string) (*LenovoTask, error)
}

// LenovoRepositoryUpdateParameters are the inputs to UpdateFromRepository (device schema:
// RepoURI is the only required field).
type LenovoRepositoryUpdateParameters struct {
    RepoURI      string // required
    RepoUserName string
    RepoPassword string
    RepoMountOpt string // CIFS/NFS mount options
    GroupRequest bool
}

// LenovoTask is a Lenovo Redfish Task (state kept as the raw string, like DellJob).
type LenovoTask struct {
    ID              string
    Name            string
    State           string // raw Redfish TaskState ("Running","Completed","Exception",…)
    Message         string
    PercentComplete int32
}

func (t *LenovoTask) IsCompleted() bool { return t.State == "Completed" }
func (t *LenovoTask) IsFailed() bool {
    switch t.State { case "Exception", "Cancelled", "Killed": return true }
    return false
}
func (t *LenovoTask) IsTerminal() bool { return t.IsCompleted() || t.IsFailed() }

var _ FirmwareUpdaterLenovo = (*LenovoRedfishBMC)(nil)
```

### 2.1 Action targets (the OEM URLs)

Unlike Dell (service under `/Systems/.../Oem/Dell/...`), Lenovo's actions live under
`UpdateService` (device-verified):

```go
const lenovoUpdateServicePath = "/redfish/v1/UpdateService/Actions/Oem"

func lenovoRepositoryActionTarget(action string) string {
    // action ∈ {"UpdateFromRepository","GetRepoUpdateDetail","BundleRollback"}
    return path.Join(lenovoUpdateServicePath, "LenovoUpdateService."+action)
}
```

### 2.2 `UpdateFromRepository` body (device-verified)

```jsonc
POST /redfish/v1/UpdateService/Actions/Oem/LenovoUpdateService.UpdateFromRepository
{ "RepoURI": "https://10.x/fw/sr650v3", "RepoUserName": "...", "RepoPassword": "...",
  "RepoMountOpt": "vers=3.0", "GroupRequest": false }
```

Implementation mirrors `DellRedfishBMC.InstallFirmwareFromRepository`: `client.Post(target,
body)` → on POST error return `isFatal=true` → extract the Task id from the `Location`
header (reuse `lenovoExtractTaskMonitorURI`) → return it. `GetTask` reuses
`lenovoParseTaskDetails`.

### 2.3 Notable differences from Dell (intentional, all device-verified)

| | Dell | Lenovo |
|---|---|---|
| Action location | `/Systems/.../Oem/Dell/DellSoftwareInstallationService` | `/UpdateService/Actions/Oem/LenovoUpdateService` |
| Result | iDRAC **Job** (Dell Jobs collection) | Redfish **Task** |
| Dry-run | `ApplyUpdate=False` + `GetRepoBasedUpdateList` | `GetRepoUpdateDetail` (a distinct action) |
| Reboot flag | `RebootNeeded` (client-controlled) | none — XCC decides per component |
| No `ListJobs` equivalent needed | uses `ListJobs`/`GetJob` (baseline diff) | a single Task id is tracked via `GetTask` |

Because Lenovo returns one Task (not a job collection to diff), `FirmwareUpdaterLenovo`
does **not** need Dell's `ListJobs` — `GetTask(taskID)` is sufficient.

## 3. Mock handlers (metal-operator `bmc/mock/server`)

Per [redfish-mock-vendor-handlers.md](redfish-mock-vendor-handlers.md) §2:

- Data: `data/UpdateService/Oem/Lenovo/UpdateTask/{index,steps,steps-fail}.json`.
- `actionHandler` entries: `hasSuffix("/LenovoUpdateService.UpdateFromRepository")` and
  `.../GetRepoUpdateDetail"`.
- Handlers `handleLenovoUpdateFromRepository` / `handleLenovoGetRepoUpdateDetail` mirror
  `handleDell*`, reusing `advanceResourceSteps(&s.lenovoTaskGen, …)` and the
  `"fail"` convention (`RepoURI` contains `"fail"` → `steps-fail.json`).
- A `ResetLenovoRepositoryUpdate()` helper (like `ResetDellRepositoryUpdate`).

## 4. Reconciling our controller scaffold (Lenovo PR #205)

Our scaffold's local `lenovoRepositoryUpdater` interface should be **replaced by the
real `bmc.FirmwareUpdaterLenovo`** once it lands. The method mapping:

| scaffold `lenovoRepositoryUpdater` | real `bmc.FirmwareUpdaterLenovo` |
|---|---|
| `GetRepoUpdateDetail(ctx, systemURI, params)` | `GetRepoUpdateDetail(ctx, systemURI)` (no params — dry-run takes none) |
| `UpdateFromRepository(ctx, systemURI, params)` | `UpdateFromRepository(ctx, systemURI, *LenovoRepositoryUpdateParameters)` |
| `GetTask(ctx, systemURI, taskID)` | `GetTask(ctx, systemURI, taskID) (*LenovoTask, …)` |

Then, in the controller: obtain it via a type assertion on the BMC client
(`updater, ok := bmcClient.(bmc.FirmwareUpdaterLenovo)`), exactly as the Dell controller
does with `FirmwareUpdaterDell`, and delete the local stub file.

## 5. Sequencing

1. **metal-operator PR** (the #1142-equivalent for Lenovo): `FirmwareUpdaterLenovo`
   interface + `LenovoRedfishBMC` methods + mock handlers + tests.
2. **maintenance-operator follow-up** (updates PR #205): drop the stub, assert
   `bmc.FirmwareUpdaterLenovo`, wire the real calls, and add a mock-backed e2e test.

Only after step 1 can the Lenovo flow be mock-tested end-to-end (controller → client →
mock). Until then the scaffold's fake-updater unit tests cover the controller logic.

## 6. References

- Dell template: [metal-operator#1142](https://github.com/ironcore-dev/metal-operator/pull/1142)
  (`bmc/redfish_dell.go`, `FirmwareUpdaterDell`, `DellJob`).
- Existing `bmc/redfish_lenovo.go` (reusable `lenovoExtractTaskMonitorURI` /
  `lenovoParseTaskDetails`; vendor dispatch `ManufacturerLenovo` in `bmc/bmc.go`).
- Device-verified Lenovo mechanism: [lenovo-redfish-updatefromrepository.md](lenovo-redfish-updatefromrepository.md).
- Mock handlers: [redfish-mock-vendor-handlers.md](redfish-mock-vendor-handlers.md).
