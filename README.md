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
oc apply -f cicd/subscription.yaml
```

Wait for the operator be in *Succeeded* state and all pods in `openshift-pipelines` in *Running* state.

```
oc apply -f cicd/sign-image.yaml -n cicd
oc apply -f cicd/pipeline.yaml -n cicd
oc apply -f cicd/pipeline-workspace-pvc.yaml -n cicd
```

Retrieve RHSSO's client secret in order to use it in `cicd/secret.yaml`. First, retrieve RHSSO admin's password.

```
oc get secrets credential-keycloak -n sso -o jsonpath='{.data.ADMIN_PASSWORD}' | base64 -d
```

Second, retrieve RHSSO admin's URL, and log in using user `admin` and the password retrieved above.

```shell
echo https://$(oc get routes keycloak -n sso -o jsonpath='{.spec.host}')/auth/admin
```

Once you're logged in, update `CLIENT_SECRET` entry, inside `cicd/secret.yaml`, with **client trusted-artifact-signer secret's value (Confidentials tab)**. After that, apply the following YAMLs.

```
oc apply -f cicd/keyless-signing-info.yaml -n cicd
oc apply -f cicd/ocp-cert-configmap.yaml -n cicd
oc apply -f cicd/quay-creds.yaml -n cicd
oc adm policy add-role-to-user edit system:serviceaccount:cicd:pipeline -n game
oc secret link pipeline quay-creds --for=mount -n cicd
```

5 - Run the pipeline

```
oc apply -f cicd/pipelinerun.yaml -n cicd
```

## Tips and tricks
* Sometimes RHTAS provisioning fails because trillian-db face some problems with pod creation. Delete all pods and scale trillian-db up.
* Sometimes pipeline run fails because there is no cluster tasks to be found. Case this happens check **pods** in **openshift-pipelines** namespace. All pods must be in *Running* state so cluster tasks are provided.