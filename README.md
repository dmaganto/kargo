# INSTALL

## QUICKSTART
```bash
curl -L https://raw.githubusercontent.com/akuity/kargo/main/hack/quickstart/kind.sh | sh
```

```bash
export GITOPS_REPO_URL="https://github.com/dmaganto/kargo-demo"
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: kargo-demo
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - stage: test
      - stage: uat
      - stage: prod
  template:
    metadata:
      name: kargo-demo-{{stage}}
      annotations:
        kargo.akuity.io/authorized-stage: kargo-demo:{{stage}}
    spec:
      project: default
      source:
        repoURL: ${GITOPS_REPO_URL}
        targetRevision: stage/{{stage}}
        path: .
      destination:
        server: https://kubernetes.default.svc
        namespace: kargo-demo-{{stage}}
      syncPolicy:
        syncOptions:
        - CreateNamespace=true
EOF
```

```bash
export GITHUB_USERNAME="danimm85@gmail.com"
export GITHUB_PAT="<RESUMED>"
```

```bash
cat <<EOF | kargo apply -f -
apiVersion: kargo.akuity.io/v1alpha1
kind: Project
metadata:
  name: kargo-demo
---
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: kargo-demo-repo
  namespace: kargo-demo
  labels:
    kargo.akuity.io/cred-type: git
stringData:
  repoURL: ${GITOPS_REPO_URL}
  username: ${GITHUB_USERNAME}
  password: ${GITHUB_PAT}
---
apiVersion: kargo.akuity.io/v1alpha1
kind: Warehouse
metadata:
  name: kargo-demo
  namespace: kargo-demo
spec:
  subscriptions:
  - image:
      repoURL: public.ecr.aws/nginx/nginx
      semverConstraint: ^1.26.0
      discoveryLimit: 5
---
apiVersion: kargo.akuity.io/v1alpha1
kind: Stage
metadata:
  name: test
  namespace: kargo-demo
spec:
  requestedFreight:
  - origin:
      kind: Warehouse
      name: kargo-demo
    sources:
      direct: true
  promotionTemplate:
    spec:
      steps:
      - uses: git-clone
        config:
          repoURL: ${GITOPS_REPO_URL}
          checkout:
          - branch: main
            path: ./src
          - branch: stage/test
            create: true
            path: ./out
      - uses: git-clear
        config:
          path: ./out
      - uses: kustomize-set-image
        as: update-image
        config:
          path: ./src/base
          images:
          - image: public.ecr.aws/nginx/nginx
      - uses: kustomize-build
        config:
          path: ./src/stages/test
          outPath: ./out/manifests.yaml
      - uses: git-commit
        as: commit
        config:
          path: ./out
          messageFromSteps:
          - update-image
      - uses: git-push
        config:
          path: ./out
          targetBranch: stage/test
      - uses: argocd-update
        config:
          apps:
          - name: kargo-demo-test
            sources:
            - repoURL: ${GITOPS_REPO_URL}
              desiredCommitFromStep: commit
---
apiVersion: kargo.akuity.io/v1alpha1
kind: Stage
metadata:
  name: uat
  namespace: kargo-demo
spec:
  requestedFreight:
  - origin:
      kind: Warehouse
      name: kargo-demo
    sources:
      stages:
      - test
  promotionTemplate:
    spec:
      steps:
      - uses: git-clone
        config:
          repoURL: ${GITOPS_REPO_URL}
          checkout:
          - branch: main
            path: ./src
          - branch: stage/uat
            create: true
            path: ./out
      - uses: git-clear
        config:
          path: ./out
      - uses: kustomize-set-image
        as: update-image
        config:
          path: ./src/base
          images:
          - image: public.ecr.aws/nginx/nginx
      - uses: kustomize-build
        config:
          path: ./src/stages/test
          outPath: ./out/manifests.yaml
      - uses: git-commit
        as: commit
        config:
          path: ./out
          messageFromSteps:
          - update-image
      - uses: git-push
        config:
          path: ./out
          targetBranch: stage/uat
      - uses: argocd-update
        config:
          apps:
          - name: kargo-demo-uat
            sources:
            - repoURL: ${GITOPS_REPO_URL}
              desiredCommitFromStep: commit
---
apiVersion: kargo.akuity.io/v1alpha1
kind: Stage
metadata:
  name: prod
  namespace: kargo-demo
spec:
  requestedFreight:
  - origin:
      kind: Warehouse
      name: kargo-demo
    sources:
      stages:
      - uat
  promotionTemplate:
    spec:
      steps:
      - uses: git-clone
        config:
          repoURL: ${GITOPS_REPO_URL}
          checkout:
          - branch: main
            path: ./src
          - branch: stage/prod
            create: true
            path: ./out
      - uses: git-clear
        config:
          path: ./out
      - uses: kustomize-set-image
        as: update-image
        config:
          path: ./src/base
          images:
          - image: public.ecr.aws/nginx/nginx
      - uses: kustomize-build
        config:
          path: ./src/stages/test
          outPath: ./out/manifests.yaml
      - uses: git-commit
        as: commit
        config:
          path: ./out
          messageFromSteps:
          - update-image
      - uses: git-push
        config:
          path: ./out
          targetBranch: stage/prod
      - uses: argocd-update
        config:
          apps:
          - name: kargo-demo-prod
            sources:
            - repoURL: ${GITOPS_REPO_URL}
              desiredCommitFromStep: commit
EOF
```


## INSTALLATION REAL CASE

```bash
cat <<EOF | kargo apply -f -
apiVersion: kargo.akuity.io/v1alpha1
kind: Project
metadata:
  name: grafana
---
apiVersion: kargo.akuity.io/v1alpha1
kind: Warehouse
metadata:
  name: grafana
  namespace: grafana
spec:
  subscriptions:
  - image:
      repoURL: grafana/grafana
      semverConstraint: ^11.0.0
      discoveryLimit: 5
  - chart:
      discoveryLimit: 5
      name: grafana
      repoURL: https://grafana.github.io/helm-charts
      semverConstraint: ^8.0.0
EOF
```