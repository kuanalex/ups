# UPS Production Cluster CP4D 5.3.1.0 to 5.4.0.5 Upgrade
## Author: Alex Kuan (alex.kuan@ibm.com)

**From:**
```
CPD: 5.3.1.0
OCP: 4.20.25
Storage: Google Cloud Netapp Volumes and Persistent Disk on Google Cloud
Internet: Air-gapped
Private container registry: Yes
Components: ibm-licensing,ibm_events_operator,ccs,cpfs,zen,cpd_platform,watsonx_orchestrate,watsonx_ai,watsonx_governance,watson_speech,voice_gateway,db2oltp,cognos_analytics
```

**To:**
```
CPD: 5.4.0.5
OCP: 4.20.25
Storage: Google Cloud Netapp Volumes and Persistent Disk on Google Cloud
Internet: Air-gapped
Private container registry: Yes
Components: ibm-licensing,ibm_events_operator,ccs,cpfs,zen,cpd_platform,watsonx_orchestrate,watsonx_ai,watsonx_governance,watson_speech,voice_gateway,db2oltp,cognos_analytics
```

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

Remember to download the CASE package for the ibm_events_operator component as well
```bash
cpd-cli manage case-download --components=ibm_events_operator --release=${VERSION} --patch_id=${PATCH_ID} --from_oci=true
```

Here is an example of the mirror-images command syntax
```bash
cpd-cli manage mirror-images --components=${COMPONENTS} --release=${VERSION} --patch_id=${PATCH_ID} --target_registry=${PRIVATE_REGISTRY_LOCATION} --arch=${IMAGE_ARCH} --case_download=false
```

The permissions required for the upgrade is ready
- OpenShift cluster administrator permissions
- IBM Software Hub administrator permissions
- Permission to access the private image registry for pushing or pulling images
- Access to the bastion node for executing the upgrade commands

---

#### Updating your environment variables script

Ensure that your environment variables script includes the correct information for the instance of IBM Software Hub that you want to upgrade

**Reference**: [Updating your environment variables script](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=cri-updating-your-environment-variables-script)

Update the fields in your cpd_vars.sh file as needed
```bash
# ------------------------------------------------------------------------------
# VERSION and PATCH_ID
# ------------------------------------------------------------------------------
export VERSION=5.4.0
export PATCH_ID=5
# ------------------------------------------------------------------------------
# Backup and restore 
# ------------------------------------------------------------------------------
export PROJECT_SCHEDULING_BR_SVC=ibm-cpd-scheduler-br-svc
export PROJECT_INST_BR_SVC=${PROJECT_CPD_INST_OPERATORS}-br-svc
export BACKUP_NAME=online-backup-$(date '+%Y%m%d-%H%M%S')
export RESTORE_NAME=${BACKUP_NAME}-restore
export OADP_PROJECT=<enter your OADP project>
# export OADP_VERSION=<OADP-version>
export NODE_AGENT_POD_CPU_LIMIT=500m
export KOPIA_POD_CPU_LIMIT=1
# export PROJECT_FUSION=ibm-spectrum-fusion-ns
# export PROJECT_PX_ADMIN_NS=<No default>
# export PROJECT_NETAPP_TRIDENT_PROTECT=trident-protect
# export NETAPP_TRIDENT_PROTECT_APP_VAULT=<AppVault name>
export BR_OPERATOR_JOB_SA=bros-job-sa
export BR_OPERATOR_SA=bros-sa
# export PROJECT_INST_BR_SVC_NEW=${PROJECT_CPD_INST_OPERATORS_NEW}-br-svc
# export PROJECT_CPD_INST_OPERATORS_NEW=<project-name>
# export OPERATOR_MAPPING=${PROJECT_CPD_INST_OPERATORS}:${PROJECT_CPD_INST_OPERATORS_NEW}
# export PROJECT_CPD_INST_OPERANDS_NEW=<project-name>
# export OPERAND_MAPPING=${PROJECT_CPD_INST_OPERANDS}:${PROJECT_CPD_INST_OPERANDS_NEW}
# export PROJECT_CPD_INSTANCE_TETHERED_1=<project-name>
# export PROJECT_CPD_INSTANCE_TETHERED_1_NEW=<project-name>
# export TETHER_MAPPING_1=${PROJECT_CPD_INSTANCE_TETHERED_1}:${PROJECT_CPD_INSTANCE_TETHERED_1_NEW}
# export PROJECT_CPD_INSTANCE_TETHERED_2=<project-name>
# export PROJECT_CPD_INSTANCE_TETHERED_2_NEW=<project-name>
# export TETHER_MAPPING_2=${PROJECT_CPD_INSTANCE_TETHERED_2}:${PROJECT_CPD_INSTANCE_TETHERED_2_NEW}
# export PROJECT_CPD_INSTANCE_TETHERED_LIST_NEW=${PROJECT_CPD_INSTANCE_TETHERED_1_NEW},${PROJECT_CPD_INSTANCE_TETHERED_2_NEW}
# export RESTORE_PROJECT_MAPPING="${OPERATOR_MAPPING},${OPERAND_MAPPING}"
# export RESTORE_PROJECT_MAPPING="${OPERATOR_MAPPING},${OPERAND_MAPPING},${TETHER_MAPPING_1},${TETHER_MAPPING_2}"

# ------------------------------------------------------------------------------
# S3 Object Storage
# ------------------------------------------------------------------------------
export S3_URL=<S3-url>
export REGION=<S3-region>
export BUCKET_NAME=<bucket-name>
export BUCKET_PREFIX=<bucket-prefix>
export ACCESS_KEY_ID=<access-key-ID>
export SECRET_ACCESS_KEY=<access-key-secret>
```

Add the br_orchestration component after the cpd_platform component, for example
```bash
export COMPONENTS=ibm-licensing,ibm_events_operator,cpd_platform,br_orchestration,watsonx_orchestrate,watsonx_ai,watsonx_governance,watson_speech,voice_gateway,db2oltp,cognos_analytics
```

Source the environment variables
```bash
source cpd_vars.sh
```

---

## Pre Upgrade Steps

**Required Tools**:

Ensure the following tools are installed and updated to the required versions
- IBM Software Hub CLI: Version 14.4.0.3
- OpenShift CLI (oc): Compatible version for your cluster
- Helm CLI: Version 4.1.4

For detailed instructions on installing or updating these tools, refer to
- [Updating client workstations](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=53-updating-client-workstations)
- [Updating IBM Software Hub CLI](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=ucw-updating-software-hub-cli)
- [Updating OpenShift CLI](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=ucw-updating-openshift-cli)
- [Installing Helm CLI](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=ucw-installing-helm-cli)

**Required Access**:
- OpenShift cluster admin access
- IBM Entitlement Key with appropriate permissions
- Access to IBM Container Registry (cp.icr.io)
- Access to private registry: UPDATE_WITH_PRIVATE_REGISTRY_URL

---

#### Backup routes and temporary patches (for watson assistant)

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

For all patches that you want to retain, use the following command
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

**Reference**: [Upgrading shared cluster components](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=pyc-upgrading-shared-cluster-components)

Upgrade IBM Licensing service
```bash
cpd-cli manage apply-cluster-components --release=${VERSION} --patch_id=${PATCH_ID} --license_acceptance=true --licensing_ns=${PROJECT_LICENSE_SERVICE}
```

Verify licensing pods are running
```bash
oc get pods -n ${PROJECT_LICENSE_SERVICE}
```

---

#### Update cluster-scoped resources for the instance

**Reference**: [Updating cluster-scoped resources for the instance](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=puish-updating-cluster-scoped-resources-instance)

Generate cluster-scoped resources for the instance
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

Upgrade the IBM Events Operator
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

**Reference**: [Upgrading IBM Software Hub](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=53-upgrading-software-hub)

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
watch -n 3 'oc get po -A -owide | egrep -v "([0-9])/\1" | egrep -v "Completed" && echo "=== ZenService Progress ===" && oc get zenservice lite-cr -o yaml | grep progress && echo "=== Ibmcpd Progress ===" && oc get ibmcpd ibmcpd-cr -o yaml | grep progress'
```

---

#### Reapply RSI Patches

**Reference**: [Upgrading Software Hub](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=uish-upgrading-software-hub)

If there are patches that apply to zen or IBM Cloud Pak foundational services pods, run the following command to apply your custom patches
```bash
cpd-cli manage apply-rsi-patches --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

Verify patches are active
```bash
cpd-cli manage get-rsi-patch-info --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --all
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

**IMPORTANT**: Before proceeding with Orchestrate upgrade, remove the following image_digests from watsonxaiifm-cr

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

Monitor watsonx_orchestrate upgrade progress
```bash
watch -n 3 'oc get po -A -owide | egrep -v "([0-9])/\1" | egrep -v "Completed" && oc get ccs,watsonxaiifm,wa,wo'
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

After completing this migration, follow the steps for 'Applying the watsonx Orchestrate 5.4.0 Patch-5 (5.4.2) Hotfix 0'

**Reference**: [Apply hot fix for IBM watsonx Orchestrate](https://www.ibm.com/support/pages/node/7247038)
**Reference**: [Applying the watsonx Orchestrate 5.4.0 Patch-5 (5.4.2) Hotfix 0](https://www.ibm.com/support/pages/node/7284300)

Set the operator and operand namespaces
```bash
export PROJECT_CPD_INST_OPERATORS=ups-wx-operators
export PROJECT_CPD_INST_OPERANDS=ups-wx-operands
```

**Note**: You will need to install Skopeo and mirror the operator and operand images before proceeding

Create 5.4.2-Hotfix0.sh
```bash
vi 5.4.2-Hotfix0.sh
```

With the following contents
```bash
#!/usr/bin/env bash
set -euo pipefail

# Function to print log messages with timestamp
log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"
}

PROJECT_CPD_INST_OPERATORS="${PROJECT_CPD_INST_OPERATORS:-}"
PROJECT_CPD_INST_OPERANDS="${PROJECT_CPD_INST_OPERANDS:-}"

if [[ -z "$PROJECT_CPD_INST_OPERATORS" ]]; then
  log "ERROR: PROJECT_CPD_INST_OPERATORS is not set."
  exit 1
fi

if [[ -z "$PROJECT_CPD_INST_OPERANDS" ]]; then
  log "ERROR: PROJECT_CPD_INST_OPERANDS is not set."
  exit 1
fi

# Operator patch label configuration
OPERATOR_PATCH_LABEL_KEY="${OPERATOR_PATCH_LABEL_KEY:-Hotfix}"
OPERATOR_PATCH_LABEL_VALUE="${OPERATOR_PATCH_LABEL_VALUE:-5.4.2-Hotfix0}"
WO_CR_NAME="wo"

# Make sure oc login is done
if ! oc whoami &>/dev/null; then
  log "ERROR: Not logged in to OpenShift. Please run 'oc login' first."
  exit 1
fi

log "✅ OpenShift login verified: $(oc whoami)"

# Backup dir for deployments
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
CLUSTER_NAME="$(oc whoami --show-console | sed 's/.*console-openshift-console\.apps\.\([^.]*\)\..*/\1/')"
BACKUP_DIR="${SCRIPT_DIR}/wxo_deployment_backups/$CLUSTER_NAME"
mkdir -p "$BACKUP_DIR"

log "📁 Backup directory: $BACKUP_DIR"

# Check WXO version
log "🔍 Checking WXO version in namespace: $PROJECT_CPD_INST_OPERANDS"
WXO_VERSION=$(oc get wo -n "$PROJECT_CPD_INST_OPERANDS" -o jsonpath='{.items[0].status.versionStatus.status}' 2>/dev/null || echo "")
CRVERSION=$(oc get wo wo -n $PROJECT_CPD_INST_OPERANDS -o jsonpath='{.spec.version}')

if [[ -z "$WXO_VERSION" ]]; then
  log "ERROR: Unable to retrieve WXO version. Please ensure WatsonxOrchestrate resource exists."
  exit 1
fi

log "   Current WXO version: $WXO_VERSION"

if [[ "$WXO_VERSION" != "5.4.0" || "$CRVERSION" != "8.0.2" ]]; then
  log "ERROR: This operator patch can only be applied when:"
  log "       WXO version  : 5.4.2"
  log "       CR version   : 8.0.2"
  log ""
  log "Current versions:"
  log "       WXO version  : ${WXO_VERSION}"
  log "       CR version   : ${CRVERSION}"
  exit 1
fi

log "✅ Version check passed (8.0.2)"
log ""

# Hardcode images here when you do not want to pass them as script arguments.
BOOTSTRAP_OPERATOR_IMAGE="icr.io/cpopen/ibm-watsonx-orchestrate-operator@sha256:0603789d433d9828e16191bbe0e5e1aa83af4d8cae31d415f155fec445842967"
COMPONENT_OPERATOR_IMAGE="icr.io/cpopen/ibm-wxo-component-operator@sha256:55066ba89814afbb0e1b48fa1484aae18d49db45d61706e5ca06368c80dfc6ea"

if [[ $# -gt 1 ]]; then
  log "Usage: $0 [image1,image2,...]"
  exit 1
fi

if [[ $# -eq 1 ]]; then
  IFS=',' read -ra IMAGES <<< "$1"
else
  IMAGES=()
  [[ -n "$BOOTSTRAP_OPERATOR_IMAGE" ]] && IMAGES+=("$BOOTSTRAP_OPERATOR_IMAGE")
  [[ -n "$COMPONENT_OPERATOR_IMAGE" ]] && IMAGES+=("$COMPONENT_OPERATOR_IMAGE")

  if [[ ${#IMAGES[@]} -eq 0 ]]; then
    log "Usage: $0 [image1,image2,...]"
    log "Either pass images as an argument or hardcode BOOTSTRAP_OPERATOR_IMAGE / COMPONENT_OPERATOR_IMAGE in the script."
    exit 1
  fi
fi

# Track patched deployments for health verification
PATCHED_DEPLOYMENTS=()

# -----------------------------
# Check and remove digest overrides from WO CR
# -----------------------------
log "🔍 Checking for digest overrides in WO CR..."
DIGEST_OVERRIDES=$(oc get wo "$WO_CR_NAME" -n "$PROJECT_CPD_INST_OPERANDS" \
  -o jsonpath='{.spec.image.digestOverrides}' 2>/dev/null || echo "")

if [[ -n "$DIGEST_OVERRIDES" && "$DIGEST_OVERRIDES" != "null" ]]; then
  log "⚠️  Digest overrides found in WO CR. Removing them before patching..."
  log "   Current digest overrides: $DIGEST_OVERRIDES"
  
  if oc patch wo "$WO_CR_NAME" -n "$PROJECT_CPD_INST_OPERANDS" --type=merge \
    -p='{"spec":{"image":{"digestOverrides":null}}}'; then
    log "✅ Successfully removed digest overrides from WO CR"
    
    # Wait a moment for the change to propagate
    sleep 2
  else
    log "✗ ERROR: Failed to remove digest overrides from WO CR"
    log "   Please remove them manually before proceeding"
    exit 1
  fi
else
  log "✅ No digest overrides found in WO CR"
fi

log ""

for IMAGE in "${IMAGES[@]}"; do
  IMAGE="$(echo "$IMAGE" | xargs)"

  # Extract image name - handle both tag (:) and digest (@) formats
  IMAGE_NAME="$(basename "$IMAGE" | cut -d'@' -f1 | cut -d':' -f1)"

  # Map image → deployment
  case "$IMAGE_NAME" in
    ibm-wxo-component-operator)
      DEPLOYMENT="ibm-wxo-componentcontroller-manager"
      ;;
    ibm-watsonx-orchestrate-operator)
      DEPLOYMENT="wo-operator"
      ;;
    *)
      log "⚠️  No deployment mapping found for image: $IMAGE_NAME"
      continue
      ;;
  esac

  log "🔍 Checking deployment '$DEPLOYMENT' for image '$IMAGE_NAME'..."

  # Backup deployment YAML before patching
  BACKUP_FILE="${BACKUP_DIR}/${DEPLOYMENT}-$(date +%Y%m%d%H%M%S).yaml"
  if oc -n "$PROJECT_CPD_INST_OPERATORS" get deploy "$DEPLOYMENT" -o yaml > "$BACKUP_FILE" 2>/dev/null; then
    log "   Backed up deployment/$DEPLOYMENT → $BACKUP_FILE"
  else
    log "   WARNING: Failed to back up deployment/$DEPLOYMENT"
  fi

  CURRENT_IMAGE="$(oc get deploy "$DEPLOYMENT" -n "$PROJECT_CPD_INST_OPERATORS" \
    -o jsonpath='{.spec.template.spec.containers[0].image}')"

  if [[ "$CURRENT_IMAGE" != *"$IMAGE_NAME"* ]]; then
    log "⚠️  Image '$IMAGE_NAME' not found in deployment '$DEPLOYMENT'. Skipping."
    continue
  fi

  log "✅ Match found. Patching deployment '$DEPLOYMENT'"
  log "   Old: $CURRENT_IMAGE"
  log "   New: $IMAGE"

  if oc patch deploy "$DEPLOYMENT" -n "$PROJECT_CPD_INST_OPERATORS" \
    --type='json' \
    -p="[{
      \"op\": \"replace\",
      \"path\": \"/spec/template/spec/containers/0/image\",
      \"value\": \"$IMAGE\"
    }]"; then
    log "🚀 Successfully patched $DEPLOYMENT"
    PATCHED_DEPLOYMENTS+=("$DEPLOYMENT")
  else
    log "✗ ERROR: Failed to patch $DEPLOYMENT"
  fi
  log ""
done

# -----------------------------
# Label WO CR with operator patch label
# -----------------------------
if [[ -n "$WO_CR_NAME" ]]; then
  log "🏷️  Managing hotfix label on WO CR..."
  
  # Check if any hotfix label exists (case-insensitive check)
  EXISTING_HOTFIX_LABEL=$(oc -n "$PROJECT_CPD_INST_OPERANDS" get wo "$WO_CR_NAME" \
    -o jsonpath='{.metadata.labels}' 2>/dev/null | grep -i '"hotfix"' || true)
  
  # Remove existing hotfix label if found
  if [[ -n "$EXISTING_HOTFIX_LABEL" ]]; then
    log "   Existing hotfix label found. Removing it..."
    # Remove both possible variations (uppercase and lowercase)
    oc -n "$PROJECT_CPD_INST_OPERANDS" label wo "$WO_CR_NAME" "Hotfix-" >/dev/null 2>&1 || true
    oc -n "$PROJECT_CPD_INST_OPERANDS" label wo "$WO_CR_NAME" "hotfix-" >/dev/null 2>&1 || true
    log "   ✅ Existing hotfix labels removed"
  fi
  
  # Apply new label
  log "   Setting label ${OPERATOR_PATCH_LABEL_KEY}=${OPERATOR_PATCH_LABEL_VALUE} on WO CR ${WO_CR_NAME}"
  if oc -n "$PROJECT_CPD_INST_OPERANDS" label wo "$WO_CR_NAME" \
    "${OPERATOR_PATCH_LABEL_KEY}=${OPERATOR_PATCH_LABEL_VALUE}" >/dev/null 2>&1; then
    
    NEW_LABEL="$(oc -n "$PROJECT_CPD_INST_OPERANDS" get wo "$WO_CR_NAME" \
      -o jsonpath="{.metadata.labels.${OPERATOR_PATCH_LABEL_KEY}}" 2>/dev/null || true)"
    
    if [[ "$NEW_LABEL" == "$OPERATOR_PATCH_LABEL_VALUE" ]]; then
      log "   ✅ Label set successfully: ${OPERATOR_PATCH_LABEL_KEY}=${OPERATOR_PATCH_LABEL_VALUE}"
    else
      log "   ⚠️  WARNING: Could not confirm label was set"
    fi
  else
    log "   ⚠️  WARNING: Failed to set label on WO CR"
  fi
else
  log "⚠️  No WO CR found, skipping label."
fi

log ""

# -----------------------------
# Verify patched deployments are healthy
# -----------------------------
if [[ ${#PATCHED_DEPLOYMENTS[@]} -gt 0 ]]; then
  log "🔍 Verifying rollout status for patched deployments..."
  
  for DEPLOYMENT in "${PATCHED_DEPLOYMENTS[@]}"; do
    log "   Checking deployment/$DEPLOYMENT..."
    
    # Wait for rollout to complete
    if oc -n "$PROJECT_CPD_INST_OPERATORS" rollout status deploy/"$DEPLOYMENT" --timeout=300s; then
      # Check Ready/Desired replica ratio
      RATIO="$(oc -n "$PROJECT_CPD_INST_OPERATORS" get deploy "$DEPLOYMENT" \
        -o jsonpath='{.status.readyReplicas}/{.status.replicas}' 2>/dev/null || echo '0/0')"
      log "   Ready/Desired: $RATIO"
      
      if [[ "$RATIO" == "1/1" ]] || [[ "$RATIO" == "2/2" ]]; then
        log "   ✅ Deployment $DEPLOYMENT is healthy (pods up and running)"
      else
        log "   ⚠️  WARNING: Deployment $DEPLOYMENT is not at 1/1 or 2/2; current $RATIO"
      fi
    else
      log "   ✗ ERROR: Rollout status for deployment/$DEPLOYMENT did not complete successfully"
    fi
  done
else
  log "ℹ️  No deployments were patched; skipping health verification."
fi

# -----------------------------
# Final message
# -----------------------------
log ""
log "------------------------------------------------------------------"
log "✅ Operator patch steps completed (${OPERATOR_PATCH_LABEL_VALUE})"
log ""
log "📁 Backups saved under: ${BACKUP_DIR}"
log ""
log "📊 Monitor the watsonx Orchestrate CR status by running:"
log "   oc get wo -n ${PROJECT_CPD_INST_OPERANDS} -o yaml | grep -E 'watsonxOrchestrateStatus|${OPERATOR_PATCH_LABEL_KEY}'"
log ""
log "✓ Ensure the watsonx Orchestrate CR status is 'Completed'"
log "✓ Ensure label ${OPERATOR_PATCH_LABEL_KEY}=${OPERATOR_PATCH_LABEL_VALUE} is present"
log ""
log "⏱️  It will take another 15–20 minutes for the updated components"
log "   to be applied and restarted."
log "------------------------------------------------------------------
```

Make the script executable
```bash
chmod 775 5.4.2-Hotfix0.sh
```
 
Run the script
```bash
nohup sh 5.4.2-Hotfix0.sh &
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
hotfix: 5.4.2
```

Confirm the completion of the hot fix by checking the Watsonx Orchestrate custom resource status
```bash
oc get wo
```

The expected output
```bash
NAME   VERSION   DEPLOYED   VERIFIED   TOTAL   INSTALLMODE         QUIESCE        RECONCILE_PROGRESS   AGE
wo     5.4.2     34         34         34      agentic_assistant   NOT_QUIESCED   100%                 Xd
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

Monitor the pods custom resources related to watsonx_ai
```bash
watch -n 3 'oc get po -A -owide | egrep -v "([0-9])/\1" | egrep -v "Completed" && oc get wmlbase,ws,watsonxai'
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
--block_storage_class=${STG_CLASS_BLOCK} \
--file_storage_class=${STG_CLASS_FILE} \
--param-file=install-options.yml \
--upgrade=true
```

Monitor watson_speech upgrade, wait until the status shows `Completed` or `Ready`
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

#### Upgrading Service Instances

Get a list of all service instances using the following command
```bash
cpd-cli service-instance list --profile=${CPD_PROFILE_NAME}
```

---

#### Upgrading service instance of analyticsengine

Upgrading the service instance
```bash
cpd-cli service-instance upgrade \
--service-type=spark \
--profile=${CPD_PROFILE_NAME} \
--all
```

Validating the service instance upgrade status
```bash
cpd-cli service-instance list --service-type=spark --profile=${CPD_PROFILE_NAME}
```

---

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

## Upgrade the cpdbr service (est. 5-10 minutes)

**References**: [Upgrading the backup and restore software for an instance that uses the IBM Fusion](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=ubrsi-fusion-backup-restore-utility)
**References**: [What's new and changed in the platform](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=new-software-hub-platform#fixlist__title__2)

After you upgrade IBM Software Hub, you must upgrade the cpdbr-tenant service and install the Backup Restore Orchestration service for the instance

Log in to Red Hat OpenShift Container Platform as a cluster administrator
```bash
${OC_LOGIN}
```

Get the name of the Data Protection Application
```bash
oc get dpa --namespace=${OADP_PROJECT}
```

Set the DPA_NAME environment variable to the name of the Data Protection Application
```bash
export DPA_NAME=<DPA-name>
```

Patch the Data Protection Application custom resource

The command that you run depends on where your cluster pulls images from

Private container registry
```bash
oc patch dataprotectionapplication ${DPA_NAME} \
--namespace=${OADP_PROJECT} \
--type=json \
-p='[
  {
    "op": "replace",
    "path": "/spec/configuration/velero/customPlugins",
    "value": \[ 
      { 
        "image": "${PRIVATE_REGISTRY_LOCATION}/cpopen/cpfs/cpfs-oadp-plugins:latest", 
        "name": "cpfs-oadp-plugin" 
      },
      { 
        "image": "${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/cpdbr-velero-plugin:${VERSION}",
        "name": "cpdbr-velero-plugin" 
      },
      { 
        "image": "${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd/swhub-velero-plugin:${VERSION}", 
        "name": "swhub-velero-plugin" 
      },
      { 
        "image": "${PRIVATE_REGISTRY_LOCATION}/db2u/db2u-velero-plugin:${VERSION}",
        "name": "db2u-velero-plugin" 
      } 
    \]
  }
]'
```

Upgrade the cpdbr-tenant service

The command that you run depends on where your cluster pulls images from

Run the following command if you are using a private container registry and the scheduling service is not installed on the cluster
```bash
cpd-cli oadp install \
--component=cpdbr-tenant \
--namespace=${OADP_PROJECT} \
--tenant-operator-namespace=${PROJECT_CPD_INST_OPERATORS} \
--private-registry-location=${PRIVATE_REGISTRY_LOCATION} \
--upgrade=true \
--log-level=debug \
--verbose
```

Confirm that the required cluster role and cluster role binding were created in the ${PROJECT_INST_BR_SVC} when you installed the cpdbr-tenant service

If they do not exist, the command creates them
```bash
BINDING_NAME="cpdbr-tenant-service-crb-${PROJECT_CPD_INST_OPERATORS}"
SHOULD_ADD=false

# Check if the exact combination of SA name and namespace exists
if oc get clusterrolebinding ${BINDING_NAME} -o json | \
   jq -e ".subjects[]? | select(.kind==\"ServiceAccount\" and .name==\"${BR_OPERATOR_JOB_SA}\" and .namespace==\"${PROJECT_INST_BR_SVC}\")" > /dev/null 2>&1; then
  echo "ServiceAccount ${BR_OPERATOR_JOB_SA} already exists in namespace ${PROJECT_INST_BR_SVC}"
else
  echo "ServiceAccount ${BR_OPERATOR_JOB_SA} in namespace ${PROJECT_INST_BR_SVC} not found, adding"
  SHOULD_ADD=true
fi

# Add the subject if needed
if [ "${SHOULD_ADD}" = true ]; then
  oc patch clusterrolebinding ${BINDING_NAME} --type=json -p="[
    {
      \"op\": \"add\",
      \"path\": \"/subjects/-\",
      \"value\": {
        \"kind\": \"ServiceAccount\",
        \"name\": \"${BR_OPERATOR_JOB_SA}\",
        \"namespace\": \"${PROJECT_INST_BR_SVC}\"
      }
    }
  ]"
fi
```

Install the Backup Restore Orchestration service for the instance
```bash
cpd-cli manage apply-br \
--license_acceptance=true \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--br_tool=ibm-fusion \
--fusion_spectrum_ns=${PROJECT_FUSION} \
--fusion_br_ns=${OADP_PROJECT} \
--br_operator_ns=${PROJECT_INST_BR_SVC} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET}
```

Give the ${BR_OPERATOR_SA} service account the edit cluster role on the required projects
```bash
oc create rolebinding bros-rolebinding-edit --clusterrole=edit --serviceaccount=${PROJECT_INST_BR_SVC}:${BR_OPERATOR_SA} -n ${PROJECT_INST_BR_SVC}
oc label rolebinding bros-rolebinding-edit -n ${PROJECT_INST_BR_SVC} component-id=br-orchestration icpdsupport/addOnId=bros
```

Give the ${BR_OPERATOR_JOB_SA} service account the edit cluster role on the required projects
```bash
# Assign the edit role in the operators project
# =======================================================================================
oc create rolebinding bros-job-sa-rb-${BR_OPERATOR_JOB_SA} \
--clusterrole=edit \
--serviceaccount=${PROJECT_INST_BR_SVC}:${BR_OPERATOR_JOB_SA} \
-n ${PROJECT_CPD_INST_OPERATORS}

oc label rolebinding bros-job-sa-rb-${BR_OPERATOR_JOB_SA} \
-n ${PROJECT_CPD_INST_OPERATORS} \
component-id=br-orchestration \
icpdsupport/addOnId=bros

# Assign the edit role in the operands project
# =======================================================================================
oc create rolebinding bros-job-sa-rb-${BR_OPERATOR_JOB_SA} \
--clusterrole=edit \
--serviceaccount=${PROJECT_INST_BR_SVC}:${BR_OPERATOR_JOB_SA} \
-n ${PROJECT_CPD_INST_OPERANDS}


oc label rolebinding bros-job-sa-rb-${BR_OPERATOR_JOB_SA} \
-n ${PROJECT_CPD_INST_OPERANDS} \
component-id=br-orchestration \
icpdsupport/addOnId=bros

if [ -n "${PROJECT_CPD_INSTANCE_TETHERED_LIST}" ]; then
    IFS=',' read -ra TETHERED_NS_LIST <<< "${PROJECT_CPD_INSTANCE_TETHERED_LIST}"
    
    for TETHERED_NS in "${TETHERED_NS_LIST[@]}"; do
      oc create rolebinding bros-job-sa-rb-${BR_OPERATOR_JOB_SA} \
      --clusterrole=edit \
      --serviceaccount=${PROJECT_INST_BR_SVC}:${BR_OPERATOR_JOB_SA} \
      -n ${TETHERED_NS}
      
      oc label rolebinding bros-job-sa-rb-${BR_OPERATOR_JOB_SA} \
      -n ${TETHERED_NS} \
      component-id=br-orchestration \
      icpdsupport/addOnId=bros

    done
fi
```

---

## Post Upgrade Validation

#### Potential Issue - Enable WxO Observability

WxO development team to provide the new procedure to enable WxO Observability

**Previous runbook**: [Enable WxO Observability](https://github.com/kuanalex/ups/blob/main/WxO_Observability_PROD_Runbook.md)

---

#### Potential Issue - Fix platform-auth-service pod in ContainerStatusUnknown

Follow the procedure in the following known issue document to resolve platform-auth-service pod in ContainerStatusUnknown issue

**Previous runbook**: [Enabling the debug trace for the platform-auth-service cause pod in ContainerStatusUnknown state and repeatedly get evicted](https://www.ibm.com/mysupport/s/defect/aCIgJ000000C9qjWAC/dt467023?language=en_US)

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
