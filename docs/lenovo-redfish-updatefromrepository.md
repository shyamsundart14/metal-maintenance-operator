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

## 3. What the repository must contain

Per the XCC "Update From Repository" documentation, the repository (CIFS / NFS /
HTTP / HTTPS reachable at `RepoURI`) holds the **Update Bundle / UXSP** contents —
**metadata files plus firmware payloads**. For CIFS/NFS mounts the metadata is
placed at the **root** of the share, and the payloads are referenced from it. XCC
**parses the metadata, self-inventories the host, and applies only the packages
applicable to this machine type that support out-of-band (OOB) update** — the
client provides no per-component list. (Format detail — plain UXSP metadata is
XML/CIM on older generations, JSON on XCC3; see
[lenovo-xcc-repository.md](lenovo-xcc-repository.md).)

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
- Lenovo XCC2 — Update From Repository: <https://pubs.lenovo.com/xcc2/updating_firmware_repository>
- Lenovo XCC3 — Update From Repository: <https://pubs.lenovo.com/xcc3/updating_firmware_repository>
- Lenovo XCC REST API — UpdateService: <https://pubs.lenovo.com/xcc-restapi/resource_updateservice>
- Companion: [dell-install-from-repository.md](dell-install-from-repository.md),
  [lenovo-xcc-repository.md](lenovo-xcc-repository.md),
  [firmware-baseline-oci.md](firmware-baseline-oci.md).
