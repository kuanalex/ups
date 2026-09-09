# UPS Production Cluster CP4D 5.3.1.0 to 5.4.0.5 Upgrade
## Author: Alex Kuan (alex.kuan@ibm.com)

**From:**
```
CPD: 5.3.1.0
OCP: 4.20.25
Storage: Google Cloud Netapp Volumes and Persistent Disk on Google Cloud
Internet: Air-gapped
Private container registry: Yes
Components: ibm-licensing,ibm_events_operator,ccs,cpfs,zen,cpd_platform,watsonx_orchestrate,watsonx_ai,watsonx_governance,watson_speech,voice_gateway,analyticsengine,db2oltp,cognos_analytics
```

**To:**
```
CPD: 5.4.0.5
OCP: 4.20.25
Storage: Google Cloud Netapp Volumes and Persistent Disk on Google Cloud
Internet: Air-gapped
Private container registry: Yes
Components: ibm-licensing,ibm_events_operator,ccs,cpfs,zen,cpd_platform,watsonx_orchestrate,watsonx_ai,watsonx_governance,watson_speech,voice_gateway,analyticsengine,db2oltp,cognos_analytics
```

---

## Prerequisites

Backup of the cluster is complete

Backup your IBM Software Hub cluster before the upgrade [Backing up and restoring IBM Software Hub](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=administering-backing-up-restoring-software-hub)

The latest olm-utils-v4 image is available [Obtaining the olm-utils-v4 image before running IBM Software Hub installation](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=pruirn-obtaining-olm-utils-v4-image)

Case packages are downloaded on the workstation [Downloading CASE packages before running IBM Software Hub upgrade](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=pruirn-downloading-case-packages)

The image mirroring completed successfully [Mirroring images to private image registry](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=mipcr-mirroring-images-directly-private-container-registry)

Here is an example of the case-download syntax
```bash
cpd-cli manage case-download --components=${COMPONENTS} --release=${VERSION} --patch_id=${PATCH_ID} --from_oci=true
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
export PROJECT_FUSION=ibm-spectrum-fusion-ns
export OADP_PROJECT=ibm-backup-restore
export PROJECT_INST_BR_SVC=${PROJECT_CPD_INST_OPERATORS}-br-svc
export BR_OPERATOR_JOB_SA=bros-job-sa
export BR_OPERATOR_SA=bros-sa
```

Source the environment variables
```bash
source cpd_vars.sh
```

---

## Pre Upgrade Steps

**Required Tools**:

Ensure the following tools are installed and updated to the required versions
- IBM Software Hub CLI: Version 14.4.0.5
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

Validate 'expose:external-regional' label in the cpd route, add the label "expose:external-regional" to your cpd-route as required
```bash
oc get route cpd -n ${PROJECT_CPD_INST_OPERANDS} -o yaml | grep -A 20 labels
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

Action should only be required for OpenShift Serverless, IBM Events Operator, OpenShift Logging (for ODS)

| Operator | Current CSV | Target for 5.4.0 | Action |
| --- | --- | --- | --- |
| OpenShift AI (RHOAI) | 2.25.9 | 2.25.x | No action required |
| NVIDIA GPU Operator | 26.3.x | 26.3.x | No action required |
| Node Feature Discovery | 4.20.0 | 4.20.x | No action required |
| OpenShift Serverless | 1.37.1 | 1.37.1 | Generate the required custom resource definitions for the IBM Events Operator |
| IBM Events Operator | 6.0.0 | 6.1.1 | Upgrade required |
| OpenShift Logging | 6.3.4 | 6.3.x | Upgrade required for ODS |

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

Generate cluster-scoped resources for the br_orchestration service
```bash
cpd-cli manage case-download --components=br_orchestration --release=${VERSION} --patch_id=${PATCH_ID} --operator_ns=${PROJECT_INST_BR_SVC} --br_operator_ns=${PROJECT_INST_BR_SVC} --cluster_resources=true
```

Run the 'oc apply -f' command returned in the terminal, for example
```bash
oc apply -f cluster_scoped_resources.yaml --server-side --force-conflicts
```

---

Generate cluster-scoped resources for the instance
```bash
cpd-cli manage case-download --components=${COMPONENTS} --release=${VERSION} --patch_id=${PATCH_ID} --operator_ns=${PROJECT_CPD_INST_OPERATORS} --cluster_resources=true
```

Run the 'oc apply -f' command returned in the terminal, for example
```bash
oc apply -f cluster_scoped_resources.yaml --server-side --force-conflicts
```

---

#### Upgrading the IBM Events Operator

**Reference**: [Upgrading the IBM Events Operator for watsonx Assistant or watsonx Orchestrate](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=puish-upgrading-events-operator)

Before proceeding with the Events Operator upgrade, make sure the kafka controller and broker pods are healthy
```bash
oc get po -n knative-eventing | grep -E 'eventing-kafka-broker|eventing-kafka-controller'
```

For example
```bash
knative-eventing-kafka-knative-eventing-kafka-broker-0       1/1     Running     0                4d14h
knative-eventing-kafka-knative-eventing-kafka-broker-1       1/1     Running     0                4d14h
knative-eventing-kafka-knative-eventing-kafka-broker-2       1/1     Running     0                4d14h
knative-eventing-kafka-knative-eventing-kafka-controller-3   1/1     Running     0                4d14h
knative-eventing-kafka-knative-eventing-kafka-controller-4   1/1     Running     0                4d14h
knative-eventing-kafka-knative-eventing-kafka-controller-5   1/1     Running     0                4d14h
```

If the kafka controller and broker pods are in CrashLoopBackOff status, check the pod logs for OOMKilled status, and if required, increase the memory via the KafkaNodePool

Scale down the Data Governor operator in ups-wx-operators namespace, then proceed with the following steps

For the broker KafkaNodePool
```bash
oc patch kafkanodepool <wo-wa-1234-ibm-abcd-broker> -n ups-wx-operands --type=merge -p '{"spec":{"resources":{"limits":{"memory":"8Gi"},"requests":{"memory":"8Gi"}}}}'
```

For the controller KafkaNodePool
```bash
oc patch kafkanodepool <wo-wa-1234-ibm-abcd-controller> -n ups-wx-operands --type=merge -p '{"spec":{"resources":{"limits":{"memory":"1Gi"},"requests":{"memory":"1Gi"}}}}'
```

Once these pods are stable, proceed with upgrading the Events operator, and continue to monitor for memory issues

---

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

Apply the prod license for IBM Software Hub
```bash
cpd-cli manage apply-entitlement --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --entitlement=cpd-enterprise
```

Apply watsonx.ai prod license
```bash
cpd-cli manage apply-entitlement --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}  --entitlement=watsonx-ai 
```

Apply watsonx.governance prod license(s)
```bash
cpd-cli manage apply-entitlement --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --entitlement=watsonx-gov-mm 
cpd-cli manage apply-entitlement --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --entitlement=watsonx-gov-rc 
```

Apply watsonx Orchestrate prod license
```bash
cpd-cli manage apply-entitlement --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --entitlement=watsonx-orchestrate
```

Apply Watson Speech prod license(s)
```bash
cpd-cli manage apply-entitlement --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --entitlement=speech-to-text
cpd-cli manage apply-entitlement --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --entitlement=text-to-speech
```

Apply Cognos Analytics prod license
```bash
cpd-cli manage apply-entitlement --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --entitlement=cognos-analytics
```

Confirm the status of the applied entitlements by checking the cpd-applied-entitlements configmap
```bash
oc get cm cpd-applied-entitlements -o yaml
```

For example
```bash
data:
  applied-entitlements: '{"cpd-enterprise": {"production": "true"}, "watsonx-ai":
    {"production": "true"}, "watsonx-gov-mm": {"production": "true"}, "watsonx-gov-rc":
    {"production": "true"}, "watsonx-orchestrate": {"production": "true"}}'
```

---

## Upgrade IBM Software Hub Platform and Services

#### Upgrade CPD platform

**Reference**: [Upgrading IBM Software Hub](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=53-upgrading-software-hub)

Login to the cluster
```bash
${CPDM_OC_LOGIN}
```

Confirm the value of 'spec/non_olm' in the Ibmcpd ibmcpd-cr custom resource yaml
```bash
echo "Ibmcpd (ibmcpd-cr): non_olm = $(oc get ibmcpd ibmcpd-cr -o jsonpath='{.spec.non_olm}')"
```

The output we expect to see is 'non_olm = true' for a helm based deployment
```bash
Ibmcpd (ibmcpd-cr): non_olm = true
```

Upgrade CPD platform
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

#### Potential Issue - EDB operator to IBM PG operator migration fails because pods do not restart

**Reference**: [EDB operator to IBM PG operator migration fails because pods do not restart](https://www.ibm.com/docs/en/cloud-paks/foundational-services/4.19.x?topic=ui-edb-operator-pg-operator-migration-fails-because-pods-do-not-restart)

In IBM Cloud Pak foundational services versions 4.19.0, 4.19.1, and 4.19.2, EDB operator to IBM PG operator migration fails in heavy workload clusters because the migration job updates pod metadata for labels and owner references before the pods restart with the new IBM PG image

Review the following symptoms and apply the suggested workaround as required

Symptom 1 - The common-service-db-pg-migration-job job takes hours but remains in the Running 0/1 status
```bash
# oc get jobs -n <data namespace> | grep db-pg-migration
common-service-db-pg-migration-job               Running    0/1           3h         3h
```

Symptom 2 - The common-service-db cluster custom resource (CR) is stuck in the Switchover in progress or Instance Status Extraction Error: HTTP communication issue status
```bash
# oc get cluster.pg.ibm.com -A
NAMESPACE   NAME                AGE    INSTANCES   READY   STATUS                   PRIMARY
zen         common-service-db   176m   2           1       Switchover in progress   common-service-db-1

# oc get cluster.pg.ibm.com
NAME                  AGE     INSTANCES   READY   STATUS                                                       PRIMARY
common-service-db   2m36s   3           2       Instance Status Extraction Error: HTTP communication issue
```

Symptom 3 - The common-service-db-x replica pod fails with an error message similar to the following example
```bash
{"level":"info","ts":"2026-07-31T09:23:44.870529463Z","logger":"wait-for-get-cluster","msg":"Encountered an error while executing get cluster. Will wait and retry","logging_pod":"common-service-db-2","error":"clusters.postgresql.k8s.enterprisedb.io \"common-service-db\" is forbidden: User \"system:serviceaccount:zen:common-service-db\" cannot get resource \"clusters\" in API group \"postgresql.k8s.enterprisedb.io\" in the namespace \"zen\""}
```

Workaround - Manually delete the failed common-service-db-x pod so that it restarts, and the pod comes back up with the correct image and permissions, repeating this step for any other failed replica 

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
    size: large
    installMode: "agentic_assistant"
    watsonxAI:
      watsonxaiifm: true
      ootbModels:
        - ibm-slate-30m-english-rtrvr
        - gpt-oss-120b
```

**IMPORTANT**: Before proceeding with Orchestrate upgrade, remove the following image_digests from watsonxaiifm-cr

Remove the image_digests section from watsonxaiifm-cr
```bash
oc patch watsonxaiifm watsonxaiifm-cr -n ${PROJECT_CPD_INST_OPERANDS} --type=json -p='[{"op": "remove", "path": "/spec/image_digests"}]'
```

Remove the image.digestOverrides from the wo custom resource
```bash
oc patch wo wo -n ups-wx-operands --type=merge -p='{"spec": {"image": {"digestOverrides": null}}}'
```

If applicable, remove the 'wo.watsonx.ibm.com/hands-off' annotation from Orchestrate rediscp custom resource
```bash
oc get rediscp wo-watson-orchestrate-rediscp -oyaml
apiVersion: redis.ibm.com/v1
kind: Rediscp
metadata:
  annotations:
    wo.watsonx.ibm.com/hands-off: "yes" -- ***this needs to be removed***
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

Monitor watsonx_orchestrate upgrade
```bash
watch -n 3 'oc get po -A -owide | egrep -v "([0-9])/\1" | egrep -v "Completed" && oc get ccs,watsonxaiifm,wa,documentprocessing,wo'
```

#### Potential Issue - Watson Orchestrate Postgres Instance Stuck

Check the status of the wo-watson-orchestrate-postgresedb cluster
```bash
oc get clusters.postgresql.k8s.enterprisedb.io wo-watson-orchestrate-postgresedb
```

Output
```bash
NAME                                AGE   INSTANCES   READY   STATUS                                       PRIMARY
wo-watson-orchestrate-postgresedb   19d   3           2       Waiting for the instances to become active   wo-watson-orchestrate-postgresedb-2
```

Note that wo-watson-orchestrate-postgresedb-2 is the PRIMARY replica in this scenario

Check the status of the orchestrate-postgresedb replica pods
```bash
wo-watson-orchestrate-postgresedb-3     0/1     CrashLoopBackOff
```

Check the logs of the replica(s) in CrashLoopBackOff
```bash
oc logs wo-watson-orchestrate-postgresedb-3 -n ${PROJECT_CPD_INST_OPERANDS} --tail=50 | grep FATAL
```

Output
```bash
Defaulted container "postgres" out of: postgres, bootstrap-controller (init)
{"level":"info","ts":"2026-08-20T04:25:14.355509372Z","logger":"postgres","msg":"record","logging_pod":"wo-watson-orchestrate-postgresedb-3","record":{"log_time":"2026-08-20 04:25:14.355 UTC","user_name":"postgres","database_name":"postgres","process_id":"37","connection_from":"[local]","session_id":"6a8681aa.25","session_line_num":"1","session_start_time":"2026-08-20 04:25:14 UTC","transaction_id":"0","error_severity":"FATAL","sql_state_code":"57P03","message":"the database system is starting up","backend_type":"client backend","query_id":"0"}}
```

---

Similar issue is reported in CSP ticket - TS022632926

---

Download and configure the kubectl-cnp plug-in
```bash
curl -sSfL https://github.com/EnterpriseDB/kubectl-cnp/raw/main/install.sh | sudo sh -s -- -b /usr/local/bin
```

Delete the broken replica pod and pvc using the cnp plug-in, for example (DO NOT DESTROY THE PRIMARY REPLICA...)
```bash
oc cnp destroy wo-watson-orchestrate-postgresedb wo-watson-orchestrate-postgresedb-3 -n ${PROJECT_CPD_INST_OPERANDS}
```

This should replace the broken replica pod with a new replica and unblock the migration shortly
```bash
oc get po | grep postgresedb 
wo-watson-orchestrate-postgresedb-1                               1/1     Running             0             3m40s
wo-watson-orchestrate-postgresedb-2                               1/1     Running             0             17s
wo-watson-orchestrate-postgresedb-4                               1/1     Running             0             4m5s
```

---

#### Potential Issue - Document Processing Operator pod stuck in CrashLoopBackOff

Update the memory values by patching the deployment
```bash
oc patch deployment ibm-documentprocessing-operator \
  -n ups-wx-operators \
  --type=json \
  -p='[
    {
      "op": "replace",
      "path": "/spec/template/spec/containers/0/resources/limits/memory",
      "value": "2Gi"
    },
    {
      "op": "replace",
      "path": "/spec/template/spec/containers/0/resources/requests/memory",
      "value": "1Gi"
    }
  ]'
```

---

#### Potential Issue - Orchestrate custom resource stuck at 97% - Deploying Milvus

During a test upgrade, Orchestrate got stuck during the Milvus deployment
```bash
oc get wo
NAME   VERSION   PATCH_VERSION     READY        DEPLOYING_COMPONENT   DEPLOYED   VERIFIED   INSTALLMODE         QUIESCE        RECONCILE_PROGRESS   AGE
wo     5.4.0     Patch 5 (8.0.2)   InProgress   milvus                45/45      44/45      agentic_assistant   NOT_QUIESCED   97%                  20d
```

Review the status messages in the Milvus custom resource
```bash
oc get wxdengine wo-milvus -n ${PROJECT_CPD_INST_OPERANDS} -o yaml | grep -A 10 status:
```

Status/message:
```bash
status:
  conditions:
  - lastTransitionTime: "2026-08-20T02:19:02Z"
    message: ""
    reason: ""
    status: "False"
    type: Successful
  - lastTransitionTime: "2026-08-20T19:16:40Z"
    message: Running reconciliation
    reason: Running
    status: "False"
    type: Running
  - ansibleResult:
      changed: 1
      completion: "2026-08-20T19:16:53.36887+00:00"
      failures: 1
      ok: 36
      skipped: 17
    lastTransitionTime: "2026-08-20T02:46:54Z"
    message: |
      The conditional check '(licensing_cr_premium.resources is defined and licensing_cr_premium.resources | length > 0 and licensing_cr_premium.resources[0].spec.set.cloudpakName in license_map) or (licensing_cr_standard.resources is defined and licensing_cr_standard.resources | length > 0 and licensing_cr_standard.resources[0].spec.set.cloudpakName in license_map) or (licensing_cr_spark.resources is defined and licensing_cr_spark.resources | length > 0 and ('spark' in wxd_addon_cr.spec.components or 'spark' in wxd_addon_premium_cr.spec.components) and licensing_cr_spark.resources[0].spec.set.cloudpakName in license_map)' failed. The error was: error while evaluating conditional ((licensing_cr_premium.resources is defined and licensing_cr_premium.resources | length > 0 and licensing_cr_premium.resources[0].spec.set.cloudpakName in license_map) or (licensing_cr_standard.resources is defined and licensing_cr_standard.resources | length > 0 and licensing_cr_standard.resources[0].spec.set.cloudpakName in license_map) or (licensing_cr_spark.resources is defined and licensing_cr_spark.resources | length > 0 and ('spark' in wxd_addon_cr.spec.components or 'spark' in wxd_addon_premium_cr.spec.components) and licensing_cr_spark.resources[0].spec.set.cloudpakName in license_map)): 'dict object' has no attribute 'spec'

      The error appears to be in '/opt/ansible/roles/common/tasks/licensing.yaml': line 112, column 7, but may
      be elsewhere in the file depending on the exact syntax problem.

      The offending line appears to be:

        when:
          - (licensing_cr_premium.resources is defined and licensing_cr_premium.resources | length > 0 and licensing_cr_premium.resources[0].spec.set.cloudpakName in license_map)
            ^ here
    reason: Failed
    status: "True"
    type: Failure
  engineStatus: Completed
  middleEndStatus: RUNNING
  middleEndStatusCode: "0"
  upgradeStatus: 7/7 - Upgrade complete
  versions:
    reconciled: 2.3.1
```

The workaround used at the time was to create the missing ibmlicensingdefinition and restart the lakehouse operator pod
```bash
cat <<EOF | oc apply -f -
apiVersion: operator.ibm.com/v1
kind: IBMLicensingDefinition
metadata:
  name: addonidwatsonxdata
  namespace: cpd-instance
  labels:
    icpdsupport/addOnId: watsonx_data
    icpdsupport/entitlement: watsonx-orchestrate
  annotations:
    cloudpakId: "6341c0866cd24bb298037e1476bd4e56"
    cloudpakName: "IBM watsonx Orchestrate Cartridge"
    productID: "0be53fb8946d4b82a770f82d60f05657"
    productMetric: "FREE"
    productName: "IBM watsonx Orchestrate"
spec:
  action: modifyOriginal
  condition:
    metadata:
      annotations:
        cloudpakInstanceId: "3edfc5f2-f5c1-4132-95bc-7aad0a7e67f6"
      labels:
        icpdsupport/addOnId: watsonx_data
  scope: cluster
  set:
    cloudpakId: "6341c0866cd24bb298037e1476bd4e56"
    cloudpakName: "IBM watsonx Orchestrate Cartridge"
    productID: "0be53fb8946d4b82a770f82d60f05657"
    productName: "IBM watsonx Orchestrate"
    productMetric: "FREE"
    productChargedContainers: ""
    productCloudpakRatio: ""
    serviceId: ""
    serviceName: ""
EOF
```

Delete the lakehouse operator pod
```bash
oc delete pod -n cpd-operators ibm-lakehouse-controller-manager-664ddf845c-xvzsh
```

Orchestrate completes Milvus deployment afterward
```bash
oc get wo
NAME   VERSION   PATCH_VERSION     READY   DEPLOYING_COMPONENT   DEPLOYED   VERIFIED   INSTALLMODE         QUIESCE        RECONCILE_PROGRESS   AGE
wo     5.4.0     Patch 5 (8.0.2)   True    All Deployed          45/45      45/45      agentic_assistant   NOT_QUIESCED   100%                 21d
```

---

#### Potential Issue - IFM Operator reports a PVC error at 82.5% progress 

Check the ifm operator log for errors
```bash
oc describe po ibm-cpd-watsonx-ai-ifm-operator-6f5894446f-kn699 -n ups-wx-operators
```

Look for this particular error message
```bash
message: |- Failed to patch object: b'{"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"PersistentVolumeClaim \\"wx-inference-proxy-cos-rev2\\" is invalid: spec.resources.requests.storage: Forbidden: field can not be less than status.capacity","reason":"Invalid","details":{"name":"wx-inference-proxy-cos-rev2","kind":"PersistentVolumeClaim","causes":[{"reason":"FieldValueForbidden","message":"Forbidden: field can not be less than status.capacity","field":"spec.resources.requests.storage"}]},"code":422}\n'
```

Kubernetes rejects an attempt to modify the PersistentVolumeClaim (PVC) named wx-inference-proxy-cos-rev2 because the configuration tried to set spec.resources.requests.storage to a value smaller than the current status.capacity

Kubernetes does not allow shrinking the storage request of an existing PVC below its provisioned capacity

Daniel created the following hotfix script which resolves the 422 Forbidden validation error where the operator tries to provision the wx-inference-proxy-cos-rev2 PVC with a hardcoded 10Gi storage size that conflicts with an existing or higher expectation

It updates the parameter template from 10Gi to 100Gi directly inside a running IFM operator pod

Confirm the script exists in this location on the bastion node and then run the IFM workaround script 
```bash
/ibm/ifm-inf-proxy-pvc-template-hotfix-5.4.2.sh
```

Monitor the ifm operator logs to ensure that the PVC issue is addressed
```bash
oc logs ibm-cpd-watsonx-ai-ifm-operator-6f5894446f-kn699 -n ups-wx-operators
```

Check for any errors in the operator pod yaml directly
```bash
oc describe po ibm-cpd-watsonx-ai-ifm-operator-6f5894446f-kn699 -n ups-wx-operators
```

---

#### Potential Issue - Watsonx Orchestrate Deployments Blocked by Duplicate Volume Mount Path Conflict

IBM Cloud Pak for Data (CPD) has a feature where it can automatically inject its own trusted certificate bundle into pods — it does this via a Kubernetes webhook that intercepts pods at the moment they're created and adds an extra volume mount pointing to /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem

In WO release 5.4.2, IBM added native support for custom certificates directly inside the WO operator — so WO components now mount their own certificate bundle at that same path (/etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem) via the wo-platform-certs volume

Both are now trying to mount something at the exact same path at the same time. Kubernetes strictly forbids this — every volume mount path in a container must be unique. So when a new pod is created, Kubernetes rejects it immediately with
```bash
spec.containers[0].volumeMounts[x].mountPath: Invalid value: "/etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem": must be unique
```

Several wxo deployments are impacted by this issue, which prevents the deployment pods from starting up properly

The following procedure describes the workaround used to address the duplicate volume mount path conflicts

Patch the 'cpd-config-ac-webhook-cfg-ups-wx-operands' mutatingwebhookconfiguration 
```bash
oc patch mutatingwebhookconfiguration cpd-config-ac-webhook-cfg-ups-wx-operands \
  --type=json \
  -p='[
    {
      "op": "add",
      "path": "/webhooks/0/objectSelector/matchExpressions/-",
      "value": {
        "key": "wo.watsonx.ibm.com/component",
        "operator": "DoesNotExist"
      }
    }
  ]'
```

Copy the contents of the custom ca secret to the wo custom secret
```bash
NS=ups-wx-operands
SOURCE_SECRET=cpd-custom-ca-certs
TARGET_SECRET=wo-custom-certs

oc get secret "$SOURCE_SECRET" -n "$NS" -o json |
jq \
  --arg name "$TARGET_SECRET" \
  --arg namespace "$NS" \
  '{
    apiVersion: "v1",
    kind: "Secret",
    metadata: {
      name: $name,
      namespace: $namespace
    },
    type: .type,
    data: .data
  }' |
oc create -f -
```

Expected result
```bash
mutatingwebhookconfiguration.admissionregistration.k8s.io/cpd-config-ac-webhook-cfg-ups-wx-operands patched
```

Monitor the deployments for Orchestrate and ensure that all of the relevant pods are able to start properly
```bash
oc get deploy -n ${PROJECT_CPD_INST_OPERANDS} | grep wo-
```

---

#### Potential Issue - wo-tenant-migration-job is skipped requiring manual job creation and execution

The wo-tenant-migration-job is supposed to run during the Orchestrate upgrade, but was skipped during the non-prod upgrade

In this scenario, the wo-tenant-migration-job will need to be run manually

Create the job yaml file
```bash
vi wo-tenant-migration-job.yaml
```

Copy the contents into the job yaml file

<details>
<summary>wo-tenant-migration-job.yaml</summary>

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  annotations:
    cloudpakName: IBM watsonx Orchestrate Cartridge for IBM Cloud Pak for Data
    productChargedContainers: All
    productCloudpakRatio: "1:1"
    productMetric: RESOURCE_UNIT
    productName: IBM watsonx Orchestrate
    productVersion: 5.4.0
  labels:
    app.kubernetes.io/component: components-services
    app.kubernetes.io/instance: wo
    app.kubernetes.io/managed-by: ibm-watson-orchestrate-operator
    app.kubernetes.io/name: watson-orchestrate
    icpdsupport/addOnId: orchestrate
    icpdsupport/app: components-services
    icpdsupport/module: components-services-orchestrate
    icpdsupport/podSelector: components-services
    wo.watsonx.ibm.com/application: watson-orchestrate
    wo.watsonx.ibm.com/component: components-services
    wo.watsonx.ibm.com/cr-name: wo
    wo.watsonx.ibm.com/operand-version: 8.0.2
  name: wo-tenant-data-service-migration
  namespace: ups-wx-operands
spec:
  backoffLimit: 3
  completionMode: NonIndexed
  completions: 1
  manualSelector: false
  parallelism: 1
  podReplacementPolicy: TerminatingOrFailed
  suspend: false
  template:
    metadata:
      annotations:
        cloudpakId: 6341c0866cd24bb298037e1476bd4e56
        cloudpakInstanceId: 53071c66-1543-42e6-a0f5-6874e4802720
        cloudpakName: IBM watsonx Orchestrate Cartridge for IBM Cloud Pak for Data
        productChargedContainers: All
        productCloudpakRatio: "1:1"
        productID: 0be53fb8946d4b82a770f82d60f05657
        productMetric: RESOURCE_UNIT
        productName: IBM watsonx Orchestrate
        productVersion: 5.4.0
      creationTimestamp: null
      labels:
        app.kubernetes.io/component: components-services
        app.kubernetes.io/instance: wo
        app.kubernetes.io/managed-by: ibm-watson-orchestrate-operator
        app.kubernetes.io/name: watson-orchestrate
        batch.kubernetes.io/job-name: wo-tenant-data-service-migration
        icpdsupport/addOnId: orchestrate
        icpdsupport/app: components-services
        icpdsupport/module: components-services-orchestrate
        icpdsupport/podSelector: components-services
        job-name: wo-tenant-data-service-migration
        wo.watsonx.ibm.com/application: watson-orchestrate
        wo.watsonx.ibm.com/component: wo-tenant-data-service-migration
        wo.watsonx.ibm.com/cr-name: wo
        wo.watsonx.ibm.com/operand-version: 8.0.2
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/arch
                operator: In
                values:
                - amd64
                - s390x
      containers:
      - args:
        - |
          set -e

          echo "=========================================="
          echo "Tenant Data Service WXO Migration Job"
          echo "=========================================="
          echo "Target DB: $DB_HOST:$DB_PORT/$DB_NAME"
          echo "=========================================="

          for var in SOURCE_POSTGRES_URL DB_HOST DB_PORT DB_USER DB_PASSWORD DB_NAME MULTI_TENANCY_PLATFORM_ACCOUNT_ID; do
            eval val=\$$var
            if [ -z "$val" ]; then
              echo "ERROR: $var is not set"
              exit 1
            fi
          done
          echo "Platform account ID: $MULTI_TENANCY_PLATFORM_ACCOUNT_ID"

          SOURCE_DB_HOST=$(python3 -c "from urllib.parse import urlparse; u=urlparse('$SOURCE_POSTGRES_URL'); print(u.hostname)")
          SOURCE_DB_PORT=$(python3 -c "from urllib.parse import urlparse; u=urlparse('$SOURCE_POSTGRES_URL'); print(u.port or 5432)")
          SOURCE_DB_USER=$(python3 -c "from urllib.parse import urlparse; u=urlparse('$SOURCE_POSTGRES_URL'); print(u.username or '')")
          SOURCE_DB_PASSWORD=$(python3 -c "from urllib.parse import urlparse; u=urlparse('$SOURCE_POSTGRES_URL'); print(u.password or '')")
          SOURCE_DB_NAME=$(python3 -c "from urllib.parse import urlparse; u=urlparse('$SOURCE_POSTGRES_URL'); print(u.path.lstrip('/').split('?')[0])")

          if [ -z "$SOURCE_DB_HOST" ]; then
            echo "ERROR: Failed to parse SOURCE_POSTGRES_URL"
            exit 1
          fi
          echo "Source DB: $SOURCE_DB_HOST:$SOURCE_DB_PORT/$SOURCE_DB_NAME"

          RETRY_COUNT=${DB_CONNECTION_RETRY_COUNT:-12}
          RETRY_DELAY=${DB_CONNECTION_RETRY_DELAY:-5}
          CONNECTION_TIMEOUT=${DB_CONNECTION_TIMEOUT:-5}

          echo "Waiting for target PostgreSQL..."
          i=1
          while [ "$i" -le "$RETRY_COUNT" ]; do
            if timeout "$CONNECTION_TIMEOUT" sh -c "echo > /dev/tcp/$DB_HOST/$DB_PORT" 2>/dev/null; then
              echo "✓ Target PostgreSQL is reachable"
              break
            fi
            echo "Attempt $i/$RETRY_COUNT: waiting for target... (retrying in ${RETRY_DELAY}s)"
            sleep "$RETRY_DELAY"
            i=$((i + 1))
          done
          if ! timeout "$CONNECTION_TIMEOUT" sh -c "echo > /dev/tcp/$DB_HOST/$DB_PORT" 2>/dev/null; then
            echo "ERROR: Could not connect to target PostgreSQL after $RETRY_COUNT attempts"
            exit 1
          fi

          echo "Waiting for source PostgreSQL..."
          i=1
          while [ "$i" -le "$RETRY_COUNT" ]; do
            if timeout "$CONNECTION_TIMEOUT" sh -c "echo > /dev/tcp/$SOURCE_DB_HOST/$SOURCE_DB_PORT" 2>/dev/null; then
              echo "✓ Source PostgreSQL is reachable"
              break
            fi
            echo "Attempt $i/$RETRY_COUNT: waiting for source... (retrying in ${RETRY_DELAY}s)"
            sleep "$RETRY_DELAY"
            i=$((i + 1))
          done
          if ! timeout "$CONNECTION_TIMEOUT" sh -c "echo > /dev/tcp/$SOURCE_DB_HOST/$SOURCE_DB_PORT" 2>/dev/null; then
            echo "ERROR: Could not connect to source PostgreSQL after $RETRY_COUNT attempts"
            exit 1
          fi

          echo "=========================================="
          echo "Checking migration status..."
          echo "=========================================="
          export PGPASSWORD="$DB_PASSWORD"

          OLD_SCHEMA_COUNT=$(psql -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d "$DB_NAME" -tAc "
            SELECT COUNT(*) FROM tenant_snapshots
            WHERE account_id = '$MULTI_TENANCY_PLATFORM_ACCOUNT_ID'
              AND tenant_data ? 'tenant'
              AND NOT tenant_data ? 'tenantName'
          " 2>/dev/null || echo "0")

          ALREADY_MIGRATED=$(psql -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d "$DB_NAME" -tAc "
            SELECT EXISTS (
              SELECT 1 FROM tenant_snapshots
              WHERE account_id = '$MULTI_TENANCY_PLATFORM_ACCOUNT_ID'
              LIMIT 1
            )" 2>/dev/null || echo "f")

          if [ "$OLD_SCHEMA_COUNT" != "0" ]; then
            echo "⚠ Found $OLD_SCHEMA_COUNT row(s) with old event-envelope schema — will heal via upsert."
          else
            echo "✓ Target DB is clean — proceeding with fresh migration."
          fi

          psql -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d "$DB_NAME" -tAc \
            "CREATE EXTENSION IF NOT EXISTS dblink;" 2>/dev/null || true

          export PGPASSWORD="$SOURCE_DB_PASSWORD"
          SOURCE_TENANT_COUNT=$(psql -h "$SOURCE_DB_HOST" -p "$SOURCE_DB_PORT" -U "$SOURCE_DB_USER" \
            -d "$SOURCE_DB_NAME" -tAc "SELECT COUNT(*) FROM tenants" 2>/dev/null || echo "0")
          SOURCE_EVENT_COUNT=$(psql -h "$SOURCE_DB_HOST" -p "$SOURCE_DB_PORT" -U "$SOURCE_DB_USER" \
            -d "$SOURCE_DB_NAME" -tAc "SELECT COUNT(*) FROM events" 2>/dev/null || echo "0")

          echo "Source — tenants: $SOURCE_TENANT_COUNT  events: $SOURCE_EVENT_COUNT"

          if [ "$SOURCE_TENANT_COUNT" -eq "0" ] && [ "$SOURCE_EVENT_COUNT" -eq "0" ]; then
            echo "INFO: Source DB is empty (fresh install). Nothing to migrate."
            exit 0
          fi

          echo "=========================================="
          echo "Migrating tenants → tenant_snapshots..."
          echo "=========================================="
          export PGPASSWORD="$DB_PASSWORD"

          psql -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d "$DB_NAME" -v ON_ERROR_STOP=1 << ENDSQL
          INSERT INTO tenant_snapshots (id, account_id, status, tenant_data, created_at, updated_at, synced_at)
          SELECT
            t.id,
            '$MULTI_TENANCY_PLATFORM_ACCOUNT_ID',
            CASE WHEN t.disabled THEN 'suspended' ELSE 'active' END,
            jsonb_build_object(
              'tenantId',      t.id,
              'tenantName',    COALESCE(t.name, t.id),
              'tenantType',    'onprem',
              'status',        CASE WHEN t.disabled THEN 'DISABLED' ELSE 'ACTIVE' END,
              'accountId',     '$MULTI_TENANCY_PLATFORM_ACCOUNT_ID',
              'crn',           '',
              'tenantCRN',     t.id,
              'createdAt',     to_char(COALESCE(t.created_on, NOW()) AT TIME ZONE 'UTC', 'YYYY-MM-DD"T"HH24:MI:SS"Z"'),
              'updatedAt',     to_char(NOW() AT TIME ZONE 'UTC', 'YYYY-MM-DD"T"HH24:MI:SS"Z"'),
              'tenantOwner',   jsonb_build_object('id', '', 'username', ''),
              'metadata',      '{}'::jsonb,
              'zenServiceInstanceInfo', jsonb_build_object(
                'zenServiceInstanceId',       t.id,
                'zenCloudPakInstanceId',       '',
                'zenControlPlaneNamespace',    '',
                'zenImageRegistryPrefix',      '',
                'zenServiceInstanceType',      'orchestrate',
                'zenServiceInstanceUID',       '',
                'zenServiceInstanceUserName',  '',
                'zenServiceInstanceVersion',   '',
                'zenServiceInstanceSecret',    ''
              ),
              'instanceId',      null,
              'planId',          'ONPREM',
              'clusterDomain',   null,
              'countryCode',     'US',
              'organizationId',  '',
              'provisioningStatus', jsonb_build_object(
                'state',            CASE WHEN t.disabled THEN 'DISABLED' ELSE 'ACTIVE' END,
                'progressStatus',   100,
                'is_isolated',      false,
                'cluster',          jsonb_build_object(
                  'clusterDomain', null,
                  'state',         CASE WHEN t.disabled THEN 'DISABLED' ELSE 'ACTIVE' END
                )
              )
            ),
            COALESCE(t.created_on, NOW()),
            NOW(),
            NOW()
          FROM dblink(
            'host=$SOURCE_DB_HOST port=$SOURCE_DB_PORT dbname=$SOURCE_DB_NAME user=$SOURCE_DB_USER password=$SOURCE_DB_PASSWORD sslmode=require',
            'SELECT id, name, disabled, created_on FROM tenants'
          ) AS t(id text, name text, disabled boolean, created_on timestamptz)
          ON CONFLICT (id) DO UPDATE SET
            tenant_data = EXCLUDED.tenant_data,
            status      = EXCLUDED.status,
            updated_at  = EXCLUDED.updated_at,
            synced_at   = EXCLUDED.synced_at;
          ENDSQL

          MIGRATED_TENANT_COUNT=$(psql -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d "$DB_NAME" -tAc \
            "SELECT COUNT(*) FROM tenant_snapshots WHERE account_id = '$MULTI_TENANCY_PLATFORM_ACCOUNT_ID'")
          echo "✓ tenant_snapshots: $MIGRATED_TENANT_COUNT rows"

          echo "=========================================="
          echo "Migrating events → events..."
          echo "=========================================="

          psql -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d "$DB_NAME" -v ON_ERROR_STOP=1 << ENDSQL
          INSERT INTO events (
            id, account_id, tenant_id, event_type, payload, processed,
            user_id, created_by, updated_by, event_id, consumer_responses,
            created_on, updated_at
          )
          SELECT
            e.id::text,
            '$MULTI_TENANCY_PLATFORM_ACCOUNT_ID',
            e.tenant_id,
            e.event_type,
            COALESCE(e.payload, '{}'::jsonb),
            COALESCE(e.processed, false),
            e.user_id,
            e.created_by,
            e.updated_by,
            e.event_id,
            e.consumer_responses,
            e.created_on,
            e.updated_at
          FROM dblink(
            'host=$SOURCE_DB_HOST port=$SOURCE_DB_PORT dbname=$SOURCE_DB_NAME user=$SOURCE_DB_USER password=$SOURCE_DB_PASSWORD sslmode=require',
            'SELECT id::text, tenant_id, event_type::text, payload, processed, user_id, created_by, updated_by, event_id, consumer_responses, created_on, updated_at FROM events'
          ) AS e(id text, tenant_id text, event_type text, payload jsonb, processed boolean,
                 user_id text, created_by text, updated_by text, event_id text,
                 consumer_responses jsonb, created_on timestamptz, updated_at timestamptz)
          ON CONFLICT (id) DO NOTHING;
          ENDSQL

          MIGRATED_EVENT_COUNT=$(psql -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d "$DB_NAME" -tAc \
            "SELECT COUNT(*) FROM events WHERE account_id = '$MULTI_TENANCY_PLATFORM_ACCOUNT_ID'")
          echo "✓ events: $MIGRATED_EVENT_COUNT rows"

          echo "Backfilling subscriptions..."
          psql -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d "$DB_NAME" -v ON_ERROR_STOP=1 << ENDSQL
          INSERT INTO subscriptions (id, tenant_id, subscription_id, crn, subscription_details, status, created_at, updated_at)
          SELECT
            gen_random_uuid(),
            id AS tenant_id,
            COALESCE(tenant_data->'subscriptions'->>'subscriptionId', id),
            NULL,
            jsonb_build_object(
              'partNumber', 'ONPREM',
              'accountId',  '$MULTI_TENANCY_PLATFORM_ACCOUNT_ID'
            ),
            COALESCE(tenant_data->'subscriptions'->>'status', 'ACTIVE'),
            created_at,
            NOW()
          FROM tenant_snapshots
          WHERE account_id = '$MULTI_TENANCY_PLATFORM_ACCOUNT_ID'
          ON CONFLICT (tenant_id, subscription_id) DO NOTHING;
          ENDSQL
          echo "✓ subscriptions backfilled"

          echo "Backfilling ip_allowlists..."
          psql -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d "$DB_NAME" -v ON_ERROR_STOP=1 << ENDSQL
          INSERT INTO ip_allowlists (id, subscription_id, crn, ip_ranges, enabled, created_at, updated_at)
          SELECT
            gen_random_uuid(),
            s.subscription_id,
            NULL,
            '[]'::jsonb,
            false,
            NOW(),
            NOW()
          FROM subscriptions s
          JOIN tenant_snapshots ts ON ts.id = s.tenant_id
          WHERE ts.account_id = '$MULTI_TENANCY_PLATFORM_ACCOUNT_ID'
          ON CONFLICT (subscription_id) DO NOTHING;
          ENDSQL
          echo "✓ ip_allowlists backfilled"

          echo "=========================================="
          echo "Migration complete. Final counts:"
          echo "=========================================="
          psql -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d "$DB_NAME" -tA << ENDSQL
          SELECT 'tenant_snapshots' AS table_name, COUNT(*) AS rows
            FROM tenant_snapshots WHERE account_id = '$MULTI_TENANCY_PLATFORM_ACCOUNT_ID'
          UNION ALL
          SELECT 'events', COUNT(*)
            FROM events WHERE account_id = '$MULTI_TENANCY_PLATFORM_ACCOUNT_ID'
          UNION ALL
          SELECT 'subscriptions', COUNT(*)
            FROM subscriptions s
            JOIN tenant_snapshots ts ON ts.id = s.tenant_id
            WHERE ts.account_id = '$MULTI_TENANCY_PLATFORM_ACCOUNT_ID'
          UNION ALL
          SELECT 'ip_allowlists', COUNT(*)
            FROM ip_allowlists il
            JOIN subscriptions s ON s.subscription_id = il.subscription_id
            JOIN tenant_snapshots ts ON ts.id = s.tenant_id
            WHERE ts.account_id = '$MULTI_TENANCY_PLATFORM_ACCOUNT_ID';
          ENDSQL
          echo "=========================================="
        command:
        - /bin/sh
        - -c
        env:
        - name: DB_HOST
          value: wo-watson-orchestrate-postgresedb-rw.cpd-instance-1.svc.cluster.local
        - name: DB_PORT
          value: "5432"
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              key: DB_USER
              name: wo-watson-orchestrate-pg-secret
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              key: DB_PASSWORD
              name: wo-watson-orchestrate-pg-secret
        - name: DB_NAME
          value: tenantdataservicedb
        - name: DB_SSLMODE
          value: require
        - name: SOURCE_POSTGRES_URL
          valueFrom:
            secretKeyRef:
              key: DB_CONNECTION_URI_ARCHER
              name: wo-watson-orchestrate-pg-secret
        - name: SOURCE_DB_SSL_MODE
          value: require
        - name: MULTI_TENANCY_PLATFORM_ACCOUNT_ID
          valueFrom:
            configMapKeyRef:
              key: MULTI_TENANCY_PLATFORM_ACCOUNT_ID
              name: product-configmap
        - name: WXO_DEPLOYMENT_PLATFORM
          value: onprem
        - name: DB_CONNECTION_RETRY_COUNT
          value: "12"
        - name: DB_CONNECTION_RETRY_DELAY
          value: "5"
        - name: DB_CONNECTION_TIMEOUT
          value: "5"
        image: cp.stg.icr.io/cp/watsonx-orchestrate/ibm-watsonx-orchestrate-onprem-utils@sha256:2a33735667284a657367b31ff800dd75a40dd8038af1f1be517a93c06345826b
        imagePullPolicy: Always
        name: migrate-wxo-data
        resources:
          limits:
            cpu: 500m
            ephemeral-storage: 1Gi
            memory: 512Mi
          requests:
            cpu: 100m
            ephemeral-storage: 100Mi
            memory: 128Mi
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            drop:
            - ALL
          privileged: false
          readOnlyRootFilesystem: false
          runAsNonRoot: true
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      imagePullSecrets:
      - name: ibm-entitlement-key
      restartPolicy: OnFailure
      schedulerName: default-scheduler
      securityContext: {}
      serviceAccount: wo-watson-orchestrate-no-perm
      serviceAccountName: wo-watson-orchestrate-no-perm
      terminationGracePeriodSeconds: 30
  ttlSecondsAfterFinished: 86400
```

</details>

Apply the job yaml
```bash
oc apply -f wo-tenant-migration-job.yaml
```

Monitor the status of the wo-tenant-migration-job job and the Orchestrate custom resource
```bash
oc get wo wo -n ${PROJECT_CPD_INST_OPERANDS} -o yaml
```

---

#### Post upgrade task 1 for Watsonx Orchestrate

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

Download and edit the [atm_endpoint_tls_migration.sql script](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=u-upgrading-from-version-53-18)
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

---

#### Post upgrade task 2 for Watsonx Orchestrate

After completing this migration, follow the steps for 'Applying the watsonx Orchestrate 5.4.0 Patch-5 (5.4.2) Hotfix 1'

**Reference**: [Apply hot fix for IBM watsonx Orchestrate](https://www.ibm.com/support/pages/node/7247038)

**Reference**: [Applying the watsonx Orchestrate 5.4.0 Patch-5 (5.4.2) Hotfix 1](https://www.ibm.com/support/pages/node/7286341)

Set the operator and operand namespaces
```bash
export PROJECT_CPD_INST_OPERATORS=ups-wx-operators
export PROJECT_CPD_INST_OPERANDS=ups-wx-operands
```

**Note**: You will need to install Skopeo and mirror the operator and operand images before proceeding

Create 5.4.2-Hotfix1.sh
```bash
vi 5.4.2-Hotfix1.sh
```

With the following contents

<details>
<summary>5.4.2-Hotfix1.sh</summary>

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
OPERATOR_PATCH_LABEL_VALUE="${OPERATOR_PATCH_LABEL_VALUE:-5.4.2-Hotfix1}"
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
  log "       WXO version  : 5.4.0"
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
BOOTSTRAP_OPERATOR_IMAGE="icr.io/cpopen/ibm-watsonx-orchestrate-operator@sha256:10e41967b0e6e985169d42d94ab35710bb9cef0e83a13a040b4b5ee838c6ed3c"
COMPONENT_OPERATOR_IMAGE="icr.io/cpopen/ibm-wxo-component-operator@sha256:2516e5b84db6cbca9357a268d454361fb6dac880d44c6bf8e2e6a634af1e6ade"

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
# uiproxy certificate fix
# -----------------------------
if oc get certificate wo-uiproxy-tls-icert \
    -n "${PROJECT_CPD_INST_OPERANDS}" >/dev/null 2>&1; then
    DNS_NAMES=$(oc get certificate wo-uiproxy-tls-icert \
        -n "${PROJECT_CPD_INST_OPERANDS}" \
        -o jsonpath='{.spec.dnsNames[*]}' 2>/dev/null)
    if echo "${DNS_NAMES}" | tr ' ' '\n' | grep -q '^wo-uiproxy-'; then
        echo "Deleting wo-uiproxy-tls-icert due to an incorrect SAN entry. The operator will recreate it with the correct SAN."
        oc delete certificate wo-uiproxy-tls-icert \
            -n "${PROJECT_CPD_INST_OPERANDS}"
    fi
else
    echo "Certificate wo-uiproxy-tls-icert not found. Skipping SAN validation."
fi

# -----------------------------
# tenant migration jobs to run
# -----------------------------
if oc get job zen-addon-config-update-job -n "${PROJECT_CPD_INST_OPERANDS}" >/dev/null 2>&1; then
    echo "Found zen-addon-config-update-job. Deleting it..."
    oc delete job zen-addon-config-update-job -n "${PROJECT_CPD_INST_OPERANDS}"
else
    echo "zen-addon-config-update-job not found. Nothing to delete."
fi
if oc get job wo-tenant-data-service-migration -n "${PROJECT_CPD_INST_OPERANDS}" >/dev/null 2>&1; then
    echo "Found wo-tenant-data-service-migration. Deleting it..."
    oc delete job wo-tenant-data-service-migration -n "${PROJECT_CPD_INST_OPERANDS}"
else
    echo "wo-tenant-data-service-migration not found. Nothing to delete."
fi

# -----------------------------
# watson-gateway fix
# -----------------------------
GW_SHA="sha256:ed87dfc283fd1bf29a078e5c91b20e86abd1fcd73dca89629a34d9ba7d826441"
GW_TAG="2.4.0"

# Hardcode gateway image here
GATEWAY_OPERATOR_IMAGE="icr.io/cpopen/watson-gateway-operator@sha256:2faa51ae7c013a41db2dc09a9b971374e61b84f267cedd086243a912b93de435"

if oc get deploy -n "$PROJECT_CPD_INST_OPERATORS" -lcomponent-id=watson-gateway >/dev/null 2>&1; then

  GW_OPERATOR_DEPLOYMENT=$(oc get deploy -n "$PROJECT_CPD_INST_OPERATORS" -lcomponent-id=watson-gateway -o jsonpath='{.items[0].metadata.name}')
  log "✅ Deployment '${GW_OPERATOR_DEPLOYMENT}' found."

  # Backup deployment YAML before patching
  BACKUP_FILE="${BACKUP_DIR}/${GW_OPERATOR_DEPLOYMENT}-$(date +%Y%m%d%H%M%S).yaml"
  if oc -n "$PROJECT_CPD_INST_OPERATORS" get deploy "$GW_OPERATOR_DEPLOYMENT" -o yaml > "$BACKUP_FILE" 2>/dev/null; then
    log "   Backed up deployment/$DEPLOYMENT → $BACKUP_FILE"
  else
    log "   WARNING: Failed to back up deployment/$GW_OPERATOR_DEPLOYMENT"
  fi

  CURRENT_IMAGE="$(oc get deploy "$GW_OPERATOR_DEPLOYMENT" -n "$PROJECT_CPD_INST_OPERATORS" \
    -o jsonpath='{.spec.template.spec.containers[0].image}')"

  log "✅ Match found. Patching deployment '$GW_OPERATOR_DEPLOYMENT'"
  log "   Old: $CURRENT_IMAGE"
  log "   New: $GATEWAY_OPERATOR_IMAGE"

  if oc patch deploy "$GW_OPERATOR_DEPLOYMENT" -n "$PROJECT_CPD_INST_OPERATORS" \
    --type='json' \
    -p="[{
      \"op\": \"replace\",
      \"path\": \"/spec/template/spec/containers/0/image\",
      \"value\": \"$GATEWAY_OPERATOR_IMAGE\"
    }]"; then
    log "🚀 Successfully patched $GW_OPERATOR_DEPLOYMENT"

    # Patch the olm-utils configmap
    BACKUP_FILE="${BACKUP_DIR}/olm-utils-cm-$(date +%Y%m%d%H%M%S).yaml"
    oc get cm olm-utils-cm -n ${PROJECT_CPD_INST_OPERANDS} -o yaml > "$BACKUP_FILE"
    oc get cm olm-utils-cm -n ${PROJECT_CPD_INST_OPERANDS} -o yaml | yq ".data.release_components_meta |= (fromyaml | .watson_gateway.cr_version = \"${GW_TAG}\" | to_yaml)" | oc apply -f -

  else
    log "✗ ERROR: Failed to patch $GW_OPERATOR_DEPLOYMENT"
  fi
  log ""

  # Wait for rollout to complete
  if oc -n "$PROJECT_CPD_INST_OPERATORS" rollout status deploy/"$GW_OPERATOR_DEPLOYMENT" --timeout=300s; then
    # Check Ready/Desired replica ratio
    RATIO="$(oc -n "$PROJECT_CPD_INST_OPERATORS" get deploy "$GW_OPERATOR_DEPLOYMENT" \
      -o jsonpath='{.status.readyReplicas}/{.status.replicas}' 2>/dev/null || echo '0/0')"
    log "   Ready/Desired: $RATIO"
    
    if [[ "$RATIO" == "1/1" ]] || [[ "$RATIO" == "2/2" ]]; then
      log "   ✅ Deployment $GW_OPERATOR_DEPLOYMENT is healthy (pods up and running)"
    else
      log "   ⚠️  WARNING: Deployment $GW_OPERATOR_DEPLOYMENT is not at 1/1 or 2/2; current $RATIO"
    fi
  else
    log "   ✗ ERROR: Rollout status for deployment/$GW_OPERATOR_DEPLOYMENT did not complete successfully"
  fi

  GW_DEPLOYMENT=$(oc get deploy -n "$PROJECT_CPD_INST_OPERANDS" -lcomponent=watson-gateway -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)

  if [[ -n "$GW_DEPLOYMENT" ]]; then
    log "✅ Deployment '${GW_DEPLOYMENT}' found. Restarting..."

    oc -n "$PROJECT_CPD_INST_OPERANDS" rollout restart deploy/"$GW_DEPLOYMENT"
    # Wait for rollout to complete
    if oc -n "$PROJECT_CPD_INST_OPERANDS" rollout status deploy/"$GW_DEPLOYMENT" --timeout=300s; then
      # Check Ready/Desired replica ratio
      RATIO="$(oc -n "$PROJECT_CPD_INST_OPERANDS" get deploy "$GW_DEPLOYMENT" \
        -o jsonpath='{.status.readyReplicas}/{.status.replicas}' 2>/dev/null || echo '0/0')"
      log "   Ready/Desired: $RATIO"
      
      if [[ "$RATIO" == "1/1" ]] || [[ "$RATIO" == "2/2" ]]; then
        log "   ✅ Deployment $GW_DEPLOYMENT is healthy (pods up and running)"
      else
        log "   ⚠️  WARNING: Deployment $GW_DEPLOYMENT is not at 1/1 or 2/2; current $RATIO"
      fi
    else
      log "   ✗ ERROR: Rollout status for deployment/$GW_DEPLOYMENT did not complete successfully"
    fi
  fi
fi

# -----------------------------
# create wo-custom-certs when customer uses cpd's cert management
# -----------------------------
if oc get secret cpd-custom-ca-certs -n "${PROJECT_CPD_INST_OPERANDS}" >/dev/null 2>&1 && \
   ! oc get secret wo-custom-certs -n "${PROJECT_CPD_INST_OPERANDS}" >/dev/null 2>&1; then
    oc get secret cpd-custom-ca-certs \
      -n "${PROJECT_CPD_INST_OPERANDS}" \
      -o yaml | \
    sed \
      -e 's/name: cpd-custom-ca-certs/name: wo-custom-certs/' \
      -e '/creationTimestamp:/d' \
      -e '/resourceVersion:/d' \
      -e '/uid:/d' \
      -e '/managedFields:/d' | \
    oc apply -f -
fi

# -----------------------------
# Legacy Cleanup
# -----------------------------

# Delete a named resource only if it exists
del() {
  local kind="$1" name="$2" ns_flag="${3:-}"
  if oc get "$kind" "$name" ${ns_flag:+-n "$ns_flag"} >/dev/null 2>&1; then
    oc delete "$kind" "$name" ${ns_flag:+-n "$ns_flag"} --ignore-not-found
  fi
}

# Delete resources by label selector; skip silently if none found
del_by_label() {
  local kind="$1" label="$2" ns="$3"
  if oc get "$kind" -n "$ns" -l "$label" --no-headers 2>/dev/null | grep -q .; then
    oc delete "$kind" -n "$ns" -l "$label" --ignore-not-found
  fi
}

# Scale a deployment to 0 only if it exists
scale_down() {
  local deploy="$1" ns="$2"
  if oc get deploy "$deploy" -n "$ns" >/dev/null 2>&1; then
    oc scale deploy "$deploy" -n "$ns" --replicas=0
  fi
}

log "Starting legacy cleanup..."

# ── UAB ────────────────────────────────────────────────────────────────────────────
log "Disabling UAB..."
oc patch wo wo -n "${PROJECT_CPD_INST_OPERANDS}" --type=merge \
  -p '{"spec":{"uab":{"enabled":false}}}' 2>/dev/null || true

log "Scaling down UAB operators..."
scale_down ba-saas-uab-wf-operator-controller-manager "${PROJECT_CPD_INST_OPERATORS}"
scale_down ibm-uab-ads-operator                       "${PROJECT_CPD_INST_OPERATORS}"
sleep 20

log "Cleaning UAB CRs and CRDs..."
del uabautomationdecisionservices wo "${PROJECT_CPD_INST_OPERANDS}"
del uabwfservices                 wo "${PROJECT_CPD_INST_OPERANDS}"
del crd uabautomationdecisionservices.uab.ba.ibm.com
del crd uabwfservices.uab.ba.ibm.com
del crd wfpsauthorings.saas.ba.ibm.com

log "Cleaning UAB WF operator resources..."
for kind in job deploy secret cm svc; do
  del_by_label "$kind" "app.kubernetes.io/managed-by=ibm-uab-wf-operator" "${PROJECT_CPD_INST_OPERANDS}"
done

log "Cleaning ADS resources..."
for kind in deploy secret cm job svc; do
  del_by_label "$kind" "app.kubernetes.io/component=ads" "${PROJECT_CPD_INST_OPERANDS}"
done

log "UAB cleanup completed."

# ── Digital Employee ────────────────────────────────────────────────────────────
log "Cleaning Digital Employee..."
if oc api-resources 2>/dev/null | grep -q "^digitalemployees"; then
  oc delete digitalemployees.wo.watsonx.ibm.com -n "${PROJECT_CPD_INST_OPERANDS}" --ignore-not-found
fi

scale_down digital-employee-operator-controller-manager "${PROJECT_CPD_INST_OPERATORS}"
sleep 20

for kind in deploy secret cm job svc; do
  del_by_label "$kind" "wo.watsonx.ibm.com/component=digital-employee" "${PROJECT_CPD_INST_OPERANDS}"
done

log "Digital Employee cleanup completed."

# ── Kafka ─────────────────────────────────────────────────────────────────────────────
log "Cleaning Kafka..."
if oc get kafka wo-watson-orchestrate-kafkaibm -n "${PROJECT_CPD_INST_OPERANDS}" >/dev/null 2>&1; then
  log "Kafka CR wo-watson-orchestrate-kafkaibm found. Deleting it..."
  del kafka wo-watson-orchestrate-kafkaibm "${PROJECT_CPD_INST_OPERANDS}"
  sleep 10

  if oc get deploy wo-archer-server -n "${PROJECT_CPD_INST_OPERANDS}" >/dev/null 2>&1; then
    log "Deleting Archer deployment wo-archer-server..."
    oc delete deploy wo-archer-server -n "${PROJECT_CPD_INST_OPERANDS}" --ignore-not-found
  else
    log "Archer deployment wo-archer-server not found; skipping."
  fi
else
  log "Kafka CR wo-watson-orchestrate-kafkaibm not found; skipping Kafka and Archer cleanup."
fi
log "Kafka cleanup completed."


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
log "------------------------------------------------------------------"
```

</details>

Make the script executable
```bash
chmod 775 5.4.2-Hotfix1.sh
```
 
Run the script
```bash
nohup sh 5.4.2-Hotfix1.sh &
```
 
Watch progress
```bash
tail -f nohup.out
```

Verify CR status and label
```bash
oc get wo -n "${PROJECT_CPD_INST_OPERANDS}" -o yaml | grep -i hotfix
```

Create 5.4.2-Hotfix1-verify.sh
```bash
vi 5.4.2-Hotfix1-verify.sh 
```

With the following content

<details>
<summary>5.4.2-Hotfix1-verify.sh</summary>

```bash
#!/usr/bin/env bash
set -eo pipefail

# ============================================================
# 542 Hotfix1 Verification Script
# Checks operator and operand deployments for expected
# image SHAs for the 5.4.2-Hotfix1 fix.
# Runs in a loop until all are verified or timeout is reached.
# ============================================================

# ---- Colour helpers ----------------------------------------
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
CYAN='\033[0;36m'
BOLD='\033[1m'
RESET='\033[0m'

# Use printf throughout — echo -e is not POSIX-portable (breaks under sh/dash)
log()  { printf "[%s] %s\n"                             "$(date '+%Y-%m-%d %H:%M:%S')" "$*"; }
ok()   { printf "${GREEN}${BOLD}[%s] OK  %s${RESET}\n" "$(date '+%Y-%m-%d %H:%M:%S')" "$*"; }
warn() { printf "${YELLOW}[%s] WARN %s${RESET}\n"       "$(date '+%Y-%m-%d %H:%M:%S')" "$*"; }
err()  { printf "${RED}[%s] ERR  %s${RESET}\n"          "$(date '+%Y-%m-%d %H:%M:%S')" "$*"; }
info() { printf "${CYAN}[%s] INFO %s${RESET}\n"         "$(date '+%Y-%m-%d %H:%M:%S')" "$*"; }

# ---- Configuration -----------------------------------------
PROJECT_CPD_INST_OPERATORS="${PROJECT_CPD_INST_OPERATORS:-}"
PROJECT_CPD_INST_OPERANDS="${PROJECT_CPD_INST_OPERANDS:-}"

if [ -z "$PROJECT_CPD_INST_OPERATORS" ]; then
  err "PROJECT_CPD_INST_OPERATORS is not set."
  exit 1
fi

if [ -z "$PROJECT_CPD_INST_OPERANDS" ]; then
  err "PROJECT_CPD_INST_OPERANDS is not set."
  exit 1
fi

# Poll interval (seconds) and overall timeout
POLL_INTERVAL="${POLL_INTERVAL:-30}"
TIMEOUT_SECONDS="${TIMEOUT_SECONDS:-3600}"   # 60 minutes

# ============================================================
# OPERATOR deployments (live in PROJECT_CPD_INST_OPERATORS)
# Format: "deployment-name=sha256digest"  — one entry per line, order preserved.
# From the hotfix patch script:
#   BOOTSTRAP_OPERATOR_IMAGE  -> wo-operator
#   COMPONENT_OPERATOR_IMAGE  -> ibm-wxo-componentcontroller-manager
# ============================================================
OPERATOR_ENTRIES=(
  "wo-operator=48d636029b579f41bc1baf4e5400bc585a1201a1a0f055cb1d6212b7cb75d2b9"
  "ibm-wxo-componentcontroller-manager=2516e5b84db6cbca9357a268d454361fb6dac880d44c6bf8e2e6a634af1e6ade"
)

# ============================================================
# OPERAND deployments (live in PROJECT_CPD_INST_OPERANDS)
# Format: "deployment-name=sha256digest"  — one entry per line, order preserved.
# ============================================================
OPERAND_ENTRIES=(
  "wo-ai-gateway=e2311681dca9da15bf0766eb4786d04869d08b24f7d7c998223f469dce5f5c3c"
  "wo-wxo-connections=04d1adea76a20e4aff886a227cbbc62110b0bdeb29ec82a1c82481fea65fcdc6"
  "wo-tenant-data-service=8f00aa42676b6f9cb76e1b49b3fd0df08971189c3a3eb95d91ffb5d090d32b08"
  "wo-channel-integrations=ffb193e1bc39d6400006220f299b729157b5a9205932b0feefa4a5e8f0780732"
  "wo-conversation-controller=34f68c8dbbb4792fff6171219573f755475117996a59ee03fc3a9ca7f837034c"
  "wo-skill-server=1f86f27749d8df98c40debcf5ec8b2e4953aa653c563696a5b7b2aa8743fb20b"
  "wo-ccaas-chat-connector=e52a2d1dc3ea1bad2167d5a3d79e9a7d91dc1686178d4cfc7cc4656b826fd32d"
  "wo-agentic-task-manager=2a1b47bc8d2c01e6ffe128627c0194ec4e00f35ec820292923c571116c72fee9"
  "wo-builder-ui=d2589f5b82198922fef13227ed6dbee048e0495dea63fbedc3b52a8ce51c9c87"
  "wo-wxo-connections-ui=ea536612ca93c32302d95c1cb89bce6af318267bbcec441f0bcf412f06c0393b"
  "wo-archer-server=798e25cd24ae0e3c9a06745870f3c580161b19e575d2dc7224d10b46f3e4278d"
  "wo-voice-controller=71739e526c17aa290fdcf5cf968a68e00462671fae6a20943767eebf95004bfe"
  "wo-socket-handler=6d296d32c2233e11bcfba17284af7a1acf04b716ce9f76a96d681372e2f8bd3b"
  "wo-tools-runtime-scheduler=49689057ca6c73e4b288c40213cc6a4523e2d985809d050c4e6cbf4016676036"
  "wo-tools-runtime-manager=928b77bf61156daa30d580d88c69f799d3a899e4b81605838e6b62b673d4f13f"
  "wo-wxo-knowledge=3919fd5b613db6121a9caee168515da09c1618fc4157135b2322de2b52169778"
  "wo-agentic-memory=694839c6c408a8c8bcaecbda4c50eeb09a2d66c357ac6639bf41a5bc62b76080"
  "wo-agent-gateway=758bd2348cb907f50e95b96ce5dd9abe27523d3bae7122db1f48cd8202a7f18a"
  "wo-uiproxy=65632b0b9f1c2022d025a5da30630f961de909f95a56d1c4c21c5ba77ccd3cb3"
)

# ============================================================
# OPERAND jobs (live in PROJECT_CPD_INST_OPERANDS)
# Format: "job-name=sha256digest"  — one entry per line, order preserved.
# Checks: image SHA matches AND job status.succeeded >= 1.
# ============================================================
JOB_ENTRIES=(
  "zen-addon-config-update-job=b99f2ce6e0deb1ad64e14b0858c27945691402d725559997022cf74b01d6e717"
  "wo-tenant-data-service-migration=b99f2ce6e0deb1ad64e14b0858c27945691402d725559997022cf74b01d6e717"
)

# ---- Helpers: extract name / sha from an "name=sha" entry --
entry_name() { echo "${1%%=*}"; }
entry_sha()  { echo "${1#*=}";  }

# ---- Pre-flight checks -------------------------------------
WO_CR_NAME="${WO_CR_NAME:-wo}"   # override if CR name differs: export WO_CR_NAME=mywo
EXPECTED_CR_VERSION="8.0.2"
EXPECTED_LABEL_KEY="Hotfix"
EXPECTED_LABEL_VALUE="5.4.2-Hotfix1"

if ! oc whoami &>/dev/null; then
  err "Not logged in to OpenShift. Please run 'oc login' first."
  exit 1
fi
ok "OpenShift login verified: $(oc whoami)"
log "Operators namespace : $PROJECT_CPD_INST_OPERATORS"
log "Operands namespace  : $PROJECT_CPD_INST_OPERANDS"
log "WO CR name          : $WO_CR_NAME"
log "Timeout             : ${TIMEOUT_SECONDS}s ($(( TIMEOUT_SECONDS / 60 )) minutes)"
log "Poll every          : ${POLL_INTERVAL}s"
printf "\n"

# Verify WO CR exists
log "Checking WO CR '${WO_CR_NAME}' in namespace: $PROJECT_CPD_INST_OPERANDS"
if ! oc -n "$PROJECT_CPD_INST_OPERANDS" get wo "$WO_CR_NAME" &>/dev/null; then
  err "WO CR '${WO_CR_NAME}' not found in namespace '$PROJECT_CPD_INST_OPERANDS'."
  err "Set WO_CR_NAME env var if your CR has a different name."
  exit 1
fi

# Verify spec.version == 8.0.2
CR_VERSION=$(oc -n "$PROJECT_CPD_INST_OPERANDS" get wo "$WO_CR_NAME" \
  -o jsonpath='{.spec.version}' 2>/dev/null || true)
if [ "$CR_VERSION" != "$EXPECTED_CR_VERSION" ]; then
  err "WO CR '${WO_CR_NAME}' spec.version is '${CR_VERSION:-<empty>}', expected '${EXPECTED_CR_VERSION}'."
  err "This script is only valid for 5.4.2-Hotfix1 (CR version ${EXPECTED_CR_VERSION})."
  exit 1
fi
ok "WO CR spec.version: ${CR_VERSION}"

# Verify Hotfix label == 5.4.2-Hotfix1
CR_LABEL=$(oc -n "$PROJECT_CPD_INST_OPERANDS" get wo "$WO_CR_NAME" \
  -o jsonpath="{.metadata.labels.${EXPECTED_LABEL_KEY}}" 2>/dev/null || true)
if [ "$CR_LABEL" != "$EXPECTED_LABEL_VALUE" ]; then
  err "WO CR label '${EXPECTED_LABEL_KEY}' is '${CR_LABEL:-<not set>}', expected '${EXPECTED_LABEL_VALUE}'."
  err "Run the hotfix patch script first before running this verify script."
  exit 1
fi
ok "WO CR label: ${EXPECTED_LABEL_KEY}=${CR_LABEL}"
printf "\n"

# Returns sha256 digest from deployment spec image (same as oc get deploy -o yaml).
# Staging/mirrored registries remap digests at pull time, so pod imageID is unreliable.
# Falls back to pod imageID only when spec image is a tag (not a digest ref).
get_running_sha() {
  local deploy="$1"
  local ns="$2"

  # Primary: spec image digest (authoritative — matches oc get deploy output)
  local spec_image=""
  spec_image=$(oc -n "$ns" get deploy "$deploy" \
      -o jsonpath='{.spec.template.spec.containers[0].image}' \
      2>/dev/null || true)
  if [ "${spec_image#*@sha256:}" != "$spec_image" ]; then
    echo "${spec_image##*@sha256:}"
    return
  fi

  # Fallback: pod imageID (only when spec uses a tag, not a digest)
  local pod="" image_id=""
  pod=$(oc -n "$ns" get pods \
          --field-selector=status.phase=Running \
          -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' \
          2>/dev/null \
        | grep "^${deploy}-" | head -1 || true)
  [ -z "$pod" ] && pod=$(oc -n "$ns" get pods -l "app=${deploy}" \
          --field-selector=status.phase=Running \
          -o jsonpath='{.items[0].metadata.name}' 2>/dev/null || true)
  if [ -n "$pod" ]; then
    image_id=$(oc -n "$ns" get pod "$pod" \
        -o jsonpath='{.status.containerStatuses[0].imageID}' 2>/dev/null || true)
    [ -n "$image_id" ] && echo "${image_id##*@sha256:}" && return
  fi

  echo ""
}

# ---- Helper: print one status row --------------------------
print_row() {
  local deploy="$1"
  local status="$2"    # VERIFIED | NOT UPDATED | MISSING
  local current="$3"
  local expected="$4"

  local color
  case "$status" in
    VERIFIED)                color="$GREEN"  ;;
    "NOT UPDATED" | MISSING) color="$RED"    ;;
    *)                       color="$YELLOW" ;;
  esac

  local got_display=""
  [ -n "$current" ] && got_display="${current:0:16}..."

  printf "  %-44s ${color}%-14s${RESET}  expected: %.16s...  got: %s\n" \
    "$deploy" "$status" "$expected" "$got_display"
}

# ---- Helper: check one group of deployments ----------------
# Usage: check_group <namespace> <entries_array_ref>
# Sets globals: _GROUP_VERIFIED  _GROUP_PENDING  _GROUP_MISSING
check_group() {
  local ns="$1"
  local entries_ref="$2"   # name of the *_ENTRIES array

  _GROUP_VERIFIED=0
  _GROUP_PENDING=0
  _GROUP_MISSING=0

  local entry deploy expected current
  eval "local -a _entries=(\"\${${entries_ref}[@]}\")"
  for entry in "${_entries[@]}"; do
    deploy=$(entry_name "$entry")
    expected=$(entry_sha  "$entry")

    if ! oc -n "$ns" get deploy "$deploy" &>/dev/null; then
      print_row "$deploy" "MISSING" "" "$expected"
      _GROUP_MISSING=$(( _GROUP_MISSING + 1 ))
      continue
    fi

    current=$(get_running_sha "$deploy" "$ns")

    if [ "$current" = "$expected" ]; then
      print_row "$deploy" "VERIFIED" "$current" "$expected"
      _GROUP_VERIFIED=$(( _GROUP_VERIFIED + 1 ))
    else
      print_row "$deploy" "NOT UPDATED" "$current" "$expected"
      _GROUP_PENDING=$(( _GROUP_PENDING + 1 ))
    fi
  done
}

# ---- Helper: check one group of jobs -----------------------
# Jobs are verified on two criteria:
#   1. spec image SHA matches expected
#   2. status.succeeded >= 1 (job completed successfully)
# Sets globals: _GROUP_VERIFIED  _GROUP_PENDING  _GROUP_MISSING
check_jobs() {
  local ns="$1"
  local entries_ref="$2"   # name of the *_ENTRIES array

  _GROUP_VERIFIED=0
  _GROUP_PENDING=0
  _GROUP_MISSING=0

  local entry job expected spec_image current_sha succeeded
  eval "local -a _entries=(\"\${${entries_ref}[@]}\")"
  for entry in "${_entries[@]}"; do
    job=$(entry_name "$entry")
    expected=$(entry_sha  "$entry")

    if ! oc -n "$ns" get job "$job" &>/dev/null; then
      print_row "$job" "MISSING" "" "$expected"
      _GROUP_MISSING=$(( _GROUP_MISSING + 1 ))
      continue
    fi

    spec_image=$(oc -n "$ns" get job "$job" \
        -o jsonpath='{.spec.template.spec.containers[0].image}' \
        2>/dev/null || true)
    current_sha="${spec_image##*@sha256:}"
    # If spec image has no digest, treat sha as empty
    [ "$spec_image" = "$current_sha" ] && current_sha=""

    succeeded=$(oc -n "$ns" get job "$job" \
        -o jsonpath='{.status.succeeded}' \
        2>/dev/null || true)
    succeeded="${succeeded:-0}"

    if [ "$current_sha" = "$expected" ] && (( succeeded >= 1 )); then
      print_row "$job" "VERIFIED" "$current_sha" "$expected"
      _GROUP_VERIFIED=$(( _GROUP_VERIFIED + 1 ))
    else
      print_row "$job" "NOT COMPLETE" "$current_sha" "$expected"
      _GROUP_PENDING=$(( _GROUP_PENDING + 1 ))
    fi
  done
}

# ---- Main verification loop --------------------------------
START_TS=$(date +%s)
ITERATION=0

_GROUP_VERIFIED=0
_GROUP_PENDING=0
_GROUP_MISSING=0

while true; do
  ITERATION=$(( ITERATION + 1 ))
  NOW=$(date +%s)
  ELAPSED=$(( NOW - START_TS ))

  if (( ELAPSED >= TIMEOUT_SECONDS )); then
    printf "\n"
    err "Timeout of ${TIMEOUT_SECONDS}s reached after ${ELAPSED}s."
    err "Deployments that did NOT reach the expected SHA:"

    printf "  ${BOLD}Operators (${PROJECT_CPD_INST_OPERATORS}):${RESET}\n"
    for entry in "${OPERATOR_ENTRIES[@]}"; do
      d=$(entry_name "$entry")
      _exp=$(entry_sha "$entry")
      _sha=$(get_running_sha "$d" "$PROJECT_CPD_INST_OPERATORS")
      [ "$_sha" != "$_exp" ] && printf "    ${RED}%s${RESET}\n" "$d"
    done

    printf "  ${BOLD}Operands (${PROJECT_CPD_INST_OPERANDS}):${RESET}\n"
    for entry in "${OPERAND_ENTRIES[@]}"; do
      d=$(entry_name "$entry")
      _exp=$(entry_sha "$entry")
      _sha=$(get_running_sha "$d" "$PROJECT_CPD_INST_OPERANDS")
      [ "$_sha" != "$_exp" ] && printf "    ${RED}%s${RESET}\n" "$d"
    done

    printf "  ${BOLD}Jobs (${PROJECT_CPD_INST_OPERANDS}):${RESET}\n"
    for entry in "${JOB_ENTRIES[@]}"; do
      d=$(entry_name "$entry")
      _exp=$(entry_sha "$entry")
      _spec=$(oc -n "$PROJECT_CPD_INST_OPERANDS" get job "$d" \
          -o jsonpath='{.spec.template.spec.containers[0].image}' 2>/dev/null || true)
      _sha="${_spec##*@sha256:}"; [ "$_spec" = "$_sha" ] && _sha=""
      _succ=$(oc -n "$PROJECT_CPD_INST_OPERANDS" get job "$d" \
          -o jsonpath='{.status.succeeded}' 2>/dev/null || true)
      { [ "$_sha" != "$_exp" ] || (( ${_succ:-0} < 1 )); } && printf "    ${RED}%s${RESET}\n" "$d"
    done
    exit 1
  fi

  REMAINING=$(( TIMEOUT_SECONDS - ELAPSED ))
  TOTAL_DEPLOYMENTS=$(( ${#OPERATOR_ENTRIES[@]} + ${#OPERAND_ENTRIES[@]} + ${#JOB_ENTRIES[@]} ))

  printf "\n"
  printf "${BOLD}================================================================${RESET}\n"
  printf "${BOLD} 5.4.2-Hotfix1 Verification -- iteration #%s${RESET}\n" "$ITERATION"
  printf "${BOLD} Elapsed : %ss  |  Remaining: %ss${RESET}\n"            "$ELAPSED" "$REMAINING"
  printf "${BOLD}================================================================${RESET}\n"

  # ---- OPERATORS section ------------------------------------
  printf "\n"
  printf "${BOLD}${CYAN}  [OPERATORS]  namespace: %s${RESET}\n" "$PROJECT_CPD_INST_OPERATORS"
  printf "${BOLD}${CYAN}  %-44s %-14s  %s${RESET}\n" "Deployment" "Status" "SHA (first 16 chars)"
  printf "  %s\n" "----------------------------------------------------------------------"

  check_group "$PROJECT_CPD_INST_OPERATORS" OPERATOR_ENTRIES
  OP_VERIFIED=$_GROUP_VERIFIED
  OP_PENDING=$_GROUP_PENDING
  OP_MISSING=$_GROUP_MISSING
  OP_TOTAL="${#OPERATOR_ENTRIES[@]}"

  printf "\n"
  printf "  Operators summary -> ${GREEN}Verified: %s${RESET}  |  ${RED}Pending: %s${RESET}  |  ${YELLOW}Missing: %s${RESET}  |  Total: %s\n" \
    "$OP_VERIFIED" "$OP_PENDING" "$OP_MISSING" "$OP_TOTAL"

  # ---- OPERANDS section -------------------------------------
  printf "\n"
  printf "${BOLD}${CYAN}  [OPERANDS]   namespace: %s${RESET}\n" "$PROJECT_CPD_INST_OPERANDS"
  printf "${BOLD}${CYAN}  %-44s %-14s  %s${RESET}\n" "Deployment" "Status" "SHA (first 16 chars)"
  printf "  %s\n" "----------------------------------------------------------------------"

  check_group "$PROJECT_CPD_INST_OPERANDS" OPERAND_ENTRIES
  OD_VERIFIED=$_GROUP_VERIFIED
  OD_PENDING=$_GROUP_PENDING
  OD_MISSING=$_GROUP_MISSING
  OD_TOTAL="${#OPERAND_ENTRIES[@]}"

  printf "\n"
  printf "  Operands summary  -> ${GREEN}Verified: %s${RESET}  |  ${RED}Pending: %s${RESET}  |  ${YELLOW}Missing: %s${RESET}  |  Total: %s\n" \
    "$OD_VERIFIED" "$OD_PENDING" "$OD_MISSING" "$OD_TOTAL"

  # ---- JOBS section -----------------------------------------
  printf "\n"
  printf "${BOLD}${CYAN}  [JOBS]       namespace: %s${RESET}\n" "$PROJECT_CPD_INST_OPERANDS"
  printf "${BOLD}${CYAN}  %-44s %-14s  %s${RESET}\n" "Job" "Status" "SHA (first 16 chars)"
  printf "  %s\n" "----------------------------------------------------------------------"

  check_jobs "$PROJECT_CPD_INST_OPERANDS" JOB_ENTRIES
  JB_VERIFIED=$_GROUP_VERIFIED
  JB_PENDING=$_GROUP_PENDING
  JB_MISSING=$_GROUP_MISSING
  JB_TOTAL="${#JOB_ENTRIES[@]}"

  printf "\n"
  printf "  Jobs summary      -> ${GREEN}Verified: %s${RESET}  |  ${RED}Pending: %s${RESET}  |  ${YELLOW}Missing: %s${RESET}  |  Total: %s\n" \
    "$JB_VERIFIED" "$JB_PENDING" "$JB_MISSING" "$JB_TOTAL"

  # ---- Overall summary --------------------------------------
  TOTAL_VERIFIED=$(( OP_VERIFIED + OD_VERIFIED + JB_VERIFIED ))
  TOTAL_PENDING=$(( OP_PENDING + OD_PENDING + JB_PENDING ))
  TOTAL_MISSING=$(( OP_MISSING + OD_MISSING + JB_MISSING ))

  printf "\n"
  printf "${BOLD}----------------------------------------------------------------${RESET}\n"
  printf "${BOLD}  OVERALL  -> ${GREEN}Verified: %s${RESET}${BOLD}  |  ${RED}Pending: %s${RESET}${BOLD}  |  ${YELLOW}Missing: %s${RESET}${BOLD}  |  Total: %s${RESET}\n" \
    "$TOTAL_VERIFIED" "$TOTAL_PENDING" "$TOTAL_MISSING" "$TOTAL_DEPLOYMENTS"
  printf "${BOLD}----------------------------------------------------------------${RESET}\n"

  if (( TOTAL_PENDING == 0 && TOTAL_MISSING == 0 )); then
    ELAPSED=$(( $(date +%s) - START_TS ))
    printf "\n"
    ok "All ${TOTAL_DEPLOYMENTS} deployments updated to expected Hotfix1 SHA values."
    printf "\n"
    printf "${BOLD}================================================================${RESET}\n"
    printf "${GREEN}${BOLD}[%s] 5.4.2-Hotfix1 verification PASSED (completed in %ss)${RESET}\n" \
      "$(date '+%Y-%m-%d %H:%M:%S')" "$ELAPSED"
    printf "${BOLD}================================================================${RESET}\n"
    exit 0
  fi

  if (( TOTAL_MISSING > 0 )); then
    warn "${TOTAL_MISSING} deployment(s) not found. They may still be deploying."
  fi

  info "Next check in ${POLL_INTERVAL}s -- press Ctrl+C to abort."
  sleep "$POLL_INTERVAL"
done
```

</details>

Make the script executable
```bash
chmod 775 5.4.2-Hotfix1-verify.sh
```

Run the script
```bash
5.4.2-Hotfix1-verify.sh
```

Verify CR status and label
```bash
oc get wo -n "${PROJECT_CPD_INST_OPERANDS}" -o yaml | grep -i hotfix
      Hotfix: 5.4.2-Hotfix1
```

Wait 30 minutes for the changes to take effect

If verification is not successful, see the following Known issue section for guidance

---

#### Potential Issue - wo-archer-server-db-schema-job stuck after applying WxO hotfix

Confirm if archer-db-schema job is in Completed state
```bash
oc get job wo-archer-server-db-schema-job
NAME                             STATUS     COMPLETIONS   DURATION   AGE
wo-archer-server-db-schema-job   Complete   1/1           43h        4d6h
```

Run the following script only if the verification using the above script fails, indicating that the job did not run successfully when the hotfix was applied
```bash
vi wxo-hotfix-db-schema-job-unblock.sh
```

With the following contents

<details>
<summary>wxo-hotfix-db-schema-job-unblock.sh</summary>

```bash
#!/usr/bin/env bash
# =============================================================================
# wxo-hotfix0-db-schema-job-unblock.sh
#
# WORKAROUND: wo-archer-server-db-schema-job stuck after applying WxO hotfix-0
#             (operand version 8.0.2 / Patch 5)
#
# ROOT CAUSE
# ----------
# The hotfix triggers <cr-name>-archer-server-db-schema-job, which runs DDL
# statements (DROP TRIGGER, COMMENT ON COLUMN, ALTER TABLE) on the 'archer'
# Postgres database. These DDL statements require an ACCESS EXCLUSIVE lock on
# the target table. Orphaned archer-server connections left in "idle in
# transaction" state by previous pod restarts hold an ACCESS SHARE lock on the
# same table, blocking the DDL indefinitely — the job pod stays Running with
# no progress for hours or days.
#
# These orphaned connections are SQLAlchemy pool connections from
# <cr-name>-archer-server pods that were recycled during the hotfix rolling
# update. The application thread is gone but the server-side Postgres backend
# was never cleaned up.
#
# WHAT THIS SCRIPT DOES
# ---------------------
#  1. Requires NS (namespace) as mandatory input.
#  2. Auto-discovers the WatsonxOrchestrate CR name within that namespace.
#  3. Locates the EDB Postgres primary pod by the fixed pattern
#     <cr-name>-watson-orchestrate-postgresedb-1.
#  4. Reports the current job status and last log line.
#  5. Finds all orphaned "idle in transaction" archer-server connections on
#     the archer DB that have been open for more than 30 minutes.
#  6. DRY-RUN (default): prints the PIDs — no changes made.
#  7. --fix mode: terminates each connection via pg_terminate_backend(),
#     verifies the lock chain clears, then confirms the job resumes.
#
# SAFETY
# ------
# Terminating these connections causes zero data loss. Here is why:
#
# In practice, all orphaned connections observed across affected clusters held
# read-only SELECT transactions. However, even in the unlikely case where an
# orphaned connection holds an uncommitted write (INSERT/UPDATE/DELETE):
#
#   1. The pod that owned the connection has already been terminated. The
#      original HTTP request or Celery task has already failed with a connection
#      error — there is no live application code waiting on the result.
#
#   2. pg_terminate_backend() causes Postgres to issue an automatic ROLLBACK on
#      the open transaction before closing the connection. This is the correct
#      and intended outcome for any transaction whose owning process is dead.
#
#   3. Committed data is never affected — pg_terminate_backend() only rolls
#      back uncommitted work. An uncommitted write from a dead pod is not
#      "owned" data; it is a failed operation that must be retried by the
#      caller regardless of what this script does.
#
# The only observable side effect is that live archer-server pods may receive
# a transient connection error when SQLAlchemy checks out a stale connection
# from the pool; SQLAlchemy's pool_pre_ping reconnects automatically.
#
# USAGE
# -----
#   # NS is mandatory — set it to your WxO application namespace.
#   # CR name is auto-discovered; no need to set it manually.
#
#   # Dry-run — safe, no changes:
#   NS=<wxo-namespace> ./wxo-hotfix0-db-schema-job-unblock.sh
#
#   # Apply the fix:
#   NS=<wxo-namespace> ./wxo-hotfix0-db-schema-job-unblock.sh --fix
#
#   # Examples:
#   NS=cpd-instance-1 ./wxo-hotfix0-db-schema-job-unblock.sh
#   NS=cpd-instance-1 ./wxo-hotfix0-db-schema-job-unblock.sh --fix
#
# REQUIREMENTS
# ------------
#   - bash 4.0+
#   - oc CLI, logged in with cluster-admin or equivalent RBAC
#   - oc exec access to the EDB Postgres pod
#
# =============================================================================

set -euo pipefail

# ── Colour helpers ────────────────────────────────────────────────────────────
RED='\033[0;31m'; YELLOW='\033[1;33m'; GREEN='\033[0;32m'
CYAN='\033[0;36m'; BOLD='\033[1m'; NC='\033[0m'
info()    { echo -e "${CYAN}[INFO]${NC}  $*"; }
warn()    { echo -e "${YELLOW}[WARN]${NC}  $*"; }
success() { echo -e "${GREEN}[OK]${NC}    $*"; }
error()   { echo -e "${RED}[ERROR]${NC} $*" >&2; }
header()  { echo -e "\n${BOLD}=== $* ===${NC}"; }

# ── Argument parsing ──────────────────────────────────────────────────────────
FIX_MODE=false
for arg in "$@"; do
  case $arg in
    --fix)    FIX_MODE=true ;;
    --help|-h)
      grep "^# " "$0" | sed 's/^# \?//'
      exit 0 ;;
    *) error "Unknown argument: $arg  (use --fix or --help)"; exit 1 ;;
  esac
done

# ── Step 1: Namespace validation & CR name discovery ─────────────────────────
header "Step 1: Namespace validation & CR name discovery"

# NS is mandatory — customer clusters use varying namespace names.
if [[ -z "${NS:-}" ]]; then
  error "NS is not set. Please provide the WxO application namespace."
  error ""
  error "Usage:  NS=<wxo-namespace> $0 [--fix]"
  error ""
  error "To find your namespace:"
  error "  oc get watsonxorchestrate --all-namespaces"
  exit 1
fi

# Verify the namespace exists and is accessible.
if ! oc get namespace "$NS" &>/dev/null; then
  error "Namespace '$NS' not found or not accessible."
  error "Verify you are logged in to the correct cluster and NS is correct."
  exit 1
fi

# Auto-discover CR name — customers may override the default 'wo'.
CR=$(oc get watsonxorchestrate -n "$NS" \
       -o jsonpath='{.items[0].metadata.name}' 2>/dev/null || true)
if [[ -z "$CR" ]]; then
  error "No WatsonxOrchestrate CR found in namespace '$NS'."
  error "Verify NS is the correct WxO application namespace."
  exit 1
fi

info "Namespace : $NS"
info "CR name   : $CR"

# ── Step 2: Postgres primary pod discovery ────────────────────────────────────
header "Step 2: Postgres primary pod discovery"

# The EDB primary pod name follows the fixed pattern:
#   <cr-name>-watson-orchestrate-postgresedb-1
# The -1 suffix always identifies the primary in EDB operator naming.
# Constructing the exact name (rather than grepping) avoids false matches
# against any unrelated Postgres instances in the same namespace.
PG_POD="${CR}-watson-orchestrate-postgresedb-1"

if ! oc get pod "$PG_POD" -n "$NS" &>/dev/null; then
  error "Postgres primary pod '$PG_POD' not found in namespace '$NS'."
  error "Expected pod name: <cr-name>-watson-orchestrate-postgresedb-1"
  error "Verify the EDB Postgres cluster is running:"
  error "  oc get pods -n $NS | grep postgresedb"
  exit 1
fi

PG_PHASE=$(oc get pod "$PG_POD" -n "$NS" \
             -o jsonpath='{.status.phase}' 2>/dev/null || echo "Unknown")
if [[ "$PG_PHASE" != "Running" ]]; then
  error "Postgres primary pod '$PG_POD' is not Running (phase: $PG_PHASE)."
  error "The Postgres cluster must be healthy before running this script."
  exit 1
fi

info "Postgres primary pod: $PG_POD (phase: $PG_PHASE)"

# Helper — run SQL, return raw -tAq output (no headers, no alignment)
pg_exec() {
  oc exec -n "$NS" "$PG_POD" -- \
    psql -U postgres -tAq -c "$1" 2>/dev/null
}

# ── Step 3: Job status ────────────────────────────────────────────────────────
header "Step 3: ${CR}-archer-server-db-schema-job status"

JOB_NAME="${CR}-archer-server-db-schema-job"

if ! oc get job "$JOB_NAME" -n "$NS" &>/dev/null; then
  warn "Job '$JOB_NAME' not found in namespace '$NS'."
  warn "It may have already completed or not yet been triggered."
  warn "Continuing to check for orphaned connections anyway..."
else
  ACTIVE=$(oc get job "$JOB_NAME" -n "$NS" \
             -o jsonpath='{.status.active}' 2>/dev/null || echo "0")
  SUCCEEDED=$(oc get job "$JOB_NAME" -n "$NS" \
                -o jsonpath='{.status.succeeded}' 2>/dev/null || echo "0")
  FAILED_CNT=$(oc get job "$JOB_NAME" -n "$NS" \
                 -o jsonpath='{.status.failed}' 2>/dev/null || echo "0")
  START_TIME=$(oc get job "$JOB_NAME" -n "$NS" \
                 -o jsonpath='{.status.startTime}' 2>/dev/null || echo "")

  info "Job status — active=$ACTIVE  succeeded=$SUCCEEDED  failed=$FAILED_CNT"
  [[ -n "$START_TIME" ]] && info "Started at: $START_TIME"

  if [[ "$SUCCEEDED" == "1" ]]; then
    success "Job has already completed successfully. No action needed."
    exit 0
  fi

  if [[ "$ACTIVE" == "1" ]]; then
    STUCK_POD=$(oc get pods -n "$NS" \
                  --selector="job-name=${JOB_NAME}" \
                  --field-selector=status.phase=Running \
                  -o jsonpath='{.items[0].metadata.name}' 2>/dev/null || true)
    if [[ -n "$STUCK_POD" ]]; then
      warn "Job pod is running: $STUCK_POD"
      LAST_LOG=$(oc logs "$STUCK_POD" -n "$NS" --tail=1 2>/dev/null \
                 | grep -v "^$" || true)
      info "Last log line: ${LAST_LOG:-<no output yet>}"
    fi
  fi
fi

# ── Step 4: Orphaned connection discovery ─────────────────────────────────────
header "Step 4: Orphaned 'idle in transaction' connections (archer DB)"

info "Querying pg_stat_activity for archer-server connections idle in transaction > 30 min..."

# application_name is set by SQLAlchemy via get_application_name() in wxo-server.
# Four variants are possible depending on SERVER_TYPE and sync/async engine:
#   wxo-server                      — FastAPI sync engine (archer-server pod)
#   async-wxo-server                — FastAPI async engine (archer-server pod)
#   conversation-controller         — Celery sync engine (SERVER_TYPE=CELERY)
#   async-conversation-controller   — Celery async engine (SERVER_TYPE=CELERY)
# All variants append ' - <ip>:<port>', e.g. 'async-wxo-server - 10.0.0.1:12345'.
# The regex below matches all four prefixes regardless of IP:port suffix.
ORPHAN_QUERY="
SELECT pid,
       application_name,
       EXTRACT(EPOCH FROM (now() - query_start))::bigint AS age_seconds,
       left(regexp_replace(query, E'[\n\r]+', ' ', 'g'), 80) AS query_snippet
FROM pg_stat_activity
WHERE datname = 'archer'
  AND state = 'idle in transaction'
  AND application_name ~ '^(wxo-server|async-wxo-server|conversation-controller|async-conversation-controller) - '
  AND query_start < now() - interval '30 minutes'
ORDER BY query_start ASC;"

# Collect PIDs into array for termination loop; also build display rows
declare -a ORPHAN_PIDS=()
ORPHAN_DISPLAY=""

while IFS='|' read -r pid app age_sec query; do
  pid=$(echo "$pid" | tr -d ' ')
  age_sec=$(echo "$age_sec" | tr -d ' ')
  [[ -z "$pid" || ! "$pid" =~ ^[0-9]+$ ]] && continue
  ORPHAN_PIDS+=("$pid")
  age_human=$(printf '%dd %02dh %02dm' \
    $((age_sec/86400)) $(((age_sec%86400)/3600)) $(((age_sec%3600)/60)))
  ORPHAN_DISPLAY+=$(printf "  %-10s %-38s %-14s %s\n" \
    "$pid" "${app:0:38}" "$age_human" "${query:0:55}")
  ORPHAN_DISPLAY+=$'\n'
done < <(pg_exec "$ORPHAN_QUERY")

ORPHAN_COUNT=${#ORPHAN_PIDS[@]}

if [[ $ORPHAN_COUNT -eq 0 ]]; then
  success "No orphaned idle-in-transaction archer-server connections found."
  info "If the job is still stuck, check for other blockers:"
  echo ""
  echo "  oc exec -n $NS $PG_POD -- psql -U postgres -c \\"
  echo "  \"SELECT blocked.pid, left(blocked.query,60) AS blocked_q,"
  echo "          blocker.pid AS blocker_pid, blocker.state,"
  echo "          blocker.application_name"
  echo "   FROM pg_stat_activity blocked"
  echo "   JOIN pg_stat_activity blocker"
  echo "     ON blocker.pid = ANY(pg_blocking_pids(blocked.pid))"
  echo "   WHERE cardinality(pg_blocking_pids(blocked.pid)) > 0;\""
  exit 0
fi

warn "Found $ORPHAN_COUNT orphaned connection(s) — all hold READ-ONLY transactions:"
echo ""
printf "  %-10s %-38s %-14s %s\n" "PID" "APPLICATION" "IDLE DURATION" "QUERY (truncated)"
printf "  %-10s %-38s %-14s %s\n" "----------" "--------------------------------------" \
  "--------------" "-------------------------------------------------------"
echo -n "$ORPHAN_DISPLAY"
echo ""

# ── Dry-run gate ──────────────────────────────────────────────────────────────
if [[ "$FIX_MODE" == false ]]; then
  echo -e "${YELLOW}DRY-RUN MODE — no changes made.${NC}"
  echo ""
  echo "All $ORPHAN_COUNT connection(s) above can be safely terminated."
  echo "They hold read-only SELECT transactions — zero data loss on termination."
  echo ""
  echo "To apply the fix:"
  echo "  NS=$NS $0 --fix"
  exit 0
fi

# ── Step 5: Terminate orphaned connections ────────────────────────────────────
header "Step 5: Terminating $ORPHAN_COUNT orphaned connection(s)"

TERMINATED=0
NOT_FOUND=0

for pid in "${ORPHAN_PIDS[@]}"; do
  info "Terminating PID $pid..."
  RESULT=$(pg_exec "SELECT pg_terminate_backend($pid);" || echo "error")
  RESULT=$(echo "$RESULT" | tr -d ' \n')
  if [[ "$RESULT" == "t" ]]; then
    success "PID $pid terminated."
    ((TERMINATED++))
  else
    warn "PID $pid: returned '$RESULT' (may already have exited — safe to ignore)."
    ((NOT_FOUND++))
  fi
done

echo ""
info "Terminated: $TERMINATED  |  Already gone: $NOT_FOUND"

# ── Step 6: Verify lock chain cleared ─────────────────────────────────────────
header "Step 6: Verifying lock chain is cleared"

sleep 3

REMAINING=$(pg_exec "
SELECT count(*)
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocker
  ON blocker.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE cardinality(pg_blocking_pids(blocked.pid)) > 0
  AND blocked.datname = 'archer';" || echo "?")
REMAINING=$(echo "$REMAINING" | tr -d ' ')

if [[ "$REMAINING" == "0" ]]; then
  success "Lock chain cleared — no blocking sessions remain on archer DB."
else
  warn "$REMAINING blocking session(s) still present on archer DB."
  warn "There may be additional blockers not matching the archer-server pattern."
  warn "Run the full lock chain query to investigate:"
  echo ""
  echo "  oc exec -n $NS $PG_POD -- psql -U postgres -c \\"
  echo "  \"SELECT blocked.pid, left(blocked.query,60) AS blocked_q,"
  echo "          blocker.pid AS blocker_pid, blocker.state,"
  echo "          blocker.application_name"
  echo "   FROM pg_stat_activity blocked"
  echo "   JOIN pg_stat_activity blocker"
  echo "     ON blocker.pid = ANY(pg_blocking_pids(blocked.pid))"
  echo "   WHERE cardinality(pg_blocking_pids(blocked.pid)) > 0;\""
fi

# ── Step 7: Job progress confirmation ─────────────────────────────────────────
header "Step 7: Confirming job progress"

sleep 5

STUCK_POD=$(oc get pods -n "$NS" \
              --selector="job-name=${JOB_NAME}" \
              --field-selector=status.phase=Running \
              -o jsonpath='{.items[0].metadata.name}' 2>/dev/null || true)

if [[ -n "$STUCK_POD" ]]; then
  NEW_LOG=$(oc logs "$STUCK_POD" -n "$NS" --tail=1 2>/dev/null \
              | grep -v "^$" || true)
  info "Latest log from $STUCK_POD:"
  echo "  ${NEW_LOG:-<no output yet>}"
  echo ""
  info "Monitor completion (blocks until done):"
  echo "  oc wait job/${JOB_NAME} -n $NS \\"
  echo "    --for=condition=Complete --timeout=3600s"
else
  JOB_DONE=$(oc get job "$JOB_NAME" -n "$NS" \
               -o jsonpath='{.status.conditions[?(@.type=="Complete")].status}' \
               2>/dev/null || true)
  if [[ "$JOB_DONE" == "True" ]]; then
    success "Job completed successfully!"
  else
    info "Job pod not yet visible — may be restarting after lock cleared."
    info "Re-check in ~30s:  oc get job $JOB_NAME -n $NS"
  fi
fi

echo ""
success "Workaround applied."
info "Once the job completes, WxO reconciliation resumes automatically."
info "Expected final CR state: RECONCILE_PROGRESS=100%  READY=True"
info "Monitor with:"
echo "  oc get watsonxorchestrate $CR -n $NS"
```

</details>

Make the script executable
```bash
chmod 775 wxo-hotfix-db-schema-job-unblock.sh
```

Run the script
```bash
./wxo-hotfix-db-schema-job-unblock.sh
```

---

Make sure to add the resource configurations from rediscp instance in the 5.4.2 wo custom resource within the spec section
```bash
spec:
    redis_resources:
      limits:
        cpu: "2"
        ephemeral-storage: 1Gi
        memory: 50Gi
      requests:
        cpu: "1"
        ephemeral-storage: 10Mi
        memory: 40Gi
    persistentVolume:
      accessModes:
      - ReadWriteOnce
      size: 150Gi
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

Monitor watsonx_ai upgrade
```bash
watch -n 3 'oc get po -A -owide | egrep -v "([0-9])/\1" | egrep -v "Completed" && oc get ccs,wmlbase,ws,NotebookRuntime,watsonxai'
```

---

#### Potential Issue - WML Operator PVC Sizing and Memory Issues

During wx_ai upgrade, the WML operator can encounter an error related to PVC sizing and memory

Monitor the WML operator logs and yaml for similar symptoms as the previous IFM operator issue
```bash
oc logs ibm-cpd-wml-operator-6d5b5f795b-x258l -n ups-wx-operators | grep -i error
```

Monitor the WML operator yaml for similar symptoms
```bash
oc describe po ibm-cpd-wml-operator-6d5b5f795b-x258l -n ups-wx-operators
```

This script addresses Watson Machine Learning (WML) job memory exhaustion and undersized storage volumes by restarting the WML operator pod and modifying its internal templates to increase default job memory limits to 1Gi and PVC storage capacities to 100Gi

Confirm the script exists in this location on the bastion node and then run the WML workaround script 
```bash
/ibm/wml-pvc-template-hotfix-COMPLETE-5.4.2.sh
```

Monitor the WML operator logs to ensure that the PVC and memory issue(s) are addressed
```bash
oc logs ibm-cpd-wml-operator-6d5b5f795b-x258 -n ups-wx-operators
```

Check for any errors in the operator pod yaml directly
```bash
oc describe po ibm-cpd-wml-operator-6d5b5f795b-x258 -n ups-wx-operators
```

Monitor the watsonxai relevant custom resources
```bash
watch -n 3 'oc get po -A -owide | egrep -v "([0-9])/\1" | egrep -v "Completed" && oc get ccs,wmlbase,ws,NotebookRuntime,watsonxai'
```

Monitor wml reconciliation
```bash
oc get wmlbase wml-cr -n ups-wx-operands -o custom-columns="STATUS:.status.wmlStatus,PROGRESS:.status.progress,MESSAGE:.status.progressMessage,RUNNING:.status.conditions[?(@.type==\"Running\")].status,FAILURE:.status.conditions[?(@.type==\"Failure\")].status"

```

Monitor wx_ai reconciliation
```bash
oc get watsonxai watsonxai-cr -n ups-wx-operands -o custom-columns="STATUS:.status.watsonxaiStatus,PROGRESS:.status.progress,MESSAGE:.status.progressMessage"
```

---

Check the watsonxai custom resource status
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=watsonx_ai
```

---

#### Upgrade Watsonx Governance

Update the install-options.yml file in the cpd-cli-workspace/olm-utils-workspace/work directory
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

Review and remove the image_digests section from woservice aiopenscale custom resource
```bash
oc patch woservice aiopenscale -n ups-wx-operands --type='json' -p='[{"op": "remove", "path": "/spec/image_digests"}]'
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
watch -n 3 'oc get po -A -owide | egrep -v "([0-9])/\1" | egrep -v "Completed" && oc get woservice,Db2aaserviceService,openpagesinstances,watsonxaiifm,watsonxgovernance'
```

Check the watsonx governance custom resource stauts
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=watsonx_governance
```

---

#### Upgrade Watson Speech

Upgrade Watson Speech
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

Monitor watson_speech upgrade
```bash
watch -n 3 'oc get po -A -owide | grep -E -v "([0-9])/\1" | grep -E -v "Completed" && oc get watsonspeech speech-cr -o yaml | grep progress'
```

Check the watson_speech custom resource status
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=watson_speech
```

After upgrading speech update the speech-cr, adding this doNotManage section
```bash
spec:
  global:
    doNotManage:
    - configmap/speech-cr-stt-runtime
    - configmap/speech-cr-tts-runtime
```

---

#### Upgrade Voice Gateway

Upgrade Voice Gateway
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
watch -n 3 'oc get po -A -owide | grep -E -v "([0-9])/\1" | grep -E -v "Completed" && oc get voicegateway voicegateway-cr -o yaml | grep -A 10 status'
```

Check the voice_gateway custom resource status
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=voice_gateway
```

---

#### Upgrade Analytics Engine

Upgrade Analytics Engine service
```bash
cpd-cli manage install-components \
--license_acceptance=true \
--components=analyticsengine \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET} \
--upgrade=true
```

Monitor Analytics Engine upgrade
```bash
watch -n 3 'oc get po -A -owide | grep -E -v "([0-9])/\1" | grep -E -v "Completed" && oc get analyticsengine -o yaml | grep progress'
```

Check the Analytics Engine custom resource status
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=analyticsengine
```

---

#### Upgrade Db2oltp

Upgrade Db2 service
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
watch -n 3 'oc get po -A -owide | grep -E -v "([0-9])/\1" | grep -E -v "Completed" && oc get Db2oltpService db2oltp-cr -o yaml | grep progress'
```

Check the db2oltp custom resource status
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=db2oltp
```

---

#### Upgrade Cognos Analytics

Before starting Cognos upgrade, ensure that the 'spec.enableInstanaMetricCollection' field is set to 'false' in the CAService custom resource
```bash
oc get CAService ca-addon-cr  -o yaml | grep -A 10 enableInstanaMetricCollection
```

**Note**: If 'enableInstanaMetricCollection' is set to 'true' this can prevent the Cognos Analytics upgrade from completing, awaiting confirmation from Development if we can set 'enableInstanaMetricCollection' to 'false' prior to upgrade of Cognos Analytics

Upgrade Cognos Analytics
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
watch -n 3 'oc get po -A -owide | grep -E -v "([0-9])/\1" | grep -E -v "Completed" && oc get caservices ca-addon-cr -o yaml | grep progress'
```

Check the Cognos analytics custom resource status
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=cognos_analytics
```

---

#### Potential Issue - Cognos Analytics upgrade blocked ibm-cognos-addon-sp-deployment in CrashLoopBackOff

During the UPS non-prod upgrade of Cognos Analytics, we encountered an issue with the ibm-cognos-addon-sp-deployment in CrashLoopBackOff

The permanent fix has been delivered to 31.0.0 branch which is targeted for CPD 6.0.0 release, meanwhile here is the documentation for the workaround

Fix the CrashLoopBackOff by first patching the caservice custom resource
```bash
oc patch caservices ca-addon-cr \
  -n <NAMESPACE> \
  --type=merge \
  -p '{"spec":{"enableInstanaMetricCollection": false}}'
```

Wait for the pod to recover
```bash
oc get pods -n <NAMESPACE> -l app=ibm-cognos-addon-sp -w
# Wait for: ibm-cognos-addon-sp-deployment-xxx   1/1   Running   0
```

Wait for CAService to complete
```bash
oc get caservices ca-addon-cr -n <NAMESPACE> -w
# Wait for STATUS: Completed, PROGRESS: 100%
```

Set environment variables
```bash
NAMESPACE=ups-wx-operands
```

Get admin credentials
```bash
# Check IAM mode
isIAMEnabled=$(oc get zenservice lite-cr -n ${NAMESPACE} \
  -o jsonpath={.spec.iamIntegration})
echo "IAM enabled: $isIAMEnabled"

# If true:
ca_password=$(oc -n ${NAMESPACE} get secret platform-auth-idp-credentials \
  -o jsonpath='{.data.admin_password}' | base64 --decode)
ca_user=$(oc -n ${NAMESPACE} get secret platform-auth-idp-credentials \
  -o jsonpath='{.data.admin_username}' | base64 --decode)

# If false:
ca_password=$(oc get secret admin-user-details \
  -o jsonpath='{.data.initial_admin_password}' -n ${NAMESPACE} | base64 --decode)
ca_user="admin"

echo "User: $ca_user  Pass length: ${#ca_password}"
```

Get a bearer token from inside the sp pod
```bash
SP_CONTAINER=$(oc get po -n ${NAMESPACE} | grep ibm-cognos-addon-sp | awk '{print $1}')
INSTANCE_ID=$(oc get po -n ${NAMESPACE} | grep artifacts | cut -d '-' -f1 | cut -d 'a' -f2)

echo "SP pod:      $SP_CONTAINER"
echo "Instance ID: $INSTANCE_ID"

# Get token via internal nginx (works on both IAM and non-IAM clusters)
BEARER=$(oc exec -n ${NAMESPACE} -it ${SP_CONTAINER} -- \
  curl -s -k -X POST \
  -H 'Content-Type: application/json' \
  -d "{\"username\":\"${ca_user}\",\"password\":\"${ca_password}\"}" \
  "https://internal-nginx-svc.${NAMESPACE}.svc.cluster.local:12443/icp4d-api/v1/authorize" \
  | jq -r '.token')

JWT_TOKEN="Authorization: Bearer $BEARER"
echo "Token acquired: ${BEARER:0:30}..."
# Must start with eyJ — if empty, check credentials from Step 2
```

Verify current Zen status
```bash
ZENURL="https://zen-core-api-svc.${NAMESPACE}.svc:4444/v3/service_instances"

before_status=$(oc exec -n ${NAMESPACE} -it ${SP_CONTAINER} -- \
  curl -s -L "${ZENURL}/${INSTANCE_ID}" -H "${JWT_TOKEN}" -k)

echo $before_status | jq -r '.service_instance | {provision_status, addon_version}'
```

Patch the CAServiceInstance CR
```bash
oc patch CAServiceInstance ca${INSTANCE_ID}-cr \
  --type merge \
  -p '{"spec":{"version":"30.0.4"}}' \
  -n ${NAMESPACE}
```

Update Zen service instance metadata to UPGRADED
```bash
oc exec -n ${NAMESPACE} -it ${SP_CONTAINER} -- \
  curl -s -X PATCH -L "${ZENURL}/${INSTANCE_ID}/meta" \
  -H "${JWT_TOKEN}" -k \
  -H 'Content-Type: application/json' \
  -d '{"provision_status": "UPGRADED", "service_instance_version": "30.0.4"}'
```

Verify the cognos instance status
```bash
cpd-cli service-instance list \
  --profile=${CPD_PROFILE_NAME} \
  --service-type=cognos-analytics-app
```

Expected result
```bash
Expected: Version: 30.0.4  |  Provision status: UPGRADED  |  Upgrade version option: []
```

**Permanent Fix**: Fixed in ibm-cognos-addon-sp:2.2.8 (digest sha256:450720b2...), shipping in CA 31.0.0, the OpenTelemetry dependencies have been updated for Node.js v22 compatibility

Once 31.0.0 is available, or if a hotfix digest is provided for 30.0.4, apply it via
```bash
oc patch caservices ca-addon-cr -n <NAMESPACE> --type=merge \
  -p '{"spec":{"hotfix_digests":{"ibm_cognos_addon_sp":"sha256:450720b26835a99f7853063f4fd39b1e03e4e6beae1c7d67f626684009768192"}}}'
After the hotfix image is running and confirmed healthy, enableInstanaMetricCollection can be safely re-enabled if required.
```

---

#### Potential Issue - Cognos post-ca-translations-job pod ImagePullBackOff errors

Post upgrade of the cognos_analytics service, you may find a post-ca-translations-job and it's corresponding pod in error
```bash
oc get job | grep post-ca-translations
post-ca-translations-job   ImagePullBackOff   0/1

oc get po |  grep post-ca-translations
post-ca-translations-job-2kcwg   ImagePullBackOff   0/1
```

The describe of the pod yields various manifest unknown errors
```bash
cp.icr.io/cp/cpd/ca-cpd-addon-translation@sha256:347d49e0e6c457ab5d2fe353c8dec7e6ea01dc7159193753b62396fd0ed69a64: reading manifest
sha256:347d49e0e6c457ab5d2fe353c8dec7e6ea01dc7159193753b62396fd0ed69a64 in cp.icr.io/cp/cpd/ca-cpd-addon-translation: manifest unknown; artifact err: get manifest: build image source:
reading manifest sha256:347d49e0e6c457ab5d2fe353c8dec7e6ea01dc7159193753b62396fd0ed69a64 in cp.icr.io/cp/cpd/ca-cpd-addon-translation: manifest unknown
```

Verify and obtain the correct image digest from the cloud-pak github repo
```bash
https://github.com/IBM/cloud-pak/blob/master/repo/case/ibm-cognos-analytics-prod/30.0.4%2B20260721.161054.10660/OLM/images.txt
```

```bash
cp.icr.io/cp/cpd/ca-cpd-addon-translation@sha256:8a7920108d14c5d617e0f187fc01bd08e19dd6b88a4e2c07c5f1dbb41aadd970
```

Patch the caservice-cr within the spec/hotfix_digests/ca_cpd_addon_translation section with the above image digest
```bash
oc patch caservice ca-addon-cr -n ${PROJECT_CPD_INST_OPERANDS} --type merge \
  -p '{
    "spec": {
      "hotfix_digests": {
        "ca_cpd_addon_translation": "sha256:8a7920108d14c5d617e0f187fc01bd08e19dd6b88a4e2c07c5f1dbb41aadd970"
      }
    }
  }'
```

Recycle the post-ca-translations-job pod and monitor the new pod spins up with the updated image

---


## Upgrade Service Instances

After upgrading service custom resources, some services require additional instance upgrades

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
cpd-cli service-instance upgrade --service-type=spark --profile=${CPD_PROFILE_NAME} --all
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

Repeat the preceding steps to upgrade each service instance associated with this instance of IBM Software Hub

---

#### Potential Issue - Db2oltp Instance Version Not Updating

During testing, Db2oltp service instance version was not being updated to the latest version, 12.1.5.0-cn1-amd64

This was due to a mismatch in the values of the zen-database-core configmap and db2oltp json inside the zen-database-core pod

Restart the zen-database-core pod and then once the zen-database-core pod is running, re-run the upgrade for db2oltp instance with the mismatched version

---

#### Upgrade Cognos Analytics service instances

Set the INSTANCE_VERSION environment variable to the version that corresponds to the version of IBM Software Hub on your cluster
```bash
export INSTANCE_VERSION=30.0.4
```

Upgrade the service instances
```bash
cpd-cli service-instance upgrade --service-type=cognos-analytics-app --profile=${CPD_PROFILE_NAME} --version=${INSTANCE_VERSION} --all
```

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
    "value": [ 
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
    ]
  }
]'
```

Upgrade the cpdbr-tenant service

The command that you run depends on where your cluster pulls images from

Run the following command if you are using a private container registry and the scheduling service is not installed on the cluster
```bash
cpd-cli oadp install --component=cpdbr-tenant --namespace=${OADP_PROJECT} --tenant-operator-namespace=${PROJECT_CPD_INST_OPERATORS} --private-registry-location=${PRIVATE_REGISTRY_LOCATION} --upgrade=true --log-level=debug --verbose
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
oc create rolebinding bros-job-sa-rb-${BR_OPERATOR_JOB_SA} --clusterrole=edit --serviceaccount=${PROJECT_INST_BR_SVC}:${BR_OPERATOR_JOB_SA} -n ${PROJECT_CPD_INST_OPERATORS}
oc label rolebinding bros-job-sa-rb-${BR_OPERATOR_JOB_SA} -n ${PROJECT_CPD_INST_OPERATORS} component-id=br-orchestration icpdsupport/addOnId=bros

# Assign the edit role in the operands project
# =======================================================================================
oc create rolebinding bros-job-sa-rb-${BR_OPERATOR_JOB_SA} --clusterrole=edit --serviceaccount=${PROJECT_INST_BR_SVC}:${BR_OPERATOR_JOB_SA} -n ${PROJECT_CPD_INST_OPERANDS}
oc label rolebinding bros-job-sa-rb-${BR_OPERATOR_JOB_SA} -n ${PROJECT_CPD_INST_OPERANDS} component-id=br-orchestration icpdsupport/addOnId=bros

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

#### Potential Issue - Chat with docs cleanup job fails due to insufficient memory after upgrade

After upgrading 5.4.0 Patch 5, the wo-chat-with-docs-expiry-cronjob pod fails with an OOMKilled error

The pod's memory limit is set to 200Mi, which may be insufficient when processing multiple knowledge bases or chat-with-docs resources that need to be deleted

The job performs several memory-intensive operations including:
- Multiple Postgres queries and updates to remove knowledge bases and related sub-resources
- Milvus vector store cleanup operations
- S3 document deletion
- HTTP requests to TRM (Tools Runtime Manager) to remove tool deployments

To resolve this issue, increase the memory limit for the chat with docs expiry cronjob
```bash
oc patch cronjob wo-chat-with-docs-expiry-cronjob \
-n ${PROJECT_CPD_INST_OPERANDS} \
--type='json' \
-p='[
  {
    "op": "add",
    "path": "/spec/jobTemplate/spec/template/spec/containers/0/resources",
    "value": {
      "requests": {
        "memory": "512Mi"
      },
      "limits": {
        "memory": "1Gi"
      }
    }
  }
]'
```

---

#### Potential Issue - Enable WxO Observability

WxO development team to provide the new procedure to enable WxO Observability

**Reference**: [Enable WxO Observability](https://github.com/kuanalex/ups/blob/main/WxO_Observability_PROD_Runbook.md)

Check the Orchestrate custom resource yaml for this particular configuration
```bash
oc get wo wo -o yaml
```

Expected output
```bash
spec:
    agentops:
      enabled: true
```

---

#### Potential Issue - Enabling Watson Speech services to process API requests on multiple clusters

**Reference**: [Enabling Watson Speech services to process API requests on multiple clusters](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=pis-enabling-watson-speech-services-process-api-requests-multiple-clusters)

You can configure Watson Speech services for an active-active multi-cluster deployment, enabling API requests to be processed across multiple clusters

To enable an active-active multi-cluster deployment topology, you must edit the Watson Speech services custom resource to
- Enable active-active mode
- Specify the Version 4 universally unique identifier (UUID) that you want to use

Set your ACTIVE_ACTIVE_SEED environment variable to the UUID (use 'f07e930b-a471-4990-b53b-47a5ed7dcc18' for both PROD-East and PROD-Central)
```bash
export ACTIVE_ACTIVE_SEED=f07e930b-a471-4990-b53b-47a5ed7dcc18
```

Login to the cluster
```bash
${OC_LOGIN}
```

Set the INSTANCE to the name of the Watson Speech services custom resource:
```bash
export INSTANCE=$(oc get watsonspeech -n=${PROJECT_CPD_INST_OPERANDS} | grep -v NAME | awk '{print $1}')
```

Patch the custom resource to enable active-active mode and specify the UUID:
```bash
oc patch WatsonSpeech ${INSTANCE} \
 --namespace=${PROJECT_CPD_INST_OPERANDS} \
 --type=merge \
 -p "{\"spec\":{\"global\":{\"activeActiveSeed\":\"${ACTIVE_ACTIVE_SEED}\",\"activeActiveEnabled\":true}}}"
```

Wait for the Watson Speech customization pods to restart and the custom resource to reach Completed state

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

For any other issues with Watsonx Orchestrate components, you can run the check_orchestrate_health utility, found here
```bash
wget -O check_orchestrate_health_v12.sh https://raw.githubusercontent.com/watson-developer-cloud/community/master/watsonx-orchestrate/scripts/check_orchestrate_health_v12.sh ; sh check_orchestrate_health_v12.sh -t
```

Validate 'expose:external-regional' label in the cpd route, add the label "expose:external-regional" to your cpd-route as required
```bash
oc get route cpd -n ${PROJECT_CPD_INST_OPERANDS} -o yaml | grep -A 20 labels
```

---

**End of Runbook**
