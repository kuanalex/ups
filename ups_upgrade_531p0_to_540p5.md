# IBM Software Hub Upgrade from 5.3.1 to 5.4.0

## Upgrade Context
- **OCP:** 4.20.23
- **SWH:** 5.3.1 → 5.4.0
- **Components:** cpd_platform, watsonx_orchestrate, watsonx_ai, watsonx_governance, watson_speech, voice_gateway, db2oltp, cognos_analytics
- **PrivateImageRegistry:** Yes

## Table of Contents
1. [Pre-upgrade Tasks](#pre-upgrade-tasks)
2. [Upgrade Execution](#upgrade-execution)
3. [Post-upgrade Tasks](#post-upgrade-tasks)


## Prerequisites

1. Backup of the cluster is done

Backup your IBM Software Hub cluster before the upgrade.
For details, see [Backing up and restoring IBM Software Hub](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=administering-backing-up-restoring-software-hub).

**Note:**
Some services don't support the offline OADP backup. Review the backup documentation and take the dedicated approach when necessary.

2. The case download and image mirroring completed successfully

If you will run the IBM Software Hub upgrade commands in a restricted network, you must have the CASE packages for the components that you plan to upgrade on the client workstation from which you will run the upgrade commands.

Run the appropriate command depending on the site from which you plan to download the CASE packages, and don't forget to specify the --patch_id field

Github
```bash
cpd-cli manage case-download \
--components=${COMPONENTS} \
--release=${VERSION} \
--patch_id=${PATCH_ID}
```

OCI
```bash
cpd-cli manage case-download \
--components=${COMPONENTS} \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--from_oci=true
```


Reference: [Downloading CASE packages before running IBM Software Hub upgrade commands in a restricted network](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=pruirn-downloading-case-packages)

If a private container registry is in-use to host the IBM Software Hub software images, you must mirror the updated images from the IBM Entitled Registry to the private container registry.

List images
```bash
cpd-cli manage list-images \
--components=${COMPONENTS} \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--inspect_source_registry=true
```

Mirror images
```bash
cpd-cli manage mirror-images \
--components=${COMPONENTS} \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--target_registry=${PRIVATE_REGISTRY_LOCATION} \
--arch=${IMAGE_ARCH} \
--case_download=false
```

Reference: [Mirroring images to private image registry](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=mipcr-mirroring-images-directly-private-container-registry-1)

3. The permissions required for the upgrade is ready

- OpenShift cluster administrator permissions
- IBM Software Hub administrator permissions
- Permission to access the private image registry for pushing or pulling images
- Access to the bastion node for executing the upgrade commands

4. A pre-upgrade health check is made to ensure the cluster's readiness for upgrade

- The OpenShift cluster, persistent storage and IBM Software Hub platform and services are in healthy status



# Pre-upgrade

## Pre-upgrade check

## Updating your environment variables script

Ensure that your environment variables script includes the correct information for the instance of IBM Software Hub that you want to upgrade.

[Updating your environment variables script](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=cri-updating-your-environment-variables-script)

Update the version and patch_id fields in your cpd_vars file
```bash
export VERSION=5.4.0
export PATCH_ID=5
```

Source the environment variables
```bash
source ./cpd_vars.sh
```

Login to the cluster
```bash
${OC_LOGIN} && ${CPDM_OC_LOGIN}
```

Check service cr status
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

Check for unhealthy pods
```bash
oc get po -A -owide | egrep -v '([0-9])/\1' | egrep -v 'Completed'
```


### Updating the IBM Software Hub command-line interface
[Update the cpd-cli utility](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=workstations-updating-software-hub-cli)


### Updating Helm CLI
[Installing Helm](https://www.ibm.com/links?url=https%3A%2F%2Fhelm.sh%2Fdocs%2Fintro%2Finstall%2F)



# Upgrade

## Upgrading the Scheduling service

### Updating cluster-scoped resources for the Scheduling service

Generate the cluster-scoped resource definitions for the Scheduling service
```bash
cpd-cli manage case-download \
--components=scheduler \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--scheduler_ns=${PROJECT_SCHEDULING_SERVICE} \
--cluster_resources=true
```

Run the oc apply -f command returned in the terminal
```bash
oc apply -f /.../cluster_scoped_resources.yaml --server-side --force-conflicts
```

### Creating image pull secrets for the Scheduling service

Create a file named dockerconfig.json based on where your cluster pulls images from

For Private container registry
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

Create the image pull secret in the project where the Scheduling service is installed:
```bash
oc create secret docker-registry ${IMAGE_PULL_SECRET} --from-file ".dockerconfigjson=dockerconfig.json" --namespace=${PROJECT_SCHEDULING_SERVICE}
```

### Upgrading the Scheduling service
```bash
cpd-cli manage apply-scheduler \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--license_acceptance=true \
--scheduler_ns=${PROJECT_SCHEDULING_SERVICE} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET}
```

Confirm that the Scheduling service pods are Running or Completed:
```bash
oc get pods --namespace=${PROJECT_SCHEDULING_SERVICE}
```

### Upgrading the License Service

If you're not sure which project the License Service is in, run the following command:
```bash
oc get deployment -A | grep ibm-licensing-operator
```

Login to the cluster
```bash
${CPDM_OC_LOGIN}
```

### Upgrading the License Service
```bash
cpd-cli manage apply-cluster-components \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--license_acceptance=true \
--licensing_ns=${PROJECT_LICENSE_SERVICE}
```

Confirm that the License Service pods are Running or Completed:
```bash
oc get pods --namespace=${PROJECT_LICENSE_SERVICE}
```

## Preparing to upgrade IBM Software Hub

### Updating the cluster-scoped resources for the platform and services

1.Generate cluster-scoped resources for platform and services
```bash
cpd-cli manage case-download \
--components=${COMPONENTS} \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--cluster_resources=true
```

Run the oc apply -f command returned in the terminal
```bash
oc apply -f /.../cluster_scoped_resources.yaml --server-side --force-conflicts
```

### Creating image pull secrets for an instance of IBM Software Hub

Login to the cluster
```bash
${CPDM_OC_LOGIN}
```

Create a file named dockerconfig.json based on where your cluster pulls images from

For Private container registry
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
oc create secret docker-registry ${IMAGE_PULL_SECRET} --from-file ".dockerconfigjson=dockerconfig.json" --namespace=${PROJECT_CPD_INST_OPERATORS}
```

Create the image pull secret in the operands project for the instance:
```bash
oc create secret docker-registry ${IMAGE_PULL_SECRET} --from-file ".dockerconfigjson=dockerconfig.json" --namespace=${PROJECT_CPD_INST_OPERANDS}
```

### Upgrading the IBM Events Operator

#### Upgrading Red Hat OpenShift Serverless Knative Eventing

Login to the cluster
```bash
${OC_LOGIN}
```

Generate the required custom resource definitions for the IBM Events Operator
```bash
cpd-cli manage deploy-events-operator --release=${VERSION} --cluster_resources=true
```

Apply the required custom resource definitions for the IBM Events Operator
```bash
oc apply -f cpd-cli-workspace/olm-utils-workspace/work/ibm-events-operator-crds.yaml --server-side --force-conflicts
```

Log in to the cluster
```bash
${CPDM_OC_LOGIN}
```

Upgrade the Red Hat OpenShift Serverless Knative Eventing software

For Portworx storage
```bash
cpd-cli manage deploy-knative-eventing --release=${VERSION} --storage_vendor=portworx --upgrade=true
```

All other storage
```bash
cpd-cli manage deploy-knative-eventing --release=${VERSION} --block_storage_class=${STG_CLASS_BLOCK} --upgrade=true
```

#### Upgrading the IBM Events Operator

Log the cpd-cli in to the Red Hat OpenShift Container Platform cluster:
```bash
${CPDM_OC_LOGIN}
```

Run the following command to upgrade the IBM Events Operator:
```bash
cpd-cli manage deploy-events-operator --release=${VERSION} --events_operator_ns=${PROJECT_CPD_INST_OPERATORS} --events_operand_ns=${PROJECT_CPD_INST_OPERANDS}
```

### Upgrading Red Hat OpenShift AI
[Upgrading Red Hat OpenShift AI](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=ups-upgrading-red-hat-openshift-ai)

### Apply Entitlements before upgrading IBM Software Hub



## Upgrading IBM Software Hub

Login to the cluster
```bash
${CPDM_OC_LOGIN}
```

Upgrade the required operators and custom resources for the instance
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

Once the above command `cpd-cli manage install-components` is completed, make sure the status of the IBM Software Hub is in 'Completed' status
```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=cpd_platform,zen
```

### Applying the RSI patches

Run the following command to re-apply your existing custom patches
```bash
cpd-cli manage apply-rsi-patches --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

Check the RSI patches status again
```bash
cpd-cli manage get-rsi-patch-info --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --all
```

## Upgrading Watson Speech to Text and Text to Speech API services

### Run the cpd-cli manage login-to-ocp command to log in to the cluster
```
${CPDM_OC_LOGIN}
```

### Upgrading the operator and custom resource for the service

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

Once the above command `cpd-cli manage install-components` completed successfully, you can run the `cpd-cli manage get-cr-status` command for the validation.

```bash
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=watson_speech
```

### Upgrading Voice Gateway

Login to the cluster
```bash
${CPDM_OC_LOGIN}
```

Upgrading the operator and custom resource for the service
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
--upgrade=true
```

Once the above command `cpd-cli manage install-components` completed successfully, you can run the `cpd-cli manage get-cr-status` command for the validation.

```
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} --components=voice_gateway
```

## Upgrading Db2

### Run the cpd-cli manage login-to-ocp command to log in to the cluster
```bash
${CPDM_OC_LOGIN}
```

### Upgrading the operator and custom resource for the service

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
  --upgrade=true
```

Once the above command `cpd-cli manage install-components` completed successfully, you can run the `cpd-cli manage get-cr-status` command for the validation.

```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=db2oltp
```

## Upgrading Cognos Analytics

### Run the cpd-cli manage login-to-ocp command to log in to the cluster
```
${CPDM_OC_LOGIN}
```

### Upgrading the operator and custom resource for the service

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
  --upgrade=true
```

Once the above command `cpd-cli manage install-components` completed successfully, you can run the `cpd-cli manage get-cr-status` command for the validation.

```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=cognos_analytics
```

## Upgrading IBM watsonx.ai

### Run the cpd-cli manage login-to-ocp command to log in to the cluster
```
${CPDM_OC_LOGIN}
```

### Upgrading the operator and custom resource for the service

```bash
cpd-cli manage install-components \
  --license_acceptance=true \
  --components=watsonx_ai \
  --release=${VERSION} \
  --patch_id=${PATCH_ID} \
  --operator_ns=${PROJECT_CPD_INST_OPERATORS} \
  --instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --image_pull_prefix=${IMAGE_PULL_PREFIX} \
  --image_pull_secret=${IMAGE_PULL_SECRET} \
  --upgrade=true
```

Once the above command `cpd-cli manage install-components` completed successfully, you can run the `cpd-cli manage get-cr-status` command for the validation.

```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=watsonx_ai
```

## Upgrading IBM Watsonx Governance

### Run the cpd-cli manage login-to-ocp command to log in to the cluster
```
${CPDM_OC_LOGIN}
```

### Upgrading the operator and custom resource for the service

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
  --upgrade=true
```

Once the above command `cpd-cli manage install-components` completed successfully, you can run the `cpd-cli manage get-cr-status` command for the validation.

```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=watsonx_governance
```

## Upgrading WatsonX Orchestrate

### Run the cpd-cli manage login-to-ocp command to log in to the cluster
```
${CPDM_OC_LOGIN}
```

### Upgrading the operator and custom resource for the service

**Important:** Before upgrading watsonx Orchestrate, review and update the `install-options.yml` file if needed. For detailed information about the install options, see [Reviewing the install-options.yml file](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=orchestrate-reviewing-install-optionsyml-file).


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
  --install_options=/tmp/work/install-options.yml \
  --upgrade=true
```

Once the above command `cpd-cli manage install-components` completed successfully, you can run the `cpd-cli manage get-cr-status` command for the validation.

```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=watsonx_orchestrate
```


# Post-upgrade


## Setting up a profile to upgrade service instances
Create a profile on the workstation from which you will upgrade the service instances.

<br>
**Note:**
The profile must be associated with a IBM Software Hub user who has either the following permissions:
<br>
- Create service instances (can_provision)
- Manage service instances (manage_service_instances)

<br>

1.Generate the API key that you need for user authentication by going to your `Profile and settings` page in the web client and clicking `Generate API key`.

<br>

2. Set the following environment variables.
<br>

- Set the API_KEY to the API key that you obtained in the preceding step.
```
export API_KEY=<api-key>
```

- Set the CPD_USERNAME environment variable to your username.
```
export CPD_USERNAME=<user-name>
```

- Set the LOCAL_USER environment variable to the name that you want to use for the local user configuration.

```
export LOCAL_USER=<local-user>
```

- Set the CPD_PROFILE_NAME environment variable to the name that you want to use for the profile.
```
export CPD_PROFILE_NAME=<cpd-profile-name>
```

- Set the CPD_PROFILE_URL environment variable to the URL of the instance that you want to connect to.
```
export CPD_PROFILE_URL=<cpd-url>
```

To get the URL of the web client, run the following command.
```
oc get route cpd \
        --namespace=${PROJECT_CPD_INST_OPERANDS}
```
 The command returns output with the following format.
```
cpd-namespace.apps.OCP-default-domain
```

Add https:// to the value that is returned. For example, `https://cpd-namespace.apps.OCP-default-domain`.

- Create a local user configuration to store your username and API key by using the config users set command.
```
cpd-cli config users set ${LOCAL_USER} \
    --username ${CPD_USERNAME} \
    --apikey ${API_KEY}
```

Anytime you regenerate your API key, you must rerun this command to update your local user configuration.

<br>

- Create a profile to store the URL and to associate the profile with your local user configuration by using the config profiles set command.
```
cpd-cli config profiles set ${CPD_PROFILE_NAME} \
    --user ${LOCAL_USER} \
    --url ${CPD_PROFILE_URL}
```
For example:
```
    users:
    - name: 682-d-engineer_1
      user:
        username: cpadmin
        apikey: {base64: c1BPZnhRZ1pyTEVyeWdkdUpEdGtqT3lpMTdpcm9HWm5JSDcza3VDcQ==}

    - name: my_profile
      profile:
        type: private
        url: https://cpd-cpd-instance.apps.ivt564.cp.example.ibm.com
        user: 682-d-engineer_1
    ....
```

- Validate your configuration by using below command.

You can now run cpd-cli commands with this profile as shown in the following example.
```
cpd-cli service-instance list \
--profile=${CPD_PROFILE_NAME}
```


## Upgrading service instances
### Upgrading service instance of db2oltp

- Upgrading the service instance

```
cpd-cli service-instance upgrade \
--service-type=db2oltp \
--profile=${CPD_PROFILE_NAME} \
--all
```

- Validating the service instance upgrade status.

```
cpd-cli service-instance list \
--service-type=db2oltp \
--profile=${CPD_PROFILE_NAME}
```
### Upgrading service instance of cognos_analytics

- Upgrading the service instance

```
cpd-cli service-instance upgrade \
--service-type=cognos-analytics-app \
--profile=${CPD_PROFILE_NAME} \
--all
```

- Validating the service instance upgrade status.

```
cpd-cli service-instance list \
--service-type=cognos-analytics-app \
--profile=${CPD_PROFILE_NAME}
```

### Apply hot fixes (if applicable)

If IBM Support has provided specific hotfixes for your environment, apply them according to the instructions provided.

**Note:** Check with IBM Support to determine if any hotfixes are required for your specific configuration and version ().

### Check if there are any remaining special tasks

> ⚠️ **Important:**


Check the IBM Software Hub 5.4.x documentation for any special migration tasks required when upgrading from 5.3.x.

For additional post-upgrade setup steps, see [Setting up services](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=1-setting-up-services).

Check if there are any remaining special tasks for your services that need to be completed after the upgrade.

### Post-upgrade service setup

After completing the upgrade, you may need to perform additional setup tasks for your services. For detailed information about post-upgrade service configuration, see:

- [Setting up services](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=1-setting-up-services)
- [Post-upgrade tasks](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=upgrading-post-upgrade-tasks)



