# CICD using Tekton and Red Hat Trusted Artifact Signer
Repository dedicated to CICD pipeline that generates application's container images and sign them using Red Hat Trusted Artifact Signer

## Install
```
oc new-project trusted-artifact-signer
oc new-project sso
oc new-project game
oc new-project cicd

oc apply -f rhsso/operator-group.yaml
oc apply -f rhsso/subscription.yaml
oc apply -f rhsso/ -n sso

oc apply -f rhtas/subscription.yaml
oc apply -f rhtas/ -n trusted-artifact-signer

oc apply -f cicd/subscription.yaml
oc adm policy add-role-to-user edit system:serviceaccount:cicd:pipeline -n game
oc apply -f cicd/sign-image.yaml -n cicd
oc apply -f cicd/pipeline.yaml -n cicd
oc apply -f cicd/pipelinerun.yaml -n cicd
```