## This repository contains few basic examples on how to deploy applicatons using flux

## Kustomize

Git Repository

```

kubectl apply -f - <<EOF
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: GitRepository
metadata:
  name: abtest-repo
  namespace: kommander-flux
spec:
  interval: 1m0s
  ref:
    branch: main
  timeout: 20s
  url: https://github.com/arbhoj/sample-k8s-app
EOF
```

#### Kustomize

```
kubectl apply -f - <<EOF
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: testkustomization
  namespace: ${WORKSPACE_NAMESPACE}
spec:
  interval: 1m0s
  path: ./basic
  prune: true
  sourceRef:
    kind: GitRepository
    name: abtest-repo
    namespace: kommander-flux
EOF
```

## HelmRelease

#### HelmRepo
```
kubectl apply -f - <<EOF
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  labels:
    kustomize.toolkit.fluxcd.io/name: helm-repository
    kustomize.toolkit.fluxcd.io/namespace: kommander-flux
  name: csi-driver-smb
  namespace: kommander-flux
spec:
  interval: 10m
  timeout: 1m
  url: https://raw.githubusercontent.com/kubernetes-csi/csi-driver-smb/master/charts
EOF
```

#### HelmRelease

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
  namespace: ${WORKSPACE_NAMESPACE}
spec:
  chart:
    spec:
      chart: csi-driver-smb
      sourceRef:
        kind: HelmRepository
        name: csi-driver-smb
        namespace: kommander-flux
      version: v1.3.0
  install:
    createNamespace: true
    remediation:
      retries: 30
  interval: 15s
  releaseName: csi-driver-smb
  targetNamespace: csi-driver-smb
  upgrade:
    remediation:
      retries: 30
EOF
```
