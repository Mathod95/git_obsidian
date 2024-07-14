---
tags:
  - KIND
  - ARGOCD
  - SSO
  - GITOPS
  - APP/KEYCLOAK
source: https://medium.com/@charled.breteche/kind-keycloak-and-argocd-with-sso-9f3536dd7f61
---




# Kind, Keycloak and ArgoCD with SSO

![](https://miro.medium.com/v2/resize:fit:700/0*yZUhGcM6qv8ktWcb.png) 
In this story I’m going to deploy  [Keycloak](https://www.keycloak.org/)  (an identity provider) and  [ArgoCD](https://argo-cd.readthedocs.io/en/stable)  (a GitOps continuous delivery tool for Kubernetes) together on a  **local Kubernetes cluster** 
Keycloak will be configured by terraform and ArgoCD will use Keycloak to enable  **SSO authentication** 
I’m going to cover the following topics:
- Local Kubernetes cluster creation
- Keycloak deployment (with helm)
- Keycloak configuration (with terraform)
- ArgoCD deployment (with helm)
- Finally, deploy  `metrics-server`  through an ArgoCD application



# Keycloak overview

> 
Keycloak is an Open Source Identity and Access Management solution for modern Applications and Services.

Keycloak lets us manage users, groups, applications, roles … and much more.
In this story I will use it to manage users, groups, and applications to support SSO authentication through the OIDC protocol.


# ArgoCD overview

> 
Argo CD is a declarative, GitOps continuous delivery tool for Kubernetes.

GitOps is a modern approach to do Kubernetes cluster management and application delivery, you can read more about it  [here](https://www.weave.works/technologies/gitops) 
ArgoCD implements the GitOps approach in a simple and user friendly way.
I’m going to deploy and configure ArgoCD to use Keycloak SSO authentication to provide secure access.


# Create a local Kubernetes cluster

First, we need a local Kubernetes cluster.
I’m going to use a setup similar to what I already described in previous stories: [


## Kind, Cilium, MetalLB, and still no kube-proxy



### In a previous story I explained how to run a Kubernetes cluster locally with Kind, Cilium, and without kube-proxy.

medium.com ](https://medium.com/@charled.breteche/kind-cilium-metallb-and-no-kube-proxy-a9fe66ddfad6?source=post_page-----9f3536dd7f61--------------------------------) [


## Caching docker images for local Kind clusters



### In previous stories I wrote about creating local Kubernetes clusters with Kind, and how to run various core components…

medium.com ](https://medium.com/@charled.breteche/caching-docker-images-for-local-kind-clusters-252fac5434aa?source=post_page-----9f3536dd7f61--------------------------------) [


## Using dnsmasq for local Kind clusters



### Running a local Kubernetes cluster is easy, you can get a cluster up in a few minutes with Kind.

medium.com ](https://medium.com/@charled.breteche/using-dnsmasq-with-a-local-kind-clusters-9a27c8987073?source=post_page-----9f3536dd7f61--------------------------------)
Yo can find a  [convenient script](https://github.com/eddycharly/kind-playground/blob/main/cluster.sh)  in my GitHub repository to quickly create a local cluster with  ****  **Cilium**  **MetalLB**  **ingress-nginx**  and  **dnsmasq**  or simply run:

```
bash <(curl -s https://raw.githubusercontent.com/eddycharly/kind-playground/main/cluster.sh)
```


After a wile, the cluster should be up and running, we can now deploy Keycloak and ArgoCD.


# Deploy Keycloak

Now we have a local cluster, we need to deploy Keycloak. We can do that easily with Helm.
I will set Keycloak admin credentials with  `admin`  `admin` . Those credentials will then be used to configure Keycloak with terraform in the next step.
Running the command below should get us a Keycloak instance up and running:

```
helm upgrade --install --wait --atomic --namespace keycloak --create-namespace --repo https://codecentric.github.io/helm-charts keycloak keycloak --values - <<EOF
extraEnv: |
  - name: KEYCLOAK_USER
    value: admin
  - name: KEYCLOAK_PASSWORD
    value: admin
ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: nginx
  rules:
    - host: keycloak.kind.cluster
      paths:
        - path: /
          pathType: Prefix
  tls: null
EOF
```


Once executed, Keycloak should be browsable at  [http://keycloak.kind.cluster](http://keycloak.kind.cluster/)  and we should be able to log in the admin console with  `admin`  `admin` 
![](https://miro.medium.com/v2/resize:fit:700/1*iaboWVTeAOzRFKurOykeAQ.png) Keycloak admin console
Anyway, we won’t need to log in the admin console at all, we will configure our Keycloak instance with terraform instead.


# Configure Keycloak with terraform

At this point, Keycloak is running but not configured yet.
In order to use it with ArgoCD we need to create users, groups and assign groups to users.
We also need to create an application to allow ArgoCD to consume the Keycloak  **OIDC endpoint**  and authenticate users using SSO.
We could configure this manually in the admin console, following the  [ArgoCD docs](https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/keycloak) 
However it’s a time consuming and error prone task, I will use the  [Keycloak terraform provider](https://registry.terraform.io/providers/mrparkers/keycloak/latest/docs)  instead to get the configuration applied automatically.
The terraform code below does a couple of things:
- Declares and configures the provider
- Creates an openid client application
- Creates an openid client scope to return user groups membeship
- Configures openid client application default scopes
- Creates a couple of groups and users
- Configures user groups membership
- Outputs the openid client application secret


```
terraform {
  required_providers {
    keycloak = {
      source  = "mrparkers/keycloak"
      version = "3.6.0"
    }
  }
}
locals {
  realm_id = "master"
  groups   = ["argocd-dev", "argocd-admin"]
  user_groups = {
    user-dev       = ["argocd-dev"]
    user-admin     = ["argocd-admin"]
    user-dev-admin = ["argocd-dev", "argocd-admin"]
  }
}
# configure keycloak provider
provider "keycloak" {
  client_id = "admin-cli"
  username  = "admin"
  password  = "admin"
  url       = "http://keycloak.kind.cluster"
}
# create argocd openid client
resource "keycloak_openid_client" "argocd" {
  realm_id              = local.realm_id
  client_id             = "argocd"
  name                  = "argocd"
  enabled               = true
  access_type           = "CONFIDENTIAL"
  standard_flow_enabled = true
  valid_redirect_uris = [
    "http://argocd.kind.cluster/auth/callback"
  ]
}
# create groups openid client scope
resource "keycloak_openid_client_scope" "groups" {
  realm_id               = local.realm_id
  name                   = "groups"
  include_in_token_scope = true
  gui_order              = 1
}
resource "keycloak_openid_group_membership_protocol_mapper" "groups" {
  realm_id        = local.realm_id
  client_scope_id = keycloak_openid_client_scope.groups.id
  name            = "groups"
  claim_name      = "groups"
  full_path       = false
}
# configure argocd openid client default scopes
resource "keycloak_openid_client_default_scopes" "client_default_scopes" {
  realm_id  = local.realm_id
  client_id = keycloak_openid_client.argocd.id
  default_scopes = [
    "profile",
    "email",
    "roles",
    "web-origins",
    keycloak_openid_client_scope.groups.name,
  ]
}
# create groups
resource "keycloak_group" "groups" {
  for_each = toset(local.groups)
  realm_id = local.realm_id
  name     = each.key
}
# create users
resource "keycloak_user" "users" {
  for_each   = local.user_groups
  realm_id   = local.realm_id
  username   = each.key
  enabled    = true
  email      = "${@domain.com">each.key}@domain.com"
  first_name = each.key
  last_name  = each.key
  initial_password {
    value = each.key
  }
}
# configure use groups membership
resource "keycloak_user_groups" "user_groups" {
  for_each  = local.user_groups
  realm_id  = local.realm_id
  user_id   = keycloak_user.users[each.key].id
  group_ids = [for g in each.value : keycloak_group.groups[g].id]
}
# output argocd openid client secret
output "client-secret" {
  value     = keycloak_openid_client.argocd.client_secret
  sensitive = true
}
```


Copy the configuration above in a file called  `keycloak.tf`  and run  `terraform init && terraform apply`  to apply it.
This will configure Keycloak to be  **ready to use by ArgoCD**  with the following groups  `argocd-admin`  and  `argocd-dev`  and users  `user-admin`  `user-dev`  and  `user-dev-admin`  (passwords are the same as user names).


# Deploy ArgoCD with SSO

The next step is to deploy ArgoCD and configure it to use Keycloak OIDC identity provider.
Again, we can deploy ArgoCD using Helm.
We will need to provide configuration values to configure Keycloak SSO authentication:

```
CLIENT_SECRET=$(terraform output -raw -state=$TF_STATE client-secret)
helm upgrade --install --wait --atomic --namespace argocd --create-namespace  --repo https://argoproj.github.io/argo-helm argocd argo-cd --values - <<EOF
redis:
  enabled: true
redis-ha:
  enabled: false
server:
  config:
    url: http://argocd.kind.cluster
    application.instanceLabelKey: argocd.argoproj.io/instance
    admin.enabled: 'false'
    resource.exclusions: |
      - apiGroups:
          - cilium.io
        kinds:
          - CiliumIdentity
        clusters:
          - '*'
    oidc.config: |
      name: Keycloak
      issuer: http://keycloak.kind.cluster/auth/realms/master
      clientID: argocd
      clientSecret: $CLIENT_SECRET
      requestedScopes: ['openid', 'profile', 'email', 'groups']
  rbacConfig:
    policy.default: role:readonly
    policy.csv: |
      g, argocd-admin, role:admin
  extraArgs:
    - --insecure
  ingress:
    annotations:
      kubernetes.io/ingress.class: nginx
    enabled: true
    hosts:
      - argocd.kind.cluster
EOF
```


Once deployed, ArgoCD should be browsable at  [http://argocd.kind.cluster](http://argocd.kind.cluster/)  showing the  `LOG IN VIA KEYCLOAK`  button.
![](https://miro.medium.com/v2/resize:fit:700/1*r0vlkg8z2jsacwW_Cz3Ptw.png) ArgoCD login page
Clicking  `LOG IN VIA KEYCLOAK`  will redirect you to the Keycloak login page.
![](https://miro.medium.com/v2/resize:fit:700/1*i_IO06ttvr3MQ8CCKA_Awg.png) Keycloak authentication step
Authenticate with the  `login`  `password`  of a user we created in the previous step ( `user-admin`  `user-admin`  for example), you should be redirected to the ArgoCD dashboard.
![](https://miro.medium.com/v2/resize:fit:700/1*dR-T78H3yAPbVtVqVX7Guw.png) ArgoCD dashboard
ArgoCD dashboard is empty as we haven’t created any application yet, we’ll do that in the next step.


# Deploy an ArgoCD application

To complete this article, I will deploy  `metrics-server`  through an ArgoCD application.
An ArgoCD application is a Kubernetes resource that describes the application to be deployed.
Base on the application definition, ArgoCD will take care of the whole application lifecycle (creation, update, deletion…).
As an example, I will use the  `metrics-server`  helm chart to tell ArgoCD how to deploy it:

```
kubectl apply -n argocd -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: metrics-server
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://charts.bitnami.com/bitnami
    targetRevision: '*'
    chart: metrics-server
    helm:
      values: |
        extraArgs:
          kubelet-insecure-tls: true
        apiService:
          create: true
  destination:
    server: https://kubernetes.default.svc
    namespace: kube-system
  revisionHistoryLimit: 3
  syncPolicy:
    syncOptions:
      - PruneLast=true
      - ApplyOutOfSyncOnly=true
    automated:
      prune: true
      selfHeal: true
EOF
```


Going back to the ArgoCD dashboard at  [http://argocd.kind.cluster](http://argocd.kind.cluster/)  should now show the  `metrics-server`  application 🎉
![](https://miro.medium.com/v2/resize:fit:700/1*kuWw5of-vNgXvfU2XYQ4Kw.png) metrics-server application in ArgoCD


# Wrapping it up

I this story I covered the fact that  **Keycloak and ArgoCD are easy to install and configure**  and can easily be completely automated with the help of terraform.
This brings us a very convenient and modern setup to play with  **GitOps in a local Kubernetes cluster**  allowing us to deploy applications in a way that is pretty close to what would be done in a production cluster.
You can find all the code provided in this article in my GitHub repository: [


## GitHub - eddycharly/kind-playground: Playing with local Kubernetes cluster with Kind, Cilium…



### This repository contains code, scripts and manifests i use to play with local Kubernetes clusters with Kind, Cilium…

github.com ](https://github.com/eddycharly/kind-playground?source=post_page-----9f3536dd7f61--------------------------------)