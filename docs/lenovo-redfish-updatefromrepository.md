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
  "RepoURI":      "sftp://10.x.x.x/firmware/sr650v3",   // REQUIRED — repository server address
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
- **Reboot / boot-source** remains the open production concern (live ESXi):
  repository updates reboot the host, and the `BareMetal` path can boot into a
  non-disk environment — the controller must guarantee the host returns to its
  production OS (the same boot-from-disk requirement as
  [firmware-update-design.md](firmware-update-design.md) §7a).
- **Open items to verify next on a live XCC:** the exact `GetRepoUpdateDetail`
  request/response (dry-run shape), the Task/job status structure a repo update
  reports, and whether `UpdateFromRepository` honours an apply-time / deferral
  option for staging.

## 9. References

- Live XCC `GET /redfish/v1/UpdateService` + `/redfish/v1/schemas/LenovoUpdateService.v1_0_0.json` (device-captured)
- Repository JSON schema (§3) characterised from the real bundle
  `lnvgy_bundle_svcpack_ka-j9ltd-01a2-24a.0_platform_comp` (System Firmware / Platform
  Bundle `24a.0`, 2024-03-01; 305 components, JSON metadata)
- Lenovo XCC2 — Update From Repository: <https://pubs.lenovo.com/xcc2/updating_firmware_repository>
- Lenovo XCC3 — Update From Repository: <https://pubs.lenovo.com/xcc3/updating_firmware_repository>
- Lenovo XCC REST API — UpdateService: <https://pubs.lenovo.com/xcc-restapi/resource_updateservice>
- Companion: [dell-install-from-repository.md](dell-install-from-repository.md),
  [lenovo-xcc-repository.md](lenovo-xcc-repository.md),
  [firmware-baseline-oci.md](firmware-baseline-oci.md).
