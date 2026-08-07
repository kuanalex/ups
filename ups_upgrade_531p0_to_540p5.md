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

## Pre-requisites

### 1. Backup of the cluster is done

Backup your Cloud Pak for Data cluster before the upgrade.

**Note:**
Make sure there are no scheduled backups conflicting with the scheduled upgrade.

### 2. The image mirroring completed successfully

Since you are using a private container registry, you must mirror the updated images from the IBM® Entitled Registry to your private container registry at `<YOUR_PRIVATE_REGISTRY>`.

### 3. The CASE files and cluster resource files downloaded successfully

Before upgrading IBM Scheduling, the IBM Software Hub platform, or any services, you must download the required cluster‑scoped resources—such as ClusterRoles and ClusterRoleBindings—for the components you plan to upgrade. Ensure that these files are available on the bastion node for use during the upgrade.

For more information, see [Downloading CASE packages](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=pruirn-downloading-case-packages-1).

Since the IBM Scheduling service is included in your upgrade, verify it's installed:
```bash
oc get scheduling -A
```

### 4. The permissions required for the upgrade is ready

- **OpenShift cluster permissions**
  
  An OpenShift cluster administrator can complete all of the installation tasks.
  
- **IBM Software Hub permissions**
  
  The Cloud Pak for Data administrator role or permissions is required for upgrading the service instances.

- **Registry permissions**
  - Permission to access the private image registry at `<YOUR_PRIVATE_REGISTRY>` for pushing or pulling images

- **Bastion node access**
  - Access to the bastion node for executing the upgrade commands

### 5. A pre-upgrade health check is made to ensure the cluster's readiness for upgrade

- The OpenShift cluster, persistent storage, IBM Software Hub platform and services are in healthy status.



# Pre-upgrade

**Note:**
Sourcing the latest environment variables used by this environment before proceeding with the following procedures. For more information, see [Updating your environment variables script](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=cri-updating-your-environment-variables-script-1).

```bash
source ./cpd_vars.sh
```

## Pre-upgrade check

### Checking the health of your cluster

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


## Updating the IBM Software Hub command-line interface

### Updating the IBM Software Hub command-line interface

Update the cpd-cli utility to the latest version. For detailed documentation and instructions, see [Update the cpd-cli utility](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=workstations-updating-software-hub-cli).

### Obtaining the olm-utils-v4 image

All IBM Software Hub customers are entitled to use the olm-utils-v4 image. The cpd-cli uses podman to pull and manage the olm-utils-v4 container image. When the workstation is connected to the internet, run the following command to update the olm-utils-v4 image on the workstation:

```bash
cpd-cli manage restart-container
```

Wait for the cpd-cli to return the following messages:

```
[SUCCESS] ... Successfully pulled the container image icr.io/cpopen/cpd/olm-utils-v4:${VERSION}
[SUCCESS] ... Successfully started the container olm-utils-play-v4
[SUCCESS] ... Container olm-utils-play-v4 has been re-created
```

The version of the olm-utils-v4 image should be the same as the version of IBM Software Hub that you plan to install.

For more information, see [Obtaining the olm-utils-v4 image](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=pruirn-obtaining-olm-utils-v4-image-1).

### Installing Helm CLI

[Installing Helm](https://www.ibm.com/links?url=https%3A%2F%2Fhelm.sh%2Fdocs%2Fintro%2Finstall%2F)

# Upgrade

## Updating your environment variables script

> ⚠️ **Important:**
Ensure that your environment variables script includes the correct information for the instance of IBM Software Hub that you want to upgrade.

[Updating your environment variables script](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=cri-updating-your-environment-variables-script)

### Editing your environment variables file

1. Open your existing environment variable shell script in a text editor.

2. Locate the `VERSION` entry and specify the version of IBM Software Hub that you want to upgrade to:
```bash
export VERSION=5.4.0
```

3. Set or update the `PATCH_ID` environment variable based on the patch that you want to install:
```bash
export PATCH_ID=<target_patch_number>
```



# Upgrade

## Upgrading the Scheduling service

### Updating cluster-scoped resources for the Scheduling service

1.Generate the cluster-scoped resource definitions for the Scheduling service:
```
cpd-cli manage case-download \
--components=scheduler \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--scheduler_ns=${PROJECT_SCHEDULING_SERVICE} \
--cluster_resources=true
```

2.Change to the work directory. The default location of the work directory is `cpd-cli-workspace/olm-utils-workspace/work`.
<br>

3.Log in to Red Hat® OpenShift® Container Platform as a cluster administrator.

```
${OC_LOGIN}
```

4.Apply the cluster-scoped resources for the Scheduling service from the cluster_scoped_resources.yaml file:
```
oc apply -f cluster_scoped_resources.yaml \
--server-side \
--force-conflicts
```

### Creating image pull secrets for the Scheduling service
1.Create a file named dockerconfig.json based on where your cluster pulls images from:

For Private container registry:

```
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


2.Create the image pull secret in the project where the Scheduling service is installed:
```
oc create secret docker-registry ${IMAGE_PULL_SECRET} \
--from-file ".dockerconfigjson=dockerconfig.json" \
--namespace=${PROJECT_SCHEDULING_SERVICE}
```

### Upgrading the Scheduling service
```
cpd-cli manage apply-scheduler \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--license_acceptance=true \
--scheduler_ns=${PROJECT_SCHEDULING_SERVICE} \
--image_pull_prefix=${IMAGE_PULL_PREFIX} \
--image_pull_secret=${IMAGE_PULL_SECRET}
```

Confirm that the Scheduling service pods are Running or Completed:

```
oc get pods --namespace=${PROJECT_SCHEDULING_SERVICE}
```

## Upgrading the License Service

### Get the project of the License service

If you're not sure which project the License Service is in, run the following command:
```
oc get deployment -A | grep ibm-licensing-operator
```

### Log in to the Red Hat OpenShift Container Platform cluster
```
${CPDM_OC_LOGIN}
```

### Upgrading the License Service

```
cpd-cli manage apply-cluster-components \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--license_acceptance=true \
--licensing_ns=${PROJECT_LICENSE_SERVICE}
```
Confirm that the License Service pods are Running or Completed:

```
oc get pods --namespace=${PROJECT_LICENSE_SERVICE}
```

## Preparing to upgrade IBM Software Hub

### Updating the cluster-scoped resources for the platform and services

1.Generate cluster-scoped resources for platform and services

```
cpd-cli manage case-download \
--components=${COMPONENTS} \
--release=${VERSION} \
--patch_id=${PATCH_ID} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--cluster_resources=true
```

2.Change to the `work` directory. 
<br>
The default location of the work directory is `cpd-cli-workspace/olm-utils-workspace/work`.

```
cd cpd-cli-workspace/olm-utils-workspace/work
```

3.Log in to Red Hat® OpenShift® Container Platform as a cluster administrator
```
${OC_LOGIN}
```

4.Apply the cluster-scoped resources for the platform and services
```
oc apply -f cluster_scoped_resources.yaml \
--server-side \
--force-conflicts
```

### Creating image pull secrets for an instance of IBM Software Hub
1.Log in to Red Hat® OpenShift® Container Platform as a user with sufficient permissions to complete the task.
```
${OC_LOGIN}
```

2.Create a file named dockerconfig.json based on where your cluster pulls images from.
For Private container registry:

```
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


3.Create the image pull secret in the operators project for the instance.

```
oc create secret docker-registry ${IMAGE_PULL_SECRET} \
--from-file ".dockerconfigjson=dockerconfig.json" \
--namespace=${PROJECT_CPD_INST_OPERATORS}
```

4.Create the image pull secret in the operands project for the instance:
```
oc create secret docker-registry ${IMAGE_PULL_SECRET} \
--from-file ".dockerconfigjson=dockerconfig.json" \
--namespace=${PROJECT_CPD_INST_OPERANDS}
```


### Upgrading the IBM Events Operator

#### Upgrading Red Hat OpenShift Serverless Knative Eventing

1.Log in to Red Hat OpenShift Container Platform as a cluster administrator.
```
${OC_LOGIN}
```

Generate the required custom resource definitions for the IBM Events Operator:

```
cpd-cli manage deploy-events-operator \
--release=${VERSION} \
--cluster_resources=true
```

Apply the required custom resource definitions for the IBM Events Operator:

```
oc apply \
-f cpd-cli-workspace/olm-utils-workspace/work/ibm-events-operator-crds.yaml \
--server-side \
--force-conflicts
```

Log the cpd-cli in to the Red Hat OpenShift Container Platform cluster:

```
${CPDM_OC_LOGIN}
```

Upgrade the Red Hat OpenShift Serverless Knative Eventing software.
<br>

For Portworx storage
```
    cpd-cli manage deploy-knative-eventing \
    --release=${VERSION} \
    --storage_vendor=portworx \
    --upgrade=true
```

All other storage
```
    cpd-cli manage deploy-knative-eventing \
    --release=${VERSION} \
    --block_storage_class=${STG_CLASS_BLOCK} \
    --upgrade=true
```

#### Upgrading the IBM Events Operator

Log the cpd-cli in to the Red Hat OpenShift Container Platform cluster:
```
${CPDM_OC_LOGIN}
```

Run the following command to upgrade the IBM Events Operator:
```
cpd-cli manage deploy-events-operator \
--release=${VERSION} \
--events_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--events_operand_ns=${PROJECT_CPD_INST_OPERANDS}
```

### Upgrading Red Hat OpenShift AI
[Upgrading Red Hat OpenShift AI](https://www.ibm.com/docs/en/software-hub/5.4.x?topic=ups-upgrading-red-hat-openshift-ai)

## Upgrading IBM Software Hub

### Run the cpd-cli manage login-to-ocp command to log in to the cluster
```
${CPDM_OC_LOGIN}
```

### Upgrading the required operators and custom resources for the instance
```
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

Once the above command `cpd-cli manage install-components` is completed, make sure the status of the IBM Software Hub is in 'Completed' status.
```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \ 
--components=cpd_platform
```

### Applying the RSI patches
Run the following command to re-apply your existing custom patches.
```
cpd-cli manage apply-rsi-patches --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

Check the RSI patches status again: 
```
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

```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=watson_speech
```

## Upgrading Voice Gateway

### Run the cpd-cli manage login-to-ocp command to log in to the cluster
```
${CPDM_OC_LOGIN}
```

### Upgrading the operator and custom resource for the service

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
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=voice_gateway
```

## Upgrading Db2

### Run the cpd-cli manage login-to-ocp command to log in to the cluster
```
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



