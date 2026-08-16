<!--
SPDX-FileCopyrightText: 2025 SAP SE or an SAP affiliate company and IronCore contributors
SPDX-License-Identifier: Apache-2.0
-->

# Design: firmware baselines as OCI artifacts

**Status:** Design / proposal
**Date:** 2026-06-14
**Layer:** Sits **on top of** the per-component firmware controllers
(`NICVersion`, `BIOSVersion`, …). Does not change how firmware is applied — only
how Ops *declares intent* and how firmware is *stored and distributed*.

**Companion documents:**
[firmware-update-design.md](firmware-update-design.md) (the SimpleUpdate engine) ·
[component-firmware-proposal.md](component-firmware-proposal.md) (the upstream CRD ask) ·
[nic-discovery-findings.md](nic-discovery-findings.md).

---

## 1. The problem this solves — Ops ergonomics

Today (and in the base design) a `FirmwareUpdate`/`*Version` manifest requires Ops
to type, **per component**, a target version *and* a firmware path:

```yaml
components:
  - type: NIC
    modelSelector: cx6dx
    targetVersion: "22.35.10.12"
    image: { URI: "https://fw-repo/.../cx6dx-22.35.10.12.bin" }
  - type: BIOS
    targetVersion: "2.19.0"
    image: { URI: "https://fw-repo/.../bios-2.19.0.exe" }
  # … one block per component …
```

**Ops does not want to type versions and paths.** They want to edit a Git manifest
and name a single **baseline**:

```yaml
spec:
  buildingBlockSelector: { matchLabels: { kubernetes.metal.cloud.sap/bb: bb085 } }
  firmwareBaseline: "ghcr.io/myorg/firmware-baselines/compute:2026-Q1"
```

The versions and artifact locations move **out of the CR** and **into the
baseline**. This document defines the baseline as an **OCI artifact**.

## 2. What a baseline is

A baseline is a named, versioned, signed **manifest-of-manifests**: it maps every
component type to the firmware version and binary that constitute a tested,
known-good fleet state. Conceptually it is what HPE's SPP, Dell's catalog, and
Lenovo's UpdateXpress bundle are — a curated set — but expressed as an
OCI-native artifact rather than a vendor ISO.

Example logical content of `compute:2026-Q1`:

```
cx6dx     → version 22.35.10.12, blob <digest>
cx5ex     → version 16.35.30.06, blob <digest>
bcm57508  → version 23.31.18.10, blob <digest>
i350      → version 22.5.7,      blob <digest>
bios      → version 2.19.0,      blob <digest>
```

## 3. Why OCI artifact (vs. Git file, vs. ISO)

| Property | OCI artifact | plain Git file | vendor ISO/SPP |
|---|---|---|---|
| Immutable, digest-pinned | ✅ | partial | ✅ |
| Signed + attestable (cosign/SBOM) | ✅ | manual | vendor-signed only |
| Carries the firmware binaries too | ✅ (as blobs) | ✗ (paths only) | ✅ |
| GitOps-native distribution (Flux/ArgoCD) | ✅ | ✅ | ✗ |
| Uses existing registry plumbing/RBAC | ✅ | n/a | ✗ |
| Vendor-neutral | ✅ | ✅ | ✗ (per vendor) |

The **baseline itself is stored as an OCI artifact** (baseline-as-artifact): an OCI
manifest listing the components + versions, with each firmware binary as a
referenced OCI **blob**. So one signed, tagged reference (`…/compute:2026-Q1`)
carries both the *mapping* and the *bits*.

> **Not an ISO.** An ISO (or any bundle) is inert packaging; on its own it cannot
> decide which file applies to which component on which host. That matching is
> always done either by our controller (via Redfish discovery — our design) or by
> a BMC's own catalog engine (the vendor-console model, out of scope). The OCI
> baseline is a *storage/distribution* format, not a substitute for discovery
> (see §6).

## 4. Artifact structure (sketch)

Using the OCI image-manifest with a firmware-specific artifact type:

```jsonc
// manifest — artifactType marks it as a firmware baseline
{
  "schemaVersion": 2,
  "artifactType": "application/vnd.metal.ironcore.firmware.baseline.v1+json",
  "config": { "mediaType": "application/vnd.metal.ironcore.firmware.baseline.config.v1+json",
              "digest": "sha256:…" },       // the component→version map
  "layers": [
    { "mediaType": "application/vnd.metal.ironcore.firmware.blob",
      "digest": "sha256:…", "size": 4194304,
      "annotations": { "firmware.metal.ironcore.dev/component": "cx6dx",
                       "firmware.metal.ironcore.dev/version":   "22.35.10.12" } },
    { "mediaType": "application/vnd.metal.ironcore.firmware.blob",
      "digest": "sha256:…", "size": 8388608,
      "annotations": { "firmware.metal.ironcore.dev/component": "bios",
                       "firmware.metal.ironcore.dev/version":   "2.19.0" } }
    // … one layer per component firmware …
  ]
}
```

The `config` blob is the human-authored **baseline map** (component token → version
→ which layer). A new baseline = build this artifact, sign it, push it, tag it.

## 5. Resolution flow (what the controller does)

The manifest names a baseline; the controller does the rest:

```
Ops CR:  firmwareBaseline: ghcr.io/myorg/firmware-baselines/compute:2026-Q1
        │
1. Resolve — pull the baseline OCI artifact; verify its signature (cosign);
             read the config → { cx6dx: 22.35.10.12, bios: 2.19.0, … }
        │
2. Per host — DISCOVER via Redfish FirmwareInventory (unchanged, §6):
             which components exist here, their current versions, Targets @odata.id
        │
3. Match — intersect discovered components with the resolved baseline:
             host has a cx6dx at 22.31 → baseline says 22.35 → needs update
             host has no cx5ex → baseline's cx5ex entry is irrelevant here
        │
4. Serve — extract the needed firmware blob from the artifact, verify digest,
           place it on a BMC-reachable HTTPS endpoint (BMC cannot pull OCI)
        │
5. Apply — SimpleUpdate with image.URI = that HTTPS URL, Targets = discovered @odata.id
           (staging + gated reboot exactly as firmware-update-design.md)
```

Steps 2–5 are the **existing** design. The baseline layer adds only steps 1 and 3
(resolve + match) — everything below stays the same.

## 6. What the baseline does NOT do — discovery is still required

This is the load-bearing caveat. **A baseline is fleet-wide truth; it is not
per-host reality.** It says "the compute fleet's ConnectX-6 Dx should be at
22.35.10.12." It does **not** know:

- **which** hosts actually have a ConnectX-6 Dx (or in which slot),
- what **version** each is currently running,
- the **`@odata.id`** that `SimpleUpdate` needs in `Targets`.

Those come only from querying each host's Redfish `FirmwareInventory` at reconcile
time. So the baseline removes the **version+path typing** (the Ops win) but the
**per-host discovery stays** — it is what maps "baseline says cx6dx→22.35" onto
"this host's actual card in slot 4, currently at 22.31." Baseline and discovery are
complementary, not substitutes.

## 7. The unpack-and-serve requirement

Redfish `SimpleUpdate` is a **pull** by the BMC over HTTP(S)/TFTP; iDRAC/iLO/XCC
**cannot** speak the OCI distribution protocol. So an OCI ref can never be the
literal `image.URI`. The controller (or a sidecar/init-job it manages) must:

1. pull + verify the baseline artifact,
2. extract the needed firmware blob (digest-checked),
3. serve it on an **internal HTTPS endpoint reachable from the BMC management
   network**, and
4. hand *that* URL to `SimpleUpdate`.

If the firmware repo and the OCI registry can be fronted by the same
BMC-reachable host (unpacking on the fly), this shim is thin. Otherwise it is a
small dedicated component. This is the one genuinely new piece of infrastructure
the OCI route requires.

## 8. Relationship to the component CRDs

- **The `*Version` CRDs** (`NICVersion`, etc. — see
  [component-firmware-proposal.md](component-firmware-proposal.md)) still take a
  concrete `version` + firmware URL. They are unchanged.
- **The baseline layer resolves** a baseline name into those concrete values and
  populates/creates the `*Version` resources (or the `FirmwareUpdate` components)
  accordingly. It is an **Ops-facing convenience layer above** the executor CRDs,
  not a replacement for them.

This keeps the upstream `metal-operator` ask focused on the executor CRDs; the
baseline/OCI layer can live in `metal-maintenance-operator` (or wherever the
fleet-level orchestration lives) without upstream changes.

## 9. Open questions

1. **Baseline authoring** — who builds/signs the OCI baseline, and from what
   source (a vendor SPP/catalog cracked into blobs, or hand-curated)? This is the
   curation burden; keeping it small is what makes the model sustainable.
2. **Registry reachability** — can the BMC management network reach an HTTPS
   endpoint co-located with the registry, to keep the unpack-and-serve shim thin?
3. **Signature policy** — is cosign verification of the baseline mandatory before
   any host is touched? (Recommended: yes — it is firmware that reboots prod hosts.)
4. **Partial baselines** — does a baseline always list all component types, or may
   it be sparse (only the components being rolled this quarter)? Sparse is more
   flexible; full is more auditable.
5. **Version-vs-baseline coexistence** — should a CR be allowed to mix a
   `firmwareBaseline` with per-component overrides, or is it strictly one or the
   other? (Recommend: baseline, with optional explicit override per component.)
