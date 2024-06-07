# CICD using Tekton and Red Hat Trusted Artifact Signer
Repository dedicated to CICD pipeline that generates application's container images and sign them using Red Hat Trusted Artifact Signer

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
oc apply -f rhsso/ -n sso
```

Wait for all pods be in *Running* state
```
oc get pods -w -n sso
```

3 - Install Red Hat Trusted Artifact Signer
```
oc apply -f rhtas/subscription.yaml
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

Wait operator's provisioning.

```
oc apply -f cicd/sign-image.yaml -n cicd
oc apply -f cicd/pipeline.yaml -n cicd
oc apply -f cicd/pipeline-workspace-pvc.yaml -n cicd
```

Retrieve RHSSO's client secret in order to use it in `cicd/secret.yaml`. First, retrieve RHSSO admin's password.

```
oc get secrets credential-keycloak -n sso -o jsonpath='{.data.ADMIN_PASSWORD}' | base64 -d
```

Once is retrieved, update `CLIENT_SECRET` entry inside `cicd/secret.yaml`. Apply the following YAMLs.

```
oc apply -f cicd/secret.yaml -n cicd
oc apply -f cicd/ocp-cert-configmap.yaml -n cicd
oc adm policy add-role-to-user edit system:serviceaccount:cicd:pipeline -n game

5 - Run the pipeline

```
oc apply -f cicd/pipelinerun.yaml -n cicd
```