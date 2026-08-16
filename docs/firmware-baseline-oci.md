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
and name a single **quarterly baseline release**:

```yaml
spec:
  buildingBlockSelector:  { matchLabels: { kubernetes.metal.cloud.sap/bb: bb085 } }
  firmwareBaselineFamily:  "ghcr.io/myorg/firmware-baselines"
  firmwareBaselineRelease: "2026-Q1"
```

The versions and artifact locations move **out of the CR** and **into the
baseline** (an **OCI artifact**, §3). Note there is **no server model in the CR**:
the controller reads each host's `kubernetes.metal.cloud.sap/type` label and
**auto-resolves** the correct per-model artifact (§2). So Ops types **one
quarterly name** and it works across a mixed fleet.

## 2. What a baseline is — one artifact per model + generation

A baseline is a named, versioned, signed **manifest-of-manifests**: it maps every
component type to the firmware version and binary that constitute a tested,
known-good state. Conceptually it is what HPE's SPP, Dell's catalog, and Lenovo's
UpdateXpress bundle are — a curated set — but OCI-native rather than a vendor ISO.

### The unit is the server model + generation, **not** the whole fleet

Firmware is **not portable across models/generations** — especially BIOS and BMC
(a Dell R740XD BIOS ≠ R760 ≠ HPE DL360 Gen10 ≠ Gen11; iDRAC ≠ iLO ≠ XCC). A single
"all models, all components" artifact would be a grab-bag of incompatible binaries,
and a CR would still have to select the slice for each host. So a baseline is
scoped to **one hardware type**, containing only the components that type has:

```
ghcr.io/myorg/firmware-baselines/dell-r740xd:2026-Q1
   bios      → 2.19.0        blob <digest>
   idrac     → <ver>         blob <digest>
   cx6dx     → 22.35.10.12   blob <digest>
   i350      → 22.5.7        blob <digest>

ghcr.io/myorg/firmware-baselines/hpe-dl360gen11:2026-Q1
   systemrom → <ver>         blob <digest>
   ilo       → <ver>         blob <digest>
   cx6dx     → 22.35.10.12   blob <digest>

ghcr.io/myorg/firmware-baselines/lenovo-sr650v3:2026-Q1
   uefi      → <ver>         blob <digest>
   xcc       → <ver>         blob <digest>
   bcm57508  → <ver>         blob <digest>
```

- **`2026-Q1` is a tag convention shared across the per-model family** — the
  quarterly release, not a single blob. Cutting a new release re-tags each
  per-model artifact `2026-Q2`, etc.
- Each per-model artifact is a **self-consistent, tested set** for that hardware —
  the only level at which "all these components at these versions" is a meaningful,
  auditable claim.
- This aligns with two facts already in the design: a **building block is
  single-model** (R3), so one CR → one model's baseline; and argora already stamps
  the model on each host as `kubernetes.metal.cloud.sap/type`, which is the key the
  controller resolves against (§5).
- **Shared components (e.g. `cx6dx`) may repeat across per-model artifacts** —
  acceptable; the OCI registry deduplicates identical blobs by digest, so the same
  NIC firmware is stored once regardless of how many baselines reference it.

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
referenced OCI **blob**. So one signed, tagged reference
(`…/dell-r740xd:2026-Q1`) carries both the *mapping* and the *bits* for one model.

> **Not an ISO.** An ISO (or any bundle) is inert packaging; on its own it cannot
> decide which file applies to which component on which host. That matching is
> always done either by our controller (via Redfish discovery — our design) or by
> a BMC's own catalog engine (the vendor-console model, out of scope). The OCI
> baseline is a *storage/distribution* format, not a substitute for discovery
> (see §6).

## 4. Artifact structure — a worked example

Every OCI artifact is three kinds of things, all stored in the registry and
addressed by SHA-256 digest: **blobs** (the big files), a **config** (small JSON
describing what it is), and a **manifest** (the index tying them together). The tag
(`dell-r740xd:2026-Q1`) is a human-friendly pointer to the manifest's digest.

Worked example for `ghcr.io/myorg/firmware-baselines/dell-r740xd:2026-Q1`.

### 4.1 Blobs — the firmware binaries (stored as-is)

| Blob (by digest) | What it is | Size |
|---|---|---|
| `sha256:a1b2…` | `cx6dx-22.35.10.12.bin` (ConnectX-6 Dx) | 4 MB |
| `sha256:c3d4…` | `i350-22.5.7.bin` (Intel I350) | 512 KB |
| `sha256:e5f6…` | `bios-r740xd-2.19.0.bin` | 8 MB |

Raw bytes — the same files you'd otherwise host on an HTTP server, deduplicated by
hash across baselines.

### 4.2 Config — the baseline map (the human-authored part)

```json
{
  "artifactType": "application/vnd.metal.ironcore.firmware.baseline.v1+json",
  "baseline": "dell-r740xd-2026-Q1",
  "serverType": "poweredge-r740xd",
  "components": [
    { "token": "cx6dx", "component": "NIC",  "version": "22.35.10.12", "blobDigest": "sha256:a1b2…" },
    { "token": "i350",  "component": "NIC",  "version": "22.5.7",      "blobDigest": "sha256:c3d4…" },
    { "token": "bios",  "component": "BIOS", "version": "2.19.0",      "blobDigest": "sha256:e5f6…" }
  ]
}
```

This is the answer to *"what does the OCI contain?"*: the firmware binaries **plus a
map** stating which binary is which component at which version. `serverType` is what
the controller matches against the host's `…/type` label (§5).

### 4.3 Manifest — the index

Layer `annotations` carry the mapping so it can be read without pulling the config:

```jsonc
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "artifactType": "application/vnd.metal.ironcore.firmware.baseline.v1+json",
  "config": {
    "mediaType": "application/vnd.metal.ironcore.firmware.baseline.config.v1+json",
    "digest": "sha256:9f8e…", "size": 412            // the §4.2 JSON
  },
  "layers": [
    { "mediaType": "application/vnd.metal.ironcore.firmware.blob",
      "digest": "sha256:a1b2…", "size": 4194304,
      "annotations": { "component": "cx6dx", "version": "22.35.10.12",
                       "org.opencontainers.image.title": "cx6dx-22.35.10.12.bin" } },
    { "mediaType": "application/vnd.metal.ironcore.firmware.blob",
      "digest": "sha256:e5f6…", "size": 8388608,
      "annotations": { "component": "bios", "version": "2.19.0",
                       "org.opencontainers.image.title": "bios-r740xd-2.19.0.bin" } }
    // … one layer per component …
  ]
}
```

Signing with cosign adds a fourth thing: a signature artifact attached to this
manifest's digest, so the controller can verify the bytes are untampered.

### 4.4 Building it (real commands, via `oras`)

```sh
oras push ghcr.io/myorg/firmware-baselines/dell-r740xd:2026-Q1 \
  --artifact-type application/vnd.metal.ironcore.firmware.baseline.v1+json \
  --config baseline-map.json:application/vnd.metal.ironcore.firmware.baseline.config.v1+json \
  cx6dx-22.35.10.12.bin:application/vnd.metal.ironcore.firmware.blob \
  i350-22.5.7.bin:application/vnd.metal.ironcore.firmware.blob \
  bios-r740xd-2.19.0.bin:application/vnd.metal.ironcore.firmware.blob

cosign sign ghcr.io/myorg/firmware-baselines/dell-r740xd:2026-Q1   # recommended
```

Cutting next quarter's baseline = new versions/blobs, re-push as `…:2026-Q2`.

## 5. Resolution flow (what the controller does)

The manifest names a baseline; the controller does the rest:

```
Ops CR:  firmwareBaselineFamily: ghcr.io/myorg/firmware-baselines
         firmwareBaselineRelease: 2026-Q1
        │
0. Resolve model — read host's kubernetes.metal.cloud.sap/type label
             (e.g. poweredge-r740xd) → artifact ref
             ghcr.io/myorg/firmware-baselines/dell-r740xd:2026-Q1
        │
1. Resolve baseline — pull that OCI artifact; verify its signature (cosign);
             read the config → { cx6dx: 22.35.10.12, i350: 22.5.7, bios: 2.19.0 }
        │
2. Per host — DISCOVER via Redfish FirmwareInventory (unchanged, §6):
             which components exist here, their current versions, Targets @odata.id
        │
3. Match — intersect discovered components with the resolved baseline:
             host has a cx6dx at 22.31 → baseline says 22.35 → needs update
             host has no cx5ex → not in this model's baseline anyway
        │
4. Serve — extract the needed firmware blob from the artifact, verify digest,
           place it on a BMC-reachable HTTPS endpoint (BMC cannot pull OCI)
        │
5. Apply — SimpleUpdate with image.URI = that HTTPS URL, Targets = discovered @odata.id
           (staging + gated reboot exactly as firmware-update-design.md)
```

Steps 2–5 are the **existing** design. The baseline layer adds only steps 0, 1 and
3 (resolve model → resolve baseline → match) — the actual apply is unchanged.

## 6. What the baseline does NOT do — discovery is still required

This is the load-bearing caveat. **A baseline is per-model truth; it is not
per-host reality.** It says "an R740XD's ConnectX-6 Dx should be at 22.35.10.12." It
does **not** know:

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
   curation burden; keeping it small is what makes the model sustainable. Note the
   per-model scoping (§2) means **one artifact per model+generation per release** —
   more artifacts to author, but each maps cleanly to a vendor's per-platform SPP.
6. **Model-label mapping** — the controller resolves the host's
   `kubernetes.metal.cloud.sap/type` slug (e.g. `poweredge-r740xd`) to a baseline
   repo name (`dell-r740xd`). Is that a direct slug match, or is a small
   type→baseline mapping table needed? A host whose `type` has no matching baseline
   in the release must be surfaced (Blocked/warning), not silently skipped.
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
