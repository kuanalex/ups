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
OCP: None
Storage: Google Cloud Netapp Volumes and Persistent Disk on Google Cloud
Internet: airgap
Private container registry: yes
Components: cpd_platform,db2oltp,watson_speech,voice_gateway,watsonx_orchestrate,watsonx_ai,cognos_analytics,watsonx_governance
```

---

# CP4D 5.3.0 → 5.3.1 Upgrade Runbook
**UPS Production Cluster**

---

## Document Information

| Field | Value |
|-------|-------|
| **Generated** | 2026-04-15 12:38:15 UTC |
| **Generator Version** | 1.2.0 |
| **Cluster Name** | UPS Production Cluster |
| **Current CPD Version** | v5.3.0 |
| **Target CPD Version** | v5.3.1 |
| **OpenShift Version** | 4.17 |
| **Storage Vendor** | Google Cloud Netapp Volumes and Persistent Disk on Google Cloud |
| **Internet Access** | Airgap |
| **Services Count** | 8 |
| **Estimated Duration** | 17 hours 5 minutes |

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Pre-Upgrade Backups](#pre-upgrade-backups)
4. [Upgrade Execution](#upgrade-execution)
5. [RSI Patch Management](#rsi-patch-management)
6. [Post-Upgrade Validation](#post-upgrade-validation)

---

## 1. Overview

This runbook provides step-by-step instructions for upgrading Cloud Pak for Data from version 5.3.0 to 5.3.1 on OpenShift 4.17.

### Upgrade Path

**Direct Upgrade**: 5.3.0 → 5.3.1

### Services to Upgrade

The following 8 services will be upgraded:

- **Cloud Pak for Data Platform**
- **Db2 OLTP**
- **Watson Speech**
- **Voice Gateway**
- **Watsonx Orchestrate**
- **Watsonx Ai**
- **Cognos Analytics**
- **Watsonx Governance**

### Key Requirements

- **CPD CLI Version**: 14.3.1.2
- **Helm Version**: 3.16.3
- **Storage**: Netapp
  - Block Storage Class: csi-gce-pd-ssd
  - File Storage Class: gcnv-standard-k8s


## Upgrade Path Validation





### ✅ Validation Status

All validation checks passed. The upgrade path is valid and can proceed.


---


---

## 2. Prerequisites

### 2.1 Required Tools

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

### 2.2 Access Requirements

- [ ] OpenShift cluster admin access
- [ ] IBM Entitlement Key with appropriate permissions
- [ ] Access to IBM Container Registry (cp.icr.io)
- [ ] Access to private registry: UPDATE_WITH_PRIVATE_REGISTRY_URL

### 2.3 Environment Variables Setup (cpd_vars.sh)

Ensure that your environment variables script includes the correct information for the instance of IBM Software Hub that you want to upgrade.

**Required Environment Variables:**
- CPD Version: `5.3.1`
- CPD CLI Version: `14.3.1.2`
- OCP URL: `UPDATE_WITH_OCP_URL`
- Operators Namespace: `cpd-operators`
- Operands Namespace: `cpd-instance`
- Block Storage Class: `csi-gce-pd-ssd`
- File Storage Class: `gcnv-standard-k8s`
- Components: `cpd_platform,db2oltp,watson_speech,voice_gateway,watsonx_orchestrate,watsonx_ai,cognos_analytics,watsonx_governance`
- Private Registry: `UPDATE_WITH_PRIVATE_REGISTRY_URL`

**Documentation**: [Updating your environment variables script](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=cri-updating-your-environment-variables-script-1)

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

### 2.4 Login to OpenShift Cluster

```bash
# Login using cpd-cli
${CPDM_OC_LOGIN}

# Or login using oc directly
${OC_LOGIN}

# Verify cluster access
oc whoami
oc get nodes
```

### 2.5 Restart OLM Utils Container

```bash
# Restart the OLM utils container with updated environment
cpd-cli manage restart-container

# Verify container is running
podman ps | grep olm-utils
```

### 2.7 Air-Gapped Environment Prerequisites


**⚠️ IMPORTANT**: This cluster uses a private container registry in an air-gapped environment. The following prerequisites must be completed before proceeding with the upgrade.

**Private Registry**: UPDATE_WITH_PRIVATE_REGISTRY_URL

---

#### Required Prerequisites for Air-Gapped Upgrades

The following steps must be completed before starting the upgrade process. Refer to IBM documentation for detailed procedures:

**1. Obtain OLM Utils v4 Image**
   - **Reference**: [Obtaining the olm-utils-v4 image](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=pruirn-obtaining-olm-utils-v4-image-2)
   - Download the OLM utils image for air-gapped operations

**2. Download CASE Packages**
   - **Reference**: [Downloading CASE packages](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=pruirn-downloading-case-packages-2)
   - Download CASE packages for all components being upgraded

**3. Mirror Images to Private Registry**
   - **Option A (Direct)**: [Mirroring images directly to private container registry](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=mipcr-mirroring-images-directly-private-container-registry-2)
   - **Option B (Intermediary)**: [Mirroring images using intermediary container registry](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=mipcr-mirroring-images-using-intermediary-container-registry-2)
   - Mirror all required images to your private registry

**4. Pull OLM Utils Image from Private Registry**
   - **Reference**: [Pulling the olm-utils-v4 image from private container registry](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=prufpcr-pulling-olm-utils-v4-image-from-private-container-registry-2)
   - Configure OLM utils to use your private registry

**5. Update Cluster-Scoped Resources for Shared Components**
   - **Reference**: [Updating cluster-scoped resources for shared cluster components](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=pyc-updating-cluster-scoped-resources-shared-cluster-components-1)
   - Update CASE packages for scheduling service and other shared components

**6. Update Cluster-Scoped Resources for Platform and Services**
   - **Reference**: [Updating cluster-scoped resources for the instance](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=puish-updating-cluster-scoped-resources-instance-1)
   - Update CASE packages for CPD platform and all services

---

#### Pre-Upgrade Checklist

Verify the following before proceeding:

- [ ] OLM utils v4 image obtained and configured
- [ ] CASE packages downloaded for all components
- [ ] Images mirrored to private registry: UPDATE_WITH_PRIVATE_REGISTRY_URL
- [ ] OLM utils configured to use private registry
- [ ] Cluster-scoped resources updated for shared components (licensing, scheduling)
- [ ] Cluster-scoped resources updated for CPD instance
- [ ] Image pull secrets configured in all required namespaces
- [ ] Catalog sources pointing to private registry
- [ ] All prerequisites validated and confirmed

---



### 2.7 Advanced Service Prerequisites

The following services require advanced infrastructure components:

#### 2.7.1 Multi-Cloud Object Gateway (MCG)

**Required for**: Watson Speech, Voice Gateway, Watsonx Ai

Multi-Cloud Object Gateway provides S3-compatible object storage for CP4D services that require large-scale data storage.

**⚠️ PREREQUISITE**: MCG/ODF must already be installed and configured. This is an upgrade runbook - storage infrastructure should already be in place.

**Verification Steps:**

```bash
# Verify ODF operator is installed
oc get csv -n openshift-storage | grep ocs-operator

# Verify NooBaa system is running
oc get noobaa -n openshift-storage

# Check MCG pods are healthy
oc get pods -n openshift-storage | grep noobaa

# Verify object bucket claims can be created
oc get crd | grep objectbucketclaim
```

**If MCG/ODF is not installed**, refer to IBM documentation:
- [Installing OpenShift Data Foundation](https://docs.openshift.com/container-platform/latest/storage/persistent_storage/persistent-storage-ocs.html)
- [Configuring Multi-Cloud Object Gateway](https://www.ibm.com/docs/en/cloud-paks/1.0?topic=foundation-multicloud-object-gateway)

**Estimated Verification Time**: 5-10 minutes



#### 2.7.2 Knative Eventing and Serving

**Required for**: Watsonx Orchestrate

Knative enables event-driven architecture and automatic scaling for CP4D services.

**Installation Steps:**

1. **Install OpenShift Serverless Operator:**

```bash
# Create namespace
oc create namespace openshift-serverless

# Install operator
cat << EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: serverless-operator
  namespace: openshift-serverless
spec:
  channel: stable
  name: serverless-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

# Wait for operator to be ready
oc wait --for=condition=Ready pod -l name=knative-openshift -n openshift-serverless --timeout=300s
```

2. **Install Knative Serving:**

```bash
# Create Knative Serving namespace
oc create namespace knative-serving

# Install Knative Serving
cat << EOF | oc apply -f -
apiVersion: operator.knative.dev/v1beta1
kind: KnativeServing
metadata:
  name: knative-serving
  namespace: knative-serving
spec:
  config:
    autoscaler:
      min-scale: "0"
      max-scale: "10"
      target-concurrency: "100"
      scale-down-delay: "30s"
    deployment:
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          cpu: 1000m
          memory: 1Gi
    domain:
      svc.cluster.local: ""
EOF

# Wait for Knative Serving to be ready
oc wait --for=condition=Ready knativeserving/knative-serving -n knative-serving --timeout=600s
```

3. **Install Knative Eventing:**

```bash
# Create Knative Eventing namespace
oc create namespace knative-eventing

# Install Knative Eventing
cat << EOF | oc apply -f -
apiVersion: operator.knative.dev/v1beta1
kind: KnativeEventing
metadata:
  name: knative-eventing
  namespace: knative-eventing
spec:
  defaultBrokerClass: MTChannelBasedBroker
  config:
    br-default-channel:
      channel-template-spec: |
        apiVersion: messaging.knative.dev/v1
        kind: InMemoryChannel
EOF

# Wait for Knative Eventing to be ready
oc wait --for=condition=Ready knativeeventing/knative-eventing -n knative-eventing --timeout=600s
```

4. **Configure service-specific Knative settings:**


**For Watsonx Orchestrate:**
- Min Scale: 0
- Max Scale: 10
- Target Concurrency: 50


**Validation:**

```bash
# KnativeServing is ready
oc get knativeserving -n knative-serving
# KnativeEventing is ready
oc get knativeeventing -n knative-eventing
```

**Estimated Time**: 20-30 minutes

**Documentation**: [https://docs.redhat.com/en/documentation/red_hat_openshift_serverless](https://docs.redhat.com/en/documentation/red_hat_openshift_serverless)



#### 2.7.3 Events Operator

**Required for**: Watsonx Orchestrate

The Events Operator manages event sources, sinks, and brokers for event-driven CP4D workflows.

**Installation Steps:**

1. **Install Events Operator:**

```bash
# Install operator via OLM
cat << EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: ibm-events-operator
  namespace: ibm-common-services
spec:
  channel: v4
  name: ibm-events-operator
  source: ibm-operator-catalog
  sourceNamespace: openshift-marketplace
EOF

# Wait for operator to be ready
oc wait --for=condition=Ready pod -l app.kubernetes.io/name=ibm-events-operator -n ibm-common-services --timeout=300s
```

2. **Create Event Broker:**

```bash
# Create Kafka-based event broker
cat << EOF | oc apply -f -
apiVersion: eventstreams.ibm.com/v1beta2
kind: EventStreams
metadata:
  name: cpd-event-broker
  namespace: ibm-common-services
spec:
  version: 4.0.0
  license:
    accept: true
    use: CloudPakForDataNonProduction
  strimziOverrides:
    kafka:
      replicas: 3
      storage:
        type: persistent-claim
        size: 100Gi
        class: {{ STG_CLASS_BLOCK }}
      config:
        log.retention.hours: 168
        offsets.topic.replication.factor: 3
        transaction.state.log.replication.factor: 3
        transaction.state.log.min.isr: 2
    zookeeper:
      replicas: 3
      storage:
        type: persistent-claim
        size: 10Gi
        class: {{ STG_CLASS_BLOCK }}
EOF

# Wait for event broker to be ready (this may take 10-15 minutes)
oc wait --for=condition=Ready eventstreams/cpd-event-broker -n ibm-common-services --timeout=900s
```

3. **Configure Event Sources:**

Supported event source types:
- http
- kafka
- cron
- container

**Validation:**

```bash
# Succeeded
oc get csv -n ibm-common-services | grep ibm-events-operator
# EventStreams is ready
oc get eventstreams -n ibm-common-services
```

**Estimated Time**: 15-25 minutes

**Documentation**: [https://www.ibm.com/docs/SSQNUZ_5.0.x/svc-welcome/events.html](https://www.ibm.com/docs/SSQNUZ_5.0.x/svc-welcome/events.html)


#### 2.7.4 Installation Sequence Summary

Follow this sequence for installing advanced prerequisites:

**Step 1**: Install Multi-Cloud Object Gateway
- Component: `mcg`
- Estimated Time: 30 minutes

**Step 2**: Install Knative Serving and Eventing
- Component: `knative`
- Estimated Time: 20 minutes

**Step 3**: Install Events Operator
- Component: `events_operator`
- Estimated Time: 15 minutes
- Prerequisites: knative

**Step 4**: Create MCG buckets for services
- Component: `mcg_buckets`
- Estimated Time: 10 minutes
- Prerequisites: mcg

**Step 5**: Create service-specific secrets
- Component: `service_secrets`
- Estimated Time: 10 minutes
- Prerequisites: mcg_buckets


#### 2.7.5 Troubleshooting

**Multi-Cloud Object Gateway:**
- **Issue**: NooBaa pods not starting
  - **Solution**: Check storage class availability and PVC binding
- **Issue**: Bucket creation fails
  - **Solution**: Verify NooBaa system is ready and has sufficient storage

**Knative:**
- **Issue**: Knative pods in CrashLoopBackOff
  - **Solution**: Check resource limits and node capacity
- **Issue**: Services not scaling
  - **Solution**: Verify autoscaler configuration and metrics server

**Events Operator:**
- **Issue**: Event broker not starting
  - **Solution**: Check Kafka cluster status and storage availability
- **Issue**: Events not flowing
  - **Solution**: Verify event source and sink configurations

---


---

## 3. Pre-Upgrade Backups

## Pre-Upgrade Backups

⚠️ **CRITICAL**: Complete all backups before proceeding with upgrade. Store backups in a secure location outside the cluster.

### Global Backups (Required for All Upgrades)

These backups are required regardless of which services are being upgraded.

#### 1. Routes Backup
Backup all routes to restore in case of upgrade failure.

```bash
# Backup all routes in operands namespace
oc get routes -n ${PROJECT_CPD_INST_OPERANDS} -o yaml > routes_backup_$(date +%Y%m%d_%H%M%S).yaml

# Verify backup
ls -lh routes_backup_*.yaml
```

**Purpose**: Routes define external access to CP4D services. This backup enables quick restoration of service endpoints.

---

#### 2. TemporaryPatch Backup
Backup all temporary patches applied to the cluster.

```bash
# Backup all temporary patches
oc get TemporaryPatch -n ${PROJECT_CPD_INST_OPERANDS} -o yaml > temporarypatch_backup_$(date +%Y%m%d_%H%M%S).yaml

# Verify backup
ls -lh temporarypatch_backup_*.yaml
```

**Purpose**: Temporary patches contain custom fixes and workarounds. These may need to be reapplied after upgrade.

---

#### 3. Custom Resource Definitions Backup
Backup all CP4D custom resources.

```bash
# Backup all CP4D CRs in operands namespace
oc get $(oc api-resources --namespaced=true --verbs=list -o name | grep -E 'cpd|ibm|watson' | paste -sd, -) \
  -n ${PROJECT_CPD_INST_OPERANDS} -o yaml > cpd_crs_backup_$(date +%Y%m%d_%H%M%S).yaml

# Verify backup
ls -lh cpd_crs_backup_*.yaml
```

**Purpose**: Custom resources define the desired state of all CP4D services. Critical for disaster recovery.

---

#### 4. ConfigMaps Backup
Backup all configuration maps.

```bash
# Backup all ConfigMaps
oc get configmaps -n ${PROJECT_CPD_INST_OPERANDS} -o yaml > configmaps_backup_$(date +%Y%m%d_%H%M%S).yaml

# Verify backup
ls -lh configmaps_backup_*.yaml
```

**Purpose**: ConfigMaps contain service configurations and settings.

---

#### 5. Secrets Backup
Backup all secrets (store securely).

```bash
# Backup all Secrets
oc get secrets -n ${PROJECT_CPD_INST_OPERANDS} -o yaml > secrets_backup_$(date +%Y%m%d_%H%M%S).yaml

# Verify backup
ls -lh secrets_backup_*.yaml

# IMPORTANT: Store this file securely and encrypt if possible
chmod 600 secrets_backup_*.yaml
```

**Purpose**: Secrets contain sensitive data like passwords, certificates, and API keys.

⚠️ **SECURITY**: Encrypt and store secrets backup in a secure location. Do not commit to version control.

---

### Service-Specific Backups

Complete the following backups for each service being upgraded:







### Backup Verification Checklist

Before proceeding with the upgrade, verify all backups are complete:

- [ ] Global backups completed (5 items)
  - [ ] Routes backup
  - [ ] TemporaryPatch backup
  - [ ] Custom Resources backup
  - [ ] ConfigMaps backup
  - [ ] Secrets backup







- [ ] All backup files stored in secure location outside cluster
- [ ] Backup files verified (non-zero size, readable)
- [ ] Secrets backups encrypted and secured

⚠️ **DO NOT PROCEED** with upgrade until all applicable backups are complete and verified.

---

### Backup Storage Recommendations

1. **Location**: Store backups on a system outside the OpenShift cluster
2. **Retention**: Keep backups for at least 30 days after successful upgrade
3. **Security**: Encrypt all backup files, especially secrets
4. **Documentation**: Document backup locations and restore procedures
5. **Testing**: Periodically test backup restoration procedures

---
---


## 4. Pre-Upgrade Health Check

### 3.1 Run Comprehensive Health Check

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

### 3.2 Check for Hot Fixes and Patches

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

### 3.3 Basic Cluster Validation

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

### 3.4 Storage Validation

```bash
# Verify storage classes
oc get sc

# Check PVC status
oc get pvc -n ${PROJECT_CPD_INST_OPERANDS}

# Verify storage vendor health
```

### 3.5 CPD Platform Validation

```bash
# Check CPD operator status
oc get pods -n ${PROJECT_CPD_INST_OPERATORS}

# Check CPD operand status
oc get pods -n ${PROJECT_CPD_INST_OPERANDS}

# Verify CPD platform version
oc get ZenService lite-cr -n ${PROJECT_CPD_INST_OPERANDS} -o jsonpath='{.status.zenStatus.versions[0].version}'

# Check CPD route
oc get route cpd -n ${PROJECT_CPD_INST_OPERANDS}

# List deployed components
cpd-cli manage list-deployed-components --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

### 3.6 Service Validation

Use the CPD CLI to check status for all services:

```bash
# Get comprehensive status for all services
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}

# Check specific service if needed (replace SERVICE_NAME)
cpd-cli manage get-cr-status \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --components=SERVICE_NAME
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

**Special Requirements for Db2 OLTP:**
- Stop Data Gate synchronization before upgrade
- Switch to Archive Logging mode
- Upgrade Db2 license before instance upgrade
- Cordon nodes for db2ckupgrade.sh job
- Update instance.json configmap after upgrade

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

**Special Requirements for Watson Speech:**
- Backup custom language models
- Verify audio file storage
- Test speech recognition after upgrade

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

**Special Requirements for Voice Gateway:**
- Requires Watson Speech Services
- Backup SIP trunk configurations
- Test telephony connections
- Verify call routing rules

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

**Special Requirements for Watsonx Orchestrate:**
- Backup automation workflows
- Export skill configurations
- Verify integration endpoints

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

**Special Requirements for Watsonx Ai:**
- Requires GPU nodes for optimal performance
- Backup foundation model configurations
- Verify model deployment endpoints
- Check prompt template library

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

**Special Requirements for Watsonx Governance:**
- Requires Watson OpenScale and Watson Knowledge Catalog
- Export governance policies
- Backup model risk assessments
- Verify compliance rules


---

**Notes:**
- `--run_storage_tests=false` is intentionally set to reduce upgrade time
- Platform upgrade typically takes 60-80 minutes
- Service upgrades vary: 20-30 minutes each depending on complexity
- Monitor pod status regularly during upgrade
- Do not interrupt the upgrade process once started
---

## 5. RSI Patch Management

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

## 6. Post-Upgrade Validation

### 5.1 Platform Validation

```bash
# Check all pods are running
oc get pods -n ${PROJECT_CPD_INST_OPERANDS} | grep -v Running | grep -v Completed

# Verify CPD version
oc get ZenService lite-cr -n ${PROJECT_CPD_INST_OPERANDS} -o jsonpath='{.status.zenStatus.versions[0].version}'

# Get comprehensive CR status for all components
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}

# List all deployed components
cpd-cli manage list-deployed-components --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}

# List all service instances
cpd-cli service-instance list --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}

# Check for any errors in recent events
oc get events -n ${PROJECT_CPD_INST_OPERANDS} --sort-by='.lastTimestamp' | tail -50
```

### 5.2 Service-Specific Validation

**Note**: The comprehensive status check above validates all services. Use the commands below only if you need to troubleshoot specific services.

```bash
# Check specific service status (replace SERVICE_NAME with actual service)
cpd-cli manage get-cr-status \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --components=SERVICE_NAME

# Check specific service pods (replace SERVICE_NAME with actual service)
oc get pods -n ${PROJECT_CPD_INST_OPERANDS} | grep SERVICE_NAME
```

---

### 5.3 Post-Upgrade Migrations


---

## 7. Final Health and Status Check Post Upgrade

### Monitor Upgrade Progress

```bash
# Monitor upgrade progress
watch -n 10 'oc get pods -n ${PROJECT_CPD_INST_OPERANDS} | grep -v Running | grep -v Completed'

# Get all events
oc get events -n ${PROJECT_CPD_INST_OPERANDS} --sort-by='.lastTimestamp'

# Check CPD status
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}

# List deployed components
cpd-cli manage list-deployed-components --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

### Get Comprehensive Status
```bash
# Check CPD status
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}

# List deployed components
cpd-cli manage list-deployed-components --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}

# Get all events
oc get events -n ${PROJECT_CPD_INST_OPERANDS} --sort-by='.lastTimestamp'
```

---

**End of Runbook**

*Generated by CP4D Upgrade Automation Tool v1.2.0*