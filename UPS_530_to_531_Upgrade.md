# UPS Production Cluster CP4D 5.3.0 to 5.3.1 Upgrade

**From:**

```
CPD: 5.3.0
OCP: 4.17
Storage: Google Cloud Netapp Volumes and Persistent Disk on Google Cloud
Internet: airgap
Private container registry: yes
Components: cpd_platform,db2oltp,watson_speech,voice_gateway,watsonx_orchestrate,watsonx_ai,cognos_analytics,watsonx_governance
```

**To:**

```
CPD: 5.3.1
OCP: 4.17
Storage: Google Cloud Netapp Volumes and Persistent Disk on Google Cloud
Internet: airgap
Private container registry: yes
Components: cpd_platform,db2oltp,watson_speech,voice_gateway,watsonx_orchestrate,watsonx_ai,cognos_analytics,watsonx_governance
```

---

## Pre-requisites

#### 1. Backup of the cluster is done

Backup your Cloud Pak for Data cluster before the upgrade.
For details, see [Backing up and restoring Cloud Pak for Data](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=administering-backing-up-restoring-software-hub).

**Note:**
Some services don't support the offline OADP backup. Review the backup documentation and take the dedicated approach when necessary.

#### 2. The image mirroring completed successfully

If a private container registry is in-use to host the IBM Cloud Pak for Data software images, you must mirror the updated images from the IBM® Entitled Registry to the private container registry.

Reference: [Mirroring images to private image registry](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=mipcr-mirroring-images-directly-private-container-registry-1)

#### 3. The permissions required for the upgrade is ready

- OpenShift cluster administrator permissions
- Cloud Pak for Data administrator permissions
- Permission to access the private image registry for pushing or pulling images
- Access to the bastion node for executing the upgrade commands

#### 4. A pre-upgrade health check is made to ensure the cluster's readiness for upgrade

- The OpenShift cluster, persistent storage and Cloud Pak for Data platform and services are in healthy status

#### 5. Backup the Routes

```bash
oc get routes -n ${PROJECT_CPD_INST_OPERANDS} -o yaml > routes_backup_$(date +%Y%m%d_%H%M%S).yaml
```

#### 6. Backup the TemporaryPatch

```bash
oc get TemporaryPatch -n ${PROJECT_CPD_INST_OPERANDS} -o yaml > temporarypatch_backup_$(date +%Y%m%d_%H%M%S).yaml
```

---

## Table of Contents

1. [Upgrade Path Validation](#upgrade-path-validation)
2. [Prerequisites](#prerequisites)
3. [Pre-Upgrade Backups](#pre-upgrade-backups)
4. [Upgrade Execution](#upgrade-execution)
5. [RSI Patch Management](#rsi-patch-management)
6. [Post-Upgrade Validation](#post-upgrade-validation)

---

## Upgrade Path Validation






---

## 1. Prerequisites

### Required Tools

Ensure the following tools are installed and configured:

```bash
# Verify OpenShift CLI
oc version

# Verify CPD CLI version 14.3.1.2
cpd-cli version

# Verify Helm version 3.16.3
helm version

# Verify jq for JSON processing
jq --version

# Verify podman for OLM utils container
podman version
```

**Install/Update CPD CLI 14.3.1.2:**

```bash
# Download and extract cpd-cli 14.3.1.2
wget https://github.com/IBM/cpd-cli/releases/download/v14.3.1.2/cpd-cli-linux-EE-14.3.1.2.tgz && \
gzip -d cpd-cli-linux-EE-14.3.1.2.tgz && \
tar -xvf cpd-cli-linux-EE-14.3.1.2.tar && \
rm -rf cpd-cli-linux-EE-14.3.1.2.tar

# Verify installation (directory name includes build number)
cd cpd-cli-linux-EE-*/ && ./cpd-cli version && cd ..

# Add to PATH
export PATH=$PWD/cpd-cli-linux-EE-*:$PATH
cpd-cli version
```

**Release Information:**
- **GitHub Release**: [v14.3.1.2](https://github.com/IBM/cpd-cli/releases/tag/v14.3.1.2)
- **Download URL**: https://github.com/IBM/cpd-cli/releases/download/v14.3.1.2/cpd-cli-linux-EE-14.3.1.2.tgz

### Access Requirements

**Required Access:**
- OpenShift cluster admin access
- IBM Entitlement Key with appropriate permissions
- Access to IBM Container Registry (cp.icr.io)
- Access to private registry: UPDATE_WITH_PRIVATE_REGISTRY_URL

### Environment Variables Setup (cpd_vars.sh)

Ensure your environment variables script is configured correctly. See [IBM Documentation](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=cri-updating-your-environment-variables-script-1).

**Verify environment variables:**

```bash
# Source your environment variables script
source cpd_vars.sh

# Verify key variables are set
echo "CPD Version: ${VERSION}"
echo "OCP URL: ${OCP_URL}"
echo "Operators Namespace: ${PROJECT_CPD_INST_OPERATORS}"
echo "Operands Namespace: ${PROJECT_CPD_INST_OPERANDS}"
echo "Block Storage: ${STG_CLASS_BLOCK}"
echo "File Storage: ${STG_CLASS_FILE}"
echo "Components: ${COMPONENTS}"
```

### Login to OpenShift Cluster

```bash
# Login using cpd-cli
${CPDM_OC_LOGIN}

# Or login using oc directly
${OC_LOGIN}

# Verify cluster access
oc whoami
oc get nodes
```

### Restart OLM Utils Container

```bash
# Restart the OLM utils container with updated environment
cpd-cli manage restart-container

# Verify container is running
podman ps | grep olm-utils
```

### Air-Gapped Environment Prerequisites


**⚠️ IMPORTANT**: Air-gapped environment detected. Complete these steps before upgrading.

**Private Registry**: UPDATE_WITH_PRIVATE_REGISTRY_URL

**Required Steps** (see [IBM Documentation](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=requirements-preparing-upgrade-in-restricted-network)):

1. **[Obtain OLM Utils v4 image](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=pruirn-obtaining-olm-utils-v4-image-2)**
2. **[Download CASE packages](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=pruirn-downloading-case-packages-2)** for all components
3. **Mirror images** to private registry ([direct](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=mipcr-mirroring-images-directly-private-container-registry-2) or [intermediary](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=mipcr-mirroring-images-using-intermediary-container-registry-2))
4. **[Pull OLM Utils](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=prufpcr-pulling-olm-utils-v4-image-from-private-container-registry-2)** from private registry
5. **[Update cluster resources](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=pyc-updating-cluster-scoped-resources-shared-cluster-components-1)** for shared components
6. **[Update cluster resources](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=puish-updating-cluster-scoped-resources-instance-1)** for CPD instance



### Advanced Service Prerequisites

Some services require additional prerequisite software upgrades. Review [IBM Documentation](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=pyc-upgrading-prerequisite-software-1) for details.

**Multicloud Object Gateway (MCG)** - Required for: Watson Speech, Voice Gateway, Watsonx Ai
[Upgrade MCG](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=ups-upgrading-multicloud-object-gateway-1) during storage or OCP upgrade.

**Red Hat OpenShift Serverless Knative** - Required for: Watsonx Orchestrate
[Upgrade Knative](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=ups-upgrading-red-hat-openshift-serverless-knative-eventing-1) to supported version.

**GPU Operators** - [Upgrade if needed](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=ups-upgrading-operators-services-that-require-gpus-1) for GPU-enabled services.

**Red Hat OpenShift AI** - [Review upgrade requirements](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=ups-upgrading-red-hat-openshift-ai-1) if using OpenShift AI.


---

## 2. Pre-Upgrade Backups

## Pre-Upgrade Backups

⚠️ **CRITICAL**: Complete all backups before proceeding with upgrade. Store backups securely outside the cluster.

### CPD Health Check and Backup

The `cpd-cli health` command creates comprehensive backups including CRs, ConfigMaps, and cluster state.

```bash
# Run comprehensive health check with backup
cpd-cli health runcommand \
  --commands=cluster,nodes,operands,operators \
  --control_plane_ns="${PROJECT_CPD_INST_OPERANDS}" \
  --operator_ns="${PROJECT_CPD_INST_OPERATORS}" \
  --log-level=debug \
  --verbose \
  --save

# The health check creates a timestamped directory with:
# - Custom Resources (CRs)
# - ConfigMaps
# - Cluster state
# - Node information
# - Operator status
```

### Additional Required Backups

#### Routes Backup
```bash
oc get routes -n ${PROJECT_CPD_INST_OPERANDS} -o yaml > routes_backup_$(date +%Y%m%d_%H%M%S).yaml
```

#### TemporaryPatch Backup
```bash
oc get TemporaryPatch -n ${PROJECT_CPD_INST_OPERANDS} -o yaml > temporarypatch_backup_$(date +%Y%m%d_%H%M%S).yaml
```

⚠️ **DO NOT PROCEED** until backups are complete and stored securely outside the cluster.
---


## 3. Pre-Upgrade Health Check

### Run Comprehensive Health Check

```bash
# Run CPD health check (includes cluster, nodes, operands, operators)
# This command also creates a backup of the current state
cpd-cli health runcommand \
--commands=cluster,nodes,operands,operators \
--control_plane_ns="${PROJECT_CPD_INST_OPERANDS}" \
--operator_ns="${PROJECT_CPD_INST_OPERATORS}" \
--log-level=debug \
--verbose \
--save

# The health check will create a timestamped directory with results
# Review the output for any critical issues before proceeding
```

### Check for Hot Fixes and Patches

```bash
# Check for image_digests in service CRs (indicates hot fixes/patches)
oc project ${PROJECT_CPD_INST_OPERANDS}

for i in $(oc api-resources | grep cpd.ibm.com | awk '{print $1}' | grep -v zenextensions); do
  echo "************* $i *************"
  for x in $(oc get $i --no-headers | awk '{print $1}'); do
    echo "--------- $x ------------"
    oc get $i $x -o jsonpath={.spec} | jq
  done
done

# Review output for any image_digests fields
# Document any hot fixes/patches that need to be removed before upgrade
```

### Basic Cluster Validation

```bash
# Check node status
oc get nodes

# Check cluster operators
oc get co

# Check cluster version
oc get clusterversion

# Verify no pods in CrashLoopBackOff
oc get pods --all-namespaces | grep -i crash

# Check for unhealthy pods
oc get po -A -owide | egrep -v '([0-9])/\1' | egrep -v 'Completed'
```

### Storage Validation

```bash
# Verify storage classes
oc get sc

# Check PVC status
oc get pvc -n ${PROJECT_CPD_INST_OPERANDS}

# Verify storage vendor health
```

### CPD Platform Validation

```bash
# Check CR status
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}

# Check for pods not in Completed status
oc get pods -n ${PROJECT_CPD_INST_OPERANDS} --field-selector=status.phase!=Succeeded

# List service instances
cpd-cli manage list-deployed-components --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

---

## 4. Upgrade Execution

### 4.1 Upgrade Shared Cluster Components

#### 4.2.1 Upgrade IBM Licensing

**Reference**: [Upgrading shared cluster components](https://www.ibm.com/docs/en/cloud-paks/cp-data/5.3.x?topic=upgrading-shared-cluster-components)

```bash
# Upgrade IBM Licensing service
cpd-cli manage apply-cluster-components \
  --release=${VERSION} \
  --license_acceptance=true \
  --licensing_ns=${PROJECT_LICENSE_SERVICE}

# Verify licensing pods are running
oc get pods -n ${PROJECT_LICENSE_SERVICE}
```

#### 4.1.2 Upgrade Scheduler (if installed)

**Reference**: [Upgrading the scheduling service](https://www.ibm.com/docs/en/cloud-paks/cp-data/5.3.x?topic=components-upgrading-scheduling-service)

```bash
# Check if scheduler is installed
oc get scheduling -A

# If scheduler exists, upgrade it
cpd-cli manage apply-scheduler \
  --release=${VERSION} \
  --license_acceptance=true \
  --scheduler_ns=${PROJECT_SCHEDULING_SERVICE}

# Verify scheduler pods are running
oc get pods -n ${PROJECT_SCHEDULING_SERVICE}
```

---

### 4.2 Upgrade Cloud Pak for Data Platform

**Reference**: [Upgrading IBM Software Hub](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=upgrading)

```bash
# Upgrade CPD platform using install-components (5.3.x method)
cpd-cli manage install-components \
  --license_acceptance=true \
  --components=cpd_platform \
  --release=${VERSION} \
  --operator_ns=${PROJECT_CPD_INST_OPERATORS} \
  --instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --image_pull_prefix=${IMAGE_PULL_PREFIX} \
  --image_pull_secret=${IMAGE_PULL_SECRET} \
  --run_storage_tests=false \
  --upgrade=true

# Monitor platform upgrade progress (this takes 60-80 minutes)
watch -n 30 'oc get ZenService lite-cr -n ${PROJECT_CPD_INST_OPERANDS} -o jsonpath="{.status.zenStatus}"'

# Check platform pods
oc get pods -n ${PROJECT_CPD_INST_OPERANDS} | grep -E "zen|usermgmt|ibm-nginx"

# Verify platform version
oc get ZenService lite-cr -n ${PROJECT_CPD_INST_OPERANDS} -o jsonpath='{.status.zenStatus.versions[0].version}'
```

---

### 4.3 Upgrade Services

**Option 1: Batch Upgrade (Recommended for Most Cases)**

Upgrade all services together for faster completion:

```bash
# Upgrade all services in batch (5.3.x method)
cpd-cli manage install-components \
  --license_acceptance=true \
  --components=${COMPONENTS} \
  --release=${VERSION} \
  --operator_ns=${PROJECT_CPD_INST_OPERATORS} \
  --instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --image_pull_prefix=${IMAGE_PULL_PREFIX} \
  --image_pull_secret=${IMAGE_PULL_SECRET} \
  --run_storage_tests=false \
  --upgrade=true

# Monitor overall upgrade progress
watch -n 30 'oc get pods -n ${PROJECT_CPD_INST_OPERANDS} | grep -v Running | grep -v Completed'

# Check upgrade status
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

**Option 2: Individual Service Upgrade (For More Control)**

Upgrade services one at a time for better monitoring and troubleshooting:

#### 4.3.2 Upgrade Db2 OLTP

```bash
# Upgrade db2oltp (5.3.x method)
cpd-cli manage install-components \
  --license_acceptance=true \
  --components=db2oltp \
  --release=${VERSION} \
  --operator_ns=${PROJECT_CPD_INST_OPERATORS} \
  --instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --image_pull_prefix=${IMAGE_PULL_PREFIX} \
  --image_pull_secret=${IMAGE_PULL_SECRET} \
  --run_storage_tests=false \
  --upgrade=true

# Monitor db2oltp upgrade
cpd-cli manage get-cr-status \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --components=db2oltp

# Check db2oltp pods
oc get pods -n ${PROJECT_CPD_INST_OPERANDS} | grep db2oltp
```

#### 4.3.3 Upgrade Watson Speech

```bash
# Upgrade watson_speech (5.3.x method)
cpd-cli manage install-components \
  --license_acceptance=true \
  --components=watson_speech \
  --release=${VERSION} \
  --operator_ns=${PROJECT_CPD_INST_OPERATORS} \
  --instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --image_pull_prefix=${IMAGE_PULL_PREFIX} \
  --image_pull_secret=${IMAGE_PULL_SECRET} \
  --run_storage_tests=false \
  --upgrade=true

# Monitor watson_speech upgrade
cpd-cli manage get-cr-status \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --components=watson_speech

# Check watson_speech pods
oc get pods -n ${PROJECT_CPD_INST_OPERANDS} | grep watson_speech
```

#### 4.3.4 Upgrade Voice Gateway

```bash
# Upgrade voice_gateway (5.3.x method)
cpd-cli manage install-components \
  --license_acceptance=true \
  --components=voice_gateway \
  --release=${VERSION} \
  --operator_ns=${PROJECT_CPD_INST_OPERATORS} \
  --instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --image_pull_prefix=${IMAGE_PULL_PREFIX} \
  --image_pull_secret=${IMAGE_PULL_SECRET} \
  --run_storage_tests=false \
  --upgrade=true

# Monitor voice_gateway upgrade
cpd-cli manage get-cr-status \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --components=voice_gateway

# Check voice_gateway pods
oc get pods -n ${PROJECT_CPD_INST_OPERANDS} | grep voice_gateway
```

#### 4.3.5 Upgrade Watsonx Orchestrate

```bash
# Upgrade watsonx_orchestrate (5.3.x method)
cpd-cli manage install-components \
  --license_acceptance=true \
  --components=watsonx_orchestrate \
  --release=${VERSION} \
  --operator_ns=${PROJECT_CPD_INST_OPERATORS} \
  --instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --image_pull_prefix=${IMAGE_PULL_PREFIX} \
  --image_pull_secret=${IMAGE_PULL_SECRET} \
  --run_storage_tests=false \
  --upgrade=true

# Monitor watsonx_orchestrate upgrade
cpd-cli manage get-cr-status \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --components=watsonx_orchestrate

# Check watsonx_orchestrate pods
oc get pods -n ${PROJECT_CPD_INST_OPERANDS} | grep watsonx_orchestrate
```

#### 4.3.6 Upgrade Watsonx Ai

```bash
# Upgrade watsonx_ai (5.3.x method)
cpd-cli manage install-components \
  --license_acceptance=true \
  --components=watsonx_ai \
  --release=${VERSION} \
  --operator_ns=${PROJECT_CPD_INST_OPERATORS} \
  --instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --image_pull_prefix=${IMAGE_PULL_PREFIX} \
  --image_pull_secret=${IMAGE_PULL_SECRET} \
  --run_storage_tests=false \
  --upgrade=true

# Monitor watsonx_ai upgrade
cpd-cli manage get-cr-status \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --components=watsonx_ai

# Check watsonx_ai pods
oc get pods -n ${PROJECT_CPD_INST_OPERANDS} | grep watsonx_ai
```

#### 4.3.7 Upgrade Cognos Analytics

```bash
# Upgrade cognos_analytics (5.3.x method)
cpd-cli manage install-components \
  --license_acceptance=true \
  --components=cognos_analytics \
  --release=${VERSION} \
  --operator_ns=${PROJECT_CPD_INST_OPERATORS} \
  --instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --image_pull_prefix=${IMAGE_PULL_PREFIX} \
  --image_pull_secret=${IMAGE_PULL_SECRET} \
  --run_storage_tests=false \
  --upgrade=true

# Monitor cognos_analytics upgrade
cpd-cli manage get-cr-status \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --components=cognos_analytics

# Check cognos_analytics pods
oc get pods -n ${PROJECT_CPD_INST_OPERANDS} | grep cognos_analytics
```

#### 4.3.8 Upgrade Watsonx Governance

```bash
# Upgrade watsonx_governance (5.3.x method)
cpd-cli manage install-components \
  --license_acceptance=true \
  --components=watsonx_governance \
  --release=${VERSION} \
  --operator_ns=${PROJECT_CPD_INST_OPERATORS} \
  --instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --image_pull_prefix=${IMAGE_PULL_PREFIX} \
  --image_pull_secret=${IMAGE_PULL_SECRET} \
  --run_storage_tests=false \
  --upgrade=true

# Monitor watsonx_governance upgrade
cpd-cli manage get-cr-status \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --components=watsonx_governance

# Check watsonx_governance pods
oc get pods -n ${PROJECT_CPD_INST_OPERANDS} | grep watsonx_governance
```


---
---

## 5. Service Instance Upgrades


## Service Instance Upgrades

**⚠️ IMPORTANT**: Some services require separate instance upgrades after the CR upgrade completes. Service instances are NOT automatically upgraded when you upgrade the service CR.

### Prerequisites

Before upgrading service instances, ensure you have:
1. ✅ Completed all CR upgrades successfully
2. ✅ Created a CPD profile with appropriate permissions
3. ✅ Set the `CPD_PROFILE_NAME` environment variable

**CPD Profile Requirements**:
- User must have `can_provision` (Create service instances) permission
- User must have `manage_service_instances` (Manage service instances) permission
- Documentation: [Creating a CPD profile](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=cli-creating-cpd-profile)

---

### Services Requiring Instance Upgrades

The following services in your configuration require manual instance upgrades:

#### 1. Db2 OLTP

**Service Type**: `db2oltp`

**Step 1**: List all Db2 instances:
```bash
cpd-cli service-instance list \
--service-type=db2oltp \
--profile=${CPD_PROFILE_NAME}
```

**Step 2**: Check instance status:
```bash
# Set instance name
export INSTANCE_NAME="<instance-name>"

# Check if instance is running
cpd-cli service-instance status ${INSTANCE_NAME} \
--profile=${CPD_PROFILE_NAME} \
--service-type=db2oltp
```

**Step 3**: Upgrade each instance individually:
```bash
cpd-cli service-instance upgrade \
--service-type=db2oltp \
--instance-name=${INSTANCE_NAME} \
--profile=${CPD_PROFILE_NAME}
```

**Documentation**: [Upgrading Db2](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=u-upgrading-db2)

**⚠️ Important**: Instance must be in running state before upgrade


---

#### 2. Watson Speech

**Service Type**: `speech`

**Upgrade Command** (all instances):
```bash
cpd-cli service-instance upgrade \
--service-type=speech \
--profile=${CPD_PROFILE_NAME} \
--all
```

**Documentation**: [Upgrading Watson Speech Services](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=u-upgrading-watson-speech-services)


---


### Service Instance Upgrade Checklist

Use this checklist to track instance upgrades:

- [ ] Db2 OLTP instances upgraded and validated
- [ ] Watson Speech instances upgraded and validated

### Notes

- **Automatic Upgrades**: If you have Db2 Data Management Console (dmc), its instances upgrade automatically - no manual action needed
- **Upgrade Order**: Upgrade service instances after all CR upgrades are complete
- **Validation**: Always validate instance upgrades by listing instances and checking their versions
- **Troubleshooting**: If an instance upgrade fails, check the instance logs and status before retrying


---

## 6. RSI Patch Management

## RSI Patch Management

If you have any custom RSI patches that patch zen pods or IBM Cloud Pak foundational services pods, reapply the patches after the upgrade.

### Step 1: List Existing RSI Patches

Run the following command to get a list of the RSI patches in the operands project:

```bash
cpd-cli manage get-rsi-patch-info \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --all
```

**Expected Output**: List of all RSI patches with their status and configuration.

### Step 2: Reapply Custom Patches

If there are patches that apply to zen or IBM Cloud Pak foundational services pods, run the following command to apply your custom patches:

```bash
cpd-cli manage apply-rsi-patches \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

**Verification**:
```bash
# Verify patches are active
cpd-cli manage get-rsi-patch-info \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --all

# Check that affected pods are running
oc get pods -n ${PROJECT_CPD_INST_OPERANDS}
```

**Reference**: [IBM Documentation - Upgrading Software Hub](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=upgrading)

---
---

## 7. Post-Upgrade Validation

### Post-Upgrade Validation

```bash
# Check CR status
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}

# Check for pods not in Completed status
oc get pods -n ${PROJECT_CPD_INST_OPERANDS} --field-selector=status.phase!=Succeeded

# Verify CPD version
oc get ZenService lite-cr -n ${PROJECT_CPD_INST_OPERANDS} -o jsonpath='{.status.zenStatus.versions[0].version}'

# List service instances
cpd-cli service-instance list --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

---

### 5.3 Post-Upgrade Migrations


---

**End of Runbook**

*Generated by CP4D Upgrade Automation Tool v1.2.0*