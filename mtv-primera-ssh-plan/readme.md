# Enable MTV 2.12 storage copy offload on HPE Primera using SSH

## 1. Verify storage placement and prerequisites

Use the existing MTV installation, VMware provider, and HPE CSI StorageClass for a single-VM cold migration pilot. **Same-array placement is unverified and must be demonstrated before enabling offload.**

Existing successful migrations are the starting point: reuse the configured Primera StorageClass and inspect the maps they used. Installing another CSI driver or creating another StorageClass is not required merely to enable MTV offload. Previous success establishes useful provisioning and migration history, but does not establish same-array placement, ESXi SSH readiness, or hardware XCOPY usage.

Identify the existing class and review its backend references and volume behavior:

```bash
oc get storageclasses \
  -o custom-columns='NAME:.metadata.name,PROVISIONER:.provisioner,BINDING:.volumeBindingMode,RECLAIM:.reclaimPolicy'
read -r -p 'Existing Primera CSI StorageClass name: ' HPE_STORAGE_CLASS
export HPE_STORAGE_CLASS
oc get storageclass "$HPE_STORAGE_CLASS" -o yaml
oc get storageprofiles.cdi.kubevirt.io "$HPE_STORAGE_CLASS" -o yaml
oc get pv -o json | jq --arg sc "$HPE_STORAGE_CLASS" '
  .items[] | select(.spec.storageClassName == $sc)
  | {name: .metadata.name, phase: .status.phase, claim: .spec.claimRef,
     driver: .spec.csi.driver, handle: .spec.csi.volumeHandle,
     mode: .spec.volumeMode, accessModes: .spec.accessModes}'
```

Confirm the expected HPE CSI provisioner (`csi.hpe.com`), backend Secret references, protocol, capacity pool, binding mode, reclaim policy, and the CDI StorageProfile's supported volume/access modes. Keep the working class settings unchanged. The backend's actual array identity still needs verification; its Secret name is not sufficient. Do not print backend passwords into the runbook or evidence files. [HPE StorageClass configuration](https://scod.hpedev.io/csi_driver/container_storage_provider/hpe_alletra_storage_mp_b10000/index.html)

For every datastore containing a pilot VM disk:

1. In vSphere, record the VMFS datastore’s backing LUN identifiers, including all extents and their NAA/WWN identifiers.
2. In Primera, match those identifiers to the corresponding virtual volumes. Record the array’s serial number or other authoritative system identifier.
3. Inspect the target HPE StorageClass and its referenced CSI backend configuration to identify the destination array.
4. Trace a representative Bound PV using that StorageClass through its CSI volume handle and HPE volume metadata to the actual Primera virtual volume. An existing migrated VM's PV can supply read-only evidence if its backend configuration is still current. Otherwise, provision a disposable block PVC for this verification. Do not write test data to an existing VM's disks.
5. Compare the source and destination **array identities**. Names, similar management addresses, or membership in a replication pair are insufficient evidence.

Record the result:

| Source datastore / LUN | Source array identity | Target StorageClass / test PV | Destination array identity | Match |
|---|---|---|---|---|
| Verified identifiers | Serial or system ID | Verified identifiers | Serial or system ID | Yes / No |

**Readiness gate:** Continue only when every source datastore and the destination volume are verified on the same array. If they differ or cannot be traced, stop offload enablement and resolve storage placement first. Red Hat describes this path as copying between source and target volumes on the same array. [Red Hat offload documentation](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.12/html/planning_your_migration_to_red_hat_openshift_virtualization/assembly_planning-migration-vmware_mtv)

Also:

- Record exact MTV, OpenShift Virtualization, ESXi, HPE CSI, and Primera OS versions; verify compatibility.
- Verify SAN connectivity and temporary destination-volume presentation to participating ESXi hosts.
- Use current evidence of successful provisioning, attachment, and guest I/O on this class; if that evidence is missing or configuration has changed, test with a disposable PVC and workload.
- Preserve existing controller and provider settings for rollback.

Check applicability of `MTV-5752`, a Primera host-set name length issue. Red Hat lists it under 2.12.3 fixes but describes no workaround; confirm your installed build’s status before production rollout. [MTV release notes](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.12/html/release_notes/ref_rn-2-12_release-notes)

## 2. Enable offload and configure ESXi SSH

Run these commands only after the readiness gate passes. Run workstation commands in Bash, keeping the variables below in the same session.

### 2.1. Select SSH and confirm the Python wrapper applies

**This procedure is for a verified Python-based populator build.** Red Hat's documentation names `secure-vmkfstools-wrapper.py`, but also contains a conflicting `.sh` example. Upstream subsequently converted the wrapper to shell, and newer source uses an extensionless wrapper. Verify the installed image's implementation before using these Python commands; selecting `esxiCloneMethod: ssh` does not select Python. Do not rename a shell script or binary to `.py` or substitute a historical Python wrapper into a newer build.

Use the Python file supplied for your exact populator build, obtained from its matching source package or Red Hat Support. This README does not supply that file. The historical [Python wrapper source](https://github.com/kubev2v/forklift/blob/76bf7107f76ca94f6f4bf85baf75f3e68d4b8869/cmd/vsphere-xcopy-volume-populator/vmkfstools-wrapper/vmkfstools_wrapper.py) establishes the `--version` and `SSH_ORIGINAL_COMMAND` behavior used below; it is not a recommendation to install that historical revision. Compare the [current release-2.12 upload implementation](https://github.com/kubev2v/forklift/blob/release-2.12/cmd/vsphere-copy-offload-populator/internal/populator/secure_script.go) when checking your build.

Use the discovery commands below to select values, then enter them at the prompts. `export` updates each environment variable in the current Bash session; rerun its prompt to change it. These discovery commands do not change cluster or storage configuration. Do not automatically select the first result when multiple environments are listed.

**`MTV_NAMESPACE`: find the namespace containing your VMware Provider.** Start by checking your current OpenShift context and listing the controller and Providers:

```bash
oc config current-context
oc get forkliftcontrollers.forklift.konveyor.io -A
oc get providers.forklift.konveyor.io -A \
  -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,TYPE:.spec.type,URL:.spec.url'

read -r -p 'Provider namespace (commonly openshift-mtv): ' MTV_NAMESPACE
export MTV_NAMESPACE
oc get providers.forklift.konveyor.io -n "$MTV_NAMESPACE"
```

The controller and populator deployment examples below assume they also reside in this namespace. If the inventory shows otherwise, use their actual namespace in the deployment query and ForkliftController patch; keep `MTV_NAMESPACE` set to the Provider namespace for Provider and SSH Secret commands. Use the discovered ForkliftController name if it differs from `forklift-controller`.

**`VSPHERE_PROVIDER`: choose a Provider whose type is `vsphere`.**

```bash
oc get providers.forklift.konveyor.io -n "$MTV_NAMESPACE" \
  -o custom-columns='NAME:.metadata.name,TYPE:.spec.type,URL:.spec.url'

read -r -p 'VMware Provider name: ' VSPHERE_PROVIDER
export VSPHERE_PROVIDER
oc get providers.forklift.konveyor.io "$VSPHERE_PROVIDER" \
  -n "$MTV_NAMESPACE" \
  -o jsonpath='{.spec.type}{"\t"}{.spec.url}{"\n"}'
```

**`ESXI_HOST`: choose the ESXi management hostname or IP for a participating host.** The Provider URL can point to vCenter and must not be used as the ESXi SSH address. Get the address from vSphere's host inventory. If `govc` is installed and already configured to authenticate to that vCenter, these optional commands list hosts and inspect a selected host:

```bash
govc find / -type h
govc host.info '/Datacenter/host/Cluster/esxi.example.com'
```

Replace the sample inventory path with a returned path. `govc` is optional; vSphere's UI provides the same selection. See the [VMware govc command reference](https://github.com/vmware/govmomi/blob/main/govc/USAGE.md).

```bash
read -r -p 'Participating ESXi management hostname or IP: ' ESXI_HOST
export ESXI_HOST
printf 'ESXi SSH target: root@%s\n' "$ESXI_HOST"
```

**`DATASTORE_UUID`: find the mounted VMFS datastore UUID on that host.** If SSH is not yet enabled, perform the SSH service setup in section 2.2 first, or run the `esxcli` command directly in an existing ESXi administrative console. Use administrative credentials and verify the host-key fingerprint when connecting.

```bash
ssh "root@${ESXI_HOST}" 'esxcli storage filesystem list'

read -r -p 'Verified mounted VMFS datastore UUID: ' DATASTORE_UUID
export DATASTORE_UUID
ssh "root@${ESXI_HOST}" "test -d '/vmfs/volumes/${DATASTORE_UUID}'"
```

Select the UUID column for the intended mounted VMFS filesystem, not its label or its vCenter `datastore-N` managed-object reference. This identifies the wrapper's datastore path; it does not replace the same-array verification in step 1. The datastore must be accessible from every participating ESXi host. [Broadcom datastore identification example](https://knowledge.broadcom.com/external/article/379915)

**`PYTHON_WRAPPER_LOCAL`: locate your release-matched Python file on the workstation.** After obtaining the correct source package or Support-provided file, search that directory. Replace the example directory below with its actual location:

```bash
rg --files '/absolute/path/to/release-matched-source' \
  -g '*vmkfstools*.py'

read -r -p 'Absolute path to the verified Python wrapper file: ' PYTHON_WRAPPER_LOCAL
export PYTHON_WRAPPER_LOCAL
test -f "$PYTHON_WRAPPER_LOCAL" && ls -l "$PYTHON_WRAPPER_LOCAL"
```

Enter the full absolute path, without surrounding quotes or a literal `~`. A matching filename alone does not establish release compatibility. If no Python wrapper is provided for the installed build, stop this Python-specific procedure.

**`WRAPPER_PATH`: derive the remote path from the selected datastore UUID.** Recompute this variable whenever `DATASTORE_UUID` changes:

```bash
export WRAPPER_PATH="/vmfs/volumes/${DATASTORE_UUID}/secure-vmkfstools-wrapper.py"

# Review all six values before continuing.
printf '%s\n' \
  "MTV_NAMESPACE=$MTV_NAMESPACE" \
  "VSPHERE_PROVIDER=$VSPHERE_PROVIDER" \
  "ESXI_HOST=$ESXI_HOST" \
  "DATASTORE_UUID=$DATASTORE_UUID" \
  "PYTHON_WRAPPER_LOCAL=$PYTHON_WRAPPER_LOCAL" \
  "WRAPPER_PATH=$WRAPPER_PATH"
```

Stop and correct any empty value or failed lookup. `ESXI_PYTHON` is discovered and set in section 2.2; the temporary key-directory variables are generated in sections 2.4 and 2.5.

Record the configured populator image and check that the local Python file matches its implementation:

```bash
oc get deployment forklift-volume-populator-controller \
  -n "$MTV_NAMESPACE" \
  -o jsonpath='{range .spec.template.spec.containers[*].env[?(@.name=="VSPHERE_XCOPY_VOLUME_POPULATOR_IMAGE")]}{.value}{"\n"}{end}'
```

If the image query returns nothing, inspect the deployment's image and environment configuration to locate the populator image reference before continuing. Record the exact image and confirm the matching Python file.

```bash
oc patch forkliftcontrollers.forklift.konveyor.io forklift-controller \
  -n "$MTV_NAMESPACE" --type=merge \
  -p '{"spec":{"feature_copy_offload":"true"}}'

oc patch providers.forklift.konveyor.io "$VSPHERE_PROVIDER" \
  -n "$MTV_NAMESPACE" --type=merge \
  -p '{"spec":{"settings":{"esxiCloneMethod":"ssh"}}}'
```

MTV uses SSH to run cloning commands on ESXi. Primera management uses its API. [Red Hat configuration procedure](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.12/html/planning_your_migration_to_red_hat_openshift_virtualization/assembly_planning-migration-vmware_mtv)

### 2.2. Enable SSH and verify Python on ESXi

Enable SSH through the vSphere UI, or run these commands in an existing ESXi administrative console:

```bash
vim-cmd hostsvc/enable_ssh
vim-cmd hostsvc/start_ssh
vim-cmd hostsvc/get_ssh_status
```

Permit TCP 22 from the migration pods' actual egress addresses. Use your existing administrative SSH credentials for installation, and verify the ESXi host-key fingerprint through a trusted channel when connecting. These administrative commands do not use the restricted MTV key.

From the workstation, discover the ESXi interpreter and confirm the datastore is mounted:

```bash
ssh "root@${ESXI_HOST}" 'command -v python; command -v python3'
ssh "root@${ESXI_HOST}" "test -d '/vmfs/volumes/${DATASTORE_UUID}'"

# Set this to the absolute interpreter path returned above.
ESXI_PYTHON='/bin/python'
ssh "root@${ESXI_HOST}" "'${ESXI_PYTHON}' --version"
```

The interpreter must support the supplied Python file. The historical wrapper uses Python 3 features. Stop if a compatible interpreter is unavailable.

### 2.3. Upload and directly test the Python file

First inspect any existing wrapper and preserve it before replacement. If MTV has already uploaded the matching file, skip the upload. Otherwise, upload the release-matched file with the administrative connection; the command below refuses to overwrite an existing path. The supplied path values must not contain shell quotes.

```bash
test -f "$PYTHON_WRAPPER_LOCAL"
ssh "root@${ESXI_HOST}" "ls -l '${WRAPPER_PATH}'"

# Run only when the file does not already exist.
ssh "root@${ESXI_HOST}" \
  "umask 077; set -C; cat > '${WRAPPER_PATH}'" \
  < "$PYTHON_WRAPPER_LOCAL"

ssh "root@${ESXI_HOST}" \
  "chown root:root '${WRAPPER_PATH}' && chmod 600 '${WRAPPER_PATH}'"

# Read-only wrapper operation; no disk clone is started.
ssh "root@${ESXI_HOST}" \
  "unset SSH_ORIGINAL_COMMAND; '${ESXI_PYTHON}' '${WRAPPER_PATH}' --version"
```

Expect the wrapper's XML response with status `0` and its version. Invoking Python explicitly does not require executable permission on the file. For a shared datastore, upload once and verify the path and interpreter on every participating host. MTV can upload its embedded wrapper during migration, so the manually installed file must match that build.

### 2.4. Install the restricted MTV public key

Create a private local working directory and extract the generated public key. Keep using the same workstation shell for the remaining steps.

```bash
umask 077
MTV_SSH_WORKDIR=$(mktemp -d "${TMPDIR:-/tmp}/mtv-esxi-ssh.XXXXXX")

oc get secrets -n "$MTV_NAMESPACE" \
  -l app.kubernetes.io/component=ssh-keys

oc get secret "offload-ssh-keys-${VSPHERE_PROVIDER}-public" \
  -n "$MTV_NAMESPACE" -o jsonpath='{.data.public-key}' \
  | base64 -d > "$MTV_SSH_WORKDIR/generated-public-key"

# Accept the RSA public key with or without generated authorized_keys options.
# Rebuild its forced command below for the verified Python implementation.
awk '{for (i=1; i<NF; i++) if ($i == "ssh-rsa") {print $i " " $(i+1); exit}}' \
  "$MTV_SSH_WORKDIR/generated-public-key" \
  > "$MTV_SSH_WORKDIR/esxi_public_key.pub"
ssh-keygen -lf "$MTV_SSH_WORKDIR/esxi_public_key.pub"

printf 'command="%s %s",no-port-forwarding,no-agent-forwarding,no-X11-forwarding,no-pty %s\n' \
  "$ESXI_PYTHON" "$WRAPPER_PATH" \
  "$(cat "$MTV_SSH_WORKDIR/esxi_public_key.pub")" \
  > "$MTV_SSH_WORKDIR/restricted_key.pub"
cat "$MTV_SSH_WORKDIR/restricted_key.pub"
```

Do not continue if extraction or fingerprint validation fails. The resulting line must invoke the verified interpreter and `.py` path, followed by the public key. It must not contain two `command=` options. If this key is already installed, inspect its existing entry first: update that entry to the intended restriction rather than leaving an unrestricted or conflicting entry for the same key.

For a new key, install it with the administrative connection, retaining existing unrelated keys:

```bash
ssh "root@${ESXI_HOST}" '
  set -eu
  umask 077
  mkdir -p /etc/ssh/keys-root
  if [ -f /etc/ssh/keys-root/authorized_keys ]; then
    cp -p /etc/ssh/keys-root/authorized_keys "/etc/ssh/keys-root/authorized_keys.before-mtv.$(date +%Y%m%d%H%M%S)"
  fi
  printf "\n" >> /etc/ssh/keys-root/authorized_keys
  cat >> /etc/ssh/keys-root/authorized_keys
  chmod 600 /etc/ssh/keys-root/authorized_keys
' < "$MTV_SSH_WORKDIR/restricted_key.pub"
```

Repeat host checks and key installation for all participating ESXi hosts. This forced command uses one fixed datastore path; recheck the procedure before expanding to other datastores or upgrading MTV.

### 2.5. Test the restricted connection and clean up

Run from a controlled workstation with access to the key Secret. This test temporarily exports the private key with mode `600` and removes that local copy on exit; MTV continues to use its Kubernetes Secret. Do not enable shell tracing. The test uses only the specified identity, disabling agent and configured identity fallback.

```bash
(
  set -e
  umask 077
  MTV_TEST_KEY="$MTV_SSH_WORKDIR/esxi_private_key"
  trap 'rm -f "$MTV_TEST_KEY"' EXIT

  oc get secret "offload-ssh-keys-${VSPHERE_PROVIDER}-private" \
    -n "$MTV_NAMESPACE" -o jsonpath='{.data.private-key}' \
    | base64 -d > "$MTV_TEST_KEY"
  chmod 600 "$MTV_TEST_KEY"

  # Pass wrapper arguments only: sshd invokes Python via the forced command.
  ssh -F /dev/null -T -o IdentityAgent=none -o IdentitiesOnly=yes \
    -o BatchMode=yes -o StrictHostKeyChecking=yes \
    -i "$MTV_TEST_KEY" "root@${ESXI_HOST}" '--version'

  # Expected: argparse rejects "id"; no shell command is executed.
  if ssh -F /dev/null -T -o IdentityAgent=none -o IdentitiesOnly=yes \
    -o BatchMode=yes -o StrictHostKeyChecking=yes \
    -i "$MTV_TEST_KEY" "root@${ESXI_HOST}" 'id'; then
    printf '%s\n' 'FAIL: unrelated command was accepted.' >&2
    exit 1
  else
    MTV_REJECT_STATUS=$?
    printf 'Rejected command exit status: %s (expected argparse status 2).\n' \
      "$MTV_REJECT_STATUS"
    test "$MTV_REJECT_STATUS" -eq 2
  fi
)
```

Require both a successful version response and an explicit wrapper rejection of `id`. Authentication, host-key, network, or missing-file errors are not successful restriction tests. Repeat for every host. Also verify connectivity from the actual migration-pod network; workstation success alone does not establish that route.

Once all host tests finish, remove the local public-key staging files:

```bash
rm -f "$MTV_SSH_WORKDIR/generated-public-key" \
  "$MTV_SSH_WORKDIR/esxi_public_key.pub" \
  "$MTV_SSH_WORKDIR/restricted_key.pub"
rmdir "$MTV_SSH_WORKDIR"
unset MTV_SSH_WORKDIR
```

The commands above configure and test SSH only. MTV starts cloning when the pilot migration runs in step 4. See [Red Hat SSH setup](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.12/html/planning_your_migration_to_red_hat_openshift_virtualization/assembly_planning-migration-vmware_mtv) for the documented key restrictions and [Forklift SSH setup](https://pkg.go.dev/github.com/kubev2v/forklift/cmd/vsphere-xcopy-volume-populator) for implementation context.

## 3. Configure Primera credentials and storage mapping

### 3.1. Inspect and decide whether to reuse each map

List maps and the Plans that reference them. These commands assume `MTV_NAMESPACE` and `VSPHERE_PROVIDER` were selected in section 2.1 and that `jq` is installed:

```bash
oc get storagemaps.forklift.konveyor.io -n "$MTV_NAMESPACE"
oc get networkmaps.forklift.konveyor.io -n "$MTV_NAMESPACE"
oc get plans.forklift.konveyor.io -n "$MTV_NAMESPACE" \
  -o custom-columns='PLAN:.metadata.name,SOURCE:.spec.provider.source.name,DESTINATION:.spec.provider.destination.name,STORAGE-MAP:.spec.map.storage.name,NETWORK-MAP:.spec.map.network.name'

read -r -p 'Existing StorageMap from a successful migration: ' EXISTING_STORAGE_MAP
export EXISTING_STORAGE_MAP
MAP_REVIEW_DIR=$(mktemp -d "${TMPDIR:-/tmp}/mtv-map-review.XXXXXX")
export MAP_REVIEW_DIR
oc get storagemaps.forklift.konveyor.io "$EXISTING_STORAGE_MAP" \
  -n "$MTV_NAMESPACE" -o json > "$MAP_REVIEW_DIR/original-storagemap.json"
jq '{name: .metadata.name, owners: .metadata.ownerReferences,
     provider: .spec.provider, entries: .spec.map}' \
  "$MAP_REVIEW_DIR/original-storagemap.json"
```

Check provider names, namespaces, and any stored UIDs against the current Providers. Match every pilot VM disk's datastore to a StorageMap entry and every NIC's source network to a NetworkMap entry. Prior success for a different VM does not prove coverage for this VM.

| Existing resource | Use in this pilot |
|---|---|
| Working Primera StorageClass | Reuse after step-1 checks; keep its CSI/backend configuration. |
| Ownerless StorageMap with correct Providers, datastore coverage, class, and Primera offload configuration | Reference unchanged from the new Plan. |
| StorageMap from ordinary network-copy migrations, or one owned by an earlier Plan | Create a clean pilot copy and configure offload there; preserve the original. |
| Ownerless NetworkMap with correct Providers, network coverage, and the intended isolated destinations | Reference unchanged; storage offload requires no network-map-specific setting. |
| NetworkMap owned by an earlier Plan | Copy its spec into a new map if its destinations are suitable. |
| NetworkMap leading to production networks or missing pilot networks | Create a separate pilot map using verified isolated destinations. |

Inspect `metadata.ownerReferences` even when a previous migration succeeded: a map owned by an old Plan can be garbage-collected with that Plan. The MTV UI's **Use an existing map** flow makes a Plan-owned copy of an ownerless map. The CLI examples here reference the named object directly, so explicitly copying the map avoids relying on the old Plan's lifecycle. [Red Hat map reuse behavior](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.12/html/planning_your_migration_to_red_hat_openshift_virtualization/assembly_planning-migration-vmware_mtv)

### 3.2. Select or create the offload Secret

Verify HTTPS connectivity to the **verified destination array’s** WSAPI endpoint, normally TCP 443, and the array account’s required permissions. [HPE integration requirements](https://scod.hpedev.io/csi_driver/container_storage_provider/hpe_alletra_storage_mp_b10000/index.html)

An existing CSI backend Secret serves the CSI driver. MTV requires the offload credential keys below. Reuse an existing dedicated MTV offload Secret only after verifying its array endpoint and account through your credential-management process; otherwise create one. Do not overwrite the working CSI Secret.

Set the selected Secret name and inspect only its metadata and key names:

```bash
read -r -p 'Existing or planned MTV Primera offload Secret name: ' OFFLOAD_SECRET_NAME
export OFFLOAD_SECRET_NAME
# For an existing Secret only; values are not displayed.
oc get secret "$OFFLOAD_SECRET_NAME" -n "$MTV_NAMESPACE" \
  -o go-template='{{.metadata.name}}{{"\n"}}{{range $key, $value := .data}}{{$key}}{{"\n"}}{{end}}'
```

For a new Secret, use this template through your credential-management process. Substitute its name and namespace to match the selected values:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: primera-offload
  namespace: openshift-mtv
type: Opaque
stringData:
  STORAGE_HOSTNAME: "https://<verified-primera-management-address>:443"
  STORAGE_USERNAME: "<array-user>"
  STORAGE_PASSWORD: "<array-password>"
  STORAGE_SKIP_SSL_VERIFICATION: "false"
```

Use the complete WSAPI URL and establish certificate trust. [Red Hat Primera Secret requirements](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.12/html/planning_your_migration_to_red_hat_openshift_virtualization/assembly_planning-migration-vmware_mtv)

### 3.3. Reuse a verified offload StorageMap

If the existing map is ownerless, has matching Providers, maps all pilot datastores to the verified class, and already uses `offloadPlugin.vsphereXcopyConfig` with `storageVendorProduct: primera3par` and the verified `secretRef`, reuse it without changes:

```bash
export STORAGE_MAP_NAME="$EXISTING_STORAGE_MAP"
```

Then skip section 3.4. A StorageMap that only specifies `destination.storageClass` can be suitable for ordinary migration but does not select XCOPY. Check each pilot entry, not just whether an offload setting appears anywhere in the map.

### 3.4. Create a separate offload StorageMap when needed

Use this path when offload settings must be added or the old map is Plan-owned. The copy command preserves the selected entries' destination settings, including explicit access/volume modes, and changes only the selected class and offload plugin. It includes only the verified pilot datastores. Supply every datastore used by the pilot VM, using vSphere IDs such as `datastore-123`, not VMFS UUIDs.

```bash
read -r -p 'Pilot datastore IDs, separated by spaces: ' -a OFFLOAD_DATASTORE_IDS
OFFLOAD_DATASTORE_IDS_JSON=$(jq -n '$ARGS.positional' --args "${OFFLOAD_DATASTORE_IDS[@]}")
export STORAGE_MAP_NAME="primera-offload-$(date +%Y%m%d%H%M%S)"

jq -e \
  --arg name "$STORAGE_MAP_NAME" \
  --arg ns "$MTV_NAMESPACE" \
  --arg sc "$HPE_STORAGE_CLASS" \
  --arg secret "$OFFLOAD_SECRET_NAME" \
  --argjson ids "$OFFLOAD_DATASTORE_IDS_JSON" '
  if ($ids | length) == 0 then error("No pilot datastore IDs supplied")
  elif (($ids - [.spec.map[].source.id]) | length) > 0 then
    error("A selected datastore ID is missing from the existing map")
  else
    {apiVersion, kind, metadata: {name: $name, namespace: $ns}, spec}
    | .spec.map = [.spec.map[]
        | select(.source.id as $id | $ids | index($id))
        | .destination.storageClass = $sc
        | .offloadPlugin = {vsphereXcopyConfig: {
            secretRef: $secret, storageVendorProduct: "primera3par"}}]
  end' "$MAP_REVIEW_DIR/original-storagemap.json" \
  > "$MAP_REVIEW_DIR/pilot-storagemap.json"

# Run only if jq succeeded; review before creating.
cat "$MAP_REVIEW_DIR/pilot-storagemap.json"
oc create --dry-run=server -f "$MAP_REVIEW_DIR/pilot-storagemap.json"
oc create -f "$MAP_REVIEW_DIR/pilot-storagemap.json"
```

The new metadata deliberately omits old owner references, finalizers, UID, resource version, managed fields, and status. Confirm copied access/volume modes support offload and the verified StorageProfile; do not automatically carry a filesystem-mode setting into a block-volume pilot. If an old entry uses source names instead of IDs, resolve them from inventory before using this ID-based copy command. Do not patch a shared map to add offload.

If no suitable old mapping exists, create a new map using this minimal template instead. Substitute the selected `STORAGE_MAP_NAME`, `OFFLOAD_SECRET_NAME`, `MTV_NAMESPACE`, Providers, datastore ID, and existing `HPE_STORAGE_CLASS`:

```yaml
apiVersion: forklift.konveyor.io/v1beta1
kind: StorageMap
metadata:
  name: primera-offload
  namespace: openshift-mtv
spec:
  provider:
    source:
      name: <vsphere-provider>
      namespace: openshift-mtv
    destination:
      name: <openshift-provider>
      namespace: openshift-mtv
  map:
    - source:
        id: <verified-datastore-id>
      destination:
        storageClass: <verified-hpe-storageclass>
      offloadPlugin:
        vsphereXcopyConfig:
          secretRef: primera-offload
          storageVendorProduct: primera3par
```

Repeat entries for additional verified datastores. For a fresh map, leave access mode unspecified initially and validate resulting PVC configuration against the working CSI StorageProfile. For a copied map, preserve compatible explicit settings. Record the final map name in `STORAGE_MAP_NAME` for step 4. [Red Hat StorageMap specification](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.12/html/migrating_your_virtual_machines_to_red_hat_openshift_virtualization/assembly_migrating-from-vmware_mtv)

## 4. Run and validate one pilot

### 4.1. Select the VM and pilot resources

Choose one noncritical VM and a downtime window. Complete the same-array gate in step 1 and the SSH tests in step 2. Confirm the vSphere account has the offload privileges: Global Settings, Datastore Browse and Low level file operations, and Host Configuration Advanced settings, Query patch, and Storage partition configuration, in addition to ordinary MTV privileges. Consolidate preexisting snapshots through your VMware workflow. [Red Hat offload prerequisites](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.12/html/planning_your_migration_to_red_hat_openshift_virtualization/assembly_planning-migration-vmware_mtv)

These examples assume migration to the **local OpenShift cluster**, with both Providers and both maps in `MTV_NAMESPACE`. They reuse `MTV_NAMESPACE` and `VSPHERE_PROVIDER` from section 2.1. Use a VM with one source network for the first pilot; add a mapping for each additional source network if needed. Every source datastore used by the VM must already have a verified entry in the step-3 StorageMap.

Discover the available resources:

```bash
oc get providers.forklift.konveyor.io -n "$MTV_NAMESPACE" \
  -o custom-columns='NAME:.metadata.name,TYPE:.spec.type,URL:.spec.url'
oc get storagemaps.forklift.konveyor.io -n "$MTV_NAMESPACE"
oc get networkmaps.forklift.konveyor.io -n "$MTV_NAMESPACE"
oc get network-attachment-definitions.k8s.cni.cncf.io -A
```

Select the Provider representing this cluster, commonly `host`. Start with an existing NetworkMap using those same Providers. Reuse it unchanged if it is ownerless and maps all pilot NICs to the intended isolated network; copy it if it is owned by an older Plan. If its destinations are unsuitable, select `new` below and choose an existing NetworkAttachmentDefinition (NAD) connected to a verified isolated test network. A NAD's name or its presence in a separate namespace does not establish network isolation; verify its backing VLAN/bridge or overlay configuration and reachability with the platform administrator. This runbook does not create a new physical network.

Obtain the source VM and network managed-object references from MTV inventory or vSphere. With an already authenticated `govc`, these optional commands print IDs for known inventory paths:

```bash
govc ls -I '/Datacenter/vm/pilot-vm'
govc ls -I '/Datacenter/network/pilot-portgroup'
```

Replace the example paths with the VM and its actual attached network. Use IDs such as `vm-123`, `network-42`, or `dvportgroup-42`; the VM ID is not its BIOS UUID. [VMware govc reference](https://github.com/vmware/govmomi/blob/main/govc/USAGE.md)

Set the pilot values in the same Bash session:

```bash
read -r -p 'Local OpenShift destination Provider name: ' DESTINATION_PROVIDER
read -r -p 'Pilot VMware VM ID (for example vm-123): ' PILOT_VM_ID
read -r -p 'Source network ID attached to the pilot VM: ' SOURCE_NETWORK_ID
export DESTINATION_PROVIDER PILOT_VM_ID SOURCE_NETWORK_ID

export TARGET_NAMESPACE='mtv-primera-pilot'
# Keep STORAGE_MAP_NAME selected or created in section 3.
printf 'Selected offload StorageMap: %s\n' "${STORAGE_MAP_NAME:?Complete section 3 first}"
export PLAN_NAME="primera-pilot-$(date +%Y%m%d%H%M%S)"
export TARGET_VM_NAME="$PLAN_NAME"
export MIGRATION_NAME="${PLAN_NAME}-run1"
PILOT_WORKDIR=$(mktemp -d "${TMPDIR:-/tmp}/mtv-primera-pilot.XXXXXX")
export PILOT_WORKDIR
printf 'Pilot manifests and evidence directory: %s\n' "$PILOT_WORKDIR"

oc get providers.forklift.konveyor.io "$DESTINATION_PROVIDER" \
  -n "$MTV_NAMESPACE" -o yaml
oc get storagemaps.forklift.konveyor.io "$STORAGE_MAP_NAME" \
  -n "$MTV_NAMESPACE" -o yaml
oc get namespace "$TARGET_NAMESPACE"
```

Select one network-map action. `reuse` references an existing map unchanged; `copy` creates a clean object with the existing spec; `new` writes the isolated-network example in section 4.2.

```bash
read -r -p 'NetworkMap action (reuse, copy, or new): ' NETWORK_MAP_ACTION
export NETWORK_MAP_ACTION
case "$NETWORK_MAP_ACTION" in
  reuse|copy)
    read -r -p 'Existing NetworkMap name: ' EXISTING_NETWORK_MAP
    export EXISTING_NETWORK_MAP
    oc get networkmaps.forklift.konveyor.io "$EXISTING_NETWORK_MAP" \
      -n "$MTV_NAMESPACE" -o json > "$PILOT_WORKDIR/original-networkmap.json"
    jq '{name: .metadata.name, owners: .metadata.ownerReferences,
         provider: .spec.provider, entries: .spec.map}' \
      "$PILOT_WORKDIR/original-networkmap.json"
    if [ "$NETWORK_MAP_ACTION" = reuse ]; then
      export NETWORK_MAP_NAME="$EXISTING_NETWORK_MAP"
      # A failed check means choose copy and rerun this selection block.
      jq -e '((.metadata.ownerReferences // []) | length) == 0' \
        "$PILOT_WORKDIR/original-networkmap.json"
    else
      export NETWORK_MAP_NAME="${PLAN_NAME}-network"
    fi
    ;;
  new)
    export NETWORK_MAP_NAME="${PLAN_NAME}-network"
    read -r -p 'Isolated test NetworkAttachmentDefinition name: ' PILOT_NAD
    read -r -p 'Namespace containing that NetworkAttachmentDefinition: ' PILOT_NAD_NAMESPACE
    export PILOT_NAD PILOT_NAD_NAMESPACE
    oc get network-attachment-definitions.k8s.cni.cncf.io "$PILOT_NAD" \
      -n "$PILOT_NAD_NAMESPACE" -o yaml
    ;;
  *) printf '%s\n' 'Invalid action: rerun this block with reuse, copy, or new.' >&2; false ;;
esac
```

Stop on failed lookups or an invalid action. For reuse/copy, inspect the referenced destination NADs with `oc get network-attachment-definitions.k8s.cni.cncf.io <name> -n <namespace> -o yaml`, substituting each entry's values. A pod-network destination also needs an explicit isolation review; it is not automatically an isolated test network. The `copy` action does not change destinations, so choose `new` if the previous map routes the pilot onto production networks.

Verify both maps' source/destination Providers match the selections, and that each pilot storage entry has `storageVendorProduct: primera3par`, the intended offload Secret, and the verified existing class. Confirm the target namespace can use all selected network destinations. If the target namespace does not exist, create it before continuing:

```bash
# Run only if the namespace lookup returned NotFound.
oc create namespace "$TARGET_NAMESPACE"
```

### 4.2. Reuse or copy the NetworkMap and write a new cold Plan

The following commands write a NetworkMap only for `new` or `copy`, and always write a new pilot Plan. Existing successful Plans are retained as history. The new Plan leaves the target VM powered off after migration so its generated configuration can be inspected before boot. The installed schema must support `spec.targetPowerState`; verify it first:

```bash
oc explain plans.spec.targetPowerState \
  --api-version=forklift.konveyor.io/v1beta1
```

If the field is unavailable, stop and reconcile the example with your installed MTV patch release. The field and its `off` value are defined in the [MTV Plan API](https://github.com/kubev2v/forklift/blob/release-2.12/pkg/apis/forklift/v1beta1/plan.go).

```bash
if [ "$NETWORK_MAP_ACTION" = new ]; then
cat > "$PILOT_WORKDIR/networkmap.yaml" <<EOF
apiVersion: forklift.konveyor.io/v1beta1
kind: NetworkMap
metadata:
  name: "${NETWORK_MAP_NAME}"
  namespace: "${MTV_NAMESPACE}"
spec:
  provider:
    source:
      name: "${VSPHERE_PROVIDER}"
      namespace: "${MTV_NAMESPACE}"
    destination:
      name: "${DESTINATION_PROVIDER}"
      namespace: "${MTV_NAMESPACE}"
  map:
    - source:
        id: "${SOURCE_NETWORK_ID}"
      destination:
        type: multus
        name: "${PILOT_NAD}"
        namespace: "${PILOT_NAD_NAMESPACE}"
EOF
elif [ "$NETWORK_MAP_ACTION" = copy ]; then
  jq --arg name "$NETWORK_MAP_NAME" --arg ns "$MTV_NAMESPACE" \
    '{apiVersion, kind, metadata: {name: $name, namespace: $ns}, spec}' \
    "$PILOT_WORKDIR/original-networkmap.json" \
    > "$PILOT_WORKDIR/networkmap.yaml"
fi

cat > "$PILOT_WORKDIR/plan.yaml" <<EOF
apiVersion: forklift.konveyor.io/v1beta1
kind: Plan
metadata:
  name: "${PLAN_NAME}"
  namespace: "${MTV_NAMESPACE}"
spec:
  warm: false
  skipGuestConversion: false
  targetPowerState: "off"
  targetNamespace: "${TARGET_NAMESPACE}"
  provider:
    source:
      name: "${VSPHERE_PROVIDER}"
      namespace: "${MTV_NAMESPACE}"
    destination:
      name: "${DESTINATION_PROVIDER}"
      namespace: "${MTV_NAMESPACE}"
  map:
    network:
      name: "${NETWORK_MAP_NAME}"
      namespace: "${MTV_NAMESPACE}"
    storage:
      name: "${STORAGE_MAP_NAME}"
      namespace: "${MTV_NAMESPACE}"
  vms:
    - id: "${PILOT_VM_ID}"
      targetName: "${TARGET_VM_NAME}"
EOF
```

The `new` example binds the source network to the selected Multus NAD. A `copy` retains the existing map's full spec while discarding old object ownership and server metadata; its output is JSON, which Kubernetes accepts even though the file uses a `.yaml` suffix. With `reuse`, no network-map file is needed. All three paths reference the selected `NETWORK_MAP_NAME` and `STORAGE_MAP_NAME` in the new Plan. [Red Hat CLI configuration examples](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.12/html/migrating_your_virtual_machines_to_red_hat_openshift_virtualization/assembly_migrating-from-vmware_mtv)

### 4.3. Validate and register the plan

Review the selected maps and Plan. Create the NetworkMap only when `new` or `copy` was selected, then validate and create the new Plan. These commands register the plan; the separate Migration resource in section 4.4 starts migration. Do not recreate or apply a manifest over a reused map.

```bash
if [ "$NETWORK_MAP_ACTION" = reuse ]; then
  oc get networkmaps.forklift.konveyor.io "$NETWORK_MAP_NAME" \
    -n "$MTV_NAMESPACE" -o yaml
else
  cat "$PILOT_WORKDIR/networkmap.yaml"
  oc create --dry-run=server -f "$PILOT_WORKDIR/networkmap.yaml" && \
    oc create -f "$PILOT_WORKDIR/networkmap.yaml"
fi

cat "$PILOT_WORKDIR/plan.yaml"
oc create --dry-run=server -f "$PILOT_WORKDIR/plan.yaml"
oc create -f "$PILOT_WORKDIR/plan.yaml"

oc wait --for=condition=Ready \
  "plans.forklift.konveyor.io/${PLAN_NAME}" \
  -n "$MTV_NAMESPACE" --timeout=300s

oc get plans.forklift.konveyor.io "$PLAN_NAME" \
  -n "$MTV_NAMESPACE" -o yaml
```

Run each command only after the previous command succeeds. Dry-run acceptance checks the API request; it does not prove SAN access, SSH connectivity, or offload functionality. If readiness times out, inspect the Plan's conditions and VM concerns and resolve them before proceeding. Do not overwrite an existing Plan if a name collision occurs.

### 4.4. Start the cold migration during the downtime window

Confirm the selected source VM, completed readiness checks, and application shutdown procedure. Retain its original disks for rollback. **Creating the Migration resource starts the migration and can shut down the source VM.** The cold example has no scheduled cutover field.

```bash
cat > "$PILOT_WORKDIR/migration.yaml" <<EOF
apiVersion: forklift.konveyor.io/v1beta1
kind: Migration
metadata:
  name: "${MIGRATION_NAME}"
  namespace: "${MTV_NAMESPACE}"
spec:
  plan:
    name: "${PLAN_NAME}"
    namespace: "${MTV_NAMESPACE}"
EOF

oc create --dry-run=server -f "$PILOT_WORKDIR/migration.yaml"
```

Execute the next command only when ready to start:

```bash
oc create -f "$PILOT_WORKDIR/migration.yaml"
```

Use a new Migration name for an intentional retry after diagnosing and cleaning up a failed attempt according to MTV guidance. [Red Hat Migration resource procedure](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.12/html/migrating_your_virtual_machines_to_red_hat_openshift_virtualization/assembly_migrating-from-vmware_mtv)

### 4.5. Monitor transfer and collect XCOPY evidence

```bash
# Watch plan condition columns; Ctrl-C ends the watch, not the migration.
oc get plans.forklift.konveyor.io "$PLAN_NAME" \
  -n "$MTV_NAMESPACE" --watch
```

In another terminal with the same variables set, or after stopping the watch:

```bash
oc get migrations.forklift.konveyor.io "$MIGRATION_NAME" \
  -n "$MTV_NAMESPACE" -o yaml
oc get plans.forklift.konveyor.io "$PLAN_NAME" \
  -n "$MTV_NAMESPACE" -o json > "$PILOT_WORKDIR/plan-status.json"
oc get pods -n "$TARGET_NAMESPACE" -o wide
oc get pvc -n "$TARGET_NAMESPACE" -o wide
oc get events -n "$TARGET_NAMESPACE" --sort-by=.metadata.creationTimestamp
```

Inspect `status.conditions` and the VM's migration pipeline in the Plan. Require `Succeeded=True` for completion. If transfer stalls or fails, select the relevant populator or conversion pod while it still exists and collect its logs:

```bash
read -r -p 'Pilot populator or conversion pod name from the pod listing: ' PILOT_POD
oc describe pod "$PILOT_POD" -n "$TARGET_NAMESPACE"
oc logs "$PILOT_POD" -n "$TARGET_NAMESPACE" \
  --all-containers=true --timestamps \
  > "$PILOT_WORKDIR/${PILOT_POD}.log"
```

For MTV 2.12.5 and later, inspect persistent per-disk `xcopyUsed` annotations. This `jq` query reports each matching status path and value without assuming a pipeline array index:

```bash
oc get plans.forklift.konveyor.io "$PLAN_NAME" \
  -n "$MTV_NAMESPACE" -o json > "$PILOT_WORKDIR/plan-status.json"

jq '.status | paths as $p
    | select($p[-1] == "xcopyUsed")
    | {path: $p, value: getpath($p)}' \
  "$PILOT_WORKDIR/plan-status.json"
```

Correlate each result with the VM and disk transfer task in the full JSON. Require affirmative XCOPY evidence for every migrated disk. Empty output is not proof of offload; `false` indicates fallback. On earlier patches, retain available populator metrics or logs before pods disappear. MTV 2.12.3 exposes `vsphere_xcopy_volume_populator_xcopy_used`, and 2.12.5 adds persistent status reporting. [MTV reporting changes](https://docs.redhat.com/en/documentation/migration_toolkit_for_virtualization/2.12/html/release_notes/ref_rn-2-12_release-notes)

### 4.6. Boot and validate the target VM

After migration succeeds, inspect the generated VM and its PVC references before starting it:

```bash
oc get virtualmachines.kubevirt.io "$TARGET_VM_NAME" \
  -n "$TARGET_NAMESPACE" -o yaml
oc get pvc -n "$TARGET_NAMESPACE" \
  -o custom-columns='NAME:.metadata.name,STATUS:.status.phase,CLASS:.spec.storageClassName,MODE:.spec.volumeMode,VOLUME:.spec.volumeName'
```

Verify the VM references the expected disks and only the intended test network. Confirm the source VM remains off. With `virtctl` installed for your OpenShift Virtualization version:

```bash
virtctl start "$TARGET_VM_NAME" -n "$TARGET_NAMESPACE"
oc get virtualmachineinstances.kubevirt.io "$TARGET_VM_NAME" \
  -n "$TARGET_NAMESPACE" --watch

# After ending the watch, open the guest's serial console if configured.
virtctl console "$TARGET_VM_NAME" -n "$TARGET_NAMESPACE"
```

Use the OpenShift web console if the guest has no serial console configured. See the [KubeVirt start and console examples](https://kubevirt.io/labs/kubernetes/lab1). VM readiness alone is insufficient: verify disk count and capacity, filesystems, representative file checksums against a pre-migration baseline, application startup, and required networking. For a Linux pilot, example guest checks are:

```bash
lsblk -f
findmnt
ip -br address
ip route
systemctl --failed
# Replace this with a representative file recorded before migration.
sha256sum /path/to/known-test-file
```

Record results and transfer/conversion/downtime measurements in the pilot evidence directory. Accept the pilot only after guest and application checks pass and every disk has XCOPY evidence. If acceptance fails, follow step 5 before returning the source to service; do not delete source disks or automatically remove target PVCs.

## 5. Rollback, rollout, and defaults

Retain the source VM and disks until acceptance. On failure, stop the destination before restarting the source; reconcile any destination writes before reverting.

Inspect temporary mappings and PVC reclaim behavior before cleanup. Restore shared settings only after checking other migrations’ dependencies.

Keep an inventory of reused versus pilot-created maps and Secrets. Cleanup must retain reused NetworkMaps, StorageMaps, the existing Primera StorageClass, CSI backend Secrets, and previous Plans. Remove only pilot-created resources after checking current references and reclaim behavior. Reusing a map never means reusing a prior VM's PVC or overwriting its disks.

Expand to small batches after pilot acceptance. Repeat storage-placement verification whenever source datastores or destination CSI backends change.

**Defaults:** MTV 2.12, MTV-generated SSH keys, reuse of the existing Primera CSI StorageClass, inspection and reuse of suitable ownerless maps, separate copies when ownership or offload settings require them, one cold pilot, isolated validation networking, and TLS verification enabled. **Same-array placement remains a required verification—not an assumption.** No cluster changes or migrations have been performed.
