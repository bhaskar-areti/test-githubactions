ARC
#################
  bhaskar-areti/
  test-githubactions/
  │  ├── .k8s
  │  │    ├── nginx.yaml
  │  │  
  │  ├── .github
  │        ├── workflows/
  │        │    ├── build-test-all.yaml
  │        │    ├── pr-unit-tests.yaml
  │        │
  │        └──-deploy-nginx.yaml 
  istio-helm-migration
  ├── clusters/
  │      └── test-cluster/
  │             └── flux-system/
  │             │     ├── kustomization.yaml
  │             │     └── gotk-sync.yaml
  │             │     └── gotk-components.yaml
  │             │
  │             └── trivy-operator.yml
  │             └── github-runners-test.yml
  │             └── istio1.25.5
  ├── apps/
  │   ├── github-runners-test
  │   │   ├── repository-runner.yaml
  │   │   ├── test-pda-api-runner.yaml
  │   ├── nginx-test/
  │      ├── deployment.yaml
  │      ├── service.yaml
  │      └── kustomization.yaml
  │  
  └── infrastructure/
  ######
  ARC flow

test-githubactions repo
        ↓
GitHub Actions (self-hosted ARC runner)
        ↓
Commit to istio-helm-migration repo
        ↓
Flux reconciliation
        ↓
Kubernetes cluster (gha-test namespace)
GitHub Actions decides when and what to deploy
Flux decides how and where to deploy
Git is the only bridge between them
No kubectl from CI or No secrets for the cluster in CI or No Flux logic in the app repo

##
two repositories and what each one does
# test-githubactions as CI / intent like “What I want to deploy”
# Istio-helm-migration as GitOps / source of truth like “What is deployed in the cluster”
Flux only watches Istio-helm-migration.
GitHub Actions only touches Git — never the cluster.


# test-githubactions (CI / GitHub Actions repo)
Only three things matter here for deployments
test-githubactions/
├── .k8s lasting.html
│   └── nginx.yaml              --> SOURCE manifest
├── .github
│   └── workflows
│       └── deploy-nginx.yaml   --> DEPLOY workflow
└── (everything else)
# .k8s/nginx.yaml -->This is the deployment template.

You edit image tags here --> This file is NOT deployed directly
It is copied by GitHub Actions into the GitOps repo
Think of it as: “Desired deployment content”

# .github/workflows/deploy-nginx.yaml -->This is the only file that connects CI → GitOps.
What it does:

Runs on ARC runner
Checks out both repos
Copies .k8s/nginx.yaml
Commits to Istio-helm-migration
Pushes to Git

This is the only deployment trigger from GitHub Actions.   If this file doesn’t run, GA is not involved

# Istio-helm-migration(Flux GitOps repo) -->This is the only repo Flux watches.
The minimal working folder structure (what actually matters)
Istio-helm-migration/
├── clusters/
│   └── test-cluster/
│       ├── flux-system/
│       │   ├── gotk-components.yaml
│       │   └── gotk-sync.yaml
│       └── apps.yaml              --> tells Flux “watch ./apps”
│
├── apps/
│   ├── github-runners-test/
│   │   └── test-githubactions-runner.yaml --> ARC runner
│   │
│   └── nginx-test/
│       ├── deployment.yaml        --> ACTUAL deployed manifest
│       └── kustomization.yaml
│
└── infrastructure/ (ignored for nginx)

 Above Files that actually participate in nginx deployment
 clusters/test-cluster/apps.yaml
This is critical.
spec:
  path: ./apps

This tells Flux: “Everything under apps/ is deployable content”
Without this file → nothing deploys.
 apps/nginx-test/deployment.yaml -->This is the only file Kubernetes actually applies.

This is what Flux reconciles -->This is what created your nginx Pod
This file is written by GitHub Actions; Flux does not care who wrote it

 apps/nginx-test/kustomization.yaml -->Just wires the app directory together.

apps/github-runners-test/test-githubactions-runner.yaml
Creates the self‑hosted runner that executes the workflow.
Without this:

GitHub Actions job would stay Pending forever

#### What happens when you update the nginx image (step by step)
We Correct way (GitHub Actions + Flux together)
You edit: test-githubactions/.k8s/nginx.yaml
GitHub Actions workflow runs
ARC runner copies the file into
    Istio-helm-migration/apps/nginx-test/deployment.yaml
GitHub Actions commits + pushes; Flux notices Git change
Flux applies the new Deployment; Kubernetes updates the Pod image

This is exactly what we just tested successfully

❌ Direct edit in GitOps repo (Flux only)
If you edit: -->Istio-helm-migration/apps/nginx-test/deployment.yaml

✅ Flux deploys
❌ GitHub Actions is not involved
This is expected — and useful — but not CI‑driven.


We now have:  Clean separation of CI and CD
 No kubectl in GitHub Actions and  No cluster credentials in CI
 Reproducible GitOps state and ARC runners scoped to repo
 Flux remains the authority
This is exactly how large organisations do Kubernetes deployments safely.

GitHub Actions writes intent → Git commits state → Flux applies state

