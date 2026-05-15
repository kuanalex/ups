# UPS Production Cluster CP4D 5.3.0 to 5.3.1 Upgrade

**From:**

```
CPD: 5.3.0
OCP: 4.18.40
Storage: Google Cloud Netapp Volumes and Persistent Disk on Google Cloud
Internet: airgap
Private container registry: yes
Components: ibm-licensing,scheduler,ibm_events_operator,ccs,cpfs,zen,cpd_platform,watsonx_orchestrate,watsonx_ai,watsonx_governance,watson_speech,voice_gateway,db2oltp,cognos_analytics
```

**To:**

```
CPD: 5.3.1
OCP: 4.18.40
Storage: Google Cloud Netapp Volumes and Persistent Disk on Google Cloud
Internet: airgap
Private container registry: yes
Components: ibm-licensing,scheduler,ibm_events_operator,ccs,cpfs,zen,cpd_platform,watsonx_orchestrate,watsonx_ai,watsonx_governance,watson_speech,voice_gateway,db2oltp,cognos_analytics
```

---

## Prerequisites

#### 1. Backup of the cluster is done

Backup your IBM Software Hub cluster before the upgrade.
For details, see [Backing up and restoring IBM Software Hub](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=administering-backing-up-restoring-software-hub).

**Note:**
Some services don't support the offline OADP backup. Review the backup documentation and take the dedicated approach when necessary.

#### 2. The image mirroring completed successfully

If a private container registry is in-use to host the IBM Software Hub software images, you must mirror the updated images from the IBM® Entitled Registry to the private container registry.

Reference: [Mirroring images to private image registry](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=mipcr-mirroring-images-directly-private-container-registry-1)

#### 3. The permissions required for the upgrade is ready

- OpenShift cluster administrator permissions
- IBM Software Hub administrator permissions
- Permission to access the private image registry for pushing or pulling images
- Access to the bastion node for executing the upgrade commands

#### 4. A pre-upgrade health check is made to ensure the cluster's readiness for upgrade

- The OpenShift cluster, persistent storage and IBM Software Hub platform and services are in healthy status

---

## Table of Contents

- [Pre Upgrade Steps](#pre-upgrade-steps)
- [Pre Upgrade Backups](#pre-upgrade-backups)
- [Upgrade Execution](#upgrade-execution)
- [Service Instance Upgrades](#service-instance-upgrades)
- [Upgrade cpdbr Service](#upgrade-cpdbr-service)
- [RSI Patch Management](#rsi-patch-management)
- [Post Upgrade Validation](#post-upgrade-validation)

---

## Pre Upgrade Steps

### Required Tools

Ensure the following tools are installed and updated to the required versions:

- **IBM Software Hub CLI**: Version 14.3.1.3
- **OpenShift CLI (oc)**: Compatible version for your cluster
- **Helm CLI**: Version 4.1.4

**Installation and Update Instructions:**

For detailed instructions on installing or updating these tools, refer to:
- [Updating client workstations](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=53-updating-client-workstations)
  - [Updating IBM Software Hub CLI](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=ucw-updating-software-hub-cli-1)
  - [Updating OpenShift CLI](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=ucw-updating-openshift-cli-1)
  - [Installing Helm CLI](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=ucw-installing-helm-cli-1)

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
echo "Private Registry: ${PRIVATE_REGISTRY_LOCATION}"
```

### Login to OpenShift Cluster
```bash
# Login using cpd-cli
${CPDM_OC_LOGIN}
```

```bash
# Or login using oc directly
${OC_LOGIN}
```

```bash
# Verify cluster access
oc whoami
oc get nodes
```

### Restart OLM Utils Container

```bash
# Restart the OLM utils container with updated environment
cpd-cli manage restart-container
```

```bash
# Verify container is running
podman ps | grep olm-utils
```

### Air Gapped Environment Prerequisites

**⚠️ IMPORTANT**: Air gapped environment detected. Complete these steps before upgrading.

**Private Registry**: UPDATE_WITH_PRIVATE_REGISTRY_URL

**Required Steps** (see [IBM Documentation](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=requirements-preparing-upgrade-in-restricted-network)):

1. **[Obtain OLM Utils v4 image](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=pruirn-obtaining-olm-utils-v4-image-2)**
2. **[Download CASE packages](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=pruirn-downloading-case-packages-2)** for all components
3. **Mirror images** to private registry ([direct](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=mipcr-mirroring-images-directly-private-container-registry-2) or [intermediary](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=mipcr-mirroring-images-using-intermediary-container-registry-2))
4. **[Pull OLM Utils](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=prufpcr-pulling-olm-utils-v4-image-from-private-container-registry-2)** from private registry

**Note**: Cluster-scoped resources and entitlements are applied in the [Upgrade Execution](#upgrade-execution) section.


### Advanced Service Prerequisites

Some services require additional prerequisite software upgrades. Review [IBM Documentation](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=pyc-upgrading-prerequisite-software-1) for details.

**Multicloud Object Gateway (MCG)** - Required for: Watson Speech, Voice Gateway, Watsonx Ai
[Upgrade MCG](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=ups-upgrading-multicloud-object-gateway-1) during storage or OCP upgrade.

**Red Hat OpenShift Serverless Knative** - Required for: Watsonx Orchestrate
[Upgrade Knative](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=ups-upgrading-red-hat-openshift-serverless-knative-eventing-1) to supported version.

**GPU Operators** - [Upgrade if needed](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=ups-upgrading-operators-services-that-require-gpus-1) for GPU-enabled services.

**Red Hat OpenShift AI** - [Review upgrade requirements](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=ups-upgrading-red-hat-openshift-ai-1) if using OpenShift AI.


---

## Pre Upgrade Backups

### Additional Required Backups

#### Routes Backup
```bash
oc get routes -n ${PROJECT_CPD_INST_OPERANDS} -o yaml > routes_backup_$(date +%Y%m%d_%H%M%S).yaml
```

#### TemporaryPatch Backup
```bash
oc get TemporaryPatch -n ${PROJECT_CPD_INST_OPERANDS} -o yaml > temporarypatch_backup_$(date +%Y%m%d_%H%M%S).yaml
```
---


## Pre Upgrade Health Check

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
# Check node, machineConfig, clusterOperators, clusterVersion
oc get nodes,mcp,co,clusterversion
```

### Storage Validation

```bash
# Verify storage classes
oc get sc
```

```bash
# Check PVC status
oc get pvc -n ${PROJECT_CPD_INST_OPERANDS}
```

### CPD Platform Validation

```bash
# Check CR status
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

```bash
# Check for pods not running correctly (excludes completed jobs)
oc get po -A -owide | egrep -v '([0-9])/\1' | egrep -v 'Completed'
```

```bash
# List service instances
cpd-cli manage list-deployed-components --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

**Note: Fix any pod issues and ensure the service CRs are in Completed status before proceeding with the upgrade**

---

## Upgrade Execution

### 4.1 Prepare Cluster for Upgrade

#### 4.1.1 Update Cluster-Scoped Resources for Shared Components

**Reference**: [Updating cluster-scoped resources for shared cluster components](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=pyc-updating-cluster-scoped-resources-shared-cluster-components-1)

```bash
# Generate cluster-scoped resource definitions for scheduling service
cpd-cli manage case-download \
--components=scheduler \
--patch_id=0 \ 
--scheduler_ns=${PROJECT_SCHEDULING_SERVICE} \
--cluster_resources=true
```

```bash
# Change to work directory
cd cpd-cli-workspace/olm-utils-workspace/work
```

```bash
# Apply cluster-scoped resources
oc apply -f cluster_scoped_resources.yaml \
  --server-side \
  --force-conflicts
```

```bash
# Optional: Keep a record
mv cluster_scoped_resources.yaml ${VERSION}-${PROJECT_SCHEDULING_SERVICE}-cluster_scoped_resources.yaml
```

```bash
# Return to base directory
cd -
```

#### 4.1.2 Update Cluster-Scoped Resources for CPD Instance

**Reference**: [Updating cluster-scoped resources for the instance](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=puish-updating-cluster-scoped-resources-instance-1)

```bash
# Generate cluster-scoped resource definitions for CPD instance
cpd-cli manage case-download \
--components=${COMPONENTS} \
--release=${VERSION} \
--patch_id=0 \ 
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--cluster_resources=true

# Change to work directory
cd cpd-cli-workspace/olm-utils-workspace/work

# Apply cluster-scoped resources
oc apply -f cluster_scoped_resources.yaml --server-side --force-conflicts

# Optional: Keep a record
mv cluster_scoped_resources.yaml ${VERSION}-${PROJECT_CPD_INST_OPERATORS}-cluster_scoped_resources.yaml

# Return to base directory
cd -
```

#### 4.1.3 Creating image pull secrets for shared cluster components

**Reference**: [Creating image pull secrets for shared cluster components](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=pyc-creating-image-pull-secrets-shared-cluster-components)

Log in to Red Hat® OpenShift® Container Platform as a user with sufficient permissions to complete the task
```bash
${OC_LOGIN}
```

Create a file named dockerconfig.json based on where your cluster pulls images from
```bash
cat <<EOF > dockerconfig.json 
{
  "auths": {
    "${PRIVATE_REGISTRY_LOCATION}": {
      "auth": "${IMAGE_PULL_CREDENTIALS}"
    }
  }
}
EOF
```

#### 4.1.4 Creating image pull secrets for the instance  

**Reference**: [Creating image pull secrets for an instance of IBM Software Hub](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=uish-creating-image-pull-secrets-instance)

Log in to Red Hat® OpenShift® Container Platform as a user with sufficient permissions to complete the task
```bash
${OC_LOGIN}
```

Create a file named dockerconfig.json based on where your cluster pulls images from
```bash
cat <<EOF > dockerconfig.json 
{
  "auths": {
    "${PRIVATE_REGISTRY_LOCATION}": {
      "auth": "${IMAGE_PULL_CREDENTIALS}"
    }
  }
}
EOF
```

Create the image pull secret in the operators project for the instance
```bash
oc create secret docker-registry ${IMAGE_PULL_SECRET} \
--from-file ".dockerconfigjson=dockerconfig.json" \
--namespace=${PROJECT_CPD_INST_OPERATORS}
```

Create the image pull secret in the operands project for the instance
```bash
oc create secret docker-registry ${IMAGE_PULL_SECRET} \
--from-file ".dockerconfigjson=dockerconfig.json" \
--namespace=${PROJECT_CPD_INST_OPERATORS}
```

#### 4.1.5 Apply Entitlements

**Reference**: [Applying your entitlements](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=aye-applying-your-entitlements-without-node-pinning-2)

```bash
# Apply IBM Cloud Pak for Data Enterprise Edition entitlement
cpd-cli manage apply-entitlement \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --entitlement=cpd-enterprise \
  --production=true

# Apply watsonx.ai license
cpd-cli manage apply-entitlement \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --entitlement=watsonx-ai \
  --production=true

# Apply watsonx.governance licenses
cpd-cli manage apply-entitlement \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --entitlement=watsonx-gov-mm \
  --production=true

cpd-cli manage apply-entitlement \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --entitlement=watsonx-gov-rc \
  --production=true

# Apply watsonx Orchestrate license
cpd-cli manage apply-entitlement \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --entitlement=watsonx-orchestrate \
  --production=true

# Apply Watson Speech licenses
cpd-cli manage apply-entitlement \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --entitlement=speech-to-text

cpd-cli manage apply-entitlement \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --entitlement=text-to-speech

# Apply Cognos Analytics license
cpd-cli manage apply-entitlement \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --entitlement=cognos-analytics
```

---

### 4.2 Upgrade Shared Cluster Components

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

#### 4.2.2 Upgrade Scheduler (if installed)

**Reference**: [Upgrading the scheduling service](https://www.ibm.com/docs/en/cloud-paks/cp-data/5.3.x?topic=components-upgrading-scheduling-service)

```bash
# Check if scheduler is installed
oc get scheduling -A
```

```bash
# If scheduler exists, upgrade it
cpd-cli manage apply-scheduler \
--release=${VERSION} \
--patch_id=0 \
--license_acceptance=true \
--scheduler_ns=${PROJECT_SCHEDULING_SERVICE}
```

```bash
# Verify scheduler pods are running
oc get pods -n ${PROJECT_SCHEDULING_SERVICE}
```

---

### 4.3 Upgrade IBM Software Hub Platform

**Reference**: [Upgrading IBM Software Hub](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=upgrading)

```bash
# Upgrade CPD platform using install-components (Est. 32 minutes)
cpd-cli manage install-components \
--license_acceptance=true \
--components=cpd_platform \
--release=${VERSION} \
--patch_id=0 \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--run_storage_tests=false \
--upgrade=true
```

```bash
# Monitor platform upgrade progress (this takes 60-80 minutes)
watch -n 30 'oc get ZenService lite-cr -n ${PROJECT_CPD_INST_OPERANDS} -o jsonpath="{.status.zenStatus}"'
```

```bash
# Check platform pods
oc get pods -n ${PROJECT_CPD_INST_OPERANDS} | grep -E "zen|usermgmt|ibm-nginx"
```

```bash
# Verify platform version
oc get ZenService lite-cr -n ${PROJECT_CPD_INST_OPERANDS} -o jsonpath='{.status.zenStatus.versions[0].version}'
```

## RSI Patch Management

If you have any custom RSI patches that patch zen pods or IBM Cloud Pak foundational services pods, reapply the patches after the upgrade.

### Step 1: List Existing RSI Patches

Run the following command to get a list of the RSI patches in the operands project:

```bash
cpd-cli manage get-rsi-patch-info --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --all
```

**Expected Output**: List of all RSI patches with their status and configuration.

### Step 2: Reapply Custom Patches

If there are patches that apply to zen or IBM Cloud Pak foundational services pods, run the following command to apply your custom patches:

```bash
cpd-cli manage apply-rsi-patches --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

```bash
# Verify patches are active
cpd-cli manage get-rsi-patch-info --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --all

# Check that affected pods are running
oc get pods -n ${PROJECT_CPD_INST_OPERANDS}
```

**Reference**: [IBM Documentation - Upgrading Software Hub](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=upgrading)

---

### 4.4 Upgrade Services

**Individual Service Upgrade (For More Control)**

#### 4.4.1 Upgrade Watsonx Orchestrate

If you plan to upgrade the previous versions of watsonx™ Orchestrate with custom upgrade options, specify the appropriate options in a file named install-options.yml in the cpd-cli work directory

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
    wxolite:
      enabled: false
    uab:
      enabled: false
    watsonxAI:
      watsonxaiifm: true
```

#### Upgrade watsonx_orchestrate
```bash
cpd-cli manage install-components \
--license_acceptance=true \
--components=watsonx_orchestrate \
--release=${VERSION} \
--patch_id=0 \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--param-file=/tmp/work/install-options.yml \
--upgrade=true
```

Monitor watsonx_orchestrate upgrade
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=watsonx_orchestrate
```

Post upgrade tasks for Watsonx Orchestrate

Login to Red Hat OpenShift cluster
```bash
$OC_LOGIN
```

Extract the current ATM server configuration from the Kubernetes secret
```bash
kubectl get secret wo-agentic-task-manager-server-env \
  -n cpd-instance-1 \
-o jsonpath='{.data.\.secret\.env}' | base64 --decode | grep SERVER_INTERNAL
```

Important: Store the value of SERVER_INTERNAL_HOSTNAME for later use, ensure that the value for SERVER_INTERNAL_PROTOCOL is set to https and SERVER_INTERNAL_PORT is set to 9045
```bash
SERVER_INTERNAL_PROTOCOL=https
SERVER_INTERNAL_HOSTNAME=wo-agentic-task-manager.cpd-instance-1.svc.cluster.local
SERVER_INTERNAL_PORT=9045
```

Download and edit the [migration script](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=u-upgrading-from-version-53-20#cli-upgrade__migration-script__title__1)
```bash
# Edit the configuration section (lines 6-7)
vi /tmp/atm_endpoint_tls_migration.sql
```

Update the configuration values to match your environment by using the following commands
```bash
SET atm_migration.old_url = 'http://<SERVER_INTERNAL_HOSTNAME>:9044';
SET atm_migration.new_url = 'https://<SERVER_INTERNAL_HOSTNAME>:9045';
```

Example configuration:
```
SET atm_migration.old_url = 'http://wo-agentic-task-manager.cpd-instance-1.svc.cluster.local:9044';
SET atm_migration.new_url = 'https://wo-agentic-task-manager.cpd-instance-1.svc.cluster.local:9045';
```

To run the migration script on PostgreSQL database
```bash
POD_NAME=$(oc get pods -l "k8s.enterprisedb.io/instanceName=wo-watson-orchestrate-postgresedb-1" -o jsonpath='{.items[0].metadata.name}')
DATABASE_NAME="archer"

oc exec -i $POD_NAME -- psql -U postgres -d $DATABASE_NAME < /tmp/atm_endpoint_tls_migration.sql
```

Run the following verification queries to validate the migration status
```bash
oc exec $POD_NAME -- psql -U postgres -d $DATABASE_NAME -c "
SELECT COUNT(*) as remaining_tools FROM tools 
WHERE binding::text LIKE '%wo-agentic-task-manager%' 
AND binding::text LIKE '%:9044%'; 
SELECT COUNT(*) as remaining_tool_versions FROM tool_version 
WHERE binding::text LIKE '%wo-agentic-task-manager%' 
AND binding::text LIKE '%:9044%';"
```

Expected results
```bash
- `remaining_tools`: 0
- `remaining_tool_versions`: 0
```

To clean up the migration log, run the following commands
```bash
oc exec $POD_NAME -- psql -U postgres -d $DATABASE_NAME -c "
-- Verify migration log entry
SELECT * FROM migration_log
WHERE migration_name = 'atm_endpoint_tls_migration_5_3_1';
"
```

Drop the migration_log table only after successful verification.
```bash
oc exec $POD_NAME -- psql -U postgres -d $DATABASE_NAME -c "
-- Drop the migration_log table 
DROP TABLE IF EXISTS migration_log;
"
```

Removing Redis CronJob - This section is applicable only if you upgrade to Version 5.3.1 Patch 2

After you upgrade to Version 5.3.1 Patch 2, you must remove the Redis Cronjob as Licensing Redis Cronjob is no longer supported

Run the following command to delete the Redis Cronjob:
```bash
oc delete cronjob wo-watson-orchestrate-redis-cronjob --ignore-not-found
```

#### 4.4.2 Upgrade Watsonx Ai

Upgrade watsonx_ai
```bash
cpd-cli manage install-components \
--license_acceptance=true \
--components=${XAI_COMPONENT_TYPE} \
--release=${VERSION} \
--patch_id=0 \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--upgrade=true
```

Monitor watsonx_ai upgrade
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=watsonx_ai
```

#### 4.4.3 Upgrade Watsonx Governance

Upgrade watsonx_governance
```bash
cpd-cli manage install-components \
--license_acceptance=true \
--components=watsonx_governance \
--release=${VERSION} \
--patch_id=0 \
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

#### 4.4.4 Upgrade Watson Speech

Before upgrading Watson Speech, confirm/update the following configurations

Increase resources for Multi cloud object gateway (Note: This was performed on Prod in the previous 5.3.0 upgrade)
```bash
oc patch -n openshift-storage storageclusters.ocs.openshift.io ocs-storagecluster --type merge --patch '{"spec": {"resources": {"noobaa-agent": {"limits": {"memory": "8Gi"},"requests": {"memory": "8Gi"}},"noobaa-core": {"limits": {"memory":"8Gi"},"requests": {"memory": "8Gi"}},"noobaa-db": {"limits": {"memory": "8Gi"},"requests": {"memory": "8Gi"}},"noobaa-endpoint": {"limits": {"memory": "8Gi"},"requests": {"memory": "8Gi"}}}}}'

oc patch -n openshift-storage backingstore noobaa-default-backing-store --type=merge --patch='{"spec":{"pvPool":{"numVolumes": 1, "resources":{"limits":{"memory": "8Gi"}}}}}'
```

For StorageCluster ocs-storagecluster, add cpu: "3" for all resources as both requests and limits, add maxCount 8 and minCount 1 as multiCloudGateway.endpoints

Patch StorageCluster ocs-storagecluster with:
```bash
oc patch storagecluster ocs-storagecluster -n openshift-storage --type merge --patch '
{
  "spec": {
    "multiCloudGateway": {
      "endpoints": {
        "maxCount": 8,
        "minCount": 1
      }
    },
    "resources": {
      "noobaa-agent": {"limits": {"cpu": "4", "memory": "8Gi"}, "requests": {"cpu": "4", "memory": "8Gi"}},
      "noobaa-core": {"limits": {"cpu": "4", "memory": "8Gi"}, "requests": {"cpu": "4", "memory": "8Gi"}},
      "noobaa-db": {"limits": {"cpu": "4", "memory": "8Gi"}, "requests": {"cpu": "4", "memory": "8Gi"}},
      "noobaa-endpoint": {"limits": {"cpu": "4", "memory": "8Gi"}, "requests": {"cpu": "4", "memory": "8Gi"}}
    }
  }
}'
```

Result should look like this:
```bash
apiVersion: ocs.openshift.io/v1
kind: StorageCluster
......
spec:
  ........
  multiCloudGateway:
    dbStorageClassName: ssd-csi
    disableLoadBalancerService: true
    endpoints:
      maxCount: 8
      minCount: 1
    reconcileStrategy: standalone
  resourceProfile: balanced
  resources:
    noobaa-agent:
      limits:
        cpu: "4"
        memory: 8Gi
      requests:
        cpu: "4"
        memory: 8Gi
    noobaa-core:
      limits:
        cpu: "4"
        memory: 8Gi
      requests:
        cpu: "4"
        memory: 8Gi
    noobaa-db:
      limits:
        cpu: "4"
        memory: 8Gi
      requests:
        cpu: "4"
        memory: 8Gi
    noobaa-endpoint:
      limits:
        cpu: "4"
        memory: 8Gi
      requests:
        cpu: "4"
        memory: 8Gi
```

For BackingStore noobaa-default-backing-store, set numVolumes to 4 and storage to 100Gi

Patch BackStore noobaa-default-backing-store with:
```bash
oc patch backingstore noobaa-default-backing-store -n openshift-storage --type merge --patch '
{
  "spec": {
    "pvPool": {
      "numVolumes": 4,
      "resources": {
        "requests": {
          "storage": "100Gi"
        }
      }
    }
  }
}'
```

Result should look like this:
```bash
apiVersion: noobaa.io/v1alpha1
kind: BackingStore
......
spec:
  pvPool:
    numVolumes: 4
    resources:
      limits:
        memory: 8Gi
      requests:
        storage: 100Gi
    secret: {}
    storageClass: ssd-csi
  type: pv-pool
```

For NooBaa noobaa, add max_connections 2400 for dbConf

Patch NooBaa noobaa with:
```bash
oc patch noobaa noobaa -n openshift-storage --type merge --patch '
{
  "spec": {
    "dbConf": "max_connections 2400\n",
    "coreResources": {
      "limits": {"cpu": "4", "memory": "8Gi"},
      "requests": {"cpu": "4", "memory": "8Gi"}
    },
    "dbResources": {
      "limits": {"cpu": "4", "memory": "8Gi"},
      "requests": {"cpu": "4", "memory": "8Gi"}
    }
  }
}'
```

Result should look like this:
```bash
apiVersion: noobaa.io/v1alpha1
kind: NooBaa
...
spec:
  ......
  coreResources:
    limits:
      cpu: "4"
      memory: 8Gi
    requests:
      cpu: "4"
      memory: 8Gi
  dbConf: |
    max_connections 2400
  dbImage: registry.redhat.io/rhel9/postgresql-15@sha256:4d707fc04f13c271b455f7b56c1fda9e232a62214ffc6213c02e41177dd4a13f
  dbResources:
```

Upgrade watson_speech
```bash
cpd-cli manage install-components \
--license_acceptance=true \
--components=watson_speech \
--release=${VERSION} \
--patch_id=0 \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--run_storage_tests=false \
--upgrade=true
```

Monitor watson_speechh upgrade
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=watson_speech
```

#### 4.4.5 Upgrade Voice Gateway

```bash
# Upgrade voice_gateway (5.3.x method)
cpd-cli manage install-components \
--license_acceptance=true \
--components=voice_gateway \
--release=${VERSION} \
--patch_id=0 \
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

#### 4.4.6 Upgrade Db2 OLTP

Upgrade db2oltp
```bash
cpd-cli manage install-components \
--license_acceptance=true \
--components=db2oltp \
--release=${VERSION} \
--patch_id=0 \
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

#### 4.4.7 Upgrade Cognos Analytics

Upgrade cognos_analytics
```bash
cpd-cli manage install-components \
--license_acceptance=true \
--components=cognos_analytics \
--release=${VERSION} \
--patch_id=0 \
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


## Service Instance Upgrades

After upgrading service CRs, some services require additional instance upgrades

### Prerequisites

1. Complete all CR upgrades successfully
2. Create a CPD profile with these permissions:
   - `can_provision` (Create service instances)
   - `manage_service_instances` (Manage service instances)
3. Set the `CPD_PROFILE_NAME` environment variable

**Documentation**: [Creating a CPD profile](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=cli-creating-cpd-profile)

---

### Upgrading Service Instances

Get a list of all service instances using the following command
```bash
cpd-cli service-instance list --profile=${CPD_PROFILE_NAME}
```

#### Upgrade Db2oltp service instances

Get the list of Db2 service instances
```bash
cpd-cli service-instance list \
--service-type=db2oltp \
--profile=${CPD_PROFILE_NAME}
```

Export the db2oltp instance name
```bash
export INSTANCE_NAME=<instance-name>
```

Run the following command to check whether your Db2 service instances is in running state:
```bash
cpd-cli service-instance status ${INSTANCE_NAME} \
--profile=${CPD_PROFILE_NAME} \
--service-type=db2oltp
```

Upgrade the service instance
```bash
cpd-cli service-instance upgrade \
--service-type=db2oltp \
--instance-name=${INSTANCE_NAME} \
--profile=${CPD_PROFILE_NAME}
```

#### Upgrade Cognos Analytics service instances

Set the INSTANCE_VERSION environment variable to the version that corresponds to the version of IBM Software Hub on your cluster
```bash
export INSTANCE_VERSION=29.1.0
```

Upgrade the service instances
```bash
cpd-cli service-instance upgrade \
--service-type=cognos-analytics-app \
--profile=${CPD_PROFILE_NAME} \
--version=${INSTANCE_VERSION} \
--all
```

#### Upgrade Openpages service instances

Get the list of OpenPages service instances
```bash
cpd-cli service-instance list \
--service-type=openpages \
--profile=${CPD_PROFILE_NAME}
```

Set the INSTANCE_NAME environment variable to the name of the service instance that you want to upgrade
```bash
export INSTANCE_NAME=<instance-name>
```

Run the following command to check whether your OpenPages service instances is in running state
```bash
cpd-cli service-instance status ${INSTANCE_NAME} \ 
--profile=${CPD_PROFILE_NAME} \ 
--service-type=openpages
```

Upgrade the service instance
```bash
cpd-cli service-instance upgrade \
--service-type=openpages \
--instance-name=${INSTANCE_NAME} \
--force-version-upgrade=true \
--profile=${CPD_PROFILE_NAME}
```

Repeat the preceding steps to upgrade each service instance associated with this instance of IBM Software Hub

---

## Upgrade cpdbr Service

You must upgrade the cpdbr service after you upgrade IBM Software Hub.

**Reference**: [Updating the cpdbr service](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=uish-updating-cpdbr-service-1)

### For Environments With Scheduling Service
```bash
cpd-cli oadp install \
--component=cpdbr-tenant \
--cpdbr-hooks-image-prefix=${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd \
--cpfs-image-prefix=${PRIVATE_REGISTRY_LOCATION}/cpopen/cpfs \
--namespace=${OADP_OPERATOR_NS} \
--tenant-operator-namespace=${PROJECT_CPD_INST_OPERATORS} \
--cpd-scheduler-namespace=${PROJECT_SCHEDULING_SERVICE} \
--skip-recipes=true \
--upgrade=true \
--log-level=debug \
--verbose
```

### For Environments Without Scheduling Service
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

**Note**: This upgrade should be performed after all services have been upgraded.

---

## Post Upgrade Validation

### Post Upgrade Validation

```bash
# Check CR status
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}

# Check for pods not running correctly (excludes completed jobs)
oc get po -A -owide | egrep -v '([0-9])/\1' | egrep -v 'Completed'

# Verify CPD version
oc get ZenService lite-cr -n ${PROJECT_CPD_INST_OPERANDS} -o jsonpath='{.status.zenStatus.versions[0].version}'

# List service instances
cpd-cli service-instance list --profile=${CPD_PROFILE_NAME}

---

**End of Runbook**
