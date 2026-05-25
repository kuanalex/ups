# Watson Speech Hotfix (05/21/26) for 5.3.1 Patch 0

This hotfix includes:

1. Maintains nl-NL_Telephony (Dutch telephony) and de-DE (German LSM) models from 5.3.0 Hotfix (03/19/26) - these are improved compared to 5.3.1 base.
2. New fr-FR_Telephony_LSM/fr-FR/fr-CA (French LSM) models - improved compared to both 5.3.0 and 5.3.1 base
3. Updated chuck runtime images for STT, TTS, and AM patcher components
4. New generic-models image

The upgrade consists of new versions of the chuck runtime images, updated model images, and a new generic-models image.

**Important:** The nl-NL_Telephony and de-DE models in this hotfix are the same versions currently running in your 5.3.0 Hotfix (03/19/26) production environment. They are included to ensure these improved models are preserved during the upgrade to 5.3.1 patch 0, as the 5.3.1 base release contains older versions of these models.

## Upgrading from 5.3.0 Hotfix (03/19/26)

### Save the Watson Speech Custom Resource

Before proceeding with the upgrade, save your current Watson Speech custom resource:

```sh
oc get WatsonSpeech speech-cr -n ${PROJECT_CPD_INST_OPERANDS} -o yaml > watson-speech-cr-backup.yaml
```

## Mirror Images

You will also need to mirror the following images in addition to the 5.3.1 base images as listed in steps 3-7. This step can also be done after upgrading to 5.3.1 patch 0.

1. Prepare the authentication credentials to access the IBM production repository. Use the same `auth.json` file that is used for CASE download and image mirroring.
An example directory path is `${HOME}/.airgap/auth.json` or you can create an `auth.json` file that contains credentials to access `icr.io` and your local private registry. (where auth value is the base64 encoded `username:password`) 
For example: 
```json
{
  "auths": {
    "icr.io": {
      "auth": "…"
    },
    "cp.icr.io": {
      "auth": "…"
    },
    "my-registry.local": {
      "auth": "…"
    }
  }
}
```
For more information about the `auth.json` file, see [containers-auth.json](https://www.ibm.com/links?url=https%3A%2F%2Fman.archlinux.org%2Fman%2Fcontainers-auth.json.5.en) syntax for the registry authentication file.

2. Install `skopeo` by using the following command:
```sh
yum install skopeo
```

Use the skopeo command to copy the patch images from the IBM production registry to the local private registry. Use the appropriate `auth.json` file, copy the patch images from the IBM production registry to the Red Hat® OpenShift® cluster registry.
To ensure that the skopeo commands run correctly, copy them into a text editor to remove any extra newline characters after the '\'. Then, copy the command from the text editor into the command line to avoid errors.

```sh
export LOCAL_REGISTRY="your_registry_hostname"
```

3. Copy the chuck image (used by multiple components) to the local registry. Replace `<FOLDER_PATH>` and `${LOCAL_REGISTRY}` with actual values.
```sh
skopeo copy --all --authfile "<FOLDER_PATH>/auth.json" --dest-tls-verify=false --src-tls-verify=false docker://cp.icr.io/cp/watson-speech/chuck@sha256:2a24058a667b1f9026d49bda2e0ddee8f0f88f4e9760eaf35961c854525eb718 docker://$LOCAL_REGISTRY/cp/watson-speech/chuck@sha256:2a24058a667b1f9026d49bda2e0ddee8f0f88f4e9760eaf35961c854525eb718
```

4. Copy the generic-models image to the local registry. Replace `<FOLDER_PATH>` and `${LOCAL_REGISTRY}` with actual values.
```sh
skopeo copy --all --authfile "<FOLDER_PATH>/auth.json" --dest-tls-verify=false --src-tls-verify=false docker://cp.icr.io/cp/watson-speech/generic-models@sha256:2dbbc1f1f8a9860765a60ba27ea181761196f776044b10388001befaf3b0046e docker://$LOCAL_REGISTRY/cp/watson-speech/generic-models@sha256:2dbbc1f1f8a9860765a60ba27ea181761196f776044b10388001befaf3b0046e
```

5. Copy the de-DE model image to the local registry. Replace `<FOLDER_PATH>` and `${LOCAL_REGISTRY}` with actual values.
```sh
skopeo copy --all --authfile "<FOLDER_PATH>/auth.json" --dest-tls-verify=false --src-tls-verify=false docker://cp.icr.io/cp/watson-speech/de-de@sha256:17eeb6230026792e6cb9471ab4cb48fae34cef6202539ecefd30bc5e0f080e6c docker://$LOCAL_REGISTRY/cp/watson-speech/de-de@sha256:17eeb6230026792e6cb9471ab4cb48fae34cef6202539ecefd30bc5e0f080e6c
```

6. Copy the fr-FR_Telephony_LSM model image to the local registry. Replace `<FOLDER_PATH>` and `${LOCAL_REGISTRY}` with actual values.
```sh
skopeo copy --all --authfile "<FOLDER_PATH>/auth.json" --dest-tls-verify=false --src-tls-verify=false docker://cp.icr.io/cp/watson-speech/fr-fr-telephony-lsm@sha256:35b79279ecb9d695a5c8c69e5bd8e172e040e87f4ed8702cdd5090f84be39fa4 docker://$LOCAL_REGISTRY/cp/watson-speech/fr-fr-telephony-lsm@sha256:35b79279ecb9d695a5c8c69e5bd8e172e040e87f4ed8702cdd5090f84be39fa4
```

7. Copy the nl-NL_Telephony model image to the local registry. Replace `<FOLDER_PATH>` and `${LOCAL_REGISTRY}` with actual values.
```sh
skopeo copy --all --authfile "<FOLDER_PATH>/auth.json" --dest-tls-verify=false --src-tls-verify=false docker://cp.icr.io/cp/watson-speech/nl-nl-telephony@sha256:940c93833760e802044b6c16e83d83696d21a464c56f723dfe68ba4d9986abc3 docker://$LOCAL_REGISTRY/cp/watson-speech/nl-nl-telephony@sha256:940c93833760e802044b6c16e83d83696d21a464c56f723dfe68ba4d9986abc3
```

## Perform the Upgrade

Once the images are in the local registry, perform the upgrade to 5.3.1 patch 0 with flag `--upgrade=true --patch_id=0`.

### Step 1: Upgrade to 5.3.1 Patch 0

**Important Note:** During the upgrade process, any custom configurations in the Watson Speech custom resource that are not managed by the cpd-cli (Helm) upgrade process may be lost. After the upgrade completes, review your saved custom resource backup (`watson-speech-cr-backup.yaml`) and manually re-apply any custom configurations that were not preserved.

Leave out the `watsonSpeech` section in the values.yaml to preserve your custom resource spec during the upgrade. Example values.yaml file:

```yaml
non_olm:
  global:
    blockStorageClass: <ocs-storagecluster-ceph-rbd>
    fileStorageClass: <ocs-storagecluster-cephfs>
```

Example cpd-cli install-components upgrade command:

```sh
cpd-cli manage install-components \
  --license_acceptance=true \
  --release=5.3.1 \
  --components=watson_speech \
  --operator_ns=${PROJECT_CPD_INST_OPERATORS} \
  --instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --block_storage_class=<ocs-storagecluster-ceph-rbd> \
  --file_storage_class=<ocs-storagecluster-cephfs> \
  --param-file=values.yaml \
  --upgrade=true \
  --patch_id=0
```

### Step 2: Wait for Upgrade to Complete

Monitor the upgrade progress:

```sh
oc get WatsonSpeech speech-cr -n ${PROJECT_CPD_INST_OPERANDS} -w
```

Wait until the status shows `Completed` or `Ready`.

### Step 3: Restore Custom Configurations (if needed)

After the upgrade completes, compare your current custom resource with the backup you created earlier:

```sh
# View the backup
cat watson-speech-cr-backup.yaml

# View the current CR
oc get WatsonSpeech speech-cr -n ${PROJECT_CPD_INST_OPERANDS} -o yaml
```

If any custom configurations were lost during the upgrade (such as custom resource request/limits/replicas, GW HPA, or other manual modifications), re-apply them by editing the custom resource:

```sh
oc edit WatsonSpeech speech-cr -n ${PROJECT_CPD_INST_OPERANDS}
```

## Apply Image Overrides

After the upgrade completes, apply the image overrides to use the new chuck runtime images and updated models.

### Edit the Watson Speech Custom Resource

Edit the Watson Speech custom resource to add the image overrides:

```sh
oc edit WatsonSpeech speech-cr -n ${PROJECT_CPD_INST_OPERANDS}
```

Add or update the following sections under `spec`:

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

**Note:** Only include the models you want to enable. Set `enabled: true` for models you wish to use.

Save and exit the editor. The Speech operator will reconcile the changes and update the deployments.

## Validating the 5.3.1 patch 0 Hotfix

After applying the image overrides, the Speech operator will reconcile the custom resource and update the components.

1. Monitor the Speech operator reconciliation:
```sh
oc get WatsonSpeech speech-cr -n ${PROJECT_CPD_INST_OPERANDS} -w
```

2. Check that the new models are being uploaded. A new `speech-cr-stt-models` job will spawn to upload the updated models to object storage:
```sh
oc get pods -n ${PROJECT_CPD_INST_OPERANDS} -w | grep speech-cr-stt-models
```

3. After the models are uploaded, verify that the new STT/TTS runtime pods are rolled out with newer chuck image:
```sh
oc get pods -n ${PROJECT_CPD_INST_OPERANDS} -w | grep -E "speech-cr-(stt|tts)-runtime"
```


## Rollback (if needed) to 5.3.1 patch 0

If you need to rollback the image overrides, you can restore the previous custom resource:

Remove the image override sections from the custom resource:

```sh
oc edit WatsonSpeech speech-cr -n ${PROJECT_CPD_INST_OPERANDS}
```

Remove the `spec.images` and `spec.global.genericModels` sections, and remove the `digest` fields from the model entries.

## Summary

This upgrade process:
1. Saves the existing Watson Speech custom resource
2. Upgrades from 5.3.0 hotfix to 5.3.1 patch 0 using minimal values.yaml
3. Applies image overrides for chuck runtime components and updated models
4. Validates that the new images and models are deployed correctly

The new models (de-DE, fr-FR_Telephony_LSM, nl-NL_Telephony) and chuck runtime images are now deployed and ready to use.