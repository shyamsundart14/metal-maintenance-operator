<!--
SPDX-FileCopyrightText: 2025 SAP SE or an SAP affiliate company and IronCore contributors
SPDX-License-Identifier: Apache-2.0
-->

# Lenovo XCC repository-based firmware update

**Status:** Analysis / design record
**Date:** 2026-08-24
**Source:** [Lenovo XCC2 — Update From Repository](https://pubs.lenovo.com/xcc2/updating_firmware_repository)
(verified against the vendor documentation).

Companion to [dell-install-from-repository.md](dell-install-from-repository.md)
(the Dell `InstallFromRepository` / PR #170 approach) and
[firmware-update-design.md](firmware-update-design.md) (the per-component
`SimpleUpdate` approach). This document records how Lenovo ThinkSystem servers
perform a repository/bundle firmware update via XCC, so the catalog-model design
can be evaluated per vendor accurately.

---

## 1. Summary — two distinct mechanisms

Lenovo XCC has a "update all firmware from a bundle/repository" capability, but it
exists in **two forms**, and they matter for how a controller would drive it:

| # | Mechanism | Interface | Client's job |
|---|---|---|---|
| **1** | **Multipart bundle push** | **Redfish** (`/mfwupdate`) | Push the whole UXSP/Update Bundle *to* XCC |
| **2** | **Remote repository sync** (`syncrep`) | **CLI / WebGUI** (Redfish not documented) | Point XCC at a remote CIFS/NFS/HTTP repo; XCC pulls |

**Both** rely on XCC to self-inventory the host against the bundle's metadata and
apply only the packages that are applicable and support out-of-band (OOB) update —
i.e. both are the **vendor-catalog model** (the BMC decides what to apply, not the
client). The difference is *how the bundle reaches XCC* and *which interface
triggers it*.

> **Correction to note.** An earlier comparison characterised Lenovo as "fitting
> the Dell `InstallFromRepository` pattern cleanly." The verified position is more
> careful: via **Redfish**, the reliable Lenovo path is the **bundle *push***
> (mechanism 1), not a Dell-style "point XCC at a remote share" action. The
> repository-*sync* mode (mechanism 2) that would match Dell most closely appears
> to be **CLI/WebGUI only** — the doc does not expose a Redfish endpoint or OEM
> action for it.

## 2. Mechanism 1 — multipart bundle push (Redfish)

The documented Redfish path. The client pushes the entire Update Bundle to XCC's
multipart update URI:

```sh
curl -k -u USER:PASS \
  -F 'UpdateFile=@./bundle.zip;type=application/octet-stream' \
  https://<xcc-ip>:443/mfwupdate
```

- `UpdateService.MultipartHttpPushUri` on XCC = **`/mfwupdate`**.
- The **Update Bundle (Service Pack / UXSP)** is a compressed archive of **JSON
  metadata files + payload binaries**. The metadata tells XCC which firmware
  images the bundle contains; XCC reserves ~2 GB of flash and cleans up old data
  when a new bundle arrives.
- XCC **parses the metadata, self-inventories the host, and batch-updates all
  applicable firmware** — both OOB and in-band (IB) packages: UEFI, XCC, PCI
  devices (incl. NICs), storage controllers, drives, etc.

**Job/status tracking:**

```
POST bundle → /redfish/v1/TaskService/Tasks/{TaskId}     (transfer + verify)
            → /redfish/v1/JobService/Jobs/{JobId}          (the update job)
            → /redfish/v1/JobService/Jobs/{JobId}/Steps/{StepName}   (per component)
```

Status properties include `TaskState` and `PercentComplete`; each component reports
its own step (e.g. a per-device success message).

## 3. Mechanism 2 — remote repository sync (`syncrep`, CLI/GUI)

This is the closest analogue to Dell's `InstallFromRepository` — XCC pulls from a
remote repository and updates autonomously — **but it is documented via WebGUI and
CLI, not Redfish.**

- **WebGUI:** enter the remote repository details → **Connect** → **Update**; XCC
  parses the metadata and applies applicable packages.
- **CLI (`syncrep`):**
  ```
  syncrep -t samba -l <url> -u <user> -p <password>
  syncrep -t http  -l http://<ip>/bundle.tgz
  syncrep -t local -l /bulk/bundle.tgz
  ```
- Supported repository transports: **CIFS/NFS/HTTP/HTTPS** (some require the XCC
  Platinum license, per the doc).

**Redfish gap:** the documentation gives **no Redfish endpoint, OEM extension, or
`UpdateFromRepository`-style action** for configuring/triggering the remote-repo
sync. So a Redfish-only controller likely **cannot** drive mechanism 2 as
documented — it would use mechanism 1 (push the bundle), or drive `syncrep`
out-of-band.

## 4. Component coverage

Both mechanisms update **all applicable firmware** — XCC selects packages from the
bundle metadata that (a) match the specific hardware and (b) support OOB update.
Coverage spans UEFI, XCC, PCI/NIC, storage controllers, and drives (OOB and IB).
As with Dell, this is comprehensive **but** the client exercises **no
per-component or per-version control** — XCC decides from the bundle.

## 5. Reboot handling

XCC manages reboot sequencing itself. The doc notes only that *"if the update
requires the XClarity Controller to be restarted … a warning message will be
displayed"* — there is **no client-supplied reboot parameter**. As with any
catalog-model update, a controller must assume reboots happen and coordinate host
drain/evacuation externally (the same live-ESXi concern raised for the Dell
approach — see [dell-install-from-repository.md](dell-install-from-repository.md) §9).

## 6. Implications for a catalog-model controller

Relative to the Dell `InstallFromRepository` design (PR #170):

- **The philosophy transfers** — hand XCC a bundle, it self-inventories and applies
  everything applicable. So a `FirmwareUpdateLenovo` is conceptually feasible.
- **The Redfish mechanics differ** — Dell = point at a share (one OEM action);
  Lenovo = **push the whole bundle** to `/mfwupdate` (multipart). A controller
  must **host/stage the bundle and push it**, not just pass a catalog URL.
- **The Dell-closest mode (repo sync) is not Redfish-documented** for Lenovo, so
  don't assume "point XCC at a share via Redfish" works — plan for the push path.
- **Status shape differs** — Lenovo uses `JobService/Jobs/{id}/Steps/{StepName}`;
  Dell uses iDRAC job IDs diffed against a baseline. A per-vendor status mapping is
  needed regardless.

### Three-vendor spectrum ("how much the BMC does for you")

```
Dell   — point iDRAC at a share + Catalog.xml (one OEM action)         ← most BMC-driven
Lenovo — push a UXSP bundle to /mfwupdate; XCC self-selects & applies
HPE    — upload each Smart Component + .compsig, build an Install Set   ← least BMC-driven
```

All three self-apply and self-reboot; the **input granularity the client must
provide increases** left→right. A single "catalog controller" abstraction is
possible, but each vendor needs its own delivery adapter (share-pointer vs.
bundle-push vs. install-set assembly).

## 7. References

- Lenovo XCC2 — Update From Repository: <https://pubs.lenovo.com/xcc2/updating_firmware_repository>
- Lenovo XCC3 — Update From Repository: <https://pubs.lenovo.com/xcc3/updating_firmware_repository>
- Lenovo XCC AMD — Update From Remote Repository: <https://pubs.lenovo.com/xcc-amd/updating_firmware_repository>
- Lenovo XCC REST API — UpdateService: <https://pubs.lenovo.com/xcc-restapi/resource_updateservice.html>
- Companion: [dell-install-from-repository.md](dell-install-from-repository.md),
  [firmware-update-design.md](firmware-update-design.md).
