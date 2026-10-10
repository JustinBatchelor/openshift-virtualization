---
name: openshift-virtualization
description: Maintain or troubleshoot JustinBatchelor/openshift-virtualization and its MTV Primera SSH copy-offload runbook, including prerequisites, migration plans, validation, and rollback. Use for this repository's published migration guidance.
---

# OpenShift Virtualization

Work from the active checkout of `JustinBatchelor/openshift-virtualization`.
The published project is `mtv-primera-ssh-plan/readme.md`, a runbook for a
single-VM cold migration pilot using MTV, VMware, and HPE Primera. Read the
relevant section before editing commands. The runbook records a proposed
procedure, not evidence that a cluster change or migration has been executed.

## Route the work

- Section 1: storage placement, compatibility, and prerequisite evidence.
- Section 2: installed populator implementation, ESXi SSH, wrappers, and Secrets.
- Section 3: maps and storage-offload configuration.
- Section 4: pilot Plan, Migration, transfer monitoring, and guest acceptance.
- Section 5: rollback, rollout, and resource ownership during cleanup.

## Preserve the operational constraints

Reuse the working VMware provider, Primera CSI StorageClass, and suitable
maps. Previous successful migrations do not prove hardware offload readiness.
Trace every source VMFS extent and destination PV to the same physical array
before enabling offload. Names, Secret references, and replication-pair
membership do not establish same-array placement.

Match the SSH wrapper to the exact installed populator build. The documented
Python procedure must not be applied to a shell or extensionless implementation
by renaming it. Selecting the SSH clone method does not select a Python wrapper.
Preserve TLS verification and obtain the build-matched wrapper through its
source package or support channel.

Inspect map ownership before reuse. Copy maps when ownership or offload changes
require isolation, discard server metadata from copies, and preserve the full
mapping specification. Do not apply a new manifest over a reused shared map.

Registering a Plan and creating a Migration have different effects: creating
the Migration can stop the source VM. Use the authorized downtime window,
retain source disks, and validate on an isolated network. Do not restart the
source while the destination can still write as the same machine.

Require affirmative XCOPY evidence for every migrated disk. A successful
migration, accepted dry run, or empty offload-status query does not establish
hardware copy offload. Preserve guest/application checks and representative
checksum comparisons as acceptance criteria.

## Verification and publication

There is no application test runner or executable deployment package in the
published project. For runbook edits, check variable continuity, resource
namespaces, shell syntax, command ordering, and the distinction between
registration, execution, and cleanup. When changing version-specific claims,
verify them against the authoritative documentation linked in the runbook.
Do not execute migration examples merely to validate documentation.

Keep real credentials out of shell examples, manifests, and evidence. Use
Secret references and prompted or generated credentials; never print backend
passwords. Keep ignored local projects and private artifacts outside published
commits, including descriptions copied from those projects.

During authorized cleanup, inventory pilot-created versus reused resources.
Retain shared maps, existing StorageClasses, CSI backend Secrets, and prior
Plans. Check PVC reclaim behavior before deleting storage. Source publication
alone does not authorize migrations, cluster changes, or disk deletion.
