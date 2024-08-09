# CICD using Tekton and Red Hat Trusted Artifact Signer
Repository dedicated to CICD pipeline that generates application's container images and sign them using Red Hat Trusted Artifact Signer. This entire environment was built using OpenShift 4.14.

## Install

1 - Create all required namespaces
```
oc new-project trusted-artifact-signer
oc new-project sso
oc new-project game
oc new-project cicd
```

2 - Install Red Hat Single Sign-On
```
oc apply -f rhsso/operator-group.yaml
oc apply -f rhsso/subscription.yaml
```

Wait for the operator be in *Succeeded* state.

```
oc apply -f rhsso/ -n sso
```

Wait for all pods be in *Running* state.

```
oc get pods -w -n sso
```

3 - Install Red Hat Trusted Artifact Signer
```
oc apply -f rhtas/subscription.yaml
```

Wait for the operator be in *Succeeded* state.

```
oc apply -f rhtas/
```

Wait all pods be in *Running* state.

```
oc get pods -w -n trusted-artifact-signer
```

4 - Install OpenShift Pipelines and CICD pipeline
```
oc apply -f cicd/install/subscription.yaml
```

Wait for the operator be in *Succeeded* state and all pods in `openshift-pipelines` in *Running* state.

```shell
oc apply -f cicd/tasks/ -n cicd
curl -sL https://raw.githubusercontent.com/tektoncd/catalog/main/task/grype/0.1/grype.yaml | oc apply -n cicd -f -
curl -sL https://raw.githubusercontent.com/tektoncd/catalog/main/task/trivy-scanner/0.2/trivy-scanner.yaml | oc apply -n cicd -f -
oc apply -f cicd/pipeline/pipeline-build-and-sign-image.yaml -n cicd
oc apply -f cicd/pipeline/pipeline-workspace-pvc.yaml -n cicd
oc apply -f cicd/pipeline/pipeline-verify-signature.yaml -n cicd
```

Retrieve RHSSO's client secret in order to use it in `cicd/secret.yaml`. First, retrieve RHSSO admin's password.

```
oc get secrets credential-keycloak -n sso -o jsonpath='{.data.ADMIN_PASSWORD}' | base64 -d
```

Second, retrieve RHSSO admin's URL, and log in using user `admin` and the password retrieved above.

```shell
echo https://$(oc get routes keycloak -n sso -o jsonpath='{.spec.host}')/auth/admin
```

Once you're logged in, update `CLIENT_SECRET` entry, inside `cicd/keyless-signing-info.yaml`, with **client trusted-artifact-signer secret's value (Confidentials tab)**.

It's necessary to update username and password from the `cicd/quay-creds.yaml` file with your **Quay.io credentials**. Without that **skopeo copy task will fail**.

You need to import **RHTAS Cosign image** to your cluster so the **sign-image** task can use it. Execute the following steps.

```shell
podman login registry.redhat.io
```

You'll be prompted to inform username and password. Pass those informations and pull the image.

```shell
podman pull registry.redhat.io/rhtas/cosign-rhel9:1.0.2-1719417920
```

Push the fresh pull image into your cluster.

```shell
podman tag registry.redhat.io/rhtas/cosign-rhel9:1.0.2-1719417920 default-route-openshift-image-registry.apps-crc.testing/cicd/rhtas-cosign-rhel9:1.0.2-1719417920
podman login default-route-openshift-image-registry.apps-crc.testing -u $(oc whoami) -p $(oc whoami -t)
podman push default-route-openshift-image-registry.apps-crc.testing/cicd/rhtas-cosign-rhel9:1.0.2-1719417920 --remove-signatures
```

After that, apply the following YAMLs.

```shell
oc apply -f cicd/config/keyless-signing-info.yaml -n cicd
oc apply -f cicd/config/ocp-cert-configmap.yaml -n cicd
oc apply -f cicd/config/quay-creds.yaml -n cicd
oc adm policy add-role-to-user edit system:serviceaccount:cicd:pipeline -n game
oc secret link pipeline quay-creds --for=mount -n cicd
```

5 - Run the pipeline

```shell
oc create -f cicd/pipeline/pr-build-and-sign-image.yaml -n cicd
oc create -f cicd/pipeline/pr-verify-signature.yaml -n cicd
```

## Tips and tricks
* Sometimes RHTAS provisioning fails because trillian-db face some problems with pod creation. Delete all pods and scale trillian-db up.
* Sometimes pipeline run fails because there is no cluster tasks to be found. Case this happens check **pods** in **openshift-pipelines** namespace. All pods must be in *Running* state so cluster tasks are provided.