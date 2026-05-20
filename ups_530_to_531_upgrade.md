# UPS Production Cluster CP4D 5.3.0 to 5.3.1 Upgrade

## Author: Alex Kuan (alex.kuan@ibm.com)

**From:**
```
CPD: 5.3.0
OCP: 4.18.40
Storage: Google Cloud Netapp Volumes and Persistent Disk on Google Cloud
Internet: airgap
Private container registry: yes
Components: ibm-licensing,ibm_events_operator,ccs,cpfs,zen,cpd_platform,watsonx_orchestrate,watsonx_ai,watsonx_governance,watson_speech,voice_gateway,db2oltp,cognos_analytics
```

**To:**
```
CPD: 5.3.1
OCP: 4.18.40
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

#### Backup of the cluster is complete

Backup your IBM Software Hub cluster before the upgrade

Reference: [Backing up and restoring IBM Software Hub](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=administering-backing-up-restoring-software-hub).

**Note**: Some services don't support the offline OADP backup. Review the backup documentation and take the dedicated approach when necessary

#### The image mirroring completed successfully

If a private container registry is in-use to host the IBM Software Hub software images, you must mirror the updated images from the IBM® Entitled Registry to the private container registry

Reference: [Mirroring images to private image registry](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=mipcr-mirroring-images-directly-private-container-registry-1)

**Note**: Since the upgrade path is to 5.3.1.0 or 5.3.1 GA, we will need to ensure we specify --patch_id=0 in the following commands
```bash
case-download 
list-images
mirror-images
list-patch
apply-patch
apply-cluster-components
apply-scheduler
apply-components
install-components
```

Here is an example of the case-download syntax
```bash
cpd-cli manage case-download \
--components=${COMPONENTS} \
--release=${VERSION} \
--patch_id=0 \
--from_oci=true
```

Here is an example of the mirror-images syntax
```bash
cpd-cli manage mirror-images \
--components=${COMPONENTS} \
--groups=${IMAGE_GROUPS} \
--release=${VERSION} \
--patch_id=0 \
--target_registry=${PRIVATE_REGISTRY_LOCATION} \
--arch=${IMAGE_ARCH} \
--case_download=false
```

#### The permissions required for the upgrade is ready
- OpenShift cluster administrator permissions
- IBM Software Hub administrator permissions
- Permission to access the private image registry for pushing or pulling images
- Access to the bastion node for executing the upgrade commands

#### A pre-upgrade health check is made to ensure the cluster's readiness for upgrade

- The OpenShift cluster, persistent storage and IBM Software Hub platform and services are in healthy status

---

## Pre Upgrade Steps

#### Required Tools

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

#### Access Requirements

**Required Access:**
- OpenShift cluster admin access
- IBM Entitlement Key with appropriate permissions
- Access to IBM Container Registry (cp.icr.io)
- Access to private registry: UPDATE_WITH_PRIVATE_REGISTRY_URL

#### Environment Variables Setup (cpd_vars.sh)

Ensure your environment variables script is configured correctly

**Reference**: [Updating your environment variables script](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=cri-updating-your-environment-variables-script-1).

Source your environment variables script and verify key variables are set
```bash
source cpd_vars.sh
echo "CPD Version: ${VERSION}"
echo "OCP URL: ${OCP_URL}"
echo "Operators Namespace: ${PROJECT_CPD_INST_OPERATORS}"
echo "Operands Namespace: ${PROJECT_CPD_INST_OPERANDS}"
echo "Block Storage: ${STG_CLASS_BLOCK}"
echo "File Storage: ${STG_CLASS_FILE}"
echo "Components: ${COMPONENTS}"
echo "Private Registry: ${PRIVATE_REGISTRY_LOCATION}"
```

Login using cpd-cli
```bash
${CPDM_OC_LOGIN}
```

Or login using oc directly
```bash
${OC_LOGIN}
```

Verify cluster access
```bash
oc whoami
oc get nodes
```

Restart the OLM utils container with updated environment
```bash
cpd-cli manage restart-container
```

Verify container is running
```bash
podman ps | grep olm-utils
```

Take a backup of the routes
```bash
oc get routes -n ${PROJECT_CPD_INST_OPERANDS} -o yaml > routes_backup_$(date +%Y%m%d_%H%M%S).yaml
```

Take a backup of the temporary patches for watson assistant
```bash
oc get TemporaryPatch -n ${PROJECT_CPD_INST_OPERANDS} -o yaml > temporarypatch_backup_$(date +%Y%m%d_%H%M%S).yaml
```

#### Air Gapped Environment Prerequisites

1. **[Obtain OLM Utils v4 image](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=pruirn-obtaining-olm-utils-v4-image-2)**
2. **[Download CASE packages](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=pruirn-downloading-case-packages-2)** for all components
3. **Mirror images** to private registry ([direct](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=mipcr-mirroring-images-directly-private-container-registry-2) or [intermediary](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=mipcr-mirroring-images-using-intermediary-container-registry-2))
4. **[Pull OLM Utils](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=prufpcr-pulling-olm-utils-v4-image-from-private-container-registry-2)** from private registry

#### Advanced Service Prerequisites

Some services require additional prerequisite software upgrades. Review [IBM Documentation](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=pyc-upgrading-prerequisite-software-1) for details

**Multicloud Object Gateway (MCG)** - Required for: Watson Speech, Voice Gateway, Watsonx Ai
[Upgrade MCG](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=ups-upgrading-multicloud-object-gateway-1) during storage or OCP upgrade

**Red Hat OpenShift Serverless Knative** - Required for: Watsonx Orchestrate
[Upgrade Knative](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=ups-upgrading-red-hat-openshift-serverless-knative-eventing-1) to supported version

**GPU Operators** - [Upgrade if needed](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=ups-upgrading-operators-services-that-require-gpus-1) for GPU-enabled services

**Red Hat OpenShift AI** - [Review upgrade requirements](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=ups-upgrading-red-hat-openshift-ai-1) if using OpenShift AI

Based on the health check review, we've determined that some of these operators will need to be upgraded as follows

| Operator | Current CSV | Target for 5.3.1 | Action |
| --- | --- | --- | --- |
| OpenShift Serverless | 1.37.1 | 1.37.x | No action required. |
| OpenShift AI (RHOAI) | 2.21.1 | 2.25.1 | Upgrade required. |
| NVIDIA GPU Operator | 25.3.3 | 26.3.x | Upgrade required. |
| Node Feature Discovery | 4.17.0 | 4.18.x | Upgrade required to match <br>OCP/ODS version. |
| IBM Events Operator | 5.2.1 | 6.0.0 | Upgrade required. |

---

## Pre Upgrade Health Check

#### Run Comprehensive Health Check

Run CPD health check (includes cluster, nodes, operands, operators)
```bash
cpd-cli health runcommand \
--commands=cluster,nodes,operands,operators \
--control_plane_ns="${PROJECT_CPD_INST_OPERANDS}" \
--operator_ns="${PROJECT_CPD_INST_OPERATORS}" \
--log-level=debug \
--verbose \
--save
```

#### Check for Hot Fixes and Patches

Check for image_digests in service CRs (indicates hot fixes/patches)
```bash
oc project ${PROJECT_CPD_INST_OPERANDS}

for i in $(oc api-resources | grep cpd.ibm.com | awk '{print $1}' | grep -v zenextensions); do
  echo "************* $i *************"
  for x in $(oc get $i --no-headers | awk '{print $1}'); do
    echo "--------- $x ------------"
    oc get $i $x -o jsonpath={.spec} | jq
  done
done
```

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

Check for pods not running correctly (excludes completed jobs)
```bash
oc get po -A -owide | egrep -v '([0-9])/\1' | egrep -v 'Completed'
```

List service instances
```bash
cpd-cli manage list-deployed-components --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

**Note: Fix any pod issues and ensure the service CRs are in Completed status before proceeding with the upgrade**

---

## Upgrade Shared Cluster Components

#### Upgrade IBM Licensing

**Reference**: [Upgrading shared cluster components](https://www.ibm.com/docs/en/cloud-paks/cp-data/5.3.x?topic=upgrading-shared-cluster-components)

Upgrade IBM Licensing service
```bash
cpd-cli manage apply-cluster-components \
--release=${VERSION} \
--patch_id=0 \
--license_acceptance=true \
--licensing_ns=${PROJECT_LICENSE_SERVICE}
```

Verify licensing pods are running
```bash
oc get pods -n ${PROJECT_LICENSE_SERVICE}
```

#### Update Cluster-Scoped Resources for CPD Instance

**Reference**: [Updating cluster-scoped resources for the instance](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=puish-updating-cluster-scoped-resources-instance-1)

Generate cluster-scoped resource definitions for CPD instance
```bash
cpd-cli manage case-download \
--components=${COMPONENTS} \
--release=${VERSION} \
--patch_id=0 \ 
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--cluster_resources=true
```

Run the 'oc apply -f' command returned in the terminal, for example
```bash
oc apply -f /root/cpd-cli-workspace/olm-utils-workspace/work/cluster_scoped_resources.yaml
```

#### Upgrading the IBM Events Operator

**Reference**: [Upgrading the IBM Events Operator for watsonx Assistant or watsonx Orchestrate](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=puish-upgrading-events-operator-1)

Log the cpd-cli in to the Red Hat OpenShift Container Platform cluster
```bash
${CPDM_OC_LOGIN}
```

Run the following command to upgrade the IBM Events Operator
```bash
cpd-cli manage deploy-events-operator \
--release=${VERSION} \
--events_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--events_operand_ns=${PROJECT_CPD_INST_OPERANDS}
```

#### Apply Entitlements

**Reference**: [Applying your entitlements](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=aye-applying-your-entitlements-without-node-pinning-2)

```bash
# Apply IBM Cloud Pak for Data Enterprise Edition entitlement
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=cpd-enterprise \
--production=false

# Apply watsonx.ai license
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=watsonx-ai \
--production=false

# Apply watsonx.governance licenses
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=watsonx-gov-mm \
--production=false

cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=watsonx-gov-rc \
--production=false

# Apply watsonx Orchestrate license
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=watsonx-orchestrate \
--production=false

# Apply Watson Speech licenses, skip for non prod
# cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=speech-to-text

# Skip for non prod
# cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=text-to-speech

# Apply Cognos Analytics license, skip for non prod
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=cognos-analytics
```

---

#### Creating image pull secrets for the instance  

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
--namespace=${PROJECT_CPD_INST_OPERANDS}
```

---

## Upgrade IBM Software Hub Platform and Services

**Reference**: [Upgrading IBM Software Hub](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=upgrading)

Remove the entire spec/image_digests from ZenService lite-cr before proceeding
```bash
oc patch zenservice lite-cr -n ${PROJECT_CPD_INST_OPERANDS} --type=json -p='[{"op": "remove", "path": "/spec/image_digests"}]'
```

Upgrade CPD platform using install-components
```bash
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

#### Reapply RSI Patches

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

**Reference**: [IBM Documentation - Upgrading Software Hub](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=uish-upgrading-software-hub-1)

---

### Upgrade Services

**Individual Service Upgrade (For More Control)**

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

Remove the image_digests sectionfrom Watsonxaiifm 
```bash
oc patch watsonxaiifm watsonxaiifm-cr -n ${PROJECT_CPD_INST_OPERANDS} --type=json -p='[{"op": "remove", "path": "/spec/image_digests"}]'
```

Remove the digestOverrides section from WatsonxOrchestrate
```bash
oc patch watsonxorchestrate wo -n ${PROJECT_CPD_INST_OPERANDS} --type=json -p='[{"op": "remove", "path": "/spec/image/digestOverrides"}]'
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

#### Apply WxO Hotfix 4

After completing this migration, follow the steps to apply IBM watsonx Orchestrate release 5.3.1 Hotfix 4

**Reference**: [Apply hot fix for IBM watsonx Orchestrate](https://www.ibm.com/support/pages/node/7247038)

Steps to be included upon 5.3.1 Hotfix 4 release
```bash
TBD
```

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

#### Upgrade Watson Speech

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

Monitor watson_speech upgrade
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=watson_speech
```

#### Upgrade Voice Gateway

Upgrade voice_gateway
```bash
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

#### Upgrade Db2 OLTP

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

#### Upgrade Cognos Analytics

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


## Upgrade Service Instances

After upgrading service CRs, some services require additional instance upgrades

#### Prerequisites

1. Complete all CR upgrades successfully
2. Create a CPD profile with these permissions:
   - `can_provision` (Create service instances)
   - `manage_service_instances` (Manage service instances)
3. Set the `CPD_PROFILE_NAME` environment variable

**Documentation**: [Creating a CPD profile](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=cli-creating-cpd-profile)

---

#### Upgrading Service Instances

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

## Upgrade Cpdbr Service

You must upgrade the cpdbr service after you upgrade IBM Software Hub.

**Reference**: [Updating the cpdbr service](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=uish-updating-cpdbr-service-1)

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

**Note**: This upgrade should be performed after all services have been upgraded.

---

## Post Upgrade Validation

Check CR status
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

Check for pods not running correctly
```bash
oc get po -A -owide | egrep -v '([0-9])/\1' | egrep -v 'Completed'
```

Verify CPD version
```bash
oc get ZenService lite-cr -n ${PROJECT_CPD_INST_OPERANDS} -o jsonpath='{.status.zenStatus.versions[0].version}'
```

List service instances
```bash
cpd-cli service-instance list --profile=${CPD_PROFILE_NAME}
```

**End of Runbook**