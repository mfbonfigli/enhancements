---
title: aws-multi-disk-setup
authors:
  - "@fbonfigl"
reviewers:
  - "@patrickdillon"
  - "@mtulio"
  - "@jcpowermac"
approvers:
  - "@patrickdillon"
api-approvers:
  - None
creation-date: 2026-07-15
last-updated: 2026-07-15
tracking-link:
  - https://redhat.atlassian.net/browse/SPLAT-2841
see-also:
  - "https://github.com/openshift/enhancements/pull/1779"
  - "https://github.com/openshift/enhancements/pull/1805"
replaces:
superseded-by:
---

# AWS Extra EBS Volumes for Control Plane Nodes

## Summary

This enhancement enables attaching additional EBS volumes to AWS control plane
nodes at install time, allowing users to place etcd data on a dedicated volume
isolated from the root disk. The feature is gated behind the `AWSMultiDisk` and
`MultiDiskSetup` feature gates (`TechPreviewNoUpgrade`) and follows the
platform-agnostic `diskSetup` pattern already implemented for Azure.

## Motivation

etcd is the most I/O-sensitive component on a control plane node. On AWS, the
root EBS volume is shared with the operating system, container runtime, and all
node-level workloads. Under write pressure from etcd, I/O contention on the
root volume degrades etcd performance, increases write latency, and can trigger
leader elections or compaction storms.

Isolating etcd onto a dedicated EBS volume — sized and typed independently
(e.g., a high-IOPS io2 volume) — eliminates this contention and allows
operators to tune the volume for etcd's specific workload characteristics.

### User Stories

- As a cluster administrator installing OpenShift on AWS, I want to attach a
  dedicated high-performance EBS volume to each control plane node dedicated to
  etcd so that etcd I/O is isolated from OS and workload traffic.

- As a cluster administrator, I want to specify the volume type (gp3, io2,
  etc.), size, and IOPS of the etcd volume in the install-config so that I can
  tune storage performance to match my etcd workload.

### Goals

- Allow users to attach one or more additional EBS volumes to control plane
  nodes via install-config.
- Allow users to configure at least one of those volumes for etcd data storage
  (`/var/lib/etcd`).
- Use the existing platform-agnostic `diskSetup` install-config mechanism.
- Expose the feature only under `TechPreviewNoUpgrade` initially.
- Work correctly on both AWS Nitro instances (NVMe devices with
  non-deterministic kernel names) and older Xen-based instances, using a
  single `/dev/xvd[b-z]` naming convention for both.

### Non-Goals

- Day-2 addition or removal of extra EBS volumes on existing clusters.
- Worker node extra disk support.
- Swap disk setup on AWS (deferred to a follow-up).
- Encryption key management for extra volumes beyond what CAPA already
  supports for `nonRootVolumes`.

## Background

The platform-agnostic `diskSetup` framework is already implemented in the
installer. EP [#1805](https://github.com/openshift/enhancements/pull/1805)
introduced the `controlPlane.diskSetup` and `compute.diskSetup` install-config
fields (`type: etcd`, `swap`, `user-defined`) and the `MultiDiskSetup` feature
gate. EP [#1779](https://github.com/openshift/enhancements/pull/1779) added the
Azure platform binding, mapping `diskSetup` entries to CAPZ `dataDisks` and
generating the corresponding MachineConfigs.

This enhancement adds the AWS platform binding to that existing framework:
mapping `diskSetup` entries to CAPA `nonRootVolumes` (EBS volumes) and
generating an AWS-specific MachineConfig that handles NVMe device discovery on
Nitro instances.

## Proposal

### Workflow Description

1. The user adds `nonRootVolumes` entries to the `controlPlane.platform.aws`
   section and `diskSetup` entries to the platform-agnostic `controlPlane`
   section of `install-config.yaml`. `diskSetup` is not AWS-specific and does
   not live inside the `platform.aws` subsection.

2. The installer validates the configuration: volume types, sizes, IOPS, and
   cross-references between `nonRootVolumes` and `diskSetup`.

3. The installer passes the `nonRootVolumes` list to the CAPA `AWSMachine`
   spec, which causes CAPA to attach the extra EBS volumes when provisioning
   the EC2 instances.

4. For each `diskSetup` entry, the installer generates a `MachineConfig` that
   contains:
   - A shell script (`/usr/local/bin/aws-nvme-disk-setup.sh`) that discovers
     the correct device for the volume, partitions it, and formats it.
   - A systemd oneshot service (`aws-disk-setup-<label>.service`) that runs the
     script after the OS has fully booted (after `ostree-remount.service`, which
     ensures `/usr` is mounted).
   - A systemd mount unit that mounts the formatted partition at the target
     path (e.g., `/var/lib/etcd`).

5. On first boot, ignition applies the `MachineConfig`, writing the script,
   service, and mount unit to disk. The service runs, discovers the NVMe device
   by reading the `bdev` field from the NVMe controller's vendor-specific data
   via `nvme amzn id-ctrl`, partitions and formats the volume, and the mount
   unit mounts it.

6. On subsequent boots, the `blkid` idempotency check in the script prevents
   re-formatting; the mount unit simply mounts the already-formatted partition.

#### Why a systemd service instead of ignition storage.disks

On AWS Nitro instances, EBS volumes appear as `/dev/nvme*n1` with kernel-
assigned ordering that cannot be predicted at ignition generation time. The
traditional `/dev/sd*` device names do not exist during the initramfs stage
when `ignition-disks.service` runs. There are no stable symlinks available in
initramfs that could identify a specific EBS volume by its user-assigned device
name (e.g., `/dev/sdb`).

RHCOS ships `61-persistent-storage-nvme-compat.rules` (udev rules that create
`/dev/disk/by-id/nvme-Amazon_Elastic_Block_Store_vol<id>` symlinks) but not
the volume-name-to-NVMe-device mapping rules that other distributions
(e.g. Amazon Linux) provide. While ignition could deliver custom udev rules,
these are applied after `ignition-disks.service` has already formatted the
disks. A reboot would not help either, as `ignition-disks.service` is
configured to run only once. The only path forward would be to ship the
required udev rules directly with RHCOS.

The approach taken here mirrors the [recommendation in the RHCOS FAQ](https://github.com/openshift/os/blob/master/docs/faq.md#q-how-do-i-configure-a-secondary-block-device-via-ignitionmc-if-the-name-varies-on-each-node): use a
systemd unit to dynamically discover
the correct device. AWS encodes the block device mapping name (e.g., `sdb`) in
the NVMe controller's vendor-specific data, readable via `nvme amzn id-ctrl`.
This is consistent across all Nitro instance families and all EBS volume types.

An alternative approach could be to introduce an enhancement to RHCOS by adding
udev rules equivalent to those in Amazon Linux's `amazon-ec2-utils`, which 
would enable the standard ignition `storage.disks` path. While this approach is
simpler, it brings additional risks, for example it cannot be put behind feature
gate and could create hard to foresee side effects for customers that have e.g. 
installed custom udev rules already in their clusters.

### API Extensions

This enhancement adds a new field to the AWS-specific install-config
`MachinePool` type.

**`pkg/types/aws/machinepool.go`**

```go
// MachinePool stores the configuration for a machine pool installed
// on AWS.
type MachinePool struct {
    // ... existing fields ...

    // nonRootVolumes defines additional EBS volumes to attach to EC2 instances
    // in the machine pool. Each volume requires a device name (e.g. /dev/sdf)
    // that must be unique across all non-root volumes. Used together with the
    // platform-agnostic diskSetup field to configure dedicated storage for
    // specific purposes such as etcd data isolation. Requires both the
    // AWSMultiDisk and MultiDiskSetup feature gates.
    //
    // +openshift:enable:FeatureGate=AWSMultiDisk
    // +optional
    NonRootVolumes []EC2NonRootVolume `json:"nonRootVolumes,omitempty"`
}

// EC2NonRootVolume defines an additional EBS volume to attach to an EC2
// instance.
type EC2NonRootVolume struct {
    // deviceName is the device name exposed to the instance (e.g. /dev/xvdb).
    // Must use the /dev/xvd[b-z] prefix, be unique across all non-root volumes,
    // and must not conflict with the root device (/dev/xvda).
    // Must match ^/dev/xvd[b-z]$.
    // +required
    DeviceName string `json:"deviceName"`

    // size defines the size of the volume in gibibytes (GiB).
    //
    // +kubebuilder:validation:Minimum=1
    // +required
    Size int `json:"size"`

    // type defines the EBS volume type. Valid values: gp2, gp3, io1, io2,
    // st1, sc1, standard.
    // +required
    Type string `json:"type"`

    // iops defines the number of provisioned IOPS. Valid for io1, io2, and
    // gp3 volume types. Required for io1 and io2. Must be 0 for types that
    // do not support IOPS provisioning.
    //
    // +kubebuilder:validation:Minimum=0
    // +optional
    IOPS int `json:"iops,omitempty"`

    // throughput defines the volume throughput in MiB/s. Only valid for gp3
    // volumes. Valid range: 125–2000 MiB/s.
    //
    // +kubebuilder:validation:Minimum=125
    // +kubebuilder:validation:Maximum=2000
    // +optional
    Throughput *int32 `json:"throughput,omitempty"`

    // kmsKeyARN is the ARN of a KMS key used to encrypt the volume at rest.
    // If omitted, the volume inherits the KMS key configured for the
    // instance's root volume.
    // +optional
    KMSKeyARN string `json:"kmsKeyARN,omitempty"`
}
```

The existing platform-agnostic `DiskSetup []Disk` field on `MachinePool`
(added by the MultiDiskSetup enhancement) is used unchanged. On AWS, each
`DiskSetup` entry references a `NonRootVolume` via `platformDiskID`, which
must match a `DeviceName` in `NonRootVolumes`.

**Example install-config snippet:**

```yaml
controlPlane:
  platform:
    aws:
      type: m6i.xlarge
      nonRootVolumes:
      - deviceName: /dev/xvdb
        size: 120
        type: gp3
  diskSetup:
  - type: etcd
    etcd:
      platformDiskID: /dev/xvdb
```

Note that `/dev/xvdb` here is the EC2 block device mapping name, not an actual
device path on Nitro instances. On Nitro, the volume appears as `/dev/nvme*n1`;
on Xen, it appears at the specified path directly. The setup script handles
both cases transparently (see Implementation Details).

### Topology Considerations

#### Hypershift / Hosted Control Planes

Not applicable. In Hypershift, control plane components including etcd run as
pods in the management cluster, not on dedicated EC2 instances provisioned by
the installer. The `diskSetup` mechanism operates on installer-provisioned
machines and does not apply to hosted control planes.

#### Standalone Clusters

Fully supported. This enhancement targets standalone cluster installations on
AWS only.

#### Single-node Deployments or MicroShift

Supported for AWS SNO IPI (standard `openshift-install create cluster` with
`replicas: 1`). AWS SNO uses the same CAPI/CAPA provisioning path as
multi-node clusters: the installer generates one `AWSMachine` with
`nonRootVolumes` populated, and the same `diskSetup` MachineConfig is applied
via MCO to the single master node. No code changes are required to support
SNO with this feature.

Bootstrap-in-place SNO (`openshift-install create single-node-ignition-config`)
does not use the CAPI machine provisioning path and is out of scope for this
enhancement.


#### OpenShift Kubernetes Engine

No specific impact. This feature depends only on the installer and the Machine
Config Operator, both of which are part of the core platform available in OKE.

### Implementation Details/Notes/Constraints

**NVMe device discovery**

On AWS Nitro instances, all EBS volumes are exposed as NVMe devices. The kernel
names (`nvme0n1`, `nvme1n1`, etc.) are non-deterministic — their order depends
on device enumeration, which varies between boots and instance types.

AWS embeds the block device mapping name (the value from `DeviceName` in the
EC2 API, e.g., `sdb`) in bytes 3072–4095 of the NVMe Identify Controller
response (vendor-specific field). The `nvme amzn id-ctrl` command reads this
field. The setup script iterates all `nvme*n1` devices, calls `nvme amzn
id-ctrl` on each, and selects the one whose `bdev` field matches the
`platformDiskID` specified in the install-config (after stripping the `/dev/`
prefix).

**Device naming and platform detection**

Only `/dev/xvd[b-z]` device names are accepted. This single naming convention
works across both AWS hypervisor generations without any branching logic based
on the device name format:

- **Nitro instances**: the volume appears as `/dev/nvme*n1`. AWS embeds the
  block device mapping name (`xvdb`) in the NVMe controller's vendor-specific
  data. The setup script iterates NVMe devices and matches on `bdev == xvdb`.
- **Xen instances**: the volume appears directly at `/dev/xvdb`. The setup
  script checks whether `/dev/$TARGET_BDEV` is a block device first; if it is,
  the NVMe enumeration is skipped entirely.

The script therefore always tries the direct path first and falls back to NVMe
enumeration — with no special-casing based on device name prefix.

**Partition label**

The partition label is derived from the disk type name (e.g., `etcd` for
`type: etcd`). Labels must consist of alphanumeric characters only; the
installer validates and rejects values that do not conform at config-validation
time rather than silently transforming them.

**systemd service ordering**

The setup service must run:
- After `ostree-remount.service` — this guarantees `/usr` is mounted and
  the script at `/usr/local/bin/aws-nvme-disk-setup.sh` is accessible.
- Before the mount unit for the target path.

`After=systemd-udev-settle.service` is intentionally omitted despite the RHCOS
PR FAQ example guidance. That unit is deprecated on modern systemd and resolves
in finite time regardless of whether all udev events have been processed. Instead,
the script calls `udevadm settle` immediately before iterating `/dev/nvme*n1` 
devices, which reliably flushes the udev event queue at runtime.

`DefaultDependencies=no` is set on the service to avoid a dependency cycle with
`local-fs.target` (which the mount unit's `RequiredBy` would otherwise create).

**Idempotency**

The script checks `blkid -t PARTLABEL=<label>` before partitioning. If the
partition already exists (e.g., after a reboot), the script exits successfully
without re-formatting.

**Partitioning and formatting**

The script uses `sfdisk` (from `util-linux`, present on RHCOS) to create a
single GPT partition spanning the entire disk, labeled with the partition label
from the install-config. It then formats the partition with `mkfs.xfs`.

`sgdisk` (from the `gdisk` package) is NOT used because it is not installed on
RHCOS.

**CAPI integration**

The `NonRootVolumes` list is mapped to `capa.Volume` entries in the
`AWSMachine.spec.nonRootVolumes` field. CAPA handles EBS volume attachment
using the EC2 `BlockDeviceMapping` API when creating the instance.

**Feature gating**

The feature requires both `AWSMultiDisk` and `MultiDiskSetup` feature gates to
be enabled. Both are available only in `TechPreviewNoUpgrade` feature sets.

### Risks and Mitigations

**Risk: NVMe device ordering changes across reboots**

The `bdev` field from the NVMe controller is stable — it reflects the EC2 block
device mapping assigned at instance creation, not the kernel enumeration order.

**Risk: Service failure blocks etcd**

If the setup service fails (e.g., volume not attached), the mount unit fails and
etcd does not start. This is intentional — silent fallback to the root disk is
worse than a clear failure.


### Drawbacks

- Additional systemd service on every boot (idempotent after first boot).
- Incompatible with the standard ignition `storage.disks` approach used by
  Azure and vSphere. On Nitro instances, no stable `/dev/sd*` path exists
  during the initramfs stage when `ignition-disks.service` runs. While
  `nvme-cli` could be pulled into the initramfs via a dracut module (the
  `bdev` field is not sysfs-accessible), this would require an RHCOS image
  change with broader impact (see Alternatives).
- If RHCOS ships AWS EBS udev rules in the future, this workaround should be
  removed.

## Alternatives (Not Implemented)

**RHCOS udev rule approach**

Adding udev rules to RHCOS that create `/dev/sdb`-style symlinks during
initramfs would allow the standard ignition `storage.disks` path — identical to
Azure and vSphere. The `bdev` field can be read either via `nvme-cli` (pullable
into the initramfs via a dracut module, as GCP already does with
`google_nvme_id`) or a small purpose-built C binary doing a single NVMe ioctl.

This approach was not taken because shipping udev rules in the RHCOS image
affects all RHCOS users on AWS, not just those using this feature:

- **Symlink proliferation**: Every EBS volume on every node gets a `/dev/sdb`-
  style symlink. Existing tooling and operator scripts referencing NVMe device
  names may behave unexpectedly.
- **Conflict with user workarounds**: Users who already deployed custom udev
  rules via MachineConfig to work around missing symlinks may see conflicts.
- **No feature gating**: RHCOS udev rules cannot be conditionally included
  based on feature gates. The rules would apply to all clusters on that image
  version regardless of opt-in.

The installer-side systemd service approach scopes all complexity to clusters
that explicitly opt in via the `AWSMultiDisk` feature gate and has no impact on
the RHCOS image or other clusters.


## Test Plan

- **Unit tests**: Validation of `NonRootVolumes` fields (type, size, IOPS,
  device name format, uniqueness). Cross-validation of `NonRootVolumes` vs
  `diskSetup` entries. MachineConfig generation for the setup service, mount
  unit, and script file.

- **e2e tests**: Install a cluster with `type: etcd` disk setup on an m6i
  instance. After install, verify:
  - `/var/lib/etcd` is mounted from the extra EBS volume
    (`findmnt /var/lib/etcd` shows the correct device)
  - etcd is healthy (`etcdctl member list` returns all members)
  - The extra volume is the correct type and size (AWS EC2 API)

- **Negative tests**: Attempt install with swap disk type on AWS (expect
  validation error). Attempt install with `platformDiskID` not matching any
  `NonRootVolumes` entry (expect validation error).

## Graduation Criteria

### Tech Preview -> GA

- RHCOS udev rule approach evaluated and either adopted or explicitly deferred
  with justification.
- No open P1/P2 bugs for the duration of one minor release in Tech Preview.
- Upgrade/downgrade testing complete.

### Removing a deprecated feature

Not applicable.

## Upgrade / Downgrade Strategy

Extra EBS volumes are attached at install time and persist across upgrades.
The `aws-disk-setup-<label>.service` and mount unit are delivered via
MachineConfig and managed by MCO. On upgrade:
- The service remains idempotent (the partition already exists).
- The mount unit continues to mount the volume from the partition label.
- No data migration is required.

On downgrade to a release that does not support this feature: since
`TechPreviewNoUpgrade` clusters cannot be upgraded or downgraded via OTA, the
only path to a different release is a full cluster reinstall. When reinstalling
with a release that predates this feature, omit the `nonRootVolumes` and
`diskSetup` fields from the install-config; the new cluster will use the root
disk for etcd as usual. The etcd data on the extra EBS volumes from the
previous install is not migrated.

## Version Skew Strategy

No version skew concerns. The feature is entirely installer-side: the
MachineConfig and systemd units are standard RHCOS primitives that do not
depend on specific OpenShift operator versions.

## Operational Aspects of API Extensions

- **`nonRootVolumes`** is an install-time-only field. It cannot be modified
  post-install without reinstalling the cluster.
- Extra EBS volumes incur additional AWS costs. Users should be aware of the
  per-GB and IOPS pricing for their chosen volume type.
- EBS volumes are deleted on instance termination. CAPA hardcodes
  `DeleteOnTermination: true` internally and does not expose this as a
  configurable field. When a control plane instance is terminated and replaced
  (for example, by CAPI machine remediation or a control plane machine
  replacement), the replacement receives a new, empty EBS volume. etcd recovers
  by receiving a snapshot from the surviving cluster members via the Raft
  protocol — the same behavior that applies to etcd data on the root disk today.
  This is safe for a 3-member (or larger) etcd cluster as long as quorum is
  maintained.

## Support Procedures

- If `aws-disk-setup-etcd.service` fails on a node, check:
  - `systemctl status aws-disk-setup-etcd.service`
  - `journalctl -u aws-disk-setup-etcd.service`
  - Whether the extra EBS volume is attached: `lsblk`
  - Whether `nvme-cli` is available: `which nvme`
- If `/var/lib/etcd` is not mounted, the mount unit will have failed. Check
  `systemctl status var-lib-etcd.mount`.
- The setup script is idempotent and can be re-run manually if needed.