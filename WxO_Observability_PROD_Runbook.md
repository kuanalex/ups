# Watsonx Orchestrate Observability Installation - PROD Runbook

## Prerequisites Checklist
- [ ] Cluster Administrator access to OpenShift cluster
- [ ] Root access to CPD installation server
- [ ] CPD CLI configured and authenticated
- [ ] IBM Entitlement Key available
- [ ] GCP access token for private registry
- [ ] Storage classes identified (block and file)
- [ ] Namespace names confirmed
- [ ] cpd_vars.sh file location confirmed

---

## Step 0: Login as Root and Source Environment

```bash
# Login as root user
sudo su - root

# Navigate to CPD installation directory
cd <cpd-installation-directory>

# Source the CPD environment variables
source cpd_vars.sh

# Verify environment is loaded
echo "CPD Instance Namespace: ${CPD_INSTANCE_NS}"
echo "CPD Operator Namespace: ${CPD_OPERATOR_NS}"
echo "Storage Class Block: ${STG_CLASS_BLOCK}"
echo "Storage Class File: ${STG_CLASS_FILE}"
echo "Private Registry: ${PRIVATE_REGISTRY_LOCATION}"
```

**Validation:** Confirm all required environment variables are set

**Note:** The cpd_vars.sh file should already contain:
- `CPD_INSTANCE_NS` (e.g., ups-wx-operands)
- `CPD_OPERATOR_NS` (e.g., ups-wx-operators)
- `STG_CLASS_BLOCK`
- `STG_CLASS_FILE`
- `PRIVATE_REGISTRY_LOCATION`
- `IBM_ENTITLEMENT_KEY`

---

## Step 1: Mirror Required Images

### Images to Mirror
```bash
IMAGES=(
  "cp.icr.io/cp/opencontent-ibm-opensearch-base-10@sha256:d8c9304f57765e88c0325a16e5cb11b03a1f7ffb290505f526ac3c894cebf1a5"
  "cp.icr.io/cp/opencontent-ibm-opensearch-min-2.19.3@sha256:a7fd1076b7cf9964860b8ea24c6ed7f8d3abfbd2662610a93568305a485b4455"
  "cp.icr.io/cp/opencontent-ibm-opensearch-plugins-2.19.3@sha256:f33e22e5929bb9475ec692cd58e6b6720998da62c9288ef68410ac4873a92eda"
  "cp.icr.io/cp/opencontent-ibm-opensearch-plugin-knn-2.19.3.0@sha256:63fa066bec1f240e7a610fd3948f568f42a0b273306d94dd904f67146a76d597"
  "cp.icr.io/cp/opencontent-ibm-opensearch-plugin-security-2.19.3.0@sha256:53e1c55af88a052eaf02537c9b2137228749c4cccaba0634455333e015115439"
  "cp.icr.io/cp/watsonx-orchestrate/agent-analytics:v5.3.1-20260320.201900"
)
```

### Mirror Script
```bash
#!/usr/bin/env bash
set -euo pipefail

SRC_AUTH="cp:$IBM_ENTITLEMENT_KEY"
DEST_AUTH="oauth2accesstoken:$(gcloud auth print-access-token)"
DEST_REGISTRY="$PRIVATE_REGISTRY_LOCATION"

copy_image () {
  local SRC="$1"
  local DEST="$2"
  echo "Copying: ${SRC} -> ${DEST}"
  skopeo copy --all \
    --src-creds "${SRC_AUTH}" \
    --dest-creds "${DEST_AUTH}" \
    "docker://${SRC}" \
    "docker://${DEST}"
}

for SRC in "${IMAGES[@]}"; do
  DEST_PATH="${SRC#*/}"
  copy_image "${SRC}" "${DEST_REGISTRY}/${DEST_PATH}"
done

echo "✅ All images mirrored successfully"
```

**Action:** Run the mirror script
```bash
chmod +x mirror-observability-images.sh
./mirror-observability-images.sh
```

**Validation:** Verify all 6 images are in private registry

### Step 1.1 Mirror images needed outside of IBM repo
```bash
skopeo copy --all --src-creds "$(jq -r '.auths["registry.redhat.io"].auth' .dockerconfigjson | base64 -d)" --dest-creds "oauth2accesstoken:$(gcloud auth print-access-token)" docker://registry.redhat.io/rhosdt/opentelemetry-rhel8-operator:rhosdt-3.7.0 docker://us-docker.pkg.dev/gcp-dct-ccca-dev/ccca-d-image-registry/rhosdt/opentelemetry-rhel8-operator:rhosdt-3.7.0


 skopeo copy --all --src-creds "$(jq -r '.auths["quay.io"].auth' .dockerconfigjson | base64 -d)" --dest-creds "oauth2accesstoken:$(gcloud auth print-access-token)" docker://quay.io/brancz/kube-rbac-proxy:v0.19.1 docker://us-docker.pkg.dev/gcp-dct-ccca-dev/ccca-d-image-registry/brancz/kube-rbac-proxy:v0.19.1

skopeo copy --all \
  --dest-creds "oauth2accesstoken:$(gcloud auth print-access-token)" \
  docker://docker.io/jaegertracing/jaeger \
  docker://us-docker.pkg.dev/gcp-dct-ccca-dev/ccca-d-image-registry/jaegertracing/jaeger
```

**Validation:** Verify all 3 images are in private registry

---

## Step 2: Login to OpenShift Cluster

```bash
cpd-cli manage login-to-ocp
```

**Validation:** Confirm successful login message

---

## Step 3: Deploy Observability Stack

```bash
cpd-cli manage deploy-observability \
  --cpd_instance_ns=${CPD_INSTANCE_NS} \
  --cpd_operator_ns=${CPD_OPERATOR_NS} \
  --image_pull_prefix=${PRIVATE_REGISTRY_LOCATION} \
  --block_storage_class=${STG_CLASS_BLOCK} \
  --file_storage_class=${STG_CLASS_FILE}
```

**Expected Duration:** 5-10 minutes

**Validation Steps:**
```bash
# Check OpenTelemetry operator
oc get csv -n ${CPD_OPERATOR_NS} | grep opentelemetry

# Check Jaeger deployment
oc get jaeger -n ${CPD_INSTANCE_NS}

# Check OpenSearch pods
oc get pods -n ${CPD_INSTANCE_NS} | grep opensearch

# Verify all pods are Running
oc get pods -n ${CPD_INSTANCE_NS} -l app.kubernetes.io/part-of=observability
```

---

## Step 4: Enable Observability in Watsonx Orchestrate

```bash
oc patch watsonxorchestrate wo \
  --type=merge \
  --patch='{"spec":{"observability":{"enabled": true}}}' \
  -n ${CPD_INSTANCE_NS}
```

**Expected Behavior:**
- Conversational controller pod will restart
- New observability-related pods will be created
- Services will reconnect (may take 3-5 minutes)

**Validation Steps:**
```bash
# Check WatsonxOrchestrate CR
oc get watsonxorchestrate wo -n ${CPD_INSTANCE_NS} -o yaml | grep -A 2 observability

# Monitor pod restarts
oc get pods -n ${CPD_INSTANCE_NS} -w

# Check conversational controller logs
oc logs -n ${CPD_INSTANCE_NS} -l component=conversational-controller --tail=50

# Verify observability components are ready
oc get pods -n ${CPD_INSTANCE_NS} | grep -E "jaeger|opensearch|otel"
```

**Validation Steps in the UI/ADK:**

1. Start a chat with a Agent and ensure that the Analyze tab for Agent Analytics is starting to get populated
2. Configure the ADK and utilize the `orchestrate observability traces` to ensure you call pull the trace spans

### Notes for Deployment

-  `wo-conversation-controller` should restart to take the change when `spec.observability.enabled` is changed
  - You can trigger the reconcile manually by deleting the `wo-operator` and `ibm-wxo-componentcontroller-manager` pods
  - Once the restart occurs, you should be able to see the changes in the UI/ADK and traces should start coming in with new chats

---

## Success Criteria

- [ ] All observability pods in Running state
- [ ] OpenSearch cluster health is GREEN
- [ ] Jaeger collector is receiving traces
- [ ] Conversational controller successfully restarted
- [ ] WatsonxOrchestrate CR shows observability.enabled: true
- [ ] No error logs in observability components

---

## Notes

- **One-time setup:** Observability stack is deployed once per cluster
- **Shared resource:** All Orchestrate instances can use the same observability stack
- **Security:** Sensitive values are automatically masked in traces
- **Performance:** Minimal impact on agent execution performance
- **Storage:** Monitor OpenSearch storage usage over time

---

**Document Version:** 1.0  
**Last Updated:** 2026-05-29  
**Prepared For:** PROD CPD Upgrade