---
title: Multi-Vendor Firmware Update

iep-number: XX

creation-date: 2026-08-27

status: implementable

authors:

- "@xkonni"
- "@stefanhipfel"
- "@davidgrun"

reviewers:

---

# IEP-XX: Multi-Vendor Firmware Update

## Table of Contents

- [Summary](#summary)
- [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals](#non-goals)
- [Proposal](#proposal)
    - [FirmwareUpdate CRD](#1-firmwareupdate-crd)
    - [FirmwareUpdateSet CRD](#2-firmwareupdateset-crd)
    - [Controller Architecture](#3-controller-architecture)
    - [Example Usage](#example-usage)
    - [Relationship to Existing Resources](#relationship-to-existing-resources)
- [Alternatives](#alternatives)
- [Open Questions](#open-questions)
- [Implementation Plan](#implementation-plan)

## Summary

Introduce a vendor-agnostic `FirmwareUpdate` / `FirmwareUpdateSet` CRD pair
that lets operators declaratively trigger full-system firmware updates across
Dell, HPE, and Lenovo servers. The common specification captures what to
update and which servers to target; the controller branches on vendor
detection to drive the update via the appropriate mechanism — Dell's
repository-based iDRAC job API, or a one-time-boot (OTB) custom image for
HPE and Lenovo.

## Motivation

The existing `BMCVersion` and `BIOSVersion` resources handle individual
firmware components (BMC firmware and BIOS) via Redfish's standard
`UpdateService.SimpleUpdate` action. This works for targeted, single-component
updates but does not cover:

1. **Repository-based bundle updates (Dell):** Dell exposes a catalog-based
   update flow: a catalog XML lists the individual firmware packages to be
   applied (NICs, RAID controllers, BIOS, BMC, …) and each package is
   submitted as a separate iDRAC job, tracked via proprietary iDRAC Job
   resources rather than standard Redfish Tasks.

2. **One-time-boot image updates (HPE, Lenovo):** These vendors require the
   BMC to perform a one-time boot from a vendor-supplied image (e.g. HPE SPP
   ISO) to apply firmware updates; there is no equivalent per-component job
   API for in-band tracking.

Because the three vendor mechanisms are fundamentally different at the
transport level, a dedicated resource designed with the branching architecture
in mind is the right fit.

### Goals

- A single `FirmwareUpdate` CRD that operators fill out the same way regardless
  of vendor.
- A `FirmwareUpdateSet` CRD mirroring the `BMCVersionSet` / `BIOSVersionSet`
  pattern: selects servers by label and stamps out per-server `FirmwareUpdate`
  instances from a template.
- Full `ServerMaintenance` integration: no firmware update proceeds until
  the server enters maintenance state.
- Retry and failure-policy knobs consistent with `BMCVersion`/`BIOSVersion`.
- Clear status reporting: per-vendor progress surfaced via common
  `state` / `conditions` fields.
- An extensible vendor-dispatch architecture so that additional vendors can be
  added in future without changes to the CRD API surface.

### Non-Goals

- Full support for vendors beyond Dell, HPE, and Lenovo in this iteration
  (though the architecture explicitly leaves room for it).
- Modelling the firmware component inventory tree in the CRD — what components
  exist and at what version is the vendor's domain (catalog, baseline image).
  The CRD covers the parameters each vendor's update mechanism requires, not
  the component tree itself.

## Proposal

### 1. `FirmwareUpdate` CRD

Scoped to a single server (`spec.serverRef`). A `FirmwareUpdateSet` creates
and owns one `FirmwareUpdate` per matching server.

```go
type FirmwareUpdateSpec struct {
    FirmwareUpdateTemplate `json:",inline"`

    // ServerRef is the Server this FirmwareUpdate targets. Immutable once set.
    // +kubebuilder:validation:XValidation:rule="self == oldSelf",message="serverRef is immutable"
    ServerRef *corev1.LocalObjectReference `json:"serverRef,omitempty"`

    // ServerMaintenanceRef is managed by the controller.
    ServerMaintenanceRef *ObjectReference `json:"serverMaintenanceRef,omitempty"`
}

type FirmwareUpdateTemplate struct {
    // Repository holds parameters for repository-based updates (Dell).
    // Exactly one of Repository or Image must be set.
    // +kubebuilder:validation:XValidation:rule="has(self.repository) != has(self.image)"
    Repository *FirmwareRepository `json:"repository,omitempty"`

    // Image holds parameters for one-time-boot image updates (HPE, Lenovo).
    Image *ImageSpec `json:"image,omitempty"`

    // ServerMaintenancePolicy controls whether the controller waits for
    // owner approval before entering maintenance. Default: OwnerApproval.
    ServerMaintenancePolicy ServerMaintenancePolicy `json:"serverMaintenancePolicy,omitempty"`

    // RetryPolicy configures automatic retries on failure.
    RetryPolicy *RetryPolicy `json:"retryPolicy,omitempty"`
}

// FirmwareRepository holds the parameters for Dell's
// InstallFromRepository / GetRepoBasedUpdateList OEM actions.
// Field names mirror RepositoryUpdateParameters in the bmc package.
type FirmwareRepository struct {
    // ShareType is the network share type (NFS, CIFS, HTTP, HTTPS).
    ShareType string `json:"shareType"`
    // Address is the share's hostname or IP.
    Address string `json:"address,omitempty"`
    // ShareName is the share name (not required for HTTP/HTTPS).
    ShareName string `json:"shareName,omitempty"`
    // CatalogFile is the catalog filename within the share.
    CatalogFile string `json:"catalogFile,omitempty"`
    // CredentialsRef is a reference to a Secret holding username and password
    // for authenticated shares. Keys: "username", "password".
    CredentialsRef *corev1.SecretReference `json:"credentialsRef,omitempty"`
    // IgnoreCertWarning, if true, ignores TLS certificate warnings.
    IgnoreCertWarning bool `json:"ignoreCertWarning,omitempty"`
    // RebootNeeded, if true, allows the BMC to reboot to apply updates.
    RebootNeeded bool `json:"rebootNeeded,omitempty"`
    // ApplySameVersions, if true, re-applies packages at the same version.
    ApplySameVersions bool `json:"applySameVersions,omitempty"`
    // ApplyDowngradeVersions, if true, allows downgrading component versions.
    ApplyDowngradeVersions bool `json:"applyDowngradeVersions,omitempty"`
}
```

```go
type FirmwareUpdateStatus struct {
    // State is the current reconciliation state.
    // +kubebuilder:validation:Enum=Pending;InProgress;Completed;Failed
    State FirmwareUpdateState `json:"state,omitempty"`

    // VendorJob tracks the in-flight vendor-specific job (Dell: iDRAC JID;
    // HPE/Lenovo: OTB task URI).
    VendorJob *VendorJobStatus `json:"vendorJob,omitempty"`

    // FailedAttempts counts failed reconciliation attempts.
    FailedAttempts int32 `json:"failedAttempts,omitempty"`

    // ObservedGeneration reflects the last processed spec generation.
    ObservedGeneration int64 `json:"observedGeneration,omitempty"`

    // Conditions follows the standard Kubernetes condition conventions.
    Conditions []metav1.Condition `json:"conditions,omitempty"`
}

// VendorJobStatus holds the ID and last-known state of a vendor job.
type VendorJobStatus struct {
    ID              string `json:"id"`
    State           string `json:"state,omitempty"`
    Message         string `json:"message,omitempty"`
    PercentComplete int32  `json:"percentComplete,omitempty"`
}
```

### 2. `FirmwareUpdateSet` CRD

Mirrors `BIOSVersionSet` exactly, selecting `Server` objects by label.

```go
type FirmwareUpdateSetSpec struct {
    // ServerSelector selects the Servers this set manages.
    ServerSelector metav1.LabelSelector `json:"serverSelector"`

    // FirmwareUpdateTemplate is the template applied to each FirmwareUpdate.
    FirmwareUpdateTemplate FirmwareUpdateTemplate `json:"firmwareUpdateTemplate,omitempty"`
}

type FirmwareUpdateSetStatus struct {
    MatchingServers           int32 `json:"matchingServers,omitempty"`
    PendingFirmwareUpdate     int32 `json:"pendingFirmwareUpdate,omitempty"`
    InProgressFirmwareUpdate  int32 `json:"inProgressFirmwareUpdate,omitempty"`
    CompletedFirmwareUpdate   int32 `json:"completedFirmwareUpdate,omitempty"`
    FailedFirmwareUpdate      int32 `json:"failedFirmwareUpdate,omitempty"`
}
```

### 3. Controller Architecture

#### `FirmwareUpdateSetReconciler`

Structurally identical to `BIOSVersionSetReconciler`:

- Queries `Server` objects matching `spec.serverSelector`.
- Creates one `FirmwareUpdate` per server from the template.
- Deletes orphaned `FirmwareUpdate` objects when a server no longer matches.
- Propagates template drift to children not currently `InProgress`.
- Propagates `ignore` / `retry` annotations.
- Aggregates child states into set-level status counters.

#### `FirmwareUpdateReconciler`

State machine: `Pending → InProgress → Completed | Failed`.

**Pending:**
- Resolves the `Server` → `BMC` → vendor via `Server.status.manufacturer`.
- If `spec.repository` is set, type-asserts the BMC client against
  `bmc.FirmwareUpdaterDell`. If not supported, transitions to `Failed`
  with a clear condition message.
- Creates a `ServerMaintenance` object; records the ref in
  `spec.serverMaintenanceRef`.
- Waits for `Server.status.state == Maintenance`.
- Transitions to `InProgress`.

**InProgress — Dell (repository):**
1. Calls `GetRepositoryUpdateList` to check whether any packages are pending.
   If none, transitions to `Completed` (nothing to do).
2. Calls `InstallFirmwareFromRepository` with `ApplyUpdate: true`.
   - If `isFatal == true`, transitions to `Failed` immediately (no retry).
   - Otherwise records the returned `jobID` in `status.vendorJob.ID`.
3. Polls `GetJob` until `IsTerminal()`.
4. If `IsCompleted()`, transitions to `Completed`.
   If `IsFailed()`, transitions to `Failed` (eligible for retry per policy).

**InProgress — HPE / Lenovo (one-time-boot):**
1. Calls the vendor-specific OTB helper (future implementation) with
   `spec.image` to boot the server from the update image.
2. Polls the Redfish Task or vendor OEM job until terminal.
3. Transitions to `Completed` or `Failed` accordingly.

**Completed:**
- Removes the `ServerMaintenance` object.
- Clears `status.vendorJob`.

**Failed:**
- Supports manual retry via `metal.ironcore.dev/operation=retry` annotation.
- Supports automatic retry if `spec.retryPolicy.maxAttempts` is not exhausted.
- Resets state to `Pending` and increments `status.failedAttempts`.

#### Vendor dispatch

Vendor is determined from `Server.status.manufacturer` (populated from
Redfish `System.Manufacturer`). Each vendor path is a separate implementation
of a `vendorHandler` interface, registered in a map. Adding a new vendor
requires only a new entry in the map — no changes to the reconciler's
`reconcileInProgress` method:

```go
// vendorHandler drives the vendor-specific InProgress logic.
type vendorHandler interface {
    reconcileInProgress(ctx context.Context, fw *FirmwareUpdate, client bmc.BMC) (ctrl.Result, error)
}

// vendorHandlers maps Server.status.manufacturer to its handler.
// dellHandler lives in firmware_dell.go, otbHandler in firmware_otb.go.
var vendorHandlers = map[string]vendorHandler{
    "Dell Inc.": &dellHandler{},
    "HPE":       &otbHandler{},
    "Lenovo":    &otbHandler{},
}

func (r *FirmwareUpdateReconciler) reconcileInProgress(ctx context.Context, fw *FirmwareUpdate, client bmc.BMC) (ctrl.Result, error) {
    h, ok := vendorHandlers[server.Status.Manufacturer]
    if !ok {
        return r.failWithCondition(ctx, fw, "UnsupportedVendor",
            fmt.Sprintf("no handler registered for manufacturer %q", server.Status.Manufacturer))
    }
    return h.reconcileInProgress(ctx, fw, client)
}
```

### Example Usage

#### Dell — repository-based update for a fleet

```yaml
apiVersion: metal.ironcore.dev/v1alpha1
kind: FirmwareUpdateSet
metadata:
  name: dell-firmware-q3-2026
  namespace: default
spec:
  serverSelector:
    matchLabels:
      metal.ironcore.dev/manufacturer: dell
  firmwareUpdateTemplate:
    repository:
      shareType: HTTPS
      ipAddress: firmware-repo.example.com
      catalogFile: Catalog.xml
      credentialsRef:
        name: dell-firmware-repo-creds
        namespace: default
      rebootNeeded: true
    serverMaintenancePolicy: OwnerApproval
    retryPolicy:
      maxAttempts: 3
```

#### HPE / Lenovo — one-time-boot update

```yaml
apiVersion: metal.ironcore.dev/v1alpha1
kind: FirmwareUpdateSet
metadata:
  name: hpe-firmware-q3-2026
spec:
  serverSelector:
    matchLabels:
      metal.ironcore.dev/manufacturer: hpe
  firmwareUpdateTemplate:
    image:
      uri: http://firmware-images.example.com/hpe-spp-2026.06.iso
      transferProtocol: HTTP
    serverMaintenancePolicy: OwnerApproval
```

### Relationship to Existing Resources

| Resource | Scope | Mechanism | Tracks |
|---|---|---|---|
| `BMCVersion` | BMC firmware, single server | `UpdateService.SimpleUpdate` | Redfish Task |
| `BIOSVersion` | BIOS, single server | `UpdateService.SimpleUpdate` | Redfish Task |
| **`FirmwareUpdate`** | **All firmware, single server** | **Dell: InstallFromRepository; HPE/Lenovo: OTB image** | **Vendor Job / OTB Task** |

`BMCVersion` and `FirmwareUpdate` are complementary and target different
things: BMC firmware always requires its own dedicated update path (resets,
vendor-specific flows) and cannot be part of a system baseline. `BIOSVersion`
and `FirmwareUpdate` may overlap for BIOS on some vendors, but serve different
use cases — targeted single-component control vs. full catalog-driven baseline
updates.

## Alternatives

**`FirmwareUpdateClass` indirection (StorageClass-style).** An alternative
design introduces a `FirmwareUpdateClass` cluster-scoped CRD (with a
`controller` ownership discriminator and `matchManufacturers` for auto-resolution)
and moves all vendor-specific parameters into separate typed CRDs
(`DellRepositoryParameters`, `OTBImageParameters`, …) referenced via
`parametersRef`. The `FirmwareUpdate` spec then reduces to `serverRef` +
`classRef`, and a single `FirmwareUpdateSet` can target a mixed-vendor fleet
without knowing vendor specifics — the Class resolves which parameters apply
per server based on `matchManufacturers`.

The trade-off: the indirection chain becomes three hops
(`FirmwareUpdate → FirmwareUpdateClass → DellRepositoryParameters`), the
CRD count grows from three to five or more, and the `controller` discriminator
adds an extension point designed for an open ecosystem of independent
provisioners — a problem we do not have with three vendors handled by a single
controller. Not adopted in this iteration. The existing `BMCVersion` and `BIOSVersion`
controllers already handle vendor-specific behaviour via internal dispatch on
`Server.status.manufacturer` without a class layer, and that pattern has
worked well. `FirmwareUpdate` follows the same convention.

**A single CRD with a required `vendor` discriminator field.** Rejected:
vendor is already derivable from `Server.status.manufacturer` and should not
be duplicated in the spec. Making operators specify it would cause drift and
prevent a `FirmwareUpdateSet` from targeting a mixed-vendor fleet.

**Separate CRDs per vendor (`DellFirmwareUpdate`, `HPEFirmwareUpdate`, ...).** Rejected:
this scatters the fleet management story and prevents a single
`FirmwareUpdateSet` from targeting a mixed fleet.

## Open Questions

1. **HPE/Lenovo OTB mechanics:** The exact Redfish or OEM path for triggering
   and monitoring the one-time-boot still needs to be confirmed against
   hardware. The `spec.image` shape deliberately mirrors `ImageSpec` from
   `BIOSVersion`; adjustments may be needed once the vendor API is
   characterised.

2. **Dry-run / preflight for Dell:** Should the controller expose a dry-run
   check as a separate observable condition (`PendingPackagesDetected`) before
   `InstallFromRepository` is issued? Alternatively, a `CheckOnly` bool in
   `FirmwareUpdateTemplate` could trigger a preflight-only reconcile.

3. **Post-update version verification:** `BMCVersion` verifies the live
   firmware version matches `spec.version` before declaring `Completed`.
   `FirmwareUpdate` cannot do the same because the target version is implicit
   in the repository catalog. Consider surfacing the post-update firmware
   inventory as a status annotation or a separate read-only object.

4. **Credential handling:** `FirmwareRepository.credentialsRef` references a
   `Secret`. Confirm the required RBAC rules and whether a projected volume or
   a direct `Get` is the right approach.

## Implementation Plan

This feature spans two repositories:

- **metal-operator** — bmc transport layer: vendor-specific client interfaces
  and implementations (`FirmwareUpdaterDell` and equivalents for HPE/Lenovo).
- **metal-maintenance-operator** — CRD types, controllers, and RBAC: all
  update controllers live here alongside `BMCVersion`, `BIOSVersion`, and
  the `ServerMaintenance` machinery they depend on.

Steps:

1. Conclude this IEP and confirm the exact vendor API flows against real
   hardware before writing any controller code.
2. Refactor ironcore-dev/metal-maintenance-operator#170 (current vendor-specific
   `FirmwareUpdateDell` draft) to align with the unified `FirmwareUpdate` /
   `FirmwareUpdateSet` design proposed here.
3. Refactor / extend the bmc transport layer in metal-operator to match the
   confirmed vendor flows.
4. HPE/Lenovo OTB path once vendor API is confirmed.
5. e2e tests extending the existing bmc mock server.
