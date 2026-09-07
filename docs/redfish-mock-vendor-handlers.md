<!--
SPDX-FileCopyrightText: 2025 SAP SE or an SAP affiliate company and IronCore contributors
SPDX-License-Identifier: Apache-2.0
-->

# Simulating repository/bundle firmware update in the Redfish mock — Lenovo & HPE

**Status:** Design / implementation note
**Date:** 2026-09-08

The "tilt emulator" used for development and e2e is **metal-operator's own Redfish
mock** (`bmc/mock/server/server.go`, built as the `mock-server` image by the
metal-operator `Tiltfile`) — **not** sushy-tools. It already simulates the
repository-update concept for **Dell**; this note documents the exact steps to add
the **Lenovo** (`UpdateFromRepository`) and **HPE** (Install Set) flows by the same
pattern, so the firmware-update controllers can be tested end-to-end without hardware.

> These changes land in **metal-operator** (`bmc/mock/server/`), not in this repo. The
> mock is consumed by both projects' test suites (`mockserver.NewMockServer(...)`).

Companions: [dell-install-from-repository.md](dell-install-from-repository.md),
[lenovo-redfish-updatefromrepository.md](lenovo-redfish-updatefromrepository.md),
[hpe-spp-manifest.md](hpe-spp-manifest.md).

---

## 1. How the mock works (the extension points)

The mock serves Redfish from **embedded JSON** (`//go:embed data/**`) and dispatches
`POST`s through an ordered **action-handler table**. The pieces you use:

- **`actionHandler{ matches, handle }`** — a URL-suffix predicate + a handler. Entries
  are registered in `NewMockServer` on `s.actionHandlers`; the first match wins and the
  default collection-create logic is skipped. **Adding an action = adding one entry +
  one handler func; `handlePost` is untouched.**
- **`hasSuffix("/…Action")`** — the usual `matches` predicate.
- **Embedded data files** under `bmc/mock/server/data/` — GETs are served from here.
  Add a file → it is served at the corresponding `/redfish/v1/...` path.
- **`s.loadResource(path)` / `s.overrides[path]` / `mergeJSON(base, update)`** — load a
  data template, seed it, and register a runtime override so subsequent GETs see the
  mutated resource.
- **`s.advanceResourceSteps(&genCounter, gen, filePath, steps, onTerminal)`** — the
  reusable async engine that walks a resource through a `steps.json` progression
  (New → Scheduled → Running… → Completed), calling `onTerminal(finalStep)` at the end.
  A generation counter cancels stale goroutines when a new action supersedes an old one.
- **`WithSystemOverride(systemID, {"Manufacturer": ...})`** — **critical for dispatch.**
  The BMC client picks a vendor-specific implementation from the System's
  `Manufacturer` (`bmc.NewRedfishBMCClient` / `DefaultVendors`). Tests must override it
  (`"Lenovo"`, `"HPE"`) so the right client is selected.

### The Dell reference (what to copy)

Dell is fully implemented and is the template:

| Piece | Dell |
|---|---|
| Action entries | `hasSuffix("/DellSoftwareInstallationService.InstallFromRepository")` → `handleDellInstallFromRepository`; `…GetRepoBasedUpdateList` → `handleDellGetRepoBasedUpdateList` |
| Data | `data/Managers/BMC/Oem/Dell/Jobs/repository/{index,steps,steps-fail}.json` |
| State | `dellRepoUpdateState` (`unchecked → pending → applied`) on `MockServer` |
| Engine | `doDellJobSteps` → `advanceResourceSteps(&s.dellJobGen, …)` |
| Fail path | `CatalogFile` containing `"fail"` selects `steps-fail.json` |
| Reset | `ResetDellRepositoryUpdate()` (simulate a BMC restart) |

`steps.json` shape (a plain progression the engine walks):

```json
[
  {"JobState": "New",       "Message": "Task successfully scheduled.", "PercentComplete": 0},
  {"JobState": "Running",   "Message": "Job execution in progress.",   "PercentComplete": 50},
  {"JobState": "Completed", "Message": "Job completed successfully.",   "PercentComplete": 100}
]
```

---

## 2. Add the Lenovo `UpdateFromRepository` flow

Lenovo maps almost 1:1 onto Dell (one OEM action + a dry-run + async task). Follow the
Dell steps, renaming.

### 2.1 Data files

Create under `bmc/mock/server/data/`:

```
UpdateService/Oem/Lenovo/UpdateTask/index.json      # the Task resource template
UpdateService/Oem/Lenovo/UpdateTask/steps.json      # success progression (copy Dell steps.json)
UpdateService/Oem/Lenovo/UpdateTask/steps-fail.json # failure progression (copy Dell steps-fail.json)
```

`UpdateTask/index.json` (mirror Dell's job template, Lenovo Task shape):

```json
{
  "@odata.type": "#Task.v1_5_0.Task",
  "@odata.id": "/redfish/v1/UpdateService/Oem/Lenovo/UpdateTask",
  "Id": "LenovoRepoUpdate",
  "Name": "UpdateFromRepository",
  "TaskState": "New",
  "TaskStatus": "OK",
  "PercentComplete": 0
}
```

(Also add the three OEM actions to `UpdateService/index.json` `Oem.Lenovo.Actions` for
completeness — `#LenovoUpdateService.UpdateFromRepository`, `.GetRepoUpdateDetail`,
`.BundleRollback` — so a `GET /UpdateService` advertises them, matching a real XCC.)

### 2.2 State + constants (on `MockServer`)

```go
const (
    lenovoTaskFilePath      = "data/UpdateService/Oem/Lenovo/UpdateTask/index.json"
    lenovoTaskStepsFilePath = "data/UpdateService/Oem/Lenovo/UpdateTask/steps.json"
    lenovoTaskFailFilePath  = "data/UpdateService/Oem/Lenovo/UpdateTask/steps-fail.json"
    lenovoTaskURI           = "/redfish/v1/UpdateService/Oem/Lenovo/UpdateTask"
)
// on MockServer:  lenovoRepoState dellRepoUpdateState (reuse the enum); lenovoTaskGen int64
```

### 2.3 Action handlers (register in `NewMockServer`)

```go
{
    matches: hasSuffix("/LenovoUpdateService.UpdateFromRepository"),
    handle:  s.handleLenovoUpdateFromRepository,
},
{
    matches: hasSuffix("/LenovoUpdateService.GetRepoUpdateDetail"),
    handle:  s.handleLenovoGetRepoUpdateDetail,
},
```

### 2.4 Handler funcs (mirror the Dell ones)

- `handleLenovoUpdateFromRepository`: unmarshal `{ "RepoURI", "GroupRequest", ... }`;
  pick `steps.json` vs `steps-fail.json` by whether **`RepoURI` contains `"fail"`** (the
  Lenovo analog of Dell's `CatalogFile` "fail" convention); seed the Task with `steps[0]`,
  bump `lenovoTaskGen`, register the override, `go s.advanceResourceSteps(&s.lenovoTaskGen,
  gen, lenovoTaskFilePath, steps, onTerminal)`; respond `202` with `Location: lenovoTaskURI`.
  `UpdateFromRepository` is always an apply (there is no per-call dry-run flag), so
  `onTerminal` moves `lenovoRepoState → applied` on `Completed`.
- `handleLenovoGetRepoUpdateDetail`: the dry-run. Return a small pending-list JSON when
  `lenovoRepoState == pending` (seed `pending` on server start, like Dell seeds it on the
  first check), else an empty list. This is the `GetRepoUpdateDetail` analog of Dell's
  `GetRepoBasedUpdateList`.

### 2.5 Test wiring

```go
ms := mockserver.NewMockServer(log, addr, mockserver.WithAuth(),
    mockserver.WithSystemOverride("437XR1138R2", map[string]any{"Manufacturer": "Lenovo"}))
```

---

## 3. Add the HPE Install Set flow

HPE is multi-step (upload → create set → invoke → track), so it needs a few more
handlers and data resources — but each is the same shape.

### 3.1 Data files

```
UpdateService/ComponentRepository/index.json         # empty collection + Oem.Hpe {ComponentCount, FreeSizeBytes, TotalSizeBytes}
UpdateService/InstallSets/index.json                 # empty collection
UpdateService/UpdateTaskQueue/index.json             # empty collection
UpdateService/InstallSets/set/index.json             # a set template (seeded on create) with the Invoke action
UpdateService/InstallSets/set/steps.json             # success progression
UpdateService/InstallSets/set/steps-fail.json        # failure progression
```

`FirmwareInventory` members must carry the HPE join fields the diff needs — override or
add members with `Oem.Hpe.Targets` and top-level `Updateable`:

```json
{ "Id": "24", "Name": "ConnectX-6 Dx 100GE 2P NIC", "Version": "22.40.1000",
  "Updateable": true,
  "Oem": { "Hpe": { "Targets": ["a6b1a447-382a-5a4f-15b3-101d15b30042"] } } }
```

(A stale `Version` like `22.40.1000` — older than the SPP's `22.49.1014` — makes the
diff select this component, so the apply path is exercised.)

### 3.2 State + constants

```go
const (
    hpeInstallSetFilePath   = "data/UpdateService/InstallSets/set/index.json"
    hpeInstallSetStepsPath  = "data/UpdateService/InstallSets/set/steps.json"
    hpeInstallSetFailPath   = "data/UpdateService/InstallSets/set/steps-fail.json"
    hpeInstallSetURI        = "/redfish/v1/UpdateService/InstallSets/set"
    hpeRepositoryPath       = "data/UpdateService/ComponentRepository/index.json"
)
// on MockServer:  hpeInstallSetGen int64; hpeStaged []string (filenames pulled via AddFromUri)
```

### 3.3 Action handlers (register in `NewMockServer`)

```go
{
    matches: hasSuffix("/HpeiLOUpdateServiceExt.AddFromUri"),
    handle:  s.handleHPEAddFromUri,
},
{ // InstallSets collection POST (create). Match the collection path, not a suffix action.
    matches: func(p string) bool { return strings.HasSuffix(p, "/UpdateService/InstallSets") },
    handle:  s.handleHPECreateInstallSet,
},
{
    matches: hasSuffix("/HpeComponentInstallSet.Invoke"),
    handle:  s.handleHPEInvokeInstallSet,
},
```

### 3.4 Handler funcs

- `handleHPEAddFromUri`: unmarshal `{ "ImageURI", "UpdateRepository", "UpdateTarget" }`;
  append the basename of `ImageURI` to `s.hpeStaged`; bump `ComponentCount` /
  decrement `FreeSizeBytes` on the ComponentRepository override (so the ~1 GB budget can
  be exercised); respond `200`/`202`. An `ImageURI` containing `"fail"` can flag the set
  to later select `steps-fail.json`.
- `handleHPECreateInstallSet`: unmarshal `{ "Name", "Sequence": [...] }`; validate each
  `Sequence[].Filename` is in `s.hpeStaged` (mirrors the real constraint "component must
  be staged first"); seed `InstallSets/set/index.json` from the template; respond `201`
  with `Location: hpeInstallSetURI`. Return the set id (`set`).
- `handleHPEInvokeInstallSet`: pick `steps.json` vs `steps-fail.json` (by whether any
  staged filename contained `"fail"`); seed the set with `steps[0]`, bump
  `hpeInstallSetGen`, `go s.advanceResourceSteps(&s.hpeInstallSetGen, gen,
  hpeInstallSetFilePath, steps, onTerminal)`; respond `202`. The controller polls the
  set (or `UpdateTaskQueue`) for terminal state — have the engine also write terminal
  state into `UpdateTaskQueue/index.json` if the controller tracks the queue.

### 3.5 Test wiring

```go
ms := mockserver.NewMockServer(log, addr, mockserver.WithAuth(),
    mockserver.WithSystemOverride("437XR1138R2", map[string]any{"Manufacturer": "HPE"}))
```

The SPP `metadata.json` the controller diffs against is served separately (a plain HTTP
fileserver over an extracted/trimmed SPP fixture) — the **mock only plays the iLO**, not
the SPP host.

---

## 4. What the mock does and does not verify

- **Verifies (contract + flow):** the controller hits the right endpoints with the right
  payloads; the async Task/Job/queue progression is handled; **both success and failure**
  paths (the `"fail"` conventions); the diff logic against a mock `FirmwareInventory`; the
  ServerMaintenance reboot-gate ordering (a reboot Redfish call arrives only after the
  gate). This is what controller CI needs.
- **Does not verify:** that a real BMC accepts the exact payloads, real firmware
  flashing, SPP staging, or vendor quirks — those still need a real (or vendor-simulated)
  BMC.

## 5. Suggested increments

1. **Lenovo** first — it is the closest to the existing Dell code (one action + dry-run +
   task), so the smallest diff.
2. **HPE** next — the multi-step Install Set; reuse `advanceResourceSteps` and the `"fail"`
   convention throughout.
3. Each ships with: data fixtures, the handler funcs, a `Reset…` helper (simulate BMC
   restart), and unit tests mirroring the existing Dell mock tests.

## 6. References

- metal-operator mock: `bmc/mock/server/server.go` (Dell handlers `handleDell*`, engine
  `advanceResourceSteps`, options `WithSystemOverride`/`WithResourceOverride`);
  data under `bmc/mock/server/data/`.
- metal-operator `Tiltfile` (`docker_build('mock-server', …)`).
- Vendor mechanisms: [dell-install-from-repository.md](dell-install-from-repository.md),
  [lenovo-redfish-updatefromrepository.md](lenovo-redfish-updatefromrepository.md),
  [hpe-spp-manifest.md](hpe-spp-manifest.md).
