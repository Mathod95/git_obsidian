---
tags:
  - APP/ARGO/CD
  - MONOREPO
source: https://medium.com/@michail.gebka/optimizing-argocd-for-monorepo-setup-7c5f548e5575
---




# Optimizing ArgoCD for Monorepo setup

Unlock valuable insights into optimizing ArgoCD performance in a Monorepo setup, drawing from our experience at  [Hypatos](https://www.hypatos.ai/) 
In our CI/CD evolution, we decided to integrate ArgoCD, Helm and GitHub Actions. Opting for a Monorepo setup for the CD part, we followed the idea of having a single declarative point of truth. This approach provides us traceability, clarity and organised structure that meet our GitOps way of working.
![](https://miro.medium.com/v2/resize:fit:361/1*GD9OktU4NQ_azjh9AnqcBw.png) 
Unfortunately growing number of applications and environments progressively  **diminished ArgoCD’s performance** . Despite attempts to remedy the situation by injecting more resources into the deployment, the expected improvements didn’t brought significant result. Instead, the  **incremental resource usage by ArgoCD translated into escalating costs without a proportional boost in efficiency.**  This challenge prompted a thorough examination of strategies to optimize ArgoCD’s performance and resource utilization.
But.. let’s start from the beginning!


# Initial setup

To understand our use case, let me briefly summarize our setup:
- Each application has its own GitHub repository containing its source code.
- Applications utilize GitHub Actions for testing and building the image.
- Each application has its Helm definition stored in a common and shared Monorepo.
- Based on conditions in GitHub Actions, a Pull Request is generated to update the image in a specific path within the Monorepo.
- Each Kubernetes cluster has its own ArgoCD instance.
-  [App of Apps pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/#app-of-apps-pattern)  is followed, where each application project has its “App Manager” storing the  [Application CRD](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#applications)  for that application.
- ArgoCD instances are  **not exposed to the public internet.** 



## Monorepo

Applications definitions inside Monorepo are organized  **by k8s cluster**  and logically  **grouped together by the projects**  they belong to.
Monorepo has the following root structure:

```
├── dev
│   ├── project_A
│   ├── project_B
├── stage
│   ├── project_A
│   ├── project_B
├── prod
│   ├── project_A
│   ├── project_B
```


Each project consists of application definitions and a project manager — an “empty” application with Application CRDs for the applications located in this group:

```
dev
└── project_A
    ├── argocd-project_A-manager
    │   ├── Chart.yaml
    │   ├── values.yaml
    │   └── templates
    │       ├── frontend_APPLICATION_DEFINITION.yaml
    │       └── backend_APPLICATION_DEFINITION.yam
    ├── frontend
    │   ├── Chart.yaml
    │   └── values.yaml
    └── backend
        ├── Chart.yaml
        └── values.yaml
```




## What about ArgoCD?

Deployed to each cluster (environment) and functioning independently from others, ArgoCD instances are also deployed declaratively, maintaining default reconciliation and synchronization settings autonomously.
![](https://miro.medium.com/v2/resize:fit:700/0*Q_vhyaoEwYTPmInA.jpg) ArgoCD as ArgoCD Application


## How does it work together

![](https://miro.medium.com/v2/resize:fit:700/1*ICV0jOCRLV3-0hatT6It6w.png) 


# Performance issues

At the outset, ArgoCD functioned seamlessly, exhibiting speed, efficiency, low resource consumption, developers satisfaction high. But a few months and dozens applications later we started hearing complains…
> 
Hey! Are there any issues with ArgoCD? My application is still not deployed

“Ok, big load” we said. Let’s add a bit more resources. But the issue kept occurring and we decided that scaling doesn’t make sense anymore.
Quick look at the logs and metrics and we knew the case. Each change to Monorepo, detected by ArgoCD, triggers  **an avalanche of apps reconciliation.** 
ArgoCD’s documentation clearly explains why is that happening:
> 
Argo CD aggressively caches generated manifests and uses the repository commit SHA as a cache key.  **A new commit to the Git repository invalidates the cache for all applications configured in the repository.**  This can negatively affect repositories with multiple applications

As our ArgoCDs clusters navigated the challenge of handling  **over 300 applications**  (per each ArgoCD) with default settings, we realized it was the time for changing the way how ArgoCD works.


# Remediation

By default ArgoCD operates in a “Pull Model”:
> 
Every 3m (by default) Argo CD checks for changes to the app manifests

![](https://miro.medium.com/v2/resize:fit:700/1*jXPC7W-4EhaQ3eyt0C49xw.png) 
Fortunately, ArgoCD provides a couple of solutions to enhance performance, particularly beneficial for Monorepo setups. One of the is  [Webhook and Manifest Paths Annotation](https://argo-cd.readthedocs.io/en/stable/operator-manual/high_availability/#webhook-and-manifest-paths-annotation) 


## Transition to a Push Model

To avoid invalidating the whole repo cache every time when a single application is updated we have transitioned to a push model, enabling us to proactively notify ArgoCD of changes, thereby triggering synchronization as needed.
We decided to implement Webhooks triggered by Github Actions:
![](https://miro.medium.com/v2/resize:fit:700/1*mi0W5yhiGXjTuH5NNAHu_w.png) 
Following the  [documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/high_availability/#webhook-and-manifest-paths-annotation)  we added the  `argocd.argoproj.io/manifest-generate-paths: .`  annotation to all Application definitions:

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
  annotations:
    # resolves to the 'guestbook' directory
    argocd.argoproj.io/manifest-generate-paths: .
spec:
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
# ...
```


Easy. Now, time to configure  [ArgoCD Github Webooks ](https://argo-cd.readthedocs.io/en/stable/operator-manual/webhook/) 
 **But hold on, it’s a bit more complicated when your ArgoCD is NOT available through public Internet** . And our ArgoCDs are not exposed outside the private networks.
Leveraging  [self-hosted GitHub Action Runners](https://github.com/actions/actions-runner-controller)  already deployed on our Kubernetes clusters, we created custom webhooks imitating the ones that  [Github Webhook](https://docs.github.com/en/webhooks/about-webhooks)  produces. We checked what exactly ArgoCD expects from the payload and after some tests we came up with a very basic Github Action:
- Trigger action on push to  ``  branch and a specific path corresponding to the evironment
-  [https://github.com/tj-actions/changed-files](https://github.com/tj-actions/changed-files)  action to detect added, modified and removed files
- Use own shared action to call ArgoCD’s API with changed files


```
name: Sync ArgoCD dev

on:
  push:
    branches:
      - main
    paths:
      - dev/**
jobs:
  webhook:
    name: Run synchronisation
    runs-on: sync-argocd-dev

    steps:
      - name: Checkout
        uses: actions/checkout@v2
        with:
          fetch-depth: 0

      - name: Get changed files
        id: changed-files
        uses: tj-actions/changed-files@v34

      - name: Send GH webhook
        uses: hypatos/shared-actions/argo-sync@main
        with:
          added-files: ${{ steps.changed-files.outputs.added_files }}
          modified-files: ${{ steps.changed-files.outputs.modified_files }}
          removed-files: ${{ steps.changed-files.outputs.deleted_files }}
          webhook-secret: ${{ secrets.ARGOCD_WEBHOOK_SECRET }}
```


 **argo-sync**  is a simple GitHub Action calling ArgoCD’s  **/api/webhook**  endpoint with the necessary data:

```
# action.yaml
name: Run Github Webhook
description: "Generates Github-Webhooks-like request to given endpoint"

inputs:
  webhook-endpoint:
    description: "Endpoint for the webhook"
    required: false
    default: "http://argocd-server.argocd/api/webhook"
  webhook-secret:
    description: "Secret for the webhook"
    required: true
  added-files:
    description: "Files that were added in the commit"
    required: false
    default: ""
  modified-files:
    description: "Files that were added in the commit"
    required: false
    default: ""
  removed-files:
    description: "Files that were added in the commit"
    required: false
    default: ""

runs:
  using: composite
  steps:
    - uses: "docker://hypatosai/argo-sync:stable"
      with:
        MODIFIED_FILES: ${{inputs.modified-files}}
        REMOVED_FILES: ${{inputs.removed-files}}
        ADDED_FILES: ${{inputs.added-files}}
        WEBHOOK_SECRET: ${{inputs.webhook-secret}}
        WEBHOOK_ENDPOINT: ${{inputs.webhook-endpoint}}
```



```
#!/bin/sh
# script.sh
echo "added files"
echo "$INPUT_ADDED_FILES"
echo "removed files"
echo "$INPUT_REMOVED_FILES"
echo "modified files"
echo "${INPUT_MODIFIED_FILES}"

jq -r --arg ADDED_FILES "${INPUT_ADDED_FILES}" --arg REMOVED_FILES "${INPUT_REMOVED_FILES}" --arg MODIFIED_FILES "${INPUT_MODIFIED_FILES}" '.head_commit += {"added": ($ADDED_FILES / " "), "removed": ($REMOVED_FILES / " "), "modified": ($MODIFIED_FILES / " ")}' $GITHUB_EVENT_PATH > /tmp/event_file_head_commit.json
jq -r --arg ADDED_FILES "${INPUT_ADDED_FILES}" --arg REMOVED_FILES "${INPUT_REMOVED_FILES}" --arg MODIFIED_FILES "${INPUT_MODIFIED_FILES}" '.commits[0] += {"added": ($ADDED_FILES / " "), "removed": ($REMOVED_FILES / " "), "modified": ($MODIFIED_FILES / " ")}' /tmp/event_file_head_commit.json > /tmp/event_file.json

CONTENT_TYPE="application/json"
RAW_FILE_DATA=`cat /tmp/event_file.json`
WEBHOOK_DATA=$(echo -n $RAW_FILE_DATA | jq -c '')
WEBHOOK_SIGNATURE=$(echo -n "$WEBHOOK_DATA" | openssl sha1 -hmac "$INPUT_WEBHOOK_SECRET" -binary | xxd -p)
WEBHOOK_SIGNATURE_256=$(echo -n "$WEBHOOK_DATA" | openssl dgst -sha256 -hmac "$INPUT_WEBHOOK_SECRET" -binary | xxd -p |tr -d '\n')
REQUEST_ID=$(uuidgen)
EVENT_NAME="push"
curl -s \
    -H "Content-Type: $CONTENT_TYPE" \
    -H "User-Agent: GitHub-Hookshot/760256b" \
    -H "X-Hub-Signature: sha1=$WEBHOOK_SIGNATURE" \
    -H "X-Hub-Signature-256: sha256=$WEBHOOK_SIGNATURE_256" \
    -H "X-GitHub-Delivery: $REQUEST_ID" \
    -H "X-GitHub-Event: $EVENT_NAME" \
    --data "$WEBHOOK_DATA" $INPUT_WEBHOOK_ENDPOINT
```


Script produces the payload based on the dummy file containing basic webhook payload from  `GITHUB_EVENT_PATH` variable. Then, creates a  `WEBHOOK_SIGNATURE`  to validate it and adds the  `X-GitHub-Delivery`  and  `X-GitHub-Event`  headers to accurately mimic the GitHub Webhook expected by ArgoCD.
 **Voilà!** 
Additionally, to optimize cache management, we’ve disabled the default  `revision-cache-expiration`  set at 3 minutes and extended it to 12 hours to still run the periodical reconciliation twice a day.
Curious about what we do in Hypatos? Check out  [https://www.hypatos.ai/](https://www.hypatos.ai/) 


## Document processing with market-leading AI & human-centric platform

Hypatos provides document hyperautomation to automate complex document processing — unleashing the potential of your people and business.
![](https://miro.medium.com/v2/resize:fit:493/0*3GAb2YZgRX9DF2u0) 