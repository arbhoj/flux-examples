This repository contains few basic examples on how to deploy applicatons using flux. 

## Kustomize

Step 1. Create a Git Repository resource with the following
- URL of the repository that contains the manifests to be applied.
- The branch to be used 

```
kubectl create ns demo

export NAMESPACE=demo

kubectl apply -f - <<EOF
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: GitRepository
metadata:
  name: demo-repo
  namespace: ${NAMESPACE}
spec:
  interval: 1m0s
  ref:
    branch: master
  timeout: 20s
  url: https://github.com/arbhoj/sample-k8s-app
EOF
```
Check if the repository is fetched successfully

```
kubectl get gitrepository -n ${NAMESPACE}
```

Step 2. Create a Kustomize resource with the following:
- Source Reference to the git repository created in the last step
- The path within the gitrepository that contains the manifests to be applied
- Patches that patch specific sections of the manifest being applied. In this example the namespace is being replaced. 
> Note: The patch specification follows [rfc6902](https://datatracker.ietf.org/doc/html/rfc6902)

```
export NAMESPACE=demo

kubectl apply -f - <<EOF
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: demokustomization
  namespace: ${NAMESPACE}
spec:
  patches:
   - patch: |
       - op: replace
         path: /metadata/namespace
         value: ${NAMESPACE}
     target:
       kind: Deployment
   - patch: |
       - op: replace
         path: /metadata/namespace
         value: ${NAMESPACE}
     target:
       kind: Service
   - patch: |
       - op: replace
         path: /metadata/namespace
         value: ${NAMESPACE}
     target:
       kind: ConfigMap
   - patch: |
       - op: replace
         path: /metadata/namespace
         value: ${NAMESPACE}
     target:
       kind: Ingress
  interval: 1m0s
  path: ./basic
  prune: true
  sourceRef:
   kind: GitRepository
   name: demo-repo
   namespace: ${NAMESPACE}
EOF
```

Check the resources deployed

```
kubectl get kustomization -n ${NAMESPACE}

kubectl get all -n ${NAMESPACE}
```

Cleanup

```
kubectl delete -n {NAMESPACE} kustomization demokustomization
kubectl delete -n {NAMESPACE} gitrepository demo-repo
```
## HelmRelease

Step 1. Create a HelmRepositoy resource with the following:
- URL of the Helm Repository to be used. 
```
export NAMESPACE=demo

kubectl apply -f - <<EOF
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  labels:
    kustomize.toolkit.fluxcd.io/name: helm-repository
    kustomize.toolkit.fluxcd.io/namespace: kommander-flux
  name: csi-driver-smb
  namespace: ${NAMESPACE}
spec:
  interval: 10m
  timeout: 1m
  url: https://raw.githubusercontent.com/kubernetes-csi/csi-driver-smb/master/charts
EOF
```

Step 2 Create a HelmRelease resource with the following:
- Reference to the HelmRepository resource created in the last step

> Note the following helm chart requires the associated serviceaccount to have permissions to `csidrivers.storage.k8s.io` at cluster scope. Hence the following RBAC needs to be created first

```
export NAMESPACE=demo
kubectl create clusterrole csi-driver-smb --verb='*' --resource=csidrivers.storage.k8s.io
kubectl create clusterrolebinding ${PROJECT}-csi-driver-smb --clusterrole csi-driver-smb --serviceaccount ${PROJECT}:${PROJECT}
```

Now deploy the HelmRelease
```
kubectl apply -f - <<EOF
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  annotations:
  labels:
    kustomize.toolkit.fluxcd.io/name: csi-driver-smb-release
    kustomize.toolkit.fluxcd.io/namespace: student
  name: csi-driver-smb
  namespace: ${NAMESPACE}
spec:
  chart:
    spec:
      chart: csi-driver-smb
      sourceRef:
        kind: HelmRepository
        name: csi-driver-smb
        namespace: ${NAMESPACE}
      version: v1.3.0
  install:
    createNamespace: false
    remediation:
      retries: 30
  interval: 15s
  serviceAccountName: ${NAMESPACE}
  releaseName: csi-driver-smb
  targetNamespace: ${NAMESPACE}
  upgrade:
    remediation:
      retries: 30
EOF
```


## Useful commands

To sync a GitRepository on demand
```
export PROJECT=demo
export GIT_REPOSITORY=demo-project-repo
kubectl -n ${PROJECT} annotate --overwrite gitrepository/${GIT_REPOSITORY} reconcile.fluxcd.io/requestedAt='$(date +%s)'

```

To sync a HelmRelease on demand
```
export PROJECT=demo
export HELM_RELEASE=csi-driver-smb
kubectl -n ${PROJECT} annotate --overwrite helmrelease/${HELM_RELEASE} reconcile.fluxcd.io/requestedAt='$(date +%s)'
```

> Note: Other Flux resources can be synced on demand in a similar fashion
