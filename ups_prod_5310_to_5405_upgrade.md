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
oc get wo -n "${PROJECT_CPD_INST_OPERANDS}" -o yaml | grep -i hotfix
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
wo     5.4.2     45         45         45      agentic_assistant   NOT_QUIESCED   100%                 Xd
```

**Important**: For any other issues with Watsonx Orchestrate components, you can run the check_orchestrate_health utility, found here
```bash
wget -O check_orchestrate_health_v12.sh https://raw.githubusercontent.com/watson-developer-cloud/community/master/watsonx-orchestrate/scripts/check_orchestrate_health_v12.sh ; sh check_orchestrate_health_v12.sh -t
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

#### Potential Issue - WML PVC Sizing and Memory Issues

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

#### Potential Issue - Enable WxO Observability

WxO development team to provide the new procedure to enable WxO Observability

**Reference**: [Enable WxO Observability](https://github.com/kuanalex/ups/blob/main/WxO_Observability_PROD_Runbook.md)

---

#### Potential Issue - Fix platform-auth-service pod in ContainerStatusUnknown

Follow the procedure in the following known issue document to resolve platform-auth-service pod in ContainerStatusUnknown issue

**Reference**: [Enabling the debug trace for the platform-auth-service cause pod in ContainerStatusUnknown state and repeatedly get evicted](https://www.ibm.com/mysupport/s/defect/aCIgJ000000C9qjWAC/dt467023?language=en_US)

---

#### Potential Issue - Enabling Watson Speech services to process API requests on multiple clusters

You can configure Watson Speech services for an active-active multi-cluster deployment, enabling API requests to be processed across multiple clusters

**Reference**: [Enabling Watson Speech services to process API requests on multiple clusters](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=pis-enabling-watson-speech-services-process-api-requests-multiple-clusters)

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
oc get route cpd -n ${PROJECT_CPD_INST_OPERANDS} -o yaml | grep -A 20 labels
```

---

**End of Runbook**
