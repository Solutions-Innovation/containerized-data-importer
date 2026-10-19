# NBD Source — High-Level Design

A new CDI `DataVolume` source type, `nbd`, that populates a target PVC from an
arbitrary NBD export.

This document describes the architecture and the interface contracts only.
Implementation details are deliberately out of scope.

## 1. Purpose and Scope

CDI owns exactly one step of a disk migration pipeline: **write the content of
an NBD export into a target PVC**.

Everything else — where the disk data comes from, which regions belong to the
current stage, when to cut over — is decided outside of CDI.

The design has two goals:

1. Let CDI consume *any* NBD export, instead of treating NBD as an internal
   implementation detail of the `vddk` source.
2. Define the contracts between CDI and its neighbours, so that the source-side
   publisher and the orchestration layer can evolve independently.

### 1.1 Roles

```
┌──────────────────────────────────────────────────────────────────────┐
│ Orchestration layer (migration control plane, separate project)      │
│   · Decides the stage: full / precopy / cutover                      │
│   · Asks the publisher to expose this stage's NBD export             │
│   · Creates or updates the DataVolume with this stage's NBD URI      │
│   · Keeps cross-stage state (ledger, synced ranges, ACK decisions)   │
└───────────────┬──────────────────────────────────┬───────────────────┘
                │ ① request export                 │ ③ hand over URI
                ▼                                  ▼
┌───────────────────────────────┐      ┌──────────────────────────────┐
│ Source publisher (separate)   │      │ CDI (scope of this design)   │
│   · Full stage: whole disk    │      │   · Connects to the NBD URI  │
│   · Precopy stage: delta      │─②──▶ │   · Queries extent tri-state │
│   · Expresses "needs copy"    │ NBD  │   · Writes per tri-state     │
│     through base:allocation   │      │   · Makes no stage decisions │
└───────────────────────────────┘      └───────────────┬──────────────┘
                                                       │ ④ write
                                                       ▼
                                              ┌─────────────────┐
                                              │  Target PVC     │
                                              │ (kept across    │
                                              │  stages)        │
                                              └─────────────────┘
```

### 1.2 Explicitly Out of Scope

| Concern | Owner |
|---|---|
| How the source computes "changed regions" (snapshots, CBT, bitmaps) | Source publisher |
| Creation and lifecycle of full / delta exports | Source publisher |
| Stage scheduling, ledger, persistence of synced ranges | Orchestration layer |
| Cutover timing and ACK logic | Orchestration layer |
| nbdkit plugins | Not needed — see §4.1 |

CDI is unaware of which stage is running and of what "changed" means. It
faithfully consumes what the source expresses over the standard NBD protocol.

## 2. Why a New Source Type

CDI currently supports `http`, `s3`, `gcs`, `registry`, `pvc`, `upload`,
`blank`, `imageio`, `vddk` and `snapshot`. NBD is not among them — it exists
only *inside* the `vddk` source, where the importer pod starts `nbdkit` with a
VDDK plugin and reads the resulting Unix socket through libnbd.

An alternative would be to extend the `vddk` source with an external NBD
connection option. That approach keeps NBD positioned as a *side channel for
reading VDDK data*: the vCenter session, the capacity check and the changed-area
query all remain vCenter-bound, which makes it unusable for non-vCenter sources.

This design positions NBD differently: **as an independent, self-describing data
source**. The extent information and the data itself come from the same NBD
connection, so no second authority (vSphere API, storage API) is involved.

## 3. End-to-End Behaviour

The source type supports two semantics, selected by a single boolean.

### 3.1 Full Stage

The target becomes byte-for-byte identical to the source.

```
Orchestration        Publisher             CDI importer         Target PVC
  │ request full export    │                     │                   │
  ├───────────────────────▶│                     │                   │
  │◀───────────────────────┤ nbd://.../disk-full │                   │
  │ create DataVolume      │                     │                   │
  │ (url=..., fullCopy=true)                     │                   │
  ├─────────────────────────────────────────────▶│                   │
  │                        │◀── connect, extents ┤                   │
  │                        │                     │ data → copy       │
  │                        │                     │ zero → write zero │
  │                        │                     │ hole → write zero │
  │                        │                     ├──────────────────▶│
  │◀────────── DataVolume Succeeded ─────────────┤                   │
```

Holes are zeroed explicitly, not skipped. For `volumeMode: Block` targets CDI's
pre-import cleanup is a no-op, so any data left on the device from a previous
use would otherwise survive inside the imported disk. Zeroing is required for
correctness, not an optional optimisation.

### 3.2 Precopy (Incremental) Stage

Only the regions the source marks as "needs transferring" are modified. The rest
of the target keeps whatever earlier stages wrote.

```
Orchestration        Publisher             CDI importer         Target PVC
  │ request delta export   │                     │        (keeps previous data)
  ├───────────────────────▶│                     │                   │
  │                        │ changed → data      │                   │
  │                        │ unchanged → hole    │                   │
  │◀───────────────────────┤ nbd://.../delta-N   │                   │
  │ update DataVolume      │                     │                   │
  │ (url=delta-N, fullCopy=false)                │                   │
  ├─────────────────────────────────────────────▶│                   │
  │                        │◀── connect, extents ┤                   │
  │                        │                     │ data → copy       │
  │                        │                     │ zero → write zero │
  │                        │                     │ hole → skip       │
  │                        │                     ├──────────────────▶│
  │◀────────── DataVolume Succeeded ─────────────┤                   │
  │ record, decide next stage                    │                   │
```

The only difference between the two stages is how `hole` is treated. CDI keeps
no stage state of its own.

## 4. Key Design Decisions

### 4.1 No nbdkit Plugin

In the VDDK and NFC import paths, nbdkit plus a plugin acts as a **protocol
bridge**: upstream speaks NFC (a proprietary VMware protocol), downstream speaks
NBD, because CDI's own code only speaks NBD. The plugin never writes to the PVC;
that is always done by the Go data source.

When the source already speaks NBD, such a bridge would be an identity
transform. The layer disappears rather than being replaced.

```
VDDK / NFC:  vCenter --NFC--> [plugin translates] --NBD--> libnbd --> PVC
NBD source:  publisher --NBD-----------------------------> libnbd --> PVC
                               ↑ this layer is gone
```

A plugin becomes necessary again only if a future source does *not* speak NBD
(a proprietary snapshot API, Ceph RBD diff, Hyper-V RCT). nbdkit and qemu-nbd
exist in this architecture only on the source side.

### 4.2 Tri-State Extent Semantics

The whole design rests on distinguishing three extent states rather than two.

| Extent state | NBD flags | Meaning | Action (full) | Action (incremental) |
|---|---|---|---|---|
| `data` | none | Real data that must be transferred | copy | copy |
| `zero` | `ZERO` | Must be transferred, content is known to be all zeroes | write zeroes | write zeroes |
| `hole,zero` | `HOLE` \| `ZERO` | Not part of this stage | write zeroes | **skip** |

Merging `HOLE` and `ZERO` — as the existing VDDK copy path does, and as common
NBD copy tools do — is harmless for a full import into a fresh PVC, but in an
incremental stage it zeroes regions the source declared untouched and therefore
**destroys data written by earlier stages**. The `nbd` source therefore
implements its own tri-state write logic instead of reusing the VDDK copy loop.

### 4.3 One Code Path for Full and Incremental

The `vddk` source needs two paths, because its extent information (vSphere CBT)
and its data (NBD) come from different places, leaving a window in which
metadata and data can disagree.

The `nbd` source has no such split: extents and data come from the same
connection. CDI walks the extent map over `[0, size)` once and acts on the
tri-state. Full and incremental differ only in the `hole` branch.

### 4.4 CDI Stays Stateless

CDI records nothing about previous stages. Each import is fully determined by
the triple *(URI, fullCopy, target PVC)*.

Consequences:

- Re-running a stage is safe; writes are at fixed offsets, so repeating a stage
  is equivalent to running it once.
- The orchestration layer sets the stage cadence freely.
- No persistent state is introduced on the CDI side.

**Failure semantics.** If an incremental stage fails midway, the target is
partially updated — neither the previous consistent state nor the intended new
one. This is inherent to in-place writes at fixed offsets and cannot be hidden
behind a transaction on the CDI side. Recovery therefore belongs to the
orchestration layer: re-run the *same* stage with the same URI (idempotent), do
not skip ahead, and fall back to a full stage if that stage's export is no
longer available. CDI only guarantees that re-running a stage is safe.

## 5. Interface Contracts

Clear contracts are what allow the three parties to evolve separately.

### 5.1 Source Publisher → CDI

**C1 — `base:allocation` must be negotiable.** Without extent queries CDI cannot
obtain tri-state information and can at best degrade to a sequential whole-disk
copy.

**C2 — "Needs transferring" must be expressed through the tri-state of §4.2.**
The error-prone case is the third row: a region that is not part of this stage
must be reported as `HOLE|ZERO`. Reporting only `ZERO` tells CDI "this region
should be all zeroes", and the resulting zero-fill erases earlier stages.

**C3 — Delta exports must be full-size and sparse, not compact diffs.** CDI
writes each extent to the *same offset* in the target and performs no address
translation. Therefore a delta export must report the full virtual disk size,
place changed regions at their real offsets, and present unchanged regions as
holes.

```
Delta export (virtual size == full disk size)
0                                                              size
├────────┬──────────┬────────────────┬──────────┬──────────────────┤
│ hole   │  data    │     hole       │  data    │      hole        │
│        │ changed  │                │ changed  │                  │
└────────┴──────────┴────────────────┴──────────┴──────────────────┘
             │                           │
             ▼ written at the same offset▼
Target PVC ──┴───────────────────────────┴─────────────────────────
         (all other regions keep their previous content)
```

Self-addressing compact diff formats are out of scope: supporting them would
require an extra mapping protocol and would break the premise that CDI is a dumb
mover. A publisher that produces a delta with a backing reference must strip it;
otherwise unallocated clusters read through to the base image, get reported as
`data`, and the delta loses its meaning.

**C4 — Size must be stable across stages and must fit the target.**

| Check | Owner | Rationale |
|---|---|---|
| Size constant across stages | Orchestration layer | CDI is stateless (§4.4) and cannot compare against a previous stage |
| Size fits the target | CDI | Rejects the import rather than truncating data |
| Size differs from the requested capacity | CDI (warning only) | Actual PVC capacity is routinely larger than requested |

CDI deliberately does **not** require exact size equality. The requested
DataVolume size, the actual PVC capacity and the device's usable size may all
differ legitimately (provisioner rounding, block alignment), so a strict
equality check would produce false rejections on some storage classes.

Note also what CDI *cannot* check: there is no second authority on capacity
here, so "did I connect to the right disk?" is the orchestration layer's
responsibility.

**C5 — `pread` is idempotent for the lifetime of a connection.** The publisher
must not switch the underlying data view while an import is in flight.

**C6 — Long-lived connections are supported.** Large disks may take hours; the
publisher must not time out the connection.

### 5.2 Orchestration Layer → CDI

**C7 — One URI per stage.** The orchestration layer hands CDI the URI for the
current stage. CDI does not interpret the URI; whether the export is full or
incremental is expressed by `fullCopy`, not by the URI.

**C8 — `fullCopy` matches the stage semantics.**

| Stage | `fullCopy` | CDI behaviour |
|---|---|---|
| Full | `true` | Pre-import cleanup for filesystem volumes; holes zeroed |
| Precopy | `false` | Target content preserved; holes skipped |

**C9 — The target PVC is retained across stages.** Incremental stages must point
at the same PVC; it must not be deleted between stages.

**C10 — Stage state lives in the orchestration layer.** CDI reports only the
success or failure of the current stage through the DataVolume/PVC phase.
Whether to run another stage, cut over, or ACK is decided outside CDI.

### 5.3 CDI's Responsibility Boundary

CDI does four things: connect, query extents, write according to the tri-state,
report the result.

CDI does **not** decide which regions changed, does not track stages, and cannot
verify that a delta export corresponds to the source state the orchestration
layer believes it does. That consistency is guaranteed jointly by the publisher
and the orchestration layer.

## 6. API Design

### 6.1 DataVolume Source

```go
type DataVolumeSource struct {
    ...
    NBD *DataVolumeSourceNBD `json:"nbd,omitempty"`
}

// DataVolumeSourceNBD provides the parameters to create a Data Volume from an
// NBD export.
type DataVolumeSourceNBD struct {
    // URL is the NBD export URI (nbd:// or nbds://).
    // The caller updates this per stage. The export must report the ranges that
    // need transferring as allocated data via the base:allocation meta context,
    // and must present the full virtual disk size with real offsets.
    URL string `json:"url"`

    // SecretRef is the name of a Secret holding the client TLS material.
    // Required for nbds://.
    // +optional
    SecretRef string `json:"secretRef,omitempty"`

    // FullCopy selects whole-disk semantics: holes are zeroed so the target
    // matches the source byte for byte. This matters for block volumes, where
    // pre-import cleanup is a no-op and stale data would otherwise survive.
    // When false (incremental stage), holes are skipped and existing target
    // content is preserved. Defaults to false.
    // +optional
    FullCopy bool `json:"fullCopy,omitempty"`
}
```

Three fields are sufficient because all stage knowledge stays outside CDI. No
checkpoint list, no stage counter, no previous-snapshot reference.

Full stage:

```yaml
spec:
  source:
    nbd:
      url: "nbds://nbd-publisher.migration.svc:10809/vm42-disk0-full"
      secretRef: "nbd-tls-certs"
      fullCopy: true
  storage:
    resources:
      requests:
        storage: 32Gi
```

Precopy stage — same target PVC, only the URL and the flag change:

```yaml
spec:
  source:
    nbd:
      url: "nbds://nbd-publisher.migration.svc:10809/vm42-disk0-delta3"
      secretRef: "nbd-tls-certs"
      fullCopy: false
```

### 6.2 Validation

Admission-time checks:

- Scheme must be `nbd://` or `nbds://`; the host must be present.
- `nbds://` requires `secretRef`.

Import-time checks (in the importer):

- The negotiated export size must fit the target's usable space; otherwise the
  import is rejected before any write.
- A size differing from the requested DataVolume capacity is logged as a
  warning, not treated as an error (C4).

### 6.3 Annotations and Environment

The DataVolume source is projected onto the target PVC as annotations, and from
there into the importer pod environment, following the pattern used by the
existing sources:

| Purpose | Carrier |
|---|---|
| Source type selector | PVC annotation, value `nbd` |
| Stage NBD URI | PVC annotation → importer environment variable |
| TLS secret name | PVC annotation → mounted secret volume |
| Full/incremental switch | PVC annotation → importer environment variable |

Client TLS material is mounted into the importer pod as a dedicated volume,
consistent with how CDI already mounts endpoint and proxy certificates.

### 6.4 Internal Interface

The new source plugs into CDI's existing data source abstraction
(`DataSourceInterface`): the data processor drives the copy, the data source owns
the read-and-write loop and the target path. This is the extension point for
data sources; nothing new is introduced.

Two properties of the implementation are architecturally relevant:

- **No scratch space.** Writes go to their final offsets and no format
  conversion is involved, so no scratch PVC is requested.
- **Phase wiring.** A full stage finishes into the resize phase; an incremental
  stage completes directly. Resizing an incremental target would operate on a
  disk that already has the right geometry and risks touching existing data.

Preallocation is not honoured by this source. Preallocation is decided when the
PVC is created; re-applying it per stage would overwrite data from earlier
stages.

## 7. Security

- `nbd://` is **plaintext and unauthenticated**. Any endpoint that can answer at
  that address can write arbitrary content into the PVC. It is acceptable only
  on a trusted in-cluster network and is not recommended for production.
- `nbds://` requires `secretRef`, enforced at admission. Peer verification stays
  on; the server certificate must carry a SAN matching the URI host, which means
  the publisher's certificate issuance must align with the address it exposes.
- Narrowing the publisher's bind address is not a substitute for TLS in this
  architecture: the consumer is an importer pod with a dynamically assigned IP,
  so the bind address must be reachable from the whole pod network. During a
  transition period a NetworkPolicy can restrict access to the importer's
  namespace, but that is defence in depth, not authentication.
- Hostname-override and TLS-priority knobs are unavailable on older libnbd
  versions shipped by current base images, so verification must rely on SAN
  matching against the URI host.

## 8. Testing Strategy

The invariant most likely to be broken by later refactoring is "an incremental
stage must not write to `hole,zero` regions". It needs explicit coverage at both
levels.

Unit level:

- Every branch of the tri-state classification.
- Incremental stage produces **no writes at all** for `hole,zero` regions.
- Full stage zeroes holes and performs pre-import cleanup.
- URI validation: scheme, host, missing secret for `nbds://`.
- Size larger than the target is rejected; a size differing from the requested
  capacity only warns.

Functional level, against a real NBD server in the cluster:

- Full import over `nbd://`, content verified.
- `nbds://` happy path, and failure on an invalid server certificate.
- **Cross-stage regression**: a full import, then a second stage that reports
  part of the disk as `hole,zero`, asserting those regions still hold the data
  written by the first stage.

## 9. Open Questions

1. **Publisher without `block_status` support.** Reject, or degrade to a
   sequential whole-disk copy? Current inclination: allow degradation when
   `fullCopy=true` (result is still correct, just slower), reject when
   `fullCopy=false`, since incremental cannot be done safely without extents.
2. **Whether to support `nbd+unix://`** for a sidecar publisher in the same pod.
   This would create a second deployment shape alongside cross-node `nbd://`.
3. **Alignment with the import populator's multi-stage API.** This design favours
   "update `url` + `fullCopy` per stage" over reusing checkpoint fields; feasibility
   on the populator path needs confirmation.
4. **Convergence with the VDDK external-NBD option**, so that two NBD ingestion
   paths do not coexist indefinitely.
5. **Progress reporting granularity.** Should an incremental stage report the
   full disk size or the bytes actually scheduled for this stage? The latter
   matches user expectation but requires a full extent scan up front.

## 10. Delivery Shape

The first iteration targets x86_64, both `nbd://` and `nbds://`, and both full
and incremental semantics, without modifying the existing VDDK data source. The
restriction to one architecture is a scope choice rather than a technical
constraint — libnbd is available on all supported architectures; VDDK is
x86_64-only because of its proprietary library, which does not apply here.

Deferred: readback verification of written ranges, multi-architecture support,
and convergence with the VDDK NBD path.

Because the contracts in §5 are fixed up front, CDI, the publisher and the
orchestration layer can be developed in parallel. A minimal `qemu-nbd` export is
enough to validate the contract before the publisher is complete.
