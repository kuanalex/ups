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
- Troubleshooting References

---

## Prerequisites

Backup of the cluster is complete

Backup your IBM Software Hub cluster before the upgrade

Reference: [Backing up and restoring IBM Software Hub](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=administering-backing-up-restoring-software-hub)

**Note**: Some services don't support the offline OADP backup. Review the backup documentation and take the dedicated approach when necessary

The image mirroring completed successfully

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
- **IBM Software Hub CLI**: Version 14.3.1.5
- **OpenShift CLI (oc)**: Compatible version for your cluster
- **Helm CLI**: Version 4.1.4

**Installation and Update Instructions:**

For detailed instructions on installing or updating these tools, refer to:
- [Updating client workstations](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=53-updating-client-workstations)
- [Updating IBM Software Hub CLI](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=ucw-updating-software-hub-cli-1)
- [Updating OpenShift CLI](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=ucw-updating-openshift-cli-1)
- [Installing Helm CLI](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=ucw-installing-helm-cli-1)

Access Requirements

**Required Access:**
- OpenShift cluster admin access
- IBM Entitlement Key with appropriate permissions
- Access to IBM Container Registry (cp.icr.io)
- Access to private registry: UPDATE_WITH_PRIVATE_REGISTRY_URL

Environment Variables Setup (cpd_vars.sh)

Ensure your environment variables script is configured correctly

**Reference**: [Updating your environment variables script](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=cri-updating-your-environment-variables-script-1)

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

List all of the temporay patches in the operands namespace
```bash
oc get temporarypatch -n ${PROJECT_CPD_INST_OPERANDS}
```

For all patches that you want to retain, use the following command:
```bash
oc label temporarypatch <patch_name> type=critical-configuration
```

For example
```bash
oc label temporarypatch wa-store-assistant-limits type=critical-configuration
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
| Node Feature Discovery | 4.17.0 | 4.18.x | Upgrade required to match OCP/ODS version. |
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

#### Potential Issue #1 - Kafka controller/broker pods in OOMKilled 

Before proceeding with the Events Operator upgrade, make sure the kafka controller and broker pods are healthy
```bash
oc get po -n knative-eventing | grep -E 'eventing-kafka-broker|eventing-kafka-controller'
```

For example
```bash
knative-eventing-kafka-knative-eventing-kafka-broker-0       1/1     Running     0              16d
knative-eventing-kafka-knative-eventing-kafka-broker-1       1/1     Running     0              16d
knative-eventing-kafka-knative-eventing-kafka-broker-2       1/1     Running     0              16d
knative-eventing-kafka-knative-eventing-kafka-controller-3   1/1     Running     0              16d
knative-eventing-kafka-knative-eventing-kafka-controller-4   1/1     Running     0              16d
knative-eventing-kafka-knative-eventing-kafka-controller-5   1/1     Running     0              16d
```

If the kafka controller and broker pods are in CrashLoopBackOff status, check the pod logs for OOMKilled status, and if required, increase the memory via the KafkaNodePool

For the broker KafkaNodePool
```bash
oc patch kafkanodepool <wo-wa-1234-ibm-abcd-broker> -n cpd-instance --type=merge -p '{"spec":{"resources":{"limits":{"memory":"8Gi"},"requests":{"memory":"8Gi"}}}}'
```

For the controller KafkaNodePool
```bash
oc patch kafkanodepool <wo-wa-1234-ibm-abcd-controller> -n cpd-instance --type=merge -p '{"spec":{"resources":{"limits":{"memory":"1Gi"},"requests":{"memory":"1Gi"}}}}'
```

Once these pods are stable, proceed with upgrading the Events operator, and continue to monitor for memory issues

Log the cpd-cli in to the Red Hat OpenShift Container Platform cluster
```bash
${CPDM_OC_LOGIN}
```

Download case packages for ibm_events_operator
```bash
cpd-cli manage case-download \
--release=${VERSION} \
--patch_id=0 \
--components=ibm_events_operator \
--from_oci=true
```

Generate cluster-scoped resource definitions for CPD instance
```bash
cpd-cli manage deploy-events-operator \
--release=${VERSION} \
--cluster_resources=true
```

Run the 'oc apply -f' command returned in the terminal, for example
```bash
oc apply -f /root/cpd-cli-workspace/olm-utils-workspace/work/cluster_scoped_resources.yaml
```

Run the following command to upgrade the IBM Events Operator
```bash
cpd-cli manage deploy-events-operator \
--release=${VERSION} \
--events_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--events_operand_ns=${PROJECT_CPD_INST_OPERANDS}
```

#### Potential Issue #2 - Kafka CR TLS Issue 

After initiating the upgrade of IBM Events Operator, monitor the Events operator logs and kafka CR for any of the following types of messages
```bash
SSL handshake failed
PKIX path validation failed
Path does not chain with any of the trust anchors
```

The TLS issue is caused by an interrupted certificate reload 

To address this, restart the Events operator pod followed by the controller pods and the broker pods

**Note**: Make sure to restart the pods one at a time to avoid taking down the cluster all at once

Identify and restart the Events operator pod
```bash
oc get po -n ups-wx-operators | grep events
```

Restart the Events operator pod
```bash
oc delete po ibm-events-cluster-operator-84bdb5dcf8-rsqlj -n ups-wx-operators
```

Identify the kafka broker and controller pods
```bash
oc get po -n knative-eventing | grep -E 'eventing-kafka-broker|eventing-kafka-controller'
```

You should see an output similar to this
```bash
knative-eventing-kafka-knative-eventing-kafka-broker-0       1/1     Running     0
knative-eventing-kafka-knative-eventing-kafka-broker-1       1/1     Running     0
knative-eventing-kafka-knative-eventing-kafka-broker-2       1/1     Running     0
knative-eventing-kafka-knative-eventing-kafka-controller-3   1/1     Running     0
knative-eventing-kafka-knative-eventing-kafka-controller-4   1/1     Running     0
knative-eventing-kafka-knative-eventing-kafka-controller-5   1/1     Running     0
```

Restart the kafka controller pods one at a time, and ensure the pod goes back into running status before restarting the next pod
```bash
oc delete po <kafka-controller-pod-name> -n knative-eventing 
```

Restart the kafka broker pods one at a time, and ensure the pod goes back into running status before restarting the next pod
```bash
oc delete po <kafka-broker-pod-name> -n knative-eventing 
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

Creating image pull secrets for the instance  

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

#### Upgrade CPD platform using install-components
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

## Upgrade Services

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

Remove the image_digests section from Watsonxaiifm 
```bash
oc patch watsonxaiifm watsonxaiifm-cr -n ${PROJECT_CPD_INST_OPERANDS} --type=json -p='[{"op": "remove", "path": "/spec/image_digests"}]'
```

In the WatsonxOrchestrate CR, there are two image digestOverrides which will need to be removed
```bash
spec:
    image:
      digestOverrides:
        wxo-server-conversation_controller: sha256:e481eae84d5a412b25ebc4b6e982face5899503b839f69c7f7cae6c05df424de
        wxo-server-server: sha256:d75246e301a1e4bd62fae7f90af32ace1344c50b5e5c710b1a5e4873e0cdbd24
```

Remove the digestOverrides section from WatsonxOrchestrate manually or by using the following patch command, which should remove the /spec/image/digestOverrides section of the yaml
```bash
oc patch watsonxorchestrate wo -n ${PROJECT_CPD_INST_OPERANDS} --type=json -p='[{"op": "remove", "path": "/spec/image/digestOverrides"}]'
```

Upgrade watsonx_orchestrate
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

**Note**: During non-prod upgrade an issue was encountered during Watsonx Orchestrate upgrade with Watson Assistant

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
oc patch temporarypatch wo-wa-data-governor-opensearch-ephemeral \
  -n ups-wx-operands \
  --type=merge \
  -p '{"spec":{"patch":{"data-governor":{"datagovernoroverride":{"spec":{"dependencies":{"opensearch":{"ephemeralDeployment":"true"}}}}}}}}'
```

Validate the patch updates
```bash
oc get patch wo-wa-data-governor-opensearch-ephemeral -o yaml
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
  -n ${PROJECT_CPD_INST_OPERANDS} \
-o jsonpath='{.data.\.secret\.env}' | base64 --decode | grep SERVER_INTERNAL
```

Important: Store the value of SERVER_INTERNAL_HOSTNAME for later use, ensure that the value for SERVER_INTERNAL_PROTOCOL is set to https and SERVER_INTERNAL_PORT is set to 9045
```bash
SERVER_INTERNAL_PROTOCOL=https
SERVER_INTERNAL_HOSTNAME=wo-agentic-task-manager.cpd-instance.svc.cluster.local
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
SET atm_migration.old_url = 'http://wo-agentic-task-manager.cpd-instance.svc.cluster.local:9044';
SET atm_migration.new_url = 'https://wo-agentic-task-manager.cpd-instance.svc.cluster.local:9045';
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

After completing this migration, follow the steps to apply IBM watsonx Orchestrate release 5.3.1 Hotfix 4

**Reference**: [Apply hot fix for IBM watsonx Orchestrate](https://www.ibm.com/support/pages/node/7247038)

**Reference**: [Applying the watsonx Orchestrate 5.3.1 hotfix (Hotfix 4)
](https://www.ibm.com/support/pages/applying-watsonx-orchestrate-531-hotfix-hotfix-4)

Set the operator and operand namespaces
```bash
export PROJECT_CPD_INST_OPERATORS=ups-wx-operators
export PROJECT_CPD_INST_OPERANDS=ups-wx-operands
```

**Note**: You will need to install Skopeo and mirror the operator and operand images before proceeding

Create hotfix5314.sh 
```bash
vi hotfix5314.sh
```

With the following contents
```bash
#!/bin/bash
# -----------------------------------------------------------------------------
# watsonx Orchestrate 5.3.1 Hotfix
# - Verifies watsonx Orchestrate version from .status.versionStatus.status.
# - Images of Operators are replaced with the image from hotfix script.
#   * HOTFIX_LABEL_VALUE for hotfix 3 is 5.3.1.4
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
echo "[$(ts)] 5.3.1 Hotfix4 steps completed."
echo "Backups saved under ${BACKUP_DIR}"
echo "Monitor the watsonx Orchestrate CR status by running:"
echo " oc get wo -n ${PROJECT_CPD_INST_OPERANDS} -o yaml | grep -E 'watsonxOrchestrateStatus|${HOTFIX_LABEL_KEY}'"
echo "Ensure the watsonx Orchestrate CR status is 'Completed' and label ${HOTFIX_LABEL_KEY}=${HOTFIX_LABEL_VALUE} is present."
echo "It will take another 15–20 minutes for the updated components to be applied and restarted."
echo "------------------------------------------------------------------"
```

Make the script executable
```bash
chmod 775 hotfix5314.sh
```
 
Run the script
```bash
nohup sh hotfix5314.sh &
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
hotfix: 5.3.1.4
```

Confirm the completion of the hotfix by check the Watsonx Orchestrate CR status
```bash
oc get wo
```

The expected output
```bash
NAME   VERSION   DEPLOYED   VERIFIED   TOTAL   INSTALLMODE         QUIESCE        RECONCILE_PROGRESS   AGE
wo     5.3.1     34         34         34      agentic_assistant   NOT_QUIESCED   100%                 9d
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

Before proceeding with the upgrade, save your current Watson Speech custom resource
```bash
oc get WatsonSpeech speech-cr -n ${PROJECT_CPD_INST_OPERANDS} -o yaml > watson-speech-cr-backup.yaml
```

**Important Note:** During the upgrade process, any custom configurations in the Watson Speech custom resource that are not managed by the cpd-cli (Helm) upgrade process may be lost

After the upgrade completes, review your saved custom resource backup (`watson-speech-cr-backup.yaml`) and manually re-apply any custom configurations that were not preserved

Leave out the `watsonSpeech` section in the values.yaml to preserve your custom resource spec during the upgrade. Example install-options.yaml file
```yaml
non_olm:
  global:
    blockStorageClass: <ocs-storagecluster-ceph-rbd>
    fileStorageClass: <ocs-storagecluster-cephfs>
```


Example cpd-cli install-components upgrade command
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
--param-file=/tmp/work/install-options.yml \
--upgrade=true
```

Monitor watson_speech upgrade, wait until the status shows `Completed` or `Ready`.
```bash
oc get WatsonSpeech speech-cr -n ${PROJECT_CPD_INST_OPERANDS} -w
```

Check the cr status
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=watson_speech
```

After the upgrade completes, compare your current custom resource with the backup you created earlier

View the backup
```bash
cat watson-speech-cr-backup.yaml
```

View the current CR
```bash
oc get WatsonSpeech speech-cr -n ${PROJECT_CPD_INST_OPERANDS} -o yaml
```

If any custom configurations were lost during the upgrade (such as custom resource request/limits/replicas, GW HPA, or other manual modifications), re-apply them by editing the custom resource
```bash
oc edit WatsonSpeech speech-cr -n ${PROJECT_CPD_INST_OPERANDS}
```

After the upgrade completes, apply the image overrides to use the new chuck runtime images and updated models
```bash
oc edit WatsonSpeech speech-cr -n ${PROJECT_CPD_INST_OPERANDS}
```

Add or update the following sections under `spec`
```yaml
spec:
  images:
    am_patcher_chuck:
      digest: sha256:2a24058a667b1f9026d49bda2e0ddee8f0f88f4e9760eaf35961c854525eb718
    stt_runtime_chuck:
      digest: sha256:2a24058a667b1f9026d49bda2e0ddee8f0f88f4e9760eaf35961c854525eb718
    tts_runtime_chuck:
      digest: sha256:2a24058a667b1f9026d49bda2e0ddee8f0f88f4e9760eaf35961c854525eb718
  global:
    genericModels:
      image: generic-models@sha256:2dbbc1f1f8a9860765a60ba27ea181761196f776044b10388001befaf3b0046e
    sttModels:
      deDe:
        digest: sha256:17eeb6230026792e6cb9471ab4cb48fae34cef6202539ecefd30bc5e0f080e6c
        enabled: true
      frFrTelephonyLSM:
        digest: sha256:35b79279ecb9d695a5c8c69e5bd8e172e040e87f4ed8702cdd5090f84be39fa4
        enabled: true
      nlNlTelephony:
        digest: sha256:940c93833760e802044b6c16e83d83696d21a464c56f723dfe68ba4d9986abc3
        enabled: true
```

**Note:** Only include the models you want to enable. Set `enabled: true` for models you wish to use

Save and exit the editor

The Speech operator will reconcile the changes and update the deployments

After applying the image overrides, the Speech operator will reconcile the custom resource and update the components

Monitor the Speech operator reconciliation
```bash
oc get WatsonSpeech speech-cr -n ${PROJECT_CPD_INST_OPERANDS} -w
```

Check that the new models are being uploaded. A new `speech-cr-stt-models` job will spawn to upload the updated models to object storage
```bash
oc get pods -n ${PROJECT_CPD_INST_OPERANDS} -w | grep speech-cr-stt-models
```

After the models are uploaded, verify that the new STT/TTS runtime pods are rolled out with newer chuck image
```bash
oc get pods -n ${PROJECT_CPD_INST_OPERANDS} -w | grep -E "speech-cr-(stt|tts)-runtime"
```

#### Upgrade Voice Gateway
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

Prerequisites

1. Complete all CR upgrades successfully
2. Create a CPD profile with these permissions:
   - `can_provision` (Create service instances)
   - `manage_service_instances` (Manage service instances)
3. Set the `CPD_PROFILE_NAME` environment variable

**Documentation**: [Creating a CPD profile](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=cli-creating-cpd-profile)

---

Upgrading Service Instances

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

Identify the caserviceinstance name
```bash
oc get caserviceinstance
```

Check the caserviceinstance-cr for hitfix_digests
```bash
oc patch caserviceinstance ca1770175442273745-cr -o yaml | grep -A 10 hotfix_digests
```

Remove any hotfix_digests in the caserviceinstances-cr with a patch command
```bash
oc patch caserviceinstance ca1770175442273745-cr -n ups-wx-operands --type=json -p=[{"op": "remove", "path": "/spec/hotfix_digests"}]
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

**Note**: This upgrade should be performed after all services have been upgraded

**Note2**: If you encounter an imagepullbackoff issue for the cpdbr tenant service pod(s) this might be caused by your cpd-cli version. The cpd-cli utility version (BUILD_ID: 3.3.1.x) dynamically appends its own build suffix to backup image queries, causing the cluster to look for non-existent tag 5.3.1.x in the air-gapped registry instead of the mirrored 5.3.1 GA baseline; downgrading to cpd-cli version 14.3.0 can force it to query the correct GA tag

**Note3**: The workaround used during UPS non prod upgrade was to tag the image in the private registry with the 5.3.1.5 tag

---

## Post Upgrade Validation

#### Enable WxO Observability

Follow the procedure in the following document to enable WxO Observability

**IMPORTANT**: [Enable WxO Observability](https://github.com/kuanalex/ups/blob/main/WxO_Observability_PROD_Runbook.md)

---

#### Fix platform-auth-service pod in ContainerStatusUnknown

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

## Troubleshooting References
- [Known issues for watsonx Orchestrate](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=issues-watsonx-orchestrate)
- [Known issues and limitations for Common core services](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=issues-common-core-services)
- [Known issues and limitations for IBM watsonx.ai](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=issues-watsonxai)
- [Known issues and limitations for watsonx.governance](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=issues-watsonxgovernance)
- [Known issues and limitations for Watson Speech services](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=issues-watson-speech-services)
- [Known issues and limitations for Db2 and Db2 Warehouse](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=issues-db2-db2-warehouse)
- [Known issues and limitations in Cognos Analytics](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=issues-cognos-analytics)

**End of Runbook**
