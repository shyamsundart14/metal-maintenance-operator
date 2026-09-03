<!--
SPDX-FileCopyrightText: 2025 SAP SE or an SAP affiliate company and IronCore contributors
SPDX-License-Identifier: Apache-2.0
-->

# Lenovo XCC `UpdateFromRepository` — a Redfish repository-update action (device-verified)

**Status:** Findings / design record — **verified against a live ThinkSystem XCC**
**Date:** 2026-09-02
**Supersedes** the "Lenovo has no Redfish repository action" conclusion in
[lenovo-xcc-repository.md](lenovo-xcc-repository.md) §1. See §6 below for the
correction.

Companion to [dell-install-from-repository.md](dell-install-from-repository.md)
(the Dell `InstallFromRepository` / PR #170 approach) and
[firmware-update-design.md](firmware-update-design.md) (per-component
`SimpleUpdate`).

---

## 1. Headline

**Lenovo XCC exposes a repository-based firmware update over Redfish.** A live
`GET /redfish/v1/UpdateService` and the XCC's own JSON schema confirm three Lenovo
OEM actions:

| Action | Target | Purpose |
|---|---|---|
| `#LenovoUpdateService.UpdateFromRepository` | `/redfish/v1/UpdateService/Actions/Oem/LenovoUpdateService.UpdateFromRepository` | Point XCC at a repository; it self-inventories and updates all applicable components |
| `#LenovoUpdateService.GetRepoUpdateDetail` | `.../Oem/LenovoUpdateService.GetRepoUpdateDetail` | Dry-run / query what is pending against the repo |
| `#LenovoUpdateService.BundleRollback` | `.../Oem/LenovoUpdateService.BundleRollback` | Roll back a bundle |

This is the "point at the repo, the BMC orchestrates the component upgrades" model
— and it is **Redfish-drivable**, not just LXCE/CLI/WebGUI.

## 2. The `UpdateFromRepository` payload (from the device schema)

Retrieved from the XCC's own schema
(`/redfish/v1/schemas/LenovoUpdateService.v1_0_0.json` →
`definitions.UpdateFromRepository.parameters`):

> *"This action shall perform firmware update, based on the repository server
> specified by input parameters."*

```jsonc
POST /redfish/v1/UpdateService/Actions/Oem/LenovoUpdateService.UpdateFromRepository
{
  "RepoURI":      "https://10.x.x.x/firmware/sr650v3",  // REQUIRED — repository server address
  "RepoUserName": "svc-firmware",                        // optional — repo credentials
  "RepoPassword": "•••••",                               // optional
  "RepoMountOpt": "vers=3.0",                            // optional — mount options (CIFS/NFS)
  "GroupRequest": false                                  // optional — group-service flag, default false
}
```

| Parameter | Required | Type | Meaning |
|---|---|---|---|
| `RepoURI` | **yes** | string | The repository server address (the only mandatory input) |
| `RepoUserName` | no | string | Repository server username |
| `RepoPassword` | no | string | Repository server password |
| `RepoMountOpt` | no | string | Mount options for the repository server (CIFS/NFS) |
| `GroupRequest` | no | bool | Whether the request comes from a group service (default `false`) |

The action is **POST-only** (`GET` returns `405 Not Allowed`) and carries **no
`@Redfish.ActionInfo`**, so the device JSON schema is the authoritative source for
its parameters. A successful invocation returns a Task (monitored like other
UpdateService operations).

## 3. What the repository must contain — the concrete JSON layout

Per the XCC "Update From Repository" documentation, the repository (CIFS / NFS /
HTTP / HTTPS reachable at `RepoURI`) holds the **Update Bundle / UXSP** contents —
**metadata files plus firmware payloads** — with the metadata placed at the **root**
of the share and the payloads referenced from it. XCC **parses the metadata,
self-inventories the host, and applies only the packages applicable to this machine
type that support out-of-band (OOB) update** — the client provides no per-component
list.

The rest of this section documents the **actual JSON metadata schema**, characterised
from a real V3-generation firmware bundle:
`lnvgy_bundle_svcpack_ka-j9ltd-01a2-24a.0_platform_comp` (System Firmware / Platform
Bundle `24a.0`, 2024-03-01; 305 components, ~914 MB, JSON metadata — the XCC3/V3
format, not the older XML/CIM). **This is the layout `RepoURI` points at.**

### 3.1 Three layers

**(a) Top-level bundle catalog** — the single root file XCC reads first
(`<bundle>_index.json`, ~238 KB). It is one `Updates[]` array (one entry per
component) plus a bundle-level `Oem` block:

```jsonc
{
  "Updates": [ /* 305 entries */ {
      "Payload":   "payloads/brcm-lnvgy_fw_nic_nxe-...anyos_comp.uxz", // relative path
      "Inventory": "..._inventory.json",       // → per-component descriptor (3.1b)
      "Locations": "..._location.json",        // → download locations (empty in offline bundle)
      "AuthenticitySignature": "..._signature.json",  // → PGP + SHA-256 (3.1c)
      "Oem": { "Inventory": [
        { "SoftwareId": "DEVICE-14E4-16D7-17AA-4105-13", "Version": "227.1.115.0",
          "OperatingSystem": "anyos", "Payload": "payloads/...uxz" }, ... ] }
  } ],
  "Oem": {
    "Description": "System Firmware (Platform) Bundle - System Support: ThinkSystem ...",
    "BundleData": { "ParentFixIds": [ /* 22 sub-package fix IDs */ ] },
    "ApplicableMachineTypes": ["7D75","7D76", ...],       // machine-type CODES, not model names
    "Version": "24a.0",
    "ReleaseDate": "2024-03-01T16.22.44Z",
    "DocumentationFiles": { "ReleaseNotes": [...], "ChangeHistory": [...] },
    "Name": "lnvgy_bundle_svcpack_ka-j9ltd-01a2-24a.0_platform_comp"
  }
}
```

**(b) Per-component quartet** — every component ships four JSON files (plus its
payload and `.chg`/`.html`/`.txt` notes), all in one flat directory:

| File | Role |
|---|---|
| `<comp>_index.json` | component manifest: `Updates[]` with `Payload`, `Oem.Inventory` (SoftwareId→Version), `Oem.FixId` (`DsNumber`, `Name`), and a per-component `Oem.ApplicableMachineTypes` |
| `<comp>_inventory.json` | rich descriptor: `Description`, `ReleaseDate`, `UpdateType` (e.g. `BMU`), `ActivationMethod` ("System reboot"), `InstallTime`, `Prerequisites` / `MinimumSupportedVersion` (SoftwareId + Version + FixId), and `Devices[]` with `AgentlessId`, `PartNumbers`, `SoftwareIds`, `FeatureCodes` |
| `<comp>_location.json` | `{ "Locations": [] }` — download URLs; empty in an offline/pre-staged bundle, populated for a hosted repo |
| `<comp>_signature.json` | `AuthenticitySignature[]` — a detached **PGP signature** ("Lenovo Group ISG RSA 4096 01 &lt;signVerify@lenovo.com&gt;") plus `Oem.Sha-256` and `SizeBytes`, for **both** the `_inventory.json` and the payload |

**(c) Payloads** — `payloads/*.lvt` and `payloads/*.uxz` (253 files, ~914 MB),
each referenced from the catalog by relative path.

### 3.2 The join key is the PCI ID (`SoftwareId` / `AgentlessId`)

Component identity is a PCI quad, present both in the filename and in `SoftwareId`:

```
brcm-lnvgy_fw_nic_nx1 . 14e4 . 1657 . 17aa . 402d - 227.0.3.1 ...
                         vendor  device subsys  subsys
                                        vendor  device
   → SoftwareId "DEVICE-14E4-1657-17AA-402D-13"   (14E4 = Broadcom, 17AA = Lenovo)
```

This is the **stable identifier** that joins *discovered hardware → applicable
firmware payload → target version* without model-name guessing — the same scheme
seen in `driverFiles` in the OS-driver UXSP. `Devices[].AgentlessId` (`17AA402D`)
is the short subsystem form XCC matches against live inventory.

### 3.3 Consequences for the controller

- **The catalog is self-sufficient for a dry-run offline.** Target versions
  (`Oem.Inventory[].Version`), reboot need (`ActivationMethod`), ordering
  (`Prerequisites` / `MinimumSupportedVersion`), and integrity (SHA-256 + PGP) are
  all present — a controller can diff catalog-vs-live-inventory itself, which is
  essentially what `GetRepoUpdateDetail` returns.
- **`ApplicableMachineTypes` uses machine-type codes** (`7D75`, `7D76`, …), not
  marketing names. The controller must map each fleet model → machine-type code to
  select the right bundle; conversely, one `platform` bundle spans many models (a
  single NIC entry here listed ~40 machine types).
- **XCC does the matching** at `UpdateFromRepository` time — the client still only
  supplies `RepoURI`. This schema is what makes that one-parameter call work.

## 3a. Licensing requirement — the repository path is license-gated (and the tier name changes per generation)

**`UpdateFromRepository` / repository-sync is not available on a base-licensed XCC.**
The CIFS / NFS / HTTPS repository transports are premium features, and — critically
for a mixed fleet — **the required license tier is named differently on each XCC
generation.** A controller (and fleet planning) must key the check off the XCC
generation, not a single tier name.

| XCC generation | Servers (this fleet) | Required license (repo over CIFS/NFS/HTTPS) | Exact wording |
|---|---|---|---|
| **XCC (gen 1)** | ThinkSystem V1 / V2 (e.g. SR650, SR850P, SR950) | **XCC Enterprise** (tiers: Standard / Advanced / Enterprise) | Repo doc omits it; Enterprise is the top tier that carries remote-media/repository features |
| **XCC2** | ThinkSystem V3 (SR650 V3, SR655 V3, SR675 V3, SR680a V3, SR850 V3, SR950 V3, …) | **XCC Platinum** | *"CIFS/NFS/HTTPS/Onboard Firmware History functionality requires XCC Platinum license."* |
| **XCC3** | ThinkSystem V4 (SR650 V4, SR850 V4, …) | **XCC Premier** | *"CIFS/NFS/HTTPS/Onboard Firmware History functionality requires XCC Premier license."* |

Notes that matter for the design:

- **HTTP (plain) is *not* listed as license-gated** on XCC2/XCC3 — only
  CIFS / NFS / **HTTPS** are. If a repository is served over plain HTTP, the premium
  license may not be required. This is a meaningful lever for a fleet that cannot
  license every BMC: a controller-hosted **HTTP** repository could sidestep the
  Platinum/Premier requirement (at the cost of TLS on the repo fetch — acceptable
  only on a trusted management network, and to be weighed against the PGP/SHA-256
  integrity the bundle metadata already carries, see §3.1c).
- **`SimpleUpdate` / multipart HTTP push is *not* gated by these tiers** — pushing a
  bundle to `/mfwupdate` or an image via `SimpleUpdate` does not need Platinum/Premier.
  So an unlicensed-for-repo BMC still has the **push** fallback (subject to the
  250 MB `MaxImageSizeBytes` ceiling, §7), just not the "point at a repo and let XCC
  pull the whole bundle" convenience.
- **Onboard Firmware History** (the MicroSD-backed bundle history that
  `BundleRollback` relies on) is gated by the **same** Platinum/Premier tier. So on a
  base-licensed XCC, both repo-update *and* bundle rollback are unavailable together.
- The tier is **per-BMC**, applied as a Feature-on-Demand (FoD) key. Fleet rollout
  therefore has a **licensing prerequisite step**: confirm/activate the correct tier
  (Enterprise / Platinum / Premier by generation) on every target BMC before the
  repository controller can drive it. The controller should **detect** the license
  (or detect the `UpdateFromRepository` action's absence / a license error) and
  surface a clear "BMC not licensed for repository update" condition rather than
  failing opaquely.

## 3b. HOW it works — component discovery and payload selection

This is the mechanism behind the one-parameter `UpdateFromRepository` call: what XCC
does *after* you hand it `RepoURI`, and how it lands on the right payload for each
device. All of the following is grounded in the real SR645 V3 / SR665 V3 bundle
(`lnvgy_bundle_svcpack_ka-j9ltd-01a2-24a.0_platform_comp`, machine types
`7D9A`–`7D9D`).

### 3b.1 The two inventories that get compared

A repository update is fundamentally a **diff of two inventories**:

1. **Live host inventory** — what XCC already knows about the server. XCC
   continuously enumerates the host's devices (over PCIe/SMBus/the management path)
   and tags each with a **stable hardware identity**: a PCI subsystem id
   (`vendor:device:subsysVendor:subsysDevice`) surfaced as an **`AgentlessId`**
   (e.g. `17AA402D`) and one or more **`SoftwareId`s**
   (e.g. `DEVICE-14E4-1657-17AA-402D-13`), each with a currently-installed version.
   This is the same identity Redfish exposes under `/redfish/v1/UpdateService/FirmwareInventory`.

2. **Repository (catalog) inventory** — what the bundle *offers*, read from the root
   catalog `<bundle>_index.json` → `Updates[]`, where every entry declares the
   `SoftwareId`s it can update and the version it would bring them to.

XCC matches **repository entries to live devices by `SoftwareId` / `AgentlessId`**,
then compares versions. **The PCI-subsystem id is the join key** — not the model
name, not the filename.

### 3b.2 Step-by-step, for one device

Take the Broadcom NX1 NIC in this bundle:

```
Live device (XCC inventory):  AgentlessId 17AA402D,
                              SoftwareId DEVICE-14E4-1657-17AA-402D-13 @ (say) 227.0.1.x

Catalog entry (Updates[i]):
  "Oem": { "Inventory": [ { "SoftwareId": "DEVICE-14E4-1657-17AA-402D-13",
                            "Version": "227.0.3.1" } ] }          ← match by SoftwareId
  "Inventory": "brcm-...402d-227.0.3.1-0342_anyos_comp_inventory.json"   ← descriptor
  "Locations": "brcm-...402d-...location.json"                    ← where to fetch (empty = local)
  "AuthenticitySignature": "brcm-...signature.json"              ← PGP + SHA-256 to verify
  "Payload": "payloads/firmware/brcm-...402d-227.0.3.1-034_anyos_comp.lvt"  ← the image
```

1. **Machine-type gate.** Each catalog entry (and the descriptor) carries
   `ApplicableMachineTypes`. XCC first checks its own machine type (`7D9A`…) is in
   that list; if not, the entry is skipped for this host. (The bundle-level
   `Oem.ApplicableMachineTypes` stamps the pack — here SR645 V3 / SR665 V3; the
   per-component list can be broader, so match on the specific host's type.)
2. **Identity match.** XCC finds a live device whose `SoftwareId` / `AgentlessId`
   equals the entry's `Oem.Inventory[].SoftwareId`. No live device with that id →
   the payload is irrelevant to this host and is not applied.
3. **Applicability / OOB gate.** The descriptor (`_inventory.json`) says whether the
   package can be applied out-of-band and how: `UpdateType` (`OOB`, `BMU`, or
   `Core`) and `ActivationMethod` (`Self-Contained`, `System reboot`,
   `Medium-specific reset`, `AC power cycle`, `Automatic`). XCC only applies packages
   it can drive out-of-band on this host.
4. **Version compare.** If the catalog `Version` differs from the installed version
   (and satisfies `Prerequisites` / `MinimumSupportedVersion`, which encode ordering
   — e.g. "XCC before UEFI"), the component is **queued**; otherwise it is skipped as
   already-current.
5. **Locate the payload.** The entry's `Payload` is a **path relative to the
   repository root** (`payloads/…/*.lvt|*.uxz`). For a mounted CIFS/NFS repo the
   image is read directly from that path; `Locations` (`_location.json`) supplies a
   download URL when the repo is remote/HTTP and the payload isn't co-located (it is
   `[]` in this offline pre-staged bundle).
6. **Verify then flash.** Before flashing, XCC checks the payload against
   `_signature.json` — the detached **PGP signature** ("Lenovo Group ISG RSA 4096")
   and the **SHA-256** — so a tampered or truncated image is rejected. Only then does
   it write the firmware and sequence any required reset per `ActivationMethod`.

### 3b.3 What this bundle actually contains (the gates are real)

Counts across the 305 components in this bundle show the gates above are not
hypothetical — components genuinely differ in how they apply:

- **`UpdateType`:** `BMU` ×250, `OOB` ×51, `Core` ×4 — i.e. most need the bare-metal
  updater assist (`BMU`), some are pure out-of-band, a few are core-firmware.
- **`ActivationMethod`:** `Self-Contained` ×259, `System reboot` ×28,
  `Medium-specific reset` ×11, `Automatic` ×5, `AC power cycle` ×1 — so **reboot is
  not uniform**; XCC decides per component from the descriptor, which is why the
  action takes no client reboot flag.

### 3b.4 Why the client stays thin — and what a controller can pre-compute

Because **XCC performs discovery, matching, applicability, version-diff, and
verification itself**, the Redfish client supplies only `RepoURI` (+ creds). The
"intelligence" lives in the repository metadata, not the caller.

That same metadata, however, is fully readable **offline**, so a controller does not
have to fly blind:

- It can pre-resolve the **model → machine-type code** (SR645 V3 → `7D9A`…) to pick
  the right bundle for a host.
- It can **replicate the version-diff** (catalog `Version` vs.
  `/redfish/v1/UpdateService/FirmwareInventory`) to predict what *would* change — the
  same result `GetRepoUpdateDetail` returns, but computed from the catalog without
  touching the BMC.
- It can read `ActivationMethod` up front to know **which updates force a reboot**,
  and gate host drain/evacuation accordingly (the live-ESXi concern in §8).

**In short:** discovery is by **stable PCI-subsystem identity** (`SoftwareId` /
`AgentlessId`), payload selection is a **`SoftwareId` → catalog entry → relative
`Payload` path** lookup gated by machine-type + OOB-applicability + version, and
integrity is enforced by per-payload **PGP + SHA-256** before flashing.

## 4. Full parity with Dell — the catalog model is a two-vendor solution over Redfish

The three Lenovo OEM actions map almost 1:1 onto Dell's repository actions used by
[PR #170](dell-install-from-repository.md):

| Purpose | Dell | Lenovo |
|---|---|---|
| Trigger repo update | `DellSoftwareInstallationService.InstallFromRepository` (share + `Catalog.xml`) | `LenovoUpdateService.UpdateFromRepository` (`RepoURI` + creds) |
| Dry-run / pending list | `GetRepositoryUpdateList` | `GetRepoUpdateDetail` |
| Rollback | rollback slots | `BundleRollback` |

**Consequence:** the catalog/repository model — one CR points at a per-model
firmware repository, the BMC self-inventories and applies all applicable components
— works uniformly for **Dell and Lenovo over Redfish**. The Dell PR #170 controller
shape (dry-run → apply → track jobs → converge, with rollback) is **directly
reusable for Lenovo** by swapping the OEM action names and the `RepoURI`-based
payload. **HPE remains the only vendor without a single repository action** (it
needs iLO Repository + Install Set assembly — see
[dell-install-from-repository.md](dell-install-from-repository.md) §8).

## 5. Do not confuse `UpdateFromRepository` with the `BareMetal` service

The XCC also exposes `/redfish/v1/UpdateService/Oem/Lenovo/FirmwareServices/BareMetal`
(`#LenovoFirmwareService`). **This is a different, heavier mechanism — not the
repository trigger.** Its actions are session/boot-oriented:

- `StartBareMetalConnection` (requires `ImageName` — an RDOC image uploaded to the
  XCC SFTP server; plus `TargetServerIP`, `BootTimeoutInMin`, ports)
- `RestartToBareMetal` — reboots the host into a bare-metal updater environment
- `StartDeferredUpdateActivation` — activate deferred updates (`CoreFirmwareOnly`)
- `CompleteBareMetal` (`NeedRestorePower`) — tear down the session
- `UEFIRecoveryOperation`

Its state fields (`BMAppStatus`: `NotReady`/`Booting`/`FullyBooted`/…, `Started`,
`FreeRdocSpaceInKB`, `BMU_Credential`) confirm it **boots the host into a
dedicated bare-metal update OS (BMU) via RDOC/SFTP** to flash components that
cannot be updated from the running host. It is an offline, multi-step,
reboot-into-updater path.

**For a repository-model controller, use `UpdateFromRepository` (on `UpdateService`),
not the `BareMetal` service.** The `BareMetal` path is noted here only to avoid
mis-wiring the controller to it, and because its reboot-into-a-boot-environment
behaviour is relevant to the live-ESXi reboot/boot-source concern (a host must not
be left in the BMU/PXE environment).

## 5a. Reboot orchestration — workload-agnostic, via `ServerMaintenance` (ESXi / KVM / bare-metal Linux)

`UpdateFromRepository` **has no reboot-control parameter** (§2) — the payload is only
`RepoURI` (+ creds); XCC decides per component from the bundle's `ActivationMethod`
(§3b.3: 28 of 305 components in the sample bundle were `System reboot`, one was
`AC power cycle`). So the controller cannot defer or suppress the reboot through the
API. **It must instead ensure the host is safe to reboot *before* it POSTs**, and it
must do so **without knowing what runs on the host** (ESXi, a KVM hypervisor, or a
bare-metal Kubernetes worker).

metal-operator already provides exactly this gate, and **Dell PR #170 already uses
it** — so the Lenovo controller follows the same pattern rather than inventing one.

### 5a.1 The gate: `ServerMaintenance` + owner-approval handshake

The gate is metal-operator's **`ServerMaintenance`** resource
(`metal.ironcore.dev`, short name `sm`) plus a **`ServerMaintenancePolicy`**:

| Policy | Behaviour |
|---|---|
| `OwnerApproval` | metal-operator marks the bound `ServerClaim` with `metal.ironcore.dev/maintenance-needed=true` and **blocks** until the workload owner drains and sets `metal.ironcore.dev/maintenance-approved=true`. Safe default for live hosts. |
| `Enforced` | proceeds **without** approval (optional higher `Priority` to jump the queue). For already-drained / spare servers only. |

Key `ServerMaintenanceSpec` fields (metal-operator v0.6.0): `ServerRef` (required),
`Policy`, `ServerPower`, `Priority`, `ServerBootConfigurationTemplate`. Status is a
single `State`: `Pending → InMaintenance → Failed`. When granted, the Server's own
`Status.State` becomes `Maintenance` and `Server.Spec.ServerMaintenanceRef` is set.

**Crucially, this is a *handshake, not an active drain*.** metal-operator does **not**
evacuate the workload itself (its `Server.Spec.Taints` / `TaintEffectEvict` is defined
but explicitly a **no-op** today — "reserved for future use"). The actual
drain/evacuation is done by **whoever owns the `ServerClaim`**, out-of-band, and then
signalled with the `maintenance-approved` label. That indirection is what makes it
workload-agnostic.

### 5a.2 Where ESXi / KVM / bare-metal differ — only in the approver

The firmware controller is identical for all three. Only the **approver** — the actor
that drains and then sets `maintenance-approved` — changes:

| Host type | Is it a K8s Node? | Drain step performed by the approver | Taints/tolerations? |
|---|---|---|---|
| **Bare-metal Linux K8s worker** | Yes | `kubectl drain` = cordon + `NoExecute` taint; DaemonSets kept via tolerations; honour PodDisruptionBudgets | **Yes — here.** This is the one case where K8s taints/tolerations do the work |
| **ESXi host** | No (not a Node) | vCenter: enter maintenance mode / DRS-evacuate VMs | No — vCenter drains |
| **KVM / libvirt host** | No (not a Node) | live-migrate or gracefully stop VMs | No — libvirt drains |

So Kubernetes taints/tolerations are **one drainer implementation** (the bare-metal
worker case), *not* the universal mechanism — the universal contract is the
`maintenance-approved` label on the `ServerClaim`.

> Self-relaxing case: if a Server has **no `ServerClaimRef`** (unclaimed / idle /
> spare), metal-operator grants maintenance **immediately even under
> `OwnerApproval`** — there is no workload to protect. Spare capacity updates without
> a manual approve.

### 5a.3 The controller flow (mirrors Dell PR #170)

1. **Dry-run, ungated.** Run `GetRepoUpdateDetail` (Lenovo) — read-only, never
   reboots, so it needs **no** `ServerMaintenance`. (Dell PR #170 does the same: its
   `RepositoryCheck` runs with `ApplyUpdate=false, RebootNeeded=false`.)
2. **Only if packages are pending**, create a `ServerMaintenance`
   (`ServerRef` = target, `Policy` from the CR spec, owner-referenced to the
   FirmwareUpdate CR).
3. **Gate the apply.** Block until `Server.Status.State == Maintenance` (exactly Dell
   PR #170's `handleServerMaintenance`, which refuses to proceed until
   `server.Status.State == ServerStateMaintenance`). Surface a
   `Waiting-for-approval` condition meanwhile.
4. **(Optional) stabilise the BMC** — PR #170 issues a BMC reset here to avoid BMCs
   that hang on subsequent operations (`GracefulRestartBMC`), then waits for the
   operation annotation to clear.
5. **POST `UpdateFromRepository`.** The reboot now happens on a drained host. (Dell
   ties `RebootNeeded = ApplyUpdate`, i.e. reboot is on for the apply pass only.)
6. **Track the Task/job to convergence**, then **delete the `ServerMaintenance`** —
   metal-operator cleans up the `maintenance-needed` / `maintenance-approved` labels
   and the owner restores its workload.

### 5a.4 CRD shape to mirror (from Dell `FirmwareUpdateDell`)

PR #170's `FirmwareUpdateDellSpec` carries the maintenance wiring the Lenovo CR
should copy — and deliberately **no** `rebootPolicy` / `applyTime` / `maintenanceWindow`
field (reboot is hardcoded on for the apply pass, gated by maintenance instead):

```go
// ServerMaintenanceRef references the ServerMaintenance the controller requested.
ServerMaintenanceRef   *metalv1alpha1.ObjectReference          // +optional
// ServerMaintenancePolicy — OwnerApproval | Enforced — enforced on the server.
ServerMaintenancePolicy *maintenancev1alpha1.ServerMaintenancePolicy // +optional
// ServerRef — the server to update (immutable).
ServerRef              *corev1.LocalObjectReference            // +required
```

**Net:** the firmware controller stays hypervisor-blind — request `ServerMaintenance`,
wait for `Maintenance` state, POST, delete — and the ESXi / KVM / K8s-worker specifics
live entirely in the **approver** that satisfies the `maintenance-approved` handshake.

## 6. Correction of the earlier conclusion

An earlier analysis ([lenovo-xcc-repository.md](lenovo-xcc-repository.md) §1)
concluded that Lenovo's repository update was **CLI/GUI (`syncrep`) only, not
Redfish**, based on (a) the XCC2 documentation page and (b) Lenovo's public
`update_firmware.psm1` example, which implements only `SimpleUpdate` and
`HttpPush`. **Both sources were incomplete.** The live XCC — and its self-served
JSON schema — prove the `UpdateFromRepository` Redfish OEM action exists with a
defined payload.

**Design lesson:** trust the **actual BMC's Redfish tree and its `/redfish/v1/JsonSchemas`**
over published docs and example scripts. Lenovo ships OEM actions that are not
fully surfaced in public documentation; querying a real device is authoritative.

## 7. Other facts from the live XCC (`GET /redfish/v1/UpdateService`)

- `MaxImageSizeBytes: 250000000` (~250 MB) — the multipart-push (`/mfwupdate`)
  size ceiling. This is why the download page warns a full ~1 GB bundle "cannot be
  processed by XCC directly." **`UpdateFromRepository` sidesteps this** — it points
  at a repository rather than pushing the bundle into XCC flash.
- `MultipartHttpPushUri: /mfwupdate` with `OperationApplyTimeSupport`:
  `Immediate` / `OnReset` / `OnStartUpdateRequest` — **staging is supported**
  (relevant to a gated-reboot design).
- `HttpPushUri: /fwupdate`; `SimpleUpdate` present (TransferProtocols TFTP / SFTP /
  HTTPS / HTTP; `Targets@Redfish.AllowableValues` includes `FirmwareInventory/BMC-Backup`).
- `BundleRepoAvailableSpaceInKB: ~2 GB`; `Oem.Lenovo.FirmwareServices` collection
  present.

## 8. Implications for the controller design

- A **repository-based controller** (à la Dell PR #170) is viable for **Dell and
  Lenovo** over Redfish. Payload differs (Dell: share + `Catalog.xml`; Lenovo:
  `RepoURI` + creds) but the state machine is the same.
- **The per-model firmware repository** (one repo per model+generation) aligns with
  the OCI-baseline granularity in [firmware-baseline-oci.md](firmware-baseline-oci.md):
  `RepoURI` would point at the per-model repo the controller resolves from the
  host's `kubernetes.metal.cloud.sap/type` label.
- **Reboot orchestration is solved by `ServerMaintenance` (§5a)** — not by an API
  parameter (there is none). The controller requests a `ServerMaintenance`
  (`OwnerApproval` for live hosts, `Enforced` for drained/spare), waits for
  `Server.Status.State == Maintenance`, then POSTs — the exact Dell PR #170 pattern.
  This makes reboot handling **workload-agnostic** (ESXi / KVM / bare-metal K8s
  worker): the per-workload drain lives in whatever **approves** the maintenance, and
  K8s taints/tolerations are only the bare-metal-worker drainer, not the universal
  mechanism.
- **Boot-source** remains a production concern (live ESXi): repository updates reboot
  the host and the `BareMetal` path can boot into a non-disk environment — the
  controller (and metal-operator's maintenance boot config) must guarantee the host
  returns to its production OS (the boot-from-disk requirement in
  [firmware-update-design.md](firmware-update-design.md) §7a).
- **Licensing is a hard prerequisite (§3a).** The repository path needs XCC
  **Enterprise (gen 1) / Platinum (XCC2, V3) / Premier (XCC3, V4)** depending on the
  BMC generation. The controller must treat "BMC licensed for repository update" as a
  precondition — detect it (or detect a license error / a missing
  `UpdateFromRepository` action) and report a clear condition, and fleet onboarding
  must include activating the correct FoD tier per BMC. Plain **HTTP** repositories
  and the `SimpleUpdate`/multipart **push** path are the unlicensed fallbacks.
- **Open items to verify next on a live XCC:** the exact `GetRepoUpdateDetail`
  request/response (dry-run shape), the Task/job status structure a repo update
  reports, whether `UpdateFromRepository` honours an apply-time / deferral option for
  staging, and **how a license error surfaces** (HTTP status / Redfish
  `MessageId`) when the action is invoked on a base-licensed XCC.

## 9. References

- Live XCC `GET /redfish/v1/UpdateService` + `/redfish/v1/schemas/LenovoUpdateService.v1_0_0.json` (device-captured)
- Repository JSON schema (§3) characterised from the real bundle
  `lnvgy_bundle_svcpack_ka-j9ltd-01a2-24a.0_platform_comp` (System Firmware / Platform
  Bundle `24a.0`, 2024-03-01; 305 components, JSON metadata)
- Lenovo XCC2 — Update From Repository (license: *"CIFS/NFS/HTTPS/Onboard Firmware
  History functionality requires XCC **Platinum** license."*):
  <https://pubs.lenovo.com/xcc2/updating_firmware_repository>
- Lenovo XCC3 — Update From Repository (license: *"CIFS/NFS/HTTPS/Onboard Firmware
  History functionality requires XCC **Premier** license."*):
  <https://pubs.lenovo.com/xcc3/updating_firmware_repository>
- Lenovo XCC (gen 1) — Update From Remote Repository (tiers Standard / Advanced /
  **Enterprise**): <https://pubs.lenovo.com/xcc/updating_firmware_repository>
- Lenovo Press — XCC support / license tiers on ThinkSystem servers:
  <https://lenovopress.lenovo.com/lp0880-xcc-support-on-thinksystem-servers>
- Lenovo XCC REST API — UpdateService: <https://pubs.lenovo.com/xcc-restapi/resource_updateservice>
- Companion: [dell-install-from-repository.md](dell-install-from-repository.md),
  [lenovo-xcc-repository.md](lenovo-xcc-repository.md),
  [firmware-baseline-oci.md](firmware-baseline-oci.md).
