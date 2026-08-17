# UPS Production Cluster CP4D 5.3.1.0 to 5.4.0.5 Upgrade

## Author: Alex Kuan (alex.kuan@ibm.com)

**From:**
```
CPD: 5.3.1.0
OCP: 4.20.25
Storage: Google Cloud Netapp Volumes and Persistent Disk on Google Cloud
Internet: airgap
Private container registry: yes
Components: ibm-licensing,ibm_events_operator,ccs,cpfs,zen,cpd_platform,watsonx_orchestrate,watsonx_ai,watsonx_governance,watson_speech,voice_gateway,db2oltp,cognos_analytics
```

**To:**
```
CPD: 5.4.0.5
OCP: 4.20.25
Storage: Google Cloud Netapp Volumes and Persistent Disk on Google Cloud
Internet: airgap
Private container registry: yes
Components: ibm-licensing,ibm_events_operator,ccs,cpfs,zen,cpd_platform,watsonx_orchestrate,watsonx_ai,watsonx_governance,watson_speech,voice_gateway,db2oltp,cognos_analytics
```

---

## Table of Contents
- Prerequisites
- Pre Upgrade Steps
- Pre Upgrade Health Check
- Upgrade Shared Cluster Components
- Upgrade IBM Software Hub Platform and Services
- Upgrade Service Instances
- Upgrade Cpdbr Service
- Post Upgrade Validation

---

## Prerequisites

Backup of the cluster is complete

Backup your IBM Software Hub cluster before the upgrade
**Reference**: [Backing up and restoring IBM Software Hub](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=administering-backing-up-restoring-software-hub)

The latest olm-utils-v4 image is available
**Reference**: [Obtaining the olm-utils-v4 image before running IBM Software Hub installation](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=pruirn-obtaining-olm-utils-v4-image)

Case packages are downloaded on the workstation
**Reference**: [Downloading CASE packages before running IBM Software Hub upgrade](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=pruirn-downloading-case-packages)

The image mirroring completed successfully
**Reference**: [Mirroring images to private image registry](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=mipcr-mirroring-images-directly-private-container-registry)

Here is an example of the case-download syntax
```bash
cpd-cli manage case-download --components=${COMPONENTS} --release=${VERSION} --patch_id=0 --from_oci=true
```

Remember to download the CASE package for the ibm_events_operator component
```bash
cpd-cli manage case-download --components=ibm_events_operator --release=${VERSION} --patch_id=${PATCH_ID} --from_oci=true
```

Here is an example of the mirror-images syntax
```bash
cpd-cli manage mirror-images --components=${COMPONENTS} --release=${VERSION} --patch_id=${PATCH_ID} --target_registry=${PRIVATE_REGISTRY_LOCATION} --arch=${IMAGE_ARCH} --case_download=false
```

The permissions required for the upgrade is ready
- OpenShift cluster administrator permissions
- IBM Software Hub administrator permissions
- Permission to access the private image registry for pushing or pulling images
- Access to the bastion node for executing the upgrade commands

A pre-upgrade health check is made to ensure the cluster's readiness for upgrade

- The OpenShift cluster, persistent storage and IBM Software Hub platform and services are in healthy status

---

## Pre Upgrade Steps

Required Tools

Ensure the following tools are installed and updated to the required versions:
- IBM Software Hub CLI: Version 14.4.0.3
- OpenShift CLI (oc): Compatible version for your cluster
- Helm CLI: Version 4.1.4

**Installation and Update Instructions:**

For detailed instructions on installing or updating these tools, refer to:
- [Updating client workstations](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=53-updating-client-workstations)
- [Updating IBM Software Hub CLI](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=ucw-updating-software-hub-cli)
- [Updating OpenShift CLI](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=ucw-updating-openshift-cli)
- [Installing Helm CLI](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=ucw-installing-helm-cli)

Access Requirements

**Required Access:**
- OpenShift cluster admin access
- IBM Entitlement Key with appropriate permissions
- Access to IBM Container Registry (cp.icr.io)
- Access to private registry: UPDATE_WITH_PRIVATE_REGISTRY_URL

---

Ensure your environment variables script is configured correctly

**Reference**: [Updating your environment variables script](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=cri-updating-your-environment-variables-script)

Source your environment variables script
```bash
source cpd_vars.sh
```

Set version and patch_id
```bash
export VERSION=5.4.0
export PATCH_ID=5
```

Login to the cluster
```bash
${OC_LOGIN} && ${CPDM_OC_LOGIN}
```

Verify nodes, machine config, cluster operators
```bash
oc get nodes,mcp,co
```

Take a backup of the routes
```bash
oc get routes -n ${PROJECT_CPD_INST_OPERANDS} -o yaml > routes_backup_$(date +%Y%m%d_%H%M%S).yaml
```

Take a backup of the temporary patches for watson assistant
```bash
oc get TemporaryPatch -n ${PROJECT_CPD_INST_OPERANDS} -o yaml > temporarypatch_backup_$(date +%Y%m%d_%H%M%S).yaml
```

List all of the temporary patches in the operands namespace
```bash
oc get TemporaryPatch -n ${PROJECT_CPD_INST_OPERANDS}
```

For all patches that you want to retain, use the following command:
```bash
oc label TemporaryPatch <patch_name> type=critical-configuration
```

For example
```bash
oc label TemporaryPatch wa-store-assistant-limits type=critical-configuration
```

---

#### Advanced Service Prerequisites

Some services require additional prerequisite software upgrades
 
**Reference**: [Upgrading prerequisite software](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=pyc-upgrading-prerequisite-software)
**Reference**: [Upgrade MCG](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=ups-upgrading-multicloud-object-gateway)
**Reference**: [Upgrading Red Hat OpenShift Serverless Knative Eventing](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=ups-upgrading-red-hat-openshift-serverless-knative-eventing)
**Reference**: [Upgrading Operators For Services That Require GPU](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=ups-upgrading-operators-services-that-require-gpus)
**Reference**: [Upgrading Red Hat OpenShift AI](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=ups-upgrading-red-hat-openshift-ai)

Based on the health check review, no action should be required for these steps

| Operator | Current CSV | Target for 5.4.0 | Action |
| --- | --- | --- | --- |
| OpenShift Serverless | 1.37.1 | 1.37.x | No action required |
| OpenShift AI (RHOAI) | 2.25.9 | 2.25.x | No action required |
| NVIDIA GPU Operator | 26.3.x | 26.3.x | No action required |
| Node Feature Discovery | 4.20.0 | 4.20.x | No action required |
| IBM Events Operator | 6.0.0 | 6.0.0 | No action required |

---

## Pre Upgrade Health Check

#### Basic Cluster Validation

Check node, machineConfig, clusterOperators, clusterVersion
```bash
oc get nodes,mcp,co,clusterversion
```

Verify storage classes
```bash
oc get sc
```

Check PVC status
```bash
oc get pvc -n ${PROJECT_CPD_INST_OPERANDS}
```

Check CR status
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

Check for pods not in Completed status
```bash
oc get po -A -owide | egrep -v '([0-9])/\1' | egrep -v 'Completed'
```

List service instances
```bash
cpd-cli service-instance list --profile=${CPD_PROFILE_NAME}
```

**Note**: Fix any pod issues and ensure the service CRs are in Completed status before proceeding with the upgrade

---

## Upgrade Shared Cluster Components

#### Upgrade IBM Licensing

**Reference**: [Upgrading shared cluster components](https://www.ibm.com/docs/en/cloud-paks/cp-data/5.4.x?topic=upgrading-shared-cluster-components)

Upgrade IBM Licensing service
```bash
cpd-cli manage apply-cluster-components --release=${VERSION} --patch_id=${PATCH_ID} --license_acceptance=true --licensing_ns=${PROJECT_LICENSE_SERVICE}
```

Verify licensing pods are running
```bash
oc get pods -n ${PROJECT_LICENSE_SERVICE}
```

---

#### Update Cluster-Scoped Resources for CPD Instance

**Reference**: [Updating cluster-scoped resources for the instance](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=puish-updating-cluster-scoped-resources-instance)

Generate cluster-scoped resource definitions for CPD instance
```bash
cpd-cli manage case-download --components=${COMPONENTS} --release=${VERSION} --patch_id=${PATCH_ID} --operator_ns=${PROJECT_CPD_INST_OPERATORS} --cluster_resources=true
```

Run the 'oc apply -f' command returned in the terminal, for example
```bash
oc apply -f /.../cpd-cli-workspace/olm-utils-workspace/work/cluster_scoped_resources.yaml
```

---

#### Upgrading the IBM Events Operator

**Reference**: [Upgrading the IBM Events Operator for watsonx Assistant or watsonx Orchestrate](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=puish-upgrading-events-operator)

Login to the cluster
```bash
${CPDM_OC_LOGIN}
```

Download case packages for ibm_events_operator
```bash
cpd-cli manage case-download --release=${VERSION} --patch_id=${PATCH_ID} --components=ibm_events_operator --from_oci=true
```

Generate cluster-scoped resource definitions for the IBM Events Operator
```bash
cpd-cli manage deploy-events-operator --release=${VERSION} --cluster_resources=true
```

Run the 'oc apply -f' command returned in the terminal, for example
```bash
oc apply -f /.../cpd-cli-workspace/olm-utils-workspace/work/cluster_scoped_resources.yaml
```

Upgrade the Red Hat OpenShift Serverless Knative Eventing software
```bash
cpd-cli manage deploy-knative-eventing --release=${VERSION} --block_storage_class=${STG_CLASS_BLOCK} --upgrade=true
```

Run the following command to upgrade the IBM Events Operator
```bash
cpd-cli manage deploy-events-operator --release=${VERSION} --events_operator_ns=${PROJECT_CPD_INST_OPERATORS} --events_operand_ns=${PROJECT_CPD_INST_OPERANDS}
```

---

#### Apply Entitlements

**Reference**: [Applying your entitlements](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=aye-applying-your-entitlements-without-node-pinning-2)

Login to the cluster
```bash
${CPDM_OC_LOGIN}
```

Apply the non-prod license for IBM Software Hub
```bash
cpd-cli manage apply-entitlement --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --entitlement=cpd-enterprise --production=false
```

Apply watsonx.ai non-prod license
```bash
cpd-cli manage apply-entitlement --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}  --entitlement=watsonx-ai --production=false
```

Apply watsonx.governance non prod licenses
```bash
cpd-cli manage apply-entitlement --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --entitlement=watsonx-gov-mm --production=false
cpd-cli manage apply-entitlement --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --entitlement=watsonx-gov-rc --production=false
```

Apply watsonx Orchestrate non prod license
```bash
cpd-cli manage apply-entitlement --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --entitlement=watsonx-orchestrate
```

Apply Watson Speech licenses, skip for non prod*
```bash
#cpd-cli manage apply-entitlement --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --entitlement=speech-to-text
#cpd-cli manage apply-entitlement --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --entitlement=text-to-speech
```

Apply Cognos Analytics license, skip for non prod*
```bash
#cpd-cli manage apply-entitlement --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --entitlement=cognos-analytics
```

---

## Upgrade IBM Software Hub Platform and Services

**Reference**: [Upgrading IBM Software Hub](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=upgrading)

Login to the cluster
```bash
${CPDM_OC_LOGIN}
```

#### Upgrade CPD platform using install-components
```bash
cpd-cli manage install-components \
--license_acceptance=true \
--components=cpd_platform \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--run_storage_tests=false \
--upgrade=true
```

Monitor platform upgrade progress (this takes 60-80 minutes)
```bash
watch -n 30 'oc get ZenService lite-cr -n ${PROJECT_CPD_INST_OPERANDS} -o jsonpath="{.status.zenStatus}"'
```

Check platform pods
```bash
oc get pods -n ${PROJECT_CPD_INST_OPERANDS} | grep -E "zen|usermgmt|ibm-nginx"
```

Verify platform version
```bash
oc get ZenService lite-cr -n ${PROJECT_CPD_INST_OPERANDS} -o jsonpath='{.status.zenStatus.versions[0].version}'
```

---

#### Reapply RSI Patches

**Reference**: [Upgrading Software Hub](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=uish-upgrading-software-hub)

If there are patches that apply to zen or IBM Cloud Pak foundational services pods, run the following command to apply your custom patches:
```bash
cpd-cli manage apply-rsi-patches --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

Verify patches are active
```bash
cpd-cli manage get-rsi-patch-info --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --all
```

Check that affected pods are running
```bash
oc get pods -n ${PROJECT_CPD_INST_OPERANDS}
```

---

## Upgrade Services

#### Upgrade Watsonx Orchestrate

If you plan to upgrade the previous versions of watsonx Orchestrate with custom upgrade options, specify the appropriate options in a file named install-options.yml in the cpd-cli work directory

You can identify the location of the work folder using below command in the cpd-cli work directory
```bash
podman inspect olm-utils-play-v4 | jq -r '.[0].Mounts' |jq -r '.[] | select(.Destination == "/tmp/work") | .Source'
```

Create the install-options.yml file in the cpd-cli-workspace/olm-utils-workspace/work directory
```bash
# ............................................................................
# watsonx Orchestrate parameters
# ............................................................................
non_olm:
  watsonxOrchestrate:
    installMode: "agentic_assistant"
    watsonxAI:
      watsonxaiifm: true
```

**IMPORTANT**: Before proceeding with Orchestrate upgrade, remove the following image_digests

Remove the image_digests section from watsonxaiifm-cr
```bash
oc patch watsonxaiifm watsonxaiifm-cr -n ${PROJECT_CPD_INST_OPERANDS} --type=json -p='[{"op": "remove", "path": "/spec/image_digests"}]'
```

Upgrade watsonx_orchestrate
```bash
cpd-cli manage install-components \
--license_acceptance=true \
--components=watsonx_orchestrate \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--param-file=/tmp/work/install-options.yml \
--upgrade=true
```

---

#### Potential Issue - Watson Assistant upgrade blocked during Watsonx Orchestrate upgrade 

The ephemeralDeployment data type was updated from Boolean to String, and this required an edit on the wo-wa-data-governor-opensearch-ephemeral temporarypatch in this section
```bash
spec:
  apiVersion: assistant.watson.ibm.com/v1
  kind: WatsonAssistant
  name: wo-wa
  patch:
    data-governor:
      datagovernoroverride:
        spec:
          dependencies:
            opensearch:
------------> ephemeralDeployment: true
```

Update the ephemeralDeployment value to a string
```bash
oc patch temporarypatch wo-wa-data-governor-opensearch-ephemeral -n ups-wx-operands --type=merge -p '{"spec":{"patch":{"data-governor":{"datagovernoroverride":{"spec":{"dependencies":{"opensearch":{"ephemeralDeployment":"true"}}}}}}}}'
```

Validate the patch updates
```bash
oc get patch wo-wa-data-governor-opensearch-ephemeral -o yaml
```

Monitor watsonx_orchestrate upgrade
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=watsonx_orchestrate
```

---

#### Post upgrade tasks for Watsonx Orchestrate

After completing this migration, follow the steps to apply the latest IBM watsonx Orchestrate release 5.4.0.5 Hot fix

**Reference**: [Apply hot fix for IBM watsonx Orchestrate](https://www.ibm.com/support/pages/node/7247038)

**Reference**: [Applying the watsonx Orchestrate 5.4.0 hot fix 1](https://www.ibm.com/support/pages/node/7247038)

Set the operator and operand namespaces
```bash
export PROJECT_CPD_INST_OPERATORS=ups-wx-operators
export PROJECT_CPD_INST_OPERANDS=ups-wx-operands
```

**Note**: You will need to install Skopeo and mirror the operator and operand images before proceeding

Create hotfix5405.sh 
```bash
vi hotfix5405.sh
```

With the following contents
```bash
#!/bin/bash
# -----------------------------------------------------------------------------
# watsonx Orchestrate 5.4.0.5 hot fix
# - Verifies watsonx Orchestrate version from .status.versionStatus.status.
# - Images of Operators are replaced with the image from hot fix script.
#   * HOTFIX_LABEL_VALUE for hot fix 1 is 5.4.0.5
#   * If an existing label value matches x.x.x.x and is higher than HOTFIX_LABEL_VALUE,
#     the script exits early after informing you
# - Deletes a fixed set of Jobs in the operands namespace and waits for all to reappear with
#   new UIDs and succeed
# -----------------------------------------------------------------------------

# -----------------------------
# Helpers
# -----------------------------
ts() { date +"%Y-%m-%d %H:%M:%S"; }

require() {
  command -v "$1" >/dev/null 2>&1 || { echo "[$(ts)] Missing required command: $1"; exit 1; }
}
get_wo_version() {
  ns="$1"
  oc get wo -n "$ns" -o jsonpath='{.items[0].status.versionStatus.status}' 2>/dev/null || true
}

get_wo_cr_version() {
  ns="$1"
  oc get wo -n "$ns" -o jsonpath='{.items[0].spec.version}' 2>/dev/null || true
}


# -----------------------------
# Patch operator deployment images
# -----------------------------

OPERATOR_IMAGES="icr.io/cpopen/ibm-watsonx-orchestrate-operator@sha256:5e75fea5876911150642c2701fd4a67f63cf3d43adfc3c78cdbd7e8ed94b952c
icr.io/cpopen/ibm-wxo-component-operator@sha256:c8bcd00b379fd85db5861c966358b536cd2da89c002fe3b58035b7c46c1f270a"

if [ -z "${PROJECT_CPD_INST_OPERATORS:-}" ]; then
  echo "[ERROR] PROJECT_CPD_INST_OPERATORS is not set. Exiting."
  exit 1
fi

# Required WO version
REQUIRED_WO_VERSION="${REQUIRED_WO_VERSION:-5.3.1}"
REQUIRED_WO_CR_VERSION="${REQUIRED_WO_CR_VERSION:-7.1.0}"

# -----------------------------
# Validations and version check
# -----------------------------
require oc

# Make sure oc login is done
if ! oc whoami &>/dev/null; then
    echo "[$(ts)] Error: Not logged in to OpenShift"
    exit 1
fi

echo "[$(ts)] Checking wo.status.versionStatus.status in ${PROJECT_CPD_INST_OPERANDS}"
WO_VER="$(get_wo_version "$PROJECT_CPD_INST_OPERANDS")"
WO_CR_VER="$(get_wo_cr_version "$PROJECT_CPD_INST_OPERANDS")"

# Check if versions match required versions
if [ "$WO_VER" != "$REQUIRED_WO_VERSION" ] || [ "$WO_CR_VER" != "$REQUIRED_WO_CR_VERSION" ]; then
    echo "[$(ts)] Error: Version mismatch!"
    echo "[$(ts)] WO Version: $WO_VER (required: $REQUIRED_WO_VERSION)"
    echo "[$(ts)] WO CR Version: $WO_CR_VER (required: $REQUIRED_WO_CR_VERSION)"
    exit 1
else
    echo "[$(ts)] Version check passed: $WO_VER , CR version: $WO_CR_VER"
fi

# Hotfix label configuration
HOTFIX_LABEL_KEY="${HOTFIX_LABEL_KEY:-hotfix}"
HOTFIX_LABEL_VALUE="${HOTFIX_LABEL_VALUE:-5.3.1.4}"
WO_CR_NAME=wo

is_semver4() {
  v="$1"
  printf '%s' "$v" | grep -Eq '^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+$'
}

# Backup dir for deployments
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
CLUSTER_NAME="$(oc whoami --show-console | sed 's/.*console-openshift-console\.apps\.\([^.]*\)\..*/\1/')"
BACKUP_DIR="${SCRIPT_DIR}/wxo_deployment_backups/$CLUSTER_NAME"
mkdir -p "$BACKUP_DIR"

if [ -n "${OPERATOR_IMAGES:-}" ]; then
  echo "[$(ts)] Updating operator deployment images in ${PROJECT_CPD_INST_OPERATORS}"

  patched_deps=""

  # Use a here-document to avoid subshell so patched_deps is preserved
  while IFS= read -r img; do
    [ -z "$img" ] && continue

    # Extract base image name (e.g., ibm-wxo-component-operator)
    base="$(basename "$img" | cut -d'@' -f1)"

    echo "[$(ts)] Processing image: $img (base=$base)"

    dep=""

    case "$base" in
      ibm-watsonx-orchestrate-operator)
        dep="wo-operator"
        ;;

      ibm-wxo-component-operator)
        dep="ibm-wxo-componentcontroller-manager"
        ;;

      *)
        dep="$(oc -n "$PROJECT_CPD_INST_OPERATORS" get deploy --no-headers 2>/dev/null \
               | grep "$base" | awk 'NR==1{print $1}')"
        ;;
    esac

    # Skip bundle/catalog images if they ever slip in
    if echo "$base" | grep -Eq 'bundle|catalog'; then
      echo "[$(ts)]   Skipping $base (bundle/catalog image – not tied to deployment)"
      continue
    fi

    if [ -z "$dep" ]; then
      echo "[$(ts)]   WARNING: no matching deployment found for $base — skipping"
      continue
    fi

    # -----------------------------
    # Backup deployment YAML
    # -----------------------------
    backup_file="${BACKUP_DIR}/${dep}-$(date +%Y%m%d%H%M%S).yaml"
    if oc -n "$PROJECT_CPD_INST_OPERATORS" get deploy "$dep" -o yaml > "$backup_file" 2>/dev/null; then
      echo "[$(ts)]   Backed up deployment/$dep → $backup_file"
    else
      echo "[$(ts)]   WARNING: failed to back up deployment/$dep"
    fi

    # Determine container name (assume first container is operator)
    cname="$(oc -n "$PROJECT_CPD_INST_OPERATORS" get deploy "$dep" \
             -o jsonpath='{.spec.template.spec.containers[0].name}' 2>/dev/null)"

    if [ -z "$cname" ]; then
      echo "[$(ts)]   WARNING: cannot determine container name for $dep — skipping"
      continue
    fi

    echo "[$(ts)]   Patching deployment/$dep container '$cname' → $img"

    if oc -n "$PROJECT_CPD_INST_OPERATORS" set image "deployment/$dep" "$cname=$img" >/dev/null 2>&1; then
      echo "[$(ts)]   ✓ Image updated for $dep"
      patched_deps="$patched_deps $dep"
    else
      echo "[$(ts)]   ✗ ERROR: failed to patch $dep"
    fi

  done <<EOF
$OPERATOR_IMAGES
EOF

# -----------------------------
# Label WO CR with configurable label and value
# -----------------------------
if [ -n "$WO_CR_NAME" ]; then
  current_label="$(oc -n "$PROJECT_CPD_INST_OPERANDS" get wo "$WO_CR_NAME" -o jsonpath="{.metadata.labels.${HOTFIX_LABEL_KEY}}" 2>/dev/null || true)"
  if [ "$current_label" = "$HOTFIX_LABEL_VALUE" ]; then
    echo "[$(ts)] WO CR ${WO_CR_NAME} already labeled ${HOTFIX_LABEL_KEY}=${HOTFIX_LABEL_VALUE}"
  else
    echo "[$(ts)] Setting label ${HOTFIX_LABEL_KEY}=${HOTFIX_LABEL_VALUE} on WO CR ${WO_CR_NAME} in ns ${PROJECT_CPD_INST_OPERANDS}"
    oc -n "$PROJECT_CPD_INST_OPERANDS" label wo "$WO_CR_NAME" "${HOTFIX_LABEL_KEY}=${HOTFIX_LABEL_VALUE}" --overwrite >/dev/null 2>&1 || true
    new_label="$(oc -n "$PROJECT_CPD_INST_OPERANDS" get wo "$WO_CR_NAME" -o jsonpath="{.metadata.labels.${HOTFIX_LABEL_KEY}}" 2>/dev/null || true)"
    if [ "$new_label" = "$HOTFIX_LABEL_VALUE" ]; then
      echo "[$(ts)] Label set: ${HOTFIX_LABEL_KEY}=${HOTFIX_LABEL_VALUE}"
    else
      echo "[$(ts)] WARNING: could not confirm ${HOTFIX_LABEL_KEY}=${HOTFIX_LABEL_VALUE} label was set"
    fi
  fi
else
  echo "[$(ts)] No WO CR found in ns ${PROJECT_CPD_INST_OPERANDS}, skipping label."
fi


  # -----------------------------
  # Verify patched deployments are healthy (1/1 or 2/2)
  # -----------------------------
  if [ -n "${patched_deps// /}" ]; then
    echo "[$(ts)] Verifying rollout status for patched deployments..."

    for dep in $patched_deps; do
      echo "[$(ts)] Checking deployment/$dep..."

      # Wait for rollout to complete
      if oc -n "$PROJECT_CPD_INST_OPERATORS" rollout status deploy/"$dep" --timeout=300s; then
        # Check Ready/Desired replica ratio
        ratio="$(oc -n "$PROJECT_CPD_INST_OPERATORS" get deploy "$dep" \
                 -o jsonpath='{.status.readyReplicas}/{.status.replicas}' 2>/dev/null || echo '0/0')"
        echo "[$(ts)]   Ready/Desired: $ratio"

        if [ "$ratio" = "1/1" ] || [ "$ratio" = "2/2" ]; then
          echo "[$(ts)]   ✓ Deployment $dep is healthy (pods up and running)."
        else
          echo "[$(ts)]   ⚠ WARNING: Deployment $dep is not at 1/1 or 2/2; current $ratio"
        fi
      else
        echo "[$(ts)]   ✗ ERROR: rollout status for deployment/$dep did not complete successfully."
      fi
    done
  else
    echo "[$(ts)] No deployments were patched; skipping health verification."
  fi

else
  echo "[$(ts)] No OPERATOR_IMAGES specified — skipping operator image patch."
fi

# Let's delete the redis cronjob and and allow the operator to create a new equivalent one. 
oc delete cronjob wo-watson-orchestrate-redis-cronjob --ignore-not-found

# -----------------------------
# Final message
# -----------------------------
echo "------------------------------------------------------------------"
echo "[$(ts)] 5.4.0 Hotfix4 steps completed."
echo "Backups saved under ${BACKUP_DIR}"
echo "Monitor the watsonx Orchestrate CR status by running:"
echo " oc get wo -n ${PROJECT_CPD_INST_OPERANDS} -o yaml | grep -E 'watsonxOrchestrateStatus|${HOTFIX_LABEL_KEY}'"
echo "Ensure the watsonx Orchestrate CR status is 'Completed' and label ${HOTFIX_LABEL_KEY}=${HOTFIX_LABEL_VALUE} is present."
echo "It will take another 15–20 minutes for the updated components to be applied and restarted."
echo "------------------------------------------------------------------"
```

Make the script executable
```bash
chmod 775 hotfix5405.sh
```
 
Run the script
```bash
nohup sh hotfix5405.sh &
```
 
Watch progress
```bash
tail -f nohup.out
```

Verify CR status and label
```bash
oc get wo -n "${PROJECT_CPD_INST_OPERANDS}" -o yaml | grep hotfix
```

Output should look like
```bash
hotfix: 5.4.0.5
```

Confirm the completion of the hotfix by check the Watsonx Orchestrate CR status
```bash
oc get wo
```

The expected output
```bash
NAME   VERSION   DEPLOYED   VERIFIED   TOTAL   INSTALLMODE         QUIESCE        RECONCILE_PROGRESS   AGE
wo     5.4.0     34         34         34      agentic_assistant   NOT_QUIESCED   100%                 9d
```

---

#### Upgrade Watsonx Ai

Export the XAI_COMPONENT_TYPE variable
```bash
XAI_COMPONENT_TYPE=watsonx_ai
```

Upgrade watsonx_ai
```bash
cpd-cli manage install-components \
--license_acceptance=true \
--components=${XAI_COMPONENT_TYPE} \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--upgrade=true
```

---

#### Potential Issue  - OOMKilled on `install-and-reconcile` job

During prod-east upgrade an issue was encountered with the 'install-and-reconsile' job during Watsonx AI upgrade

The `wml-install-and-reconcile` job reached its backofflimit of 6 because all 6 job start ups failed due to OOMKilled

This caused the `wml-cr` on the `wmlbases` resource to become stuck during the watsonX AI upgrade

The `wml-cr` on the `wmlbases` resource would not move past this stuck state at 87.5% and `InProgress` status, since it was waiting for this job to complete

Container information and log output can be found below
```bash
  Containers:
   cleanup-hibernate:
    Image:      us-docker.pkg.dev/gcp-dct-ccca-dev/ccca-d-image-registry/cp/cpd/wml-post-upgrade-cleanup-deployments@sha256:fa18900f8874755bc8c0cce9759384f5a5d5fa06a000c4d4ea8e01eec1d21f3c
    Port:       <none>
    Host Port:  <none>
    Command:
      /bin/sh
      /opt/ibm/scripts/run_upgrade_or_rollback.sh
    Limits:
      cpu:                250m
      ephemeral-storage:  200Mi
      memory:             350Mi
    Requests:
      cpu:                250m
      ephemeral-storage:  20Mi
      memory:             350Mi
```

```bash
Python version :3.11.13 (main, Jan 16 2026, 00:00:00) [GCC 11.5.0 20240719 (Red Hat 11.5.0-11)]
2026/05/30 17:44:18,067|INFO|upgrade_or_rollback_deployments.py:133: Capturing pre-upgrade runtime details...
2026/05/30 17:44:33,085|ERROR|upgrade_or_rollback_deployments.py:124:
[COMMAND]: kubectl logs $(kubectl get pods --no-headers -o custom-columns=":metadata.name" -n ups-wx-operands | grep runtime-assemblies-operator) -n ups-wx-operands | grep "icpdsupport/addOnId=wml"
[ERROR]:
```

To resolve the issue, delete the job, restart the WML operator and injected new memory values into the Operator files using the following script during the startup of the reconcile
```bash
OP_NS=ups-wx-operators
OP_LABEL='name=ibm-cpd-wml-operator'

echo "Current operator pod:"
OLD_POD=$(oc get pod -n "$OP_NS" -l "$OP_LABEL" -o jsonpath='{.items[0].metadata.name}')
echo "$OLD_POD"

echo "Deleting old operator pod..."
oc delete pod "$OLD_POD" -n "$OP_NS"

echo "Waiting for new operator pod..."
while true; do
  NEW_POD=$(oc get pod -n "$OP_NS" -l "$OP_LABEL" \
    --field-selector=status.phase=Running \
    -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)

  if [ -n "$NEW_POD" ] && [ "$NEW_POD" != "$OLD_POD" ]; then
    READY=$(oc get pod "$NEW_POD" -n "$OP_NS" -o jsonpath='{.status.containerStatuses[0].ready}' 2>/dev/null)
    if [ "$READY" = "true" ]; then
      echo "New operator pod is ready: $NEW_POD"
      break
    fi
  fi

  echo "Still waiting..."
  sleep 5
done

echo "Patching WML job templates from 350Mi to 1Gi..."
oc exec -n "$OP_NS" "$NEW_POD" -- sh -c '
for f in \
/opt/ansible/5.4.0/roles/wml-base/templates/install-reconsile-cleanup-and-hibernate.yaml.j2 \
/opt/ansible/5.4.0/roles/wml-base/templates/post-upgrade-cleanup-and-hibernate.yaml.j2 \
/opt/ansible/5.4.0/roles/wml-base/templates/pre-upgrade-check-job.yaml.j2 \
/opt/ansible/5.4.0/roles/wml-base/templates/preinstall-wml-runtime-definitions.yaml.j2 \
/opt/ansible/5.4.0/roles/wml-base/templates/wml-shutdown-restart-runtimes.yaml.j2
do
  echo "===== $f ====="
  cp "$f" "$f.bak"
  sed -i "s/memory: \"350Mi\"/memory: \"1Gi\"/g" "$f"
  grep -n "memory:" "$f" | head -20
done
'

echo "Done. Patched pod: $NEW_POD"
```

This loaded the higher memory into the job defention and allowed us to pass through the initial job

Recommended that support needs to look into this with larger enviroments

Monitor watsonx_ai upgrade
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=watsonx_ai
```

---

#### Upgrade Watsonx Governance

Create the install-options.yml file in the cpd-cli-workspace/olm-utils-workspace/work directory
```bash
---
# ............................................................................
# watsonx.governance parameters
# ............................................................................
non_olm:
  watsonxGovernance:
    installType: all
    enableFactsheet: "true"
    enableOpenpages: "true"
    enableOpenscale: "true"
#   openpagesInstanceCR: "op-wxgov-instance"
#   openPages:
#     databaseType: internal
#     database: Db2
#     dbSecretName: <secret-name>
```

Upgrade watsonx_governance
```bash
cpd-cli manage install-components \
--license_acceptance=true \
--components=watsonx_governance \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--param-file=/tmp/work/install-options.yml \
--upgrade=true
```

Monitor watsonx_governance upgrade
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=watsonx_governance
```

---


#### Upgrade Watson Speech

Upgrade watson speech
```bash
cpd-cli manage install-components \
--license_acceptance=true \
--components=watson_speech \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--upgrade=true
```

Monitor watson_speech upgrade, wait until the status shows `Completed` or `Ready`.
```bash
oc get WatsonSpeech speech-cr -n ${PROJECT_CPD_INST_OPERANDS}
```

Check the cr status
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=watson_speech
```

---

#### Upgrade Voice Gateway
```bash
cpd-cli manage install-components \
--license_acceptance=true \
--components=voice_gateway \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--run_storage_tests=false \
--upgrade=true
```

Monitor voice_gateway upgrade
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=voice_gateway
```

---

#### Upgrade Db2 OLTP
```bash
cpd-cli manage install-components \
--license_acceptance=true \
--components=db2oltp \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--run_storage_tests=false \
--upgrade=true
```

Monitor db2oltp upgrade
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=db2oltp
```

---

#### Upgrade Cognos Analytics
```bash
cpd-cli manage install-components \
--license_acceptance=true \
--components=cognos_analytics \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--run_storage_tests=false \
--upgrade=true
```

Monitor cognos_analytics upgrade
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=cognos_analytics
```

---


## Upgrade Service Instances

After upgrading service CRs, some services require additional instance upgrades

**Reference**: [Creating a CPD profile](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=cli-creating-cpd-profile)

---

Upgrading Service Instances

Get a list of all service instances using the following command
```bash
cpd-cli service-instance list --profile=${CPD_PROFILE_NAME}
```

#### Upgrade Db2oltp service instances

Get the list of Db2 service instances
```bash
cpd-cli service-instance list --service-type=db2oltp --profile=${CPD_PROFILE_NAME}
```

Export the db2oltp instance name
```bash
export INSTANCE_NAME=<instance-name>
```

Run the following command to check whether your Db2 service instances is in running state
```bash
cpd-cli service-instance status ${INSTANCE_NAME} --profile=${CPD_PROFILE_NAME} --service-type=db2oltp
```

Upgrade the service instance
```bash
cpd-cli service-instance upgrade --service-type=db2oltp --instance-name=${INSTANCE_NAME} --profile=${CPD_PROFILE_NAME}
```

---

#### Upgrade Cognos Analytics service instances

Set the INSTANCE_VERSION environment variable to the version that corresponds to the version of IBM Software Hub on your cluster
```bash
export INSTANCE_VERSION=30.0.0
```

Upgrade the service instances
```bash
cpd-cli service-instance upgrade --service-type=cognos-analytics-app --profile=${CPD_PROFILE_NAME} --version=${INSTANCE_VERSION} --all
```

#### Potential Issue - CAserviceinstance stuck trying to connect to dispatcher

While upgrading Governance, in an effort to speed up the upgrade, we upgraded the service instance of CAserviceinstance to the newer version

This appeared to put the ca-reporting service in a bad state

Even though the new version appeared on the service instance, the dispatcher was still trying to connect to the old version

This caused the old pods of the `reporting` pods to not come down and the new pods to not boot up properly

To fix this issue, we had to delete the all the CA Pods (`smarts`, `reporting`, etc) and operator to re-create them

This allowed the pods to come up properly, upgrade and report back healthy

---

#### Upgrade Openpages service instances

Get the list of OpenPages service instances
```bash
cpd-cli service-instance list --service-type=openpages --profile=${CPD_PROFILE_NAME}
```

Set the INSTANCE_NAME environment variable to the name of the service instance that you want to upgrade
```bash
export INSTANCE_NAME=<instance-name>
```

Run the following command to check whether your OpenPages service instances is in running state
```bash
cpd-cli service-instance status ${INSTANCE_NAME} --profile=${CPD_PROFILE_NAME} --service-type=openpages
```

Upgrade the service instance
```bash
cpd-cli service-instance upgrade --service-type=openpages --instance-name=${INSTANCE_NAME} --force-version-upgrade=true --profile=${CPD_PROFILE_NAME}
```

Repeat the preceding steps to upgrade each service instance associated with this instance of IBM Software Hub

---

## Upgrade cpdbr service

You must upgrade the cpdbr service after you upgrade IBM Software Hub.

**Reference**: [Updating the cpdbr service](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=uish-updating-cpdbr-service)

For Environments Without Scheduling Service
```bash
cpd-cli oadp install \
--component=cpdbr-tenant \
--cpdbr-hooks-image-prefix=${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd \
--cpfs-image-prefix=${PRIVATE_REGISTRY_LOCATION}/cpopen/cpfs \
--namespace=${OADP_OPERATOR_NS} \
--tenant-operator-namespace=${PROJECT_CPD_INST_OPERATORS} \
--skip-recipes=true \
--upgrade=true \
--log-level=debug \
--verbose
```

**Note**: This upgrade should be performed after all services have been upgraded

**Note**: If you encounter an imagepullbackoff issue for the cpdbr tenant service pod(s) this might be caused by your cpd-cli version. The cpd-cli utility version (BUILD_ID: 3.3.1.x) dynamically appends its own build suffix to backup image queries, causing the cluster to look for non-existent tag 5.4.0.x in the air-gapped registry instead of the mirrored 5.4.0 GA baseline; downgrading to cpd-cli version 14.3.0 can force it to query the correct GA tag

**Note**: The workaround used during UPS non prod upgrade was to tag the image in the private registry with the 5.4.0.5 tag

---

## Post Upgrade Validation

#### Potential Issue - Enable WxO Observability

Follow the procedure in the following document to enable WxO Observability

**IMPORTANT**: [Enable WxO Observability](https://github.com/kuanalex/ups/blob/main/WxO_Observability_PROD_Runbook.md)

---

#### Potential Issue - Fix platform-auth-service pod in ContainerStatusUnknown

Follow the procedure in the following known issue document to resolve platform-auth-service pod in ContainerStatusUnknown issue

**IMPORTANT**: [Enabling the debug trace for the platform-auth-service cause pod in ContainerStatusUnknown state and repeatedly get evicted](https://www.ibm.com/mysupport/s/defect/aCIgJ000000C9qjWAC/dt467023?language=en_US)

---

#### General validation steps

Check CR status
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

Check for pods not running correctly
```bash
oc get po -A -owide | egrep -v '([0-9])/\1' | egrep -v 'Completed'
```

List service instances
```bash
cpd-cli service-instance list --profile=${CPD_PROFILE_NAME}
```

Validate 'expose:external-regional' label in the cpd route, add the label "expose:external-regional" to your cpd-route as required
```bash
oc get route cpd -o yaml
```

---

**End of Runbook**
